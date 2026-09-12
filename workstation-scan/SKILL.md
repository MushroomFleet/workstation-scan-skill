---
name: workstation-scan
description: >-
  Scan the local machine and produce or refresh `Workstation.md` — a hardware, accelerator and
  toolchain profile for agents and humans, so an agent immediately knows which device to target,
  the fastest dtype, how much memory it can actually allocate, which frameworks genuinely work
  versus are merely installed, and which approaches are dead ends here. Creates it on first run;
  updates in place after. Use whenever the user says "use the
  workstation-scan skill", "scan this workstation", "create/update the Workstation.md", "profile
  this machine", "what are my specs", or "can this machine run X", or asks about GPU/VRAM/CPU/RAM
  — and proactively before any hardware-dependent decision: choosing model size or quantisation,
  picking CUDA vs ROCm vs XPU vs Metal vs CPU, sizing batch/thread counts, or diagnosing why a GPU
  "isn't being used". ALWAYS use this rather than answering from memory or one command: reported
  and actual capability routinely disagree, and this artifact's value is that every claim was
  measured.
---

# Workstation Scan

This skill produces and maintains **`Workstation.md`**: a single Markdown profile of the machine it
is run on, structured so an agent can parse the decisive facts in seconds and a human can read the
reasoning behind them.

The artifact exists to answer one question reliably: **what can this machine actually do, right
now?** Not what the hardware is nominally capable of, and not what is nominally installed — what
works when invoked.

This SKILL.md governs **how to scan and the create-vs-update mechanics**. The bundled grounding
governs **what goes into the document**. Read the grounding before writing anything.

## Step 0 — Read the standard (required)

Read `references/Workstation-grounding.md` in full before scanning or writing. It defines the
document structure, the frontmatter schema, the evidence-labelling contract, the severity taxonomy,
and the quality gates. It is authoritative on *what goes in*; if this file appears to conflict with
it on content, the grounding wins.

Do **not** copy the grounding into the target directory. Only `Workstation.md` is written out.

## Step 1 — Choose the mode

Check for `Workstation.md` in the working directory.

- **Absent → Create mode.** Full scan, compose fresh, write.
- **Present → Update mode.** Read it, re-scan, reconcile against current reality, rewrite in place.

**The file's presence decides the mode, not the user's wording.** If the user says "create" and a
file exists, switch to Update and say so. Preserve any `## Case Study` / `## Session Log` sections
an earlier run or a human added — append, never silently discard recorded history.

In Update mode, **explicitly diff**: state what changed since the recorded profile (driver
versions, new runtimes, freed or consumed disk, resolved constraints). A profile whose value is
trust must show its own drift.

## Step 2 — Identify the platform, then use the right commands

Detect the OS first and use only that column. **Never assume a command exists — check, and record
the failure if it does not.**

| Target | Windows | Linux | macOS |
|---|---|---|---|
| OS / build | `Get-CimInstance Win32_OperatingSystem`, `systeminfo` | `/etc/os-release`, `uname -a` | `sw_vers`, `uname -a` |
| CPU | `Get-CimInstance Win32_Processor` | `lscpu`, `/proc/cpuinfo` | `sysctl -n machdep.cpu.brand_string`, `system_profiler SPHardwareDataType` |
| Memory | `Get-CimInstance Win32_PhysicalMemory` | `/proc/meminfo`, `dmidecode -t memory` | `sysctl hw.memsize`, `system_profiler SPMemoryDataType` |
| GPU | `Get-CimInstance Win32_VideoController` | `lspci -nnk \| grep -A3 -i vga`, `glxinfo` | `system_profiler SPDisplaysDataType` |
| Accelerators | `Get-PnpDevice -Class ComputeAccelerator` | `lspci`, `/sys/class/` | `system_profiler SPHardwareDataType` (ANE) |
| Storage | `Get-CimInstance Win32_DiskDrive`, `Get-Volume` | `lsblk`, `df -h` | `diskutil list`, `df -h` |
| Board / BIOS | `Get-CimInstance Win32_BaseBoard`, `Win32_BIOS` | `dmidecode -t baseboard -t bios` | `system_profiler SPHardwareDataType` |
| Power | `powercfg /getactivescheme`, `Win32_Battery` | `tlp-stat`, `/sys/class/power_supply/` | `pmset -g` |

### ⚠️ Windows: `wmic` is removed on modern builds

On Windows 11 builds ≈26100+ `wmic` returns "command not found". **Use `Get-CimInstance`.** If a
`wmic` invocation fails, do not retry variants — switch immediately and note the OS build in the
document.

### Do not trust a single source for memory

GPU memory is reported differently by every layer and the numbers genuinely disagree. Record each
distinctly (see grounding §4):

- **OS/driver reported** (e.g. WDDM total graphics memory, registry `qwMemorySize`) — an accounting
  figure, usually the largest and least useful.
