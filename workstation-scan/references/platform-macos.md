# Platform Reference — macOS

Loaded by `SKILL.md` Step 2 when the detected platform is macOS (Darwin). This file carries the
macOS-specific commands, gotchas and probes. The shared procedure lives in `SKILL.md`; the document
standard lives in `Workstation-grounding.md`. Nothing here overrides either.

Every command in this file was executed on a real Apple Silicon Mac while writing it. Where a
command needs `sudo`, it says so — **a scan never requires sudo.** Record such items as
"requires sudo, not run" rather than guessing the value.

## M1. Detection — Apple Silicon or Intel

Decide this first; the two paths differ materially.

```bash
uname -m                          # arm64 → Apple Silicon ; x86_64 → Intel Mac OR a Rosetta shell
sysctl -n hw.optional.arm64       # 1 on Apple Silicon (present even inside a Rosetta shell)
sysctl -n sysctl.proc_translated  # 1 → THIS SHELL is running under Rosetta ; 0 → native
sw_vers                           # ProductVersion / BuildVersion
```

| Signal | Apple Silicon | Intel Mac |
|---|---|---|
| `hw.optional.arm64` | `1` | absent / `0` |
| `machdep.cpu.brand_string` | `Apple M1…M4 …` | `Intel(R) Core(TM) …` |
| GPU | On-chip, unified memory | Intel iGPU and/or discrete AMD, own VRAM |
| `torch_device` | `mps` | `mps` (Metal on AMD) or `cpu`; check |
| ANE | Present | Absent |
| MLX | Supported | Not supported (Apple Silicon only) |

**If `uname -m` prints `x86_64` but `hw.optional.arm64` is `1`, the shell is under Rosetta.** Record
it in the frontmatter (`rosetta_translated: true`) and in the directives — everything launched from
this shell inherits x86_64, including Python, and an x86_64 Python has no MPS. Prefer re-running
the scan from a native shell (`arch -arm64 zsh`).

### Intel Mac degraded path

On an Intel Mac, run M2 inventory, M5 toolchain and M6 framework probes. Skip M4 (ANE), skip MLX,
and treat the GPU section per its actual vendor: an AMD discrete GPU reports dedicated VRAM via
`system_profiler SPDisplaysDataType` (`VRAM (Total)`), `unified_memory: false`, and Metal is the
only compute route — there is no CUDA, no ROCm, and no OpenCL beyond the deprecated 1.2 framework.
State in the Platform Summary that the machine is an Intel Mac and that Apple no longer ships new
Metal features to it; that is a 🟠 High constraint for any ML work.

## M2. Inventory commands

**Never assume a command exists — check, and record the failure if it does not.**

