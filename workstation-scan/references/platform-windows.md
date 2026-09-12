# Platform Reference — Windows

Loaded by `SKILL.md` Step 2 when the detected platform is Windows. This file carries the
Windows-specific commands, gotchas and probes. The shared procedure (mode selection, probing
philosophy, verification tiers, quality gates) lives in `SKILL.md`; the document standard lives in
`Workstation-grounding.md`. Nothing here overrides either.

Windows profiling behaviour is unchanged from V1 of this skill. Every command below is the V1
command.

## W1. Detection

| Signal | Value on Windows |
|---|---|
| `platform` (frontmatter) | `win32` |
| Python `sys.platform` | `win32` |
| Shell | PowerShell (`pwsh` or Windows PowerShell). Do not assume `cmd.exe` semantics. |
| `$env:PROCESSOR_ARCHITECTURE` | `AMD64` or `ARM64` — record it; Windows on ARM changes which wheels exist. |

## W2. Inventory commands

**Never assume a command exists — check, and record the failure if it does not.**

| Target | Command |
|---|---|
| OS / build | `Get-CimInstance Win32_OperatingSystem`, `systeminfo` |
| CPU | `Get-CimInstance Win32_Processor` |
| Memory | `Get-CimInstance Win32_PhysicalMemory` |
| GPU | `Get-CimInstance Win32_VideoController` |
| Accelerators | `Get-PnpDevice -Class ComputeAccelerator` |
| Storage | `Get-CimInstance Win32_DiskDrive`, `Get-Volume` |
| Board / BIOS | `Get-CimInstance Win32_BaseBoard`, `Win32_BIOS` |
| Power | `powercfg /getactivescheme`, `Win32_Battery` |

### ⚠️ `wmic` is removed on modern builds

On Windows 11 builds ≈26100+ `wmic` returns "command not found". **Use `Get-CimInstance`.** If a
`wmic` invocation fails, do not retry variants — switch immediately and note the OS build in the
document.

### Virtualization

`Win32_Processor` exposes `VirtualizationFirmwareEnabled`; `systeminfo` lists the Hyper-V
requirements block. Record whether hardware virtualization is enabled in firmware — when it is off,
Docker Desktop and WSL2 cannot run, which is a 🔴 Blocker for container-based work.

## W3. The three memory figures on Windows

GPU memory is reported differently by every layer and the numbers genuinely disagree. Record each
distinctly (grounding §4):

| Figure | Windows source | Use |
|---|---|---|
| **OS/driver reported** | WDDM total graphics memory (`Win32_VideoController.AdapterRAM`, or the registry `qwMemorySize` under `HKLM\SYSTEM\CurrentControlSet\Control\Class\{4d36e968-…}`) | An accounting figure, usually the largest and least useful. |
| **Runtime reported** | CUDA / Level Zero / ROCm `total_memory` (e.g. `torch.cuda.get_device_properties(0).total_memory`, `torch.xpu.get_device_properties(0).total_memory`) | The allocator's budget. |
| **Actually allocatable** | Allocate increasing buffers until OOM | *This is the number that governs decisions.* |

Where they differ, **say so and name which one to plan against.** On an integrated GPU with shared
memory (Intel Arc iGPU, AMD APU) the WDDM figure includes a slice of system RAM that the runtime may
or may not grant; only the measured ceiling is trustworthy.

`AdapterRAM` is a 32-bit field and wraps above 4 GiB — read the registry `qwMemorySize` value
instead for any card with more than 4 GiB, and say which source produced the number.

## W4. Vendor and runtime tools

Run whichever apply, and record exact output. Absence is itself a finding — record "not found":

```powershell
nvidia-smi                       # NVIDIA
rocm-smi ; rocminfo              # AMD (rarely present on Windows; HIP SDK only)
sycl-ls                          # Intel oneAPI / Level Zero
vulkaninfo --summary             # Vulkan (any vendor)
clinfo                           # OpenCL
```

Runtime table for grounding §4 section 6 — check each: CUDA, ROCm, Level Zero, DirectML, Vulkan,
OpenCL, OpenVINO. Metal is always absent on Windows; record it as N/A, not "missing".

## W5. Framework probes

Then probe the **frameworks**, because this is where the common, expensive lie lives:

```python
import torch
print(torch.__version__)          # the +cpu / +cu124 / +xpu / +rocm SUFFIX IS THE FINDING
for name in ("cuda", "xpu", "mps"):
    be = getattr(torch, name, None)
    ok = bool(be and be.is_available())
    print(name, ok, be.device_count() if ok else 0)
```

```python
import onnxruntime as ort
print(ort.get_available_providers())   # CPUExecutionProvider only == no acceleration
```

Look for `DmlExecutionProvider` (DirectML), `CUDAExecutionProvider`, `OpenVINOExecutionProvider`.

### The failure this catches

A machine can have a healthy GPU, a current driver, a working vendor runtime, and `torch`
installed — and still run every workload on the CPU, because the installed wheel is the `+cpu`
build with no device kernels compiled in. `torch.xpu` / `torch.cuda` will *exist as an API surface*
and report **zero devices**. Nothing warns you; throughput is simply 10–40× lower than it should
be.

Also check for a dependency file that would silently undo a working setup — e.g. a
`requirements.txt` pinning `torch>=2.x` with no accelerator index URL, which reinstalls the CPU
wheel on the next clean install. Record it as a constraint.

## W6. Verification tier specifics

The shared verification tier (SKILL.md Step 4) applies unchanged: op coverage, dtype ranking,
allocation ceiling. On Windows:

- Use `torch.cuda.synchronize()` / `torch.xpu.synchronize()` around timings; kernel launches are
  asynchronous and an unsynchronised timer measures nothing.
- The allocation ceiling **is** the plan-against figure on a discrete GPU. On an integrated GPU
  sharing host RAM, also record free host RAM at idle — the two draw from one pool.
- Power plan (`powercfg /getactivescheme`) and AC/battery (`Win32_Battery.BatteryStatus`) are
  benchmark conditions per grounding §7. Run them; do not guess.

## W7. Frontmatter values specific to Windows

| Key | Windows value |
|---|---|
| `platform` | `win32` |
| `cpu_arch` | `x86_64` or `arm64` (from `PROCESSOR_ARCHITECTURE`) |
| `torch_device` | `cuda` / `xpu` / `cpu` (ROCm on Windows presents as `cuda` if at all — record the wheel suffix) |
| `unified_memory` | `false` for discrete GPUs; `true` for iGPU/APU shared pools |
| `vram_dedicated_gib` | From the runtime `total_memory`, or `0` for shared-memory iGPUs |
| `gpu_cores` | Execution units / SMs / CUs from the vendor tool, or `null` |
| `rosetta_translated` | `null` (not applicable) |
| `virtualization_enabled` | From `VirtualizationFirmwareEnabled` |

## W8. Agent Directive phrasing on Windows

The platform gotcha directive (grounding §3 item 6) on Windows is typically:

> - `wmic` is unavailable on this build; use `Get-CimInstance`.

Add a `forbidden` entry for every wrong-vendor tool that would fail here (e.g. on an Intel machine:
`cuda`, `bitsandbytes`, `nvidia-smi`).