- **Runtime reported** (CUDA/Level Zero/Metal/ROCm `total_memory`) — the allocator's budget.
- **Actually allocatable** — what a probe successfully allocates before OOM. *This is the number
  that governs decisions.*

Where they differ, **say so and name which one to plan against.**

## Step 3 — Probe the accelerator reality (the core of this skill)

**This step is what makes the artifact worth having. Never skip it.** Presence of hardware and
presence of a library prove nothing about whether they work together.

Run whichever apply, and record exact output:

```bash
nvidia-smi                       # NVIDIA
rocm-smi / rocminfo              # AMD
sycl-ls                          # Intel oneAPI / Level Zero
vulkaninfo --summary             # Vulkan (any vendor)
clinfo                           # OpenCL
```

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

### The failure this catches

A machine can have a healthy GPU, a current driver, a working vendor runtime, and `torch`
installed — and still run every workload on the CPU, because the installed wheel is the `+cpu`
build with no device kernels compiled in. `torch.xpu` / `torch.cuda` will *exist as an API surface*
and report **zero devices**. Nothing warns you; throughput is simply 10–40× lower than it should
be.

**Therefore:**
- A framework is "working" only if it reports **≥1 device** and completes a real operation.
- Record the **build variant suffix** verbatim. `2.14.0+cpu` and `2.14.0+xpu` are not the same fact.
- If a device is unavailable, diagnose **why** (wrong wheel / missing runtime / driver / disabled in
  firmware) and record the specific remedy.

Also check for a dependency file that would silently undo a working setup — e.g. a
`requirements.txt` pinning `torch>=2.x` with no accelerator index URL, which reinstalls the CPU
wheel on the next clean install. Record it as a constraint.

## Step 4 — Verify capability with real operations

Capability claims must be executed, not inferred. Keep this tier fast (seconds).

1. **Op coverage.** Run the operations real workloads need — for ML: conv2d, layer/group norm,
   scaled-dot-product attention across dtypes, softmax, embedding, autocast + backward, and a
   host↔device round trip. Report an *n/total passed* count. A device that loads models but lacks a
   kernel is worse than one that fails loudly.
2. **Dtype ranking.** Time a small matmul in each supported dtype (fp32, bf16, fp16). **Do not
   assume** which is fastest — matrix-engine paths can make one dtype several times faster than
   another, and vendor capability flags sometimes contradict measured behaviour. Record measured
   order and name the winner as the recommended default dtype.
3. **Allocation ceiling.** Allocate increasing buffers until OOM to find the real limit.

### Heavier benchmarking is opt-in

Sustained throughput benchmarks (full model runs, long token-generation tests) are **not** part of
a default scan — they take minutes and generate heat. Run them only when the user asks, or when a
specific decision depends on them. If run, obey the benchmarking rules in grounding §7 — above all:
**baseline and variant must be measured in the same session**, because thermal drift on laptops can
move prompt-processing figures by 1.5× on identical inputs.

## Step 5 — Write `Workstation.md`

Compose per the grounding. Non-negotiable requirements:

- **Machine-readable frontmatter first** (grounding §2), so an agent gets the decisive facts
  without parsing prose.
- **Agent Directives near the top** (grounding §3) — the imperative do/don't list. An agent that
  reads only the frontmatter and that section must still make correct choices.
- **Every claim labelled `[measured]` or `[vendor spec]`** (grounding §5). No unlabelled assertions.
- **A constraint table with severities** (grounding §6), blockers first.
- **Strengths stated as well as limits.** A profile listing only problems leads agents to
  underestimate the machine and refuse feasible work.

## Step 6 — Quality gates

Do not report completion until all pass:

- [ ] Every numeric claim is labelled `[measured]` or `[vendor spec]`.
- [ ] **Nothing is asserted that was not actually checked.** Power source, thermal state, link
      speed and driver dates are all easy to guess wrong — run the command or omit the claim.
- [ ] Accelerator availability was probed by execution, not inferred from installed packages.
- [ ] Framework build-variant suffixes recorded verbatim.
- [ ] The three memory figures are distinguished, and the plan-against number is named.
- [ ] Recommended dtype is backed by a measurement, not by reputation.
- [ ] Constraints carry severities; blockers state a concrete remedy.
- [ ] Frontmatter agrees with the body (no stale values after an update).
- [ ] In Update mode: changes since the previous profile are stated, and prior case-study /
      session-log sections are preserved.
- [ ] Internal consistency: figures repeated in several sections match, or the discrepancy is
      explained.

## Correcting the record

If a later measurement contradicts something the document already says, **fix the claim and mark
the correction inline** — do not quietly overwrite. The artifact's value rests on being trusted, so
a visible correction is worth more than a silent edit. Mark superseded constants clearly and point
to the section that replaced them.

## Reporting to the user

Summarise in a few lines: the machine, the accelerator and its working status, the single most
consequential finding (usually a blocker or a corrected assumption), and where the file was
written. Lead with anything that changes what they should do next.