| Target | Command | Notes |
|---|---|---|
| OS / build | `sw_vers`, `uname -a` | Darwin kernel version ≠ macOS version; record both |
| CPU | `sysctl -n machdep.cpu.brand_string`, `sysctl hw.ncpu hw.physicalcpu hw.logicalcpu`, `sysctl hw.perflevel0.physicalcpu hw.perflevel0.name hw.perflevel1.physicalcpu hw.perflevel1.name` | perflevel0 = Performance, perflevel1 = Efficiency. `hw.cpufrequency` is **absent** on Apple Silicon — do not report a clock |
| Cache | `sysctl hw.l1icachesize hw.l1dcachesize hw.l2cachesize hw.perflevel0.l2cachesize hw.perflevel1.l2cachesize` | L2 differs per cluster |
| Memory | `sysctl -n hw.memsize`, `system_profiler SPMemoryDataType` | Type (LPDDR5) and manufacturer; always soldered on Apple Silicon → `ram_upgradeable: false` |
| Free memory | `vm_stat`, `memory_pressure`, `sysctl -n vm.swapusage` | See M3 — page size is 16384 on Apple Silicon |
| GPU | `system_profiler SPDisplaysDataType` | `Total Number of Cores`, `Metal Support: Metal N`, attached displays |
| ANE | `ioreg -l \| grep -oE '"IOClass" = "[^"]*ANE[^"]*"' \| sort -u` | Presence only; see M4 |
| Storage | `diskutil list`, `diskutil info /`, `df -h / /System/Volumes/Data` | See M2a |
| Board / firmware | `system_profiler SPHardwareDataType` | Model Identifier, Chip, System Firmware Version. **Strip Serial Number, Hardware UUID, Provisioning UDID** — the command prints them and the grounding forbids recording them |
| Power | `pmset -g batt`, `pmset -g therm`, `system_profiler SPPowerDataType` | AC vs battery, charge %, cycle count, condition, adapter wattage |
| Low-power mode | `pmset -g custom \| grep lowpowermode` | Per power source; affects benchmarks |
| Thermal | `pmset -g therm` | Only reports *warning levels*. Actual thermal pressure and package power need `sudo powermetrics` — **requires sudo, not run** |
| Virtualization | `sysctl -n kern.hv_support` | `1` → Hypervisor.framework available (Docker/OrbStack/UTM) |
| SIP | `csrutil status` | Informational |
| Hostname | `scutil --get ComputerName`, `hostname -s` | `uname -n` may include the router domain (e.g. `.fritz.box`) — do not record it |

### M2a. Storage on APFS

`df -h /` reports the **sealed system snapshot**, not the user's data. Report the Data volume:

```bash
df -h /System/Volumes/Data          # the figure the user can actually consume
diskutil info / | grep -E 'Container Total Space|Container Free Space|Solid State|Protocol'
```

Both share one APFS container, so "free" is the same number, but `Capacity` on `/` is misleading.
Count `storage_volumes` as physical disks, not APFS volumes (a stock Mac shows one disk and six
volumes). Mounted disk images (`/dev/diskN (disk image)`) are not storage; omit them.

Disk is the tightest resource on many Macs. Below ~20 GiB free, model downloads and the swapfile
compete — record it as a 🔴 Blocker for the model class that will not fit and name the large
consumers (`du -sh ~/.ollama ~/.cache ~/Library/Caches` and any model directories found).

## M3. The three memory figures on macOS — read this before recording any number

Apple Silicon has **one physical pool**. CPU, GPU and ANE all allocate from it. The three figures
therefore mean something different from a discrete-GPU machine:

| Figure | macOS source | What it is |
|---|---|---|
| **OS reported** | `sysctl -n hw.memsize` | Total unified memory. Shared with the OS, every process and the display. |
| **Runtime reported** | Metal `recommendedMaxWorkingSetSize` — via `torch.mps.recommended_max_memory()`, `mx.device_info()["max_recommended_working_set_size"]`, or the `total=` figure in `~/.ollama/logs/server.log` | The GPU's **wired-memory budget**. Defaults to roughly 65–75 % of RAM. Governed by `sysctl iogpu.wired_limit_mb` (`0` = default). |
| **Actually allocatable** | Allocate until failure | **On unified memory this figure is a trap — see below.** |

### ⚠️ "Allocate until OOM" over-reports on unified memory

PyTorch's MPS allocator permits allocations up to `recommended_max_memory × high_watermark_ratio`
(default 1.7). On a 16 GB machine the probe **succeeded in allocating 19 GiB before the allocator
refused at a 20.13 GiB "max allowed"** — more than the physical RAM. The OS did not fail; it paged
everything else out to the SSD (`vm.swapusage` confirms). Nothing errored. The workload would have
run at swap speed.

Therefore, on macOS:

- **Plan against the runtime figure** (`recommended_max_memory`), **not** the allocation ceiling.
  Record the ceiling too, labelled as "allocatable-with-swap", so the reader understands why it
  exceeds the budget.
