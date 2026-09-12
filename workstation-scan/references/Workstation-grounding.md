# `Workstation.md` — Document Standard

Authoritative specification for the artifact produced by the `workstation-scan` skill. The SKILL.md
governs *how to scan*; this governs *what the document contains*. On any conflict about content,
this file wins.

## Contents

1. [Purpose and audience](#1-purpose-and-audience)
2. [Frontmatter schema](#2-frontmatter-schema)
3. [Agent Directives section](#3-agent-directives-section)
4. [Required sections](#4-required-sections)
5. [Evidence labelling contract](#5-evidence-labelling-contract)
6. [Constraint severity taxonomy](#6-constraint-severity-taxonomy)
7. [Benchmarking rules](#7-benchmarking-rules)
8. [Case studies and session log](#8-case-studies-and-session-log)
9. [Anti-patterns](#9-anti-patterns)
10. [Skeleton](#10-skeleton)

---

## 1. Purpose and audience

**Primary reader: an AI agent** deciding how to execute a task on this machine. It needs to learn,
in seconds and without ambiguity: the compute device to target, the fastest dtype, the memory it may
allocate, which frameworks work, and which approaches are already known to fail.

**Secondary reader: a human** who needs the reasoning, the measurements, and the caveats.

Serve both by ordering the document **decisive-first**: machine-readable frontmatter, then
imperative directives, then evidence, then detail. An agent that reads only the first two sections
must still make correct choices; a human reading on must find the justification.

The document is **descriptive, not aspirational.** It records what the machine does today. Planned
upgrades, vendor marketing figures, and recalled context never override a measurement.

### Scope boundary

In scope: hardware, accelerators, drivers, compute runtimes, toolchains, framework functionality,
measured performance, and the constraints these create.

Out of scope: project code structure, application architecture, business logic. It profiles the
*machine*, not the work. **Never record secrets, tokens, licence keys, full serial numbers, network
credentials, or private IP/hostname inventories** — a hardware profile is routinely shared and
committed. A drive model is fine; its serial number is not.

---

## 2. Frontmatter schema

Open with YAML frontmatter carrying the decisive facts. Keys are fixed so agents can rely on them.
Omit a key only when genuinely inapplicable; use `null` when unknown.

```yaml
---
scan_date: 2026-09-12
hostname: MSI
os: Windows 11 Pro
os_build: "10.0.26200"
platform: win32                        # win32 | darwin | linux
cpu_arch: x86_64                       # x86_64 | arm64
rosetta_translated: null               # macOS only: true if the scan shell ran under Rosetta
virtualization_enabled: true           # firmware VT-x/AMD-V on Windows/Linux; kern.hv_support on macOS

cpu: Intel Core Ultra 7 258V
cpu_cores: 8
cpu_threads: 8            # if == cores, there is no SMT; agents must not assume 2x
cpu_topology: 4P+4LPE     # P/E core split where the CPU has one; null otherwise
ram_total_gib: 31.5
ram_free_idle_gib: 13.2
ram_upgradeable: false

accelerator: Intel Arc 140V
accelerator_vendor: intel              # nvidia | amd | intel | apple | none
accelerator_kind: integrated           # discrete | integrated | unified
gpu_cores: 8                           # Xe-cores / SMs / CUs / Apple GPU cores; null if unknown
metal_version: null                    # macOS only, e.g. "Metal 4"
torch_device: xpu                      # cuda | rocm | xpu | mps | cpu
unified_memory: true                   # true => host<->device copies are cheap
vram_dedicated_gib: 0
vram_allocatable_gib: 16.0             # plan-against figure — see "The three memory figures"
recommended_dtype: bfloat16            # measured fastest, not assumed
recommended_dtype_reason: throughput   # throughput | footprint — why it was chosen
memory_bandwidth_gbs_effective: 83     # measured; null if not measured

python_probed: C:/proj/.venv/Scripts/python.exe   # the interpreter the framework probes ran in
python_arch: x86_64

secondary_accelerators:
  - name: Intel AI Boost NPU
    status: idle                       # working | idle | unusable
    reachable_via: OpenVINO

frameworks:
  torch: { version: 2.14.0+xpu, device: xpu, working: true }
  onnxruntime: { version: 1.29.0, providers: [CPUExecutionProvider], working: false }

storage_free_gib: 119.8
storage_volumes: 1

blockers:
  - ONNX Runtime has no accelerated execution provider
  - Hardware virtualization disabled in firmware (no Docker/WSL2)

forbidden:                             # will fail on this machine — never attempt
  - cuda
  - bitsandbytes
  - nvidia-smi
---
```

The same keys on an Apple Silicon Mac. Note which values change meaning, not just value:

```yaml
---
scan_date: 2026-09-12
hostname: SKYNET
os: macOS 26.6.2
os_build: "25G83"
platform: darwin
cpu_arch: arm64
rosetta_translated: false
virtualization_enabled: true

cpu: Apple M4
cpu_cores: 10
cpu_threads: 10                        # no SMT on Apple Silicon
cpu_topology: 4P+6E
ram_total_gib: 16.0
ram_free_idle_gib: 6.8
ram_upgradeable: false

accelerator: Apple M4
accelerator_vendor: apple
accelerator_kind: unified
gpu_cores: 8
metal_version: "Metal 4"
torch_device: mps
unified_memory: true
vram_dedicated_gib: 0
vram_allocatable_gib: 11.84            # Metal working-set budget — NOT the OOM ceiling (19 GiB, swapped)
recommended_dtype: bfloat16
recommended_dtype_reason: footprint    # measured only 8% faster than fp32; chosen for memory
memory_bandwidth_gbs_effective: null

python_probed: /Users/me/proj/.venv/bin/python
python_arch: arm64

secondary_accelerators:
  - name: Apple Neural Engine
    status: idle
    reachable_via: Core ML / onnxruntime CoreMLExecutionProvider

frameworks:
  torch: { version: 2.10.0, device: mps, working: true }
  mlx: { version: null, working: false }            # not installed
  onnxruntime: { version: null, working: false }    # not installed
  ollama: { version: 0.33.3, backend: Metal, working: true }

storage_free_gib: 7.1
storage_volumes: 1

blockers:
  - 7 GiB free disk — no room for models above ~5 GB
  - float64 unsupported on MPS

forbidden:
  - cuda
  - nvidia-smi
  - bitsandbytes
  - xpu
  - torch.float64 on mps
---
```

### Notes on specific fields

- **`cpu_threads`** — always state it. Modern CPUs may ship without SMT; an agent assuming
  `threads = 2 × cores` will oversubscribe.
- **`vram_allocatable_gib`** — the plan-against figure from §4, never the OS-reported figure. On a
  discrete GPU that is the measured OOM ceiling; on a unified-memory machine it is the runtime's
  working-set budget, because the OOM ceiling there includes swap.
- **`unified_memory`** — `true` changes design decisions: host↔device transfer is nearly free, so
  streaming/offload schemes optimise a bottleneck that does not exist, while GPU allocations consume
  the same pool as the host — and can exceed it by swapping.
- **`recommended_dtype`** — must come from a measurement. Vendor capability flags sometimes
  contradict measured throughput. **`recommended_dtype_reason`** says whether the margin was large
  enough to be a throughput choice or whether the dtype is recommended only to halve memory; an
  agent porting a "bf16 is 8× faster" assumption from one machine to another is a real failure.
- **`python_probed` / `python_arch`** — the profile describes one interpreter's reality. Name it.
  On macOS an `x86_64` here is the decisive finding.
- **Platform-inapplicable keys** (`metal_version` on Windows, `rosetta_translated` on Linux) are
  `null`, not omitted, so agents can rely on the key set.
- **`forbidden`** — the highest-value field for preventing wasted work. List tools and APIs that
  cannot work here.

---

## 3. Agent Directives section

Immediately after the frontmatter, before any hardware detail. Imperative, scannable, and about
**decisions**, not description.

Cover:

1. **The device to target**, and the exact idiom (`.to("xpu")`, `device_map="cuda"`, `mps`, …).
2. **What will fail** — wrong-vendor tooling, absent runtimes, firmware-disabled features.
3. **The dtype to prefer**, with the measured margin.
4. **The memory budget**, and that GPU allocations may share the host pool.
5. **Thread/parallelism sizing.**
6. **Platform command gotchas** (e.g. "`wmic` is unavailable; use `Get-CimInstance`").
7. **Known dead ends** — recorded so they are not re-attempted.

Write them as instructions to a reader who will act:

> - Target **`xpu`**. This machine has **no CUDA**; `.cuda()`, `bitsandbytes` and `nvidia-smi` will
>   fail. Use `torch.xpu`, SYCL, Level Zero or Vulkan.
> - Prefer **`bfloat16`** — measured **~8× faster than fp32** on this GPU.
> - Plan against **16 GiB** of GPU memory, shared with system RAM; only ~13 GiB is free at idle.
> - Use **8** worker threads, never 16 — 8 cores, no SMT.

And the same section on an Apple Silicon Mac — note the memory and dtype directives say something
structurally different, not just different numbers:

> - Target **`mps`**. No CUDA, no XPU: `.cuda()`, `bitsandbytes`, `nvidia-smi` will fail. MLX is
>   the other native path.
> - Plan against **11.8 GiB** of GPU memory (Metal working-set budget), not the 16 GiB total. The
>   allocator will grant more — measured 19 GiB — but it does so by swapping to SSD.
> - Prefer **bf16 for footprint**; it measured only **~8 % faster than fp32**. Do not expect a
>   matrix-engine speedup here.
> - **`float64` is unsupported on MPS** — cast to float32 first.
> - Use **10** threads (10 cores, no SMT); 4 are performance cores.

---

## 4. Required sections

In order. Merge or omit only where a section is genuinely inapplicable, and say so.

| # | Section | Must contain |
|---|---|---|
| 1 | **Platform Summary** | Make/model, form factor, board, firmware version + date, machine class, CPU architecture, and (macOS) whether the scan ran natively or under Rosetta. Flag thermally-constrained chassis — it changes how benchmarks must be read. |
| 2 | **CPU** | Model, cores, **threads**, SMT presence, core topology (P/E), base + measured boost, cache, **virtualization enabled or not**. |
| 3 | **Accelerator** | Model, vendor, driver + date, compute units, and **all three memory figures** (§below). State the addressing model (`cuda`/`xpu`/`mps`/…). |
| 4 | **Secondary accelerators** | NPU/ANE/other. State honestly whether anything has ever exercised it. |
| 5 | **Memory & Storage** | Capacity, type, speed, channels, **upgradeable or soldered**, free-at-idle, and free disk + volume count. Call out the tightest resource. |
| 6 | **OS & Compute Runtimes** | OS build, and a runtime table with present/absent/N-A status: CUDA, ROCm, Level Zero, Metal, Core ML, DirectML, Vulkan, OpenCL, OpenVINO. Mark vendor-impossible runtimes N/A (Metal on Windows, CUDA on a Mac), not "missing". |
| 7 | **Toolchain** | Language runtimes (with **architecture** on macOS), package managers, compilers, build tools, containers; and what is **missing** that tasks commonly need. |
| 8 | **Framework Reality Check** | The §Step-3 probes. Build-variant suffixes / interpreter architecture verbatim, the interpreter path probed, device counts, op-coverage result with named failures, provider lists. **The most important section.** |
| 9 | **Measured Performance** | Benchmarks with methodology and conditions (§7). |
| 10 | **Constraints** | Severity table (§6), blockers first, each with a remedy. |
| 11 | **Strengths** | What this machine is genuinely good at. Required — see §9. |
| 12 | **Recommended Next Steps** | Ranked, actionable; tick off completed items across updates. |

### The three memory figures

Always distinguish, and **name the one to plan against**:

| Figure | Source | Use |
|---|---|---|
| OS/driver reported | WDDM total, registry (Windows); `hw.memsize` (macOS) | Accounting only. Usually the largest. **Do not plan against it.** |
| Runtime reported | `total_memory` from CUDA / Level Zero / ROCm; Metal `recommendedMaxWorkingSetSize` (`torch.mps.recommended_max_memory()`) | The allocator's budget. **Plan against this on unified memory.** |
| Actually allocatable | Measured by allocating until failure | **Plan against this on a discrete GPU.** On unified memory, record it as *allocatable-with-swap* — evidence, not a budget. |

These commonly differ by several GiB. Reporting only the first is the single most misleading thing
this document can do. The second most misleading is planning against the OOM ceiling on a
unified-memory machine: measured on a 16 GB Apple Silicon Mac, PyTorch granted **19 GiB** before
refusing, because the OS swapped everything else out rather than fail. Always state which memory
model applies and therefore which row is the plan-against figure.

---

## 5. Evidence labelling contract

**Every factual claim carries its provenance.** Two labels:

- **`[measured]`** — this scan executed something and read the result.
- **`[vendor spec]`** — from published specifications; a ceiling, not an observation.

Rules:

1. An unlabelled numeric claim is a defect.
2. **Never label something `[measured]` that was inferred.** If a figure was not obtained by
   running a command, it is not measured.
3. **Never state a condition that was not checked.** Power source, thermal state, PCIe link width
   and charge level are frequent, embarrassing guesses. Run the command or omit the claim.
4. Where measured contradicts vendor spec, **report both and trust the measurement.** Such a gap is
   a finding: it usually reveals a real limit (thermal, bandwidth, driver).
5. Include the invoking command for anything a reader might want to re-run.

> This contract exists because the failure mode it prevents is *specifically* an assumption written
> confidently enough to be believed later. A profile whose claims cannot be distinguished by
> provenance is worse than no profile, because it will be trusted.

---

## 6. Constraint severity taxonomy

| Severity | Meaning | Agent behaviour |
|---|---|---|
| 🔴 **Blocker** | A class of work cannot run at all | Do not attempt. Apply the remedy or choose another approach. |
| 🟠 **High** | Materially limits scope; workaround exists | Plan around it explicitly. |
| 🟡 **Medium** | Affects tuning, sizing or reliability | Adjust parameters. |
| 🟢 **Low** | Informational; a papercut | Be aware. |

Each row: constraint, severity, **consequence**, and for blockers/high a **concrete remedy**. Order
blockers first. Mark resolved constraints with strikethrough and a pointer to the resolving
section rather than deleting them — the history is useful.

Typical blockers worth checking for: CPU-only framework wheels on an accelerated machine; an
x86_64 interpreter on an arm64 Mac; accelerated runtime absent; virtualization disabled in
firmware; insufficient disk for the model class; an architecture the installed engine does not
support.

---

## 7. Benchmarking rules

Benchmarks are only useful if reproducible and honestly bounded.

1. **State conditions**: power source, power plan/profile, thermal state, and whether other load
   was present. If not checked, say so rather than guessing.
2. **Same-session baselines.** Compare variant against baseline measured in the *same* session.
   Thermal drift on constrained chassis can move compute-bound figures by 1.5× on identical inputs.
   Cross-session comparison is invalid; say so where it matters.
3. **Report variance**, not just means (±, or several runs). A figure without spread hides drift.
4. **Separate compute-bound from bandwidth-bound** results — they behave differently and respond to
   different levers. Note when throughput falls off at larger sizes, which usually indicates a
   bandwidth wall rather than a compute limit.
5. **Prefer a model over a number.** Where several data points exist, fit the relationship (e.g.
   time-per-unit = fixed overhead + size ÷ bandwidth) and report the fit quality. A model predicts;
   a number only describes.
6. **State predictions before measuring** when testing a model, and record whether they held. A
   falsified prediction is a finding worth keeping.
7. **Two points cannot validate a two-parameter model** — they fit it exactly. Say so, and get a
   third point before presenting constants as established.
8. **Name the reliable metric.** If one measurement is stable and another noisy, say which to trust.

---

## 8. Case studies and session log

Optional, appended after the required sections, and **preserved across updates**.

A **Case Study** records a real task attempted on this machine: objective, what blocked it, what
worked, measurements, and closed avenues. These are disproportionately valuable — they convert
one-off debugging into reusable knowledge, and they stop future agents re-walking dead ends.

Include, where applicable:
- The blocker and its **root cause**, quoting the diagnostic output.
- The **working configuration**, with exact paths, versions and commands.
- **Closed avenues** under a clear "do not re-attempt" heading, with the reason.
- Non-obvious flags or config quirks required to make it work.

Keep them factual and dated. Do not let a case study contradict the main sections — reconcile, and
mark corrections per SKILL.md.

---

## 9. Anti-patterns

**Do not:**

- **List only problems.** A deficit-only profile causes agents to refuse feasible work. §11 is
  required for this reason.
- **Report vendor specs as achievements.** "64 TOPS" means nothing if nothing has ever run on it.
  State idle hardware as idle.
- **Give one memory number.** See §4.
- **Plan against the OOM ceiling on unified memory.** It includes swap. See §4.
- **Infer a dtype ranking.** Measure it — and report the margin, so a negligible one is not read
  as a large one.
- **Probe the wrong interpreter.** The system Python is rarely what a task runs in. Name the one
  probed.
- **Assume SMT.** Check threads.
- **Trust a library's presence as capability.** Probe the device.
- **Assert unverified conditions** (power source, thermals, link speed). The most likely error in
  the whole document.
- **Bury the decisive fact.** If the GPU is unusable, that belongs in the frontmatter and the
  directives, not in §8.
- **Record secrets, serial numbers, or private network inventories.**
- **Silently overwrite a corrected claim.** Mark corrections.
- **Let frontmatter drift from the body** after an update.

---

## 10. Skeleton

```markdown
---
<frontmatter per §2>
---

# Workstation Profile — <hostname>

**Scanned:** <date> · **Machine:** <make/model> · **OS:** <os build>
**Status:** <one line: accelerator and whether it works>

## Agent Directives
<imperative do/don't list per §3>

## 1. Platform Summary
## 2. CPU
## 3. Accelerator
## 4. Secondary Accelerators
## 5. Memory & Storage
## 6. OS & Compute Runtimes
## 7. Toolchain
## 8. Framework Reality Check
## 9. Measured Performance
## 10. Constraints
## 11. Strengths
## 12. Recommended Next Steps

---
## Case Study — <task> (<date>)     <!-- optional, preserved across updates -->

---
*Collected on <platform> via `references/platform-<os>.md` using <commands>. <Notes on unavailable commands and anything requiring elevated privileges that was not run.>*
```

Tables over prose for anything enumerable. Callouts (`>`) for facts that change decisions. Fenced
blocks for verbatim command output — quoting real output is far more convincing, and more useful,
than paraphrasing it.