- Also record **free memory at idle** — on a unified pool the practical budget is
  `min(recommended_max_memory, free RAM at idle)`. Compute free as
  `(free + inactive + speculative + purgeable pages) × 16384` from `vm_stat`, and state
  `vm.swapusage`. A machine already using swap at idle has less headroom than either figure says.
- `vram_dedicated_gib` is `0` and `unified_memory` is `true`. Host↔device copies are cheap;
  streaming / offload schemes optimise a bottleneck that does not exist here.
- The wired limit can be raised with `sudo sysctl iogpu.wired_limit_mb=N`. Record the current
  value; **do not change it** during a scan. Mention it as a remedy in Constraints when a model
  narrowly misses the budget.

## M4. Apple Neural Engine (secondary accelerator)

Detect presence via `ioreg` (`AppleT8xxxANE…` / `H1xANE…` IOClass entries). Nothing in a default
scan exercises it — **state it as idle** unless a probe proves otherwise. Routes to it, in order of
practicality:

1. **ONNX Runtime `CoreMLExecutionProvider`** — probe in M6. Present ≠ ANE used; Core ML decides
   per-op at runtime and falls back to GPU/CPU silently.
2. **`coremltools`** — converts models to `.mlpackage`; `ct.ComputeUnit.CPU_AND_NE` requests the ANE.
3. **Not reachable from PyTorch MPS or MLX.** Neither targets the ANE. Say so in the directives so
   agents do not try.

`sudo powermetrics --samplers ane_power` is the only way to observe ANE utilisation; record it as
requires-sudo. Vendor TOPS figures are `[vendor spec]` and worthless until something has run on it.

## M5. Toolchain

| Item | Command | Gotcha |
|---|---|---|
| Xcode / CLT | `xcode-select -p`, `xcodebuild -version`, `clang --version` | A beta Xcode path (`Xcode-beta.app`) is worth recording — SDK and clang versions move |
| Metal toolchain | `xcrun -sdk macosx metal --version` | On Xcode 26+ the Metal compiler is a **separate download**; failure text says `xcodebuild -downloadComponent MetalToolchain`. Absent → cannot compile `.metal` shaders (MLX / llama.cpp JIT still work; they ship pre-compiled or use runtime compilation) |
| Homebrew | `which brew`, `brew --prefix` | `/opt/homebrew` = native arm64 ; `/usr/local` = x86_64 Homebrew, usually under Rosetta. Both can coexist — record which is on `PATH` |
| Rosetta | `pgrep -q oahd && echo installed` | Installed ≠ in use. `sysctl.proc_translated` says whether *this shell* is translated |
| Python | `command -v python3 python uv conda pyenv`, `python3 -c 'import platform,sys;print(platform.machine(),sys.executable)'`, `file "$(command -v python3)"` | See M5a |
| Node / Rust / Go / Java | `node --version`, `cargo --version`, `go version`, `java -version` | Record absent ones |
| Containers | `docker --version`, `podman --version`, `orb version` | Docker on macOS is always a Linux VM; GPU is **not** passed through. Record as a constraint for any "run it in Docker" plan involving Metal |
| Ollama | `ollama --version`, `ollama list`, `grep library= ~/.ollama/logs/server.log \| tail -1` | The log line proves Metal and reports the runtime memory figure |
| Vulkan / OpenCL | `which vulkaninfo clinfo`, `ls /opt/homebrew/lib/libMoltenVK.dylib` | Normally absent; MoltenVK is the only Vulkan route |

### M5a. Python architecture — the macOS version of the CPU-wheel lie

The Windows failure is a `+cpu` wheel. The macOS equivalent is **an x86_64 Python on an arm64
machine**: it runs (under Rosetta), `pip` installs `macosx_x86_64` wheels, `torch` imports, and
`torch.backends.mps.is_available()` returns **False** with no explanation. MLX will not install at
all.

Check every interpreter that matters, not just `python3`:

```bash
python3 -c 'import platform,sys; print(platform.machine(), sys.version.split()[0], sys.executable)'
file "$(command -v python3)"                          # universal / arm64 / x86_64
find ~ -maxdepth 4 -name .venv -type d 2>/dev/null    # project venvs
ls -d ~/miniconda3 ~/miniforge3 ~/anaconda3 /opt/homebrew/opt/python@3* 2>/dev/null
```

`/usr/bin/python3` is Apple's Command Line Tools Python (3.9 on current systems, universal
binary). It has no ML packages and should not be the probe target when a project venv exists.
**Probe the interpreter a task will actually use** and record its path in the frontmatter
(`python_probed`). Record `python_arch` verbatim: `arm64` is correct; `x86_64` is a 🔴 Blocker
for GPU work with the remedy "create a native venv: `arch -arm64 python3 -m venv .venv`".

## M6. Framework probes

Run each against the interpreter chosen in M5a. Record the exact output; absence is a finding.

### PyTorch (MPS)

```python
import torch, platform
print(torch.__version__, platform.machine())   # macOS wheels carry NO +suffix; arch is the finding
mb = torch.backends.mps
print("mps built", mb.is_built(), "available", mb.is_available(), "cuda", torch.cuda.is_available())
if mb.is_available():
    print("recommended_max_memory GiB", torch.mps.recommended_max_memory() / 2**30)
```

`is_built() == True, is_available() == False` means the wheel is fine but the OS/arch is not —
almost always Rosetta. `is_built() == False` is a wrong wheel (x86_64 or Linux).

### MLX

```python
import mlx.core as mx
print(mx.__version__, mx.default_device(), mx.device_info())
# device_info gives architecture, max_buffer_length, max_recommended_working_set_size, memory_size
a = mx.random.normal((2048, 2048)); mx.eval(a @ a)     # a real op — import alone proves nothing
```

MLX is Apple-Silicon-only and always uses unified memory; there is no device to choose. Time a
matmul in fp32 / fp16 / bf16 exactly as for torch and rank them.

### ONNX Runtime (CoreML provider)

```python
import onnxruntime as ort
print(ort.__version__, ort.get_available_providers())
# want: ['CoreMLExecutionProvider', 'CPUExecutionProvider'] ; CPU-only == no acceleration
```

The stock `onnxruntime` PyPI wheel for macOS **does** include CoreML. If only
`CPUExecutionProvider` appears, an x86_64 or Linux wheel is installed.

### llama.cpp / Metal

Only runs when something is present:

```bash
# Python binding
python -c 'import llama_cpp; print(llama_cpp.__version__, "gpu offload", llama_cpp.llama_supports_gpu_offload())'
# CLI build
which llama-cli llama-server && llama-cli --version
# Ollama (a bundled llama.cpp) — the log proves the backend
grep -oE 'library=[A-Za-z]+[^\n]*total="[^"]*"' ~/.ollama/logs/server.log | tail -1
```

`llama_supports_gpu_offload() == False` on Apple Silicon means the binding was built without
Metal (`CMAKE_ARGS="-DGGML_METAL=on" pip install --no-binary llama-cpp-python llama-cpp-python`
is the remedy). Ollama's `library=Metal` line and its `total=` figure are the Metal proof and the
runtime memory figure respectively.

### What "working" means here

Unchanged from the shared rule: a framework is working only if it reports a device **and completes
a real operation**. On MPS additionally record `fp64` as unsupported (`Cannot convert a MPS Tensor
to float64`) — it is the one common op that fails, and libraries that default to float64 (some
scientific code, some diffusers schedulers) hit it.

## M7. Verification tier specifics

The shared verification tier (SKILL.md Step 4) applies. On macOS:

- **Synchronise**: `torch.mps.synchronize()` / `mx.eval(...)` around every timing. Both runtimes
  are lazy or asynchronous; an unsynchronised timer measures nothing.
- **Dtype ranking**: measure it, but expect the margin to be small. The Apple GPU has no separate
  matrix engine for bf16 the way Intel XMX or NVIDIA tensor cores do; on an M4 the fp32 / fp16 /
  bf16 matmul times were within 8 % of each other. State the measured margin, and state that the
  reason to prefer 16-bit on this platform is **memory footprint**, not throughput. That is a
  different recommendation from the Windows/Intel case and agents must not import it blindly.
- **Allocation ceiling**: run it, but label the result per M3 and do **not** put it in
  `vram_allocatable_gib`. That key takes the runtime figure on unified memory. Put the OOM number
  in the body as "allocatable-with-swap" with the swap evidence.
- **Conditions**: `pmset -g batt` (AC / battery, charge), `pmset -g custom | grep lowpowermode`,
  `pmset -g therm`, and `uptime` load average. Thermal pressure needs sudo — say "not measured".
  A MacBook Air is passively cooled: sustained runs throttle, so same-session baselines
  (grounding §7) matter even more.

## M8. Frontmatter values specific to macOS

| Key | macOS value |
|---|---|
| `platform` | `darwin` |
| `os` | `macOS <ProductVersion>` |
| `os_build` | `BuildVersion` from `sw_vers` (e.g. `"25G83"`) — quote it |
| `cpu_arch` | `arm64` or `x86_64` from `uname -m` in a **native** shell |
| `cpu_cores` / `cpu_threads` | `hw.physicalcpu` / `hw.logicalcpu` — equal on Apple Silicon (no SMT) |
| `cpu_topology` | e.g. `4P+6E` from the perflevel sysctls |
| `accelerator` | `Apple M4` (chip name; GPU has no separate model) |
| `accelerator_vendor` | `apple` |
| `accelerator_kind` | `unified` |
| `gpu_cores` | `Total Number of Cores` from `SPDisplaysDataType` |
| `metal_version` | `Metal Support` line, e.g. `Metal 4` |
| `torch_device` | `mps` |
| `unified_memory` | `true` |
| `vram_dedicated_gib` | `0` |
| `vram_allocatable_gib` | **`recommended_max_memory`**, not the OOM ceiling (M3) |
| `ram_upgradeable` | `false` |
| `python_probed` / `python_arch` | Path and arch of the interpreter the probes ran in |
| `rosetta_translated` | From `sysctl.proc_translated` |
| `virtualization_enabled` | From `kern.hv_support` |
| `secondary_accelerators` | ANE entry, `status: idle` unless exercised, `reachable_via: Core ML` |
| `frameworks` | Add `mlx` and `llama_cpp` / `ollama` entries when present |
| `forbidden` | Always: `cuda`, `nvidia-smi`, `bitsandbytes`, `rocm`, `xpu`, `torch.float64 on mps`; plus `mlx` on Intel Macs |

## M9. Agent Directive phrasing on macOS

Typical directives (grounding §3):

> - Target **`mps`** (`.to("mps")`, `device_map="mps"`). No CUDA, no ROCm, no XPU: `.cuda()`,
>   `bitsandbytes`, `nvidia-smi` will fail. MLX is the alternative native path.
> - Plan against **N GiB** of GPU memory (Metal working-set budget), not the 16 GiB total. The
>   allocator will let you exceed it — it swaps instead of failing. Watch `vm.swapusage`.
> - Prefer **fp16/bf16 for footprint**; measured throughput gain over fp32 is only ~X %.
> - **`float64` is unsupported on MPS.** Cast to float32 before `.to("mps")`.
> - The ANE is not reachable from PyTorch or MLX. Use Core ML / ONNX Runtime CoreML EP.
> - Use **10** worker threads (10 cores, no SMT); prefer the 4 performance cores for latency work.
> - Docker here is a Linux VM with no GPU. Do not plan Metal work inside a container.
> - Free disk is **N GiB** — model downloads above that will fail; clear `~/.ollama` / caches first.
