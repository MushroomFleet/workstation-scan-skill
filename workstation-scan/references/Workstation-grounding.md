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
platform: win32

cpu: Intel Core Ultra 7 258V
cpu_cores: 8
cpu_threads: 8            # if == cores, there is no SMT; agents must not assume 2x
ram_total_gib: 31.5
ram_free_idle_gib: 13.2
ram_upgradeable: false

accelerator: Intel Arc 140V
accelerator_vendor: intel              # nvidia | amd | intel | apple | none
accelerator_kind: integrated           # discrete | integrated | unified
torch_device: xpu                      # cuda | rocm | xpu | mps | cpu
unified_memory: true                   # true => host<->device copies are cheap
vram_dedicated_gib: 0
vram_allocatable_gib: 16.0             # MEASURED ceiling — plan against this
recommended_dtype: bfloat16            # measured fastest, not assumed
memory_bandwidth_gbs_effective: 83     # measured; null if not measured

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

### Notes on specific fields

- **`cpu_threads`** — always state it. Modern CPUs may ship without SMT; an agent assuming
  `threads = 2 × cores` will oversubscribe.
- **`vram_allocatable_gib`** — the measured ceiling from §4, never the OS-reported figure.
- **`unified_memory`** — `true` changes design decisions: host↔device transfer is nearly free, so
  streaming/offload schemes optimise a bottleneck that does not exist, while GPU allocations consume
  the same pool as the host.
- **`recommended_dtype`** — must come from a measurement. Vendor capability flags sometimes
  contradict measured throughput.
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

---

## 4. Required sections

In order. Merge or omit only where a section is genuinely inapplicable, and say so.

| # | Section | Must contain |
|---|---|---|
| 1 | **Platform Summary** | Make/model, form factor, board, firmware version + date, machine class. Flag thermally-constrained chassis — it changes how benchmarks must be read. |
| 2 | **CPU** | Model, cores, **threads**, SMT presence, core topology (P/E), base + measured boost, cache, **virtualization enabled or not**. |
| 3 | **Accelerator** | Model, vendor, driver + date, compute units, and **all three memory figures** (§below). State the addressing model (`cuda`/`xpu`/`mps`/…). |
| 4 | **Secondary accelerators** | NPU/ANE/other. State honestly whether anything has ever exercised it. |
| 5 | **Memory & Storage** | Capacity, type, speed, channels, **upgradeable or soldered**, free-at-idle, and free disk + volume count. Call out the tightest resource. |
| 6 | **OS & Compute Runtimes** | OS build, and a runtime table with present/absent status: CUDA, ROCm, Level Zero, Metal, DirectML, Vulkan, OpenCL, OpenVINO. |
| 7 | **Toolchain** | Language runtimes, package managers, compilers, build tools; and what is **missing** that tasks commonly need. |
| 8 | **Framework Reality Check** | The §Step-3 probes. Build-variant suffixes verbatim, device counts, op-coverage result, provider lists. **The most important section.** |
| 9 | **Measured Performance** | Benchmarks with methodology and conditions (§7). |
| 10 | **Constraints** | Severity table (§6), blockers first, each with a remedy. |
| 11 | **Strengths** | What this machine is genuinely good at. Required — see §9. |
| 12 | **Recommended Next Steps** | Ranked, actionable; tick off completed items across updates. |

### The three memory figures

Always distinguish, and **name the one to plan against**:

| Figure | Source | Use |
|---|---|---|
| OS/driver reported | WDDM total, registry, `system_profiler` | Accounting only. Usually the largest. **Do not plan against it.** |
| Runtime reported | `total_memory` from CUDA / Level Zero / Metal / ROCm | The allocator's view. |
| **Actually allocatable** | **Measured by allocating until OOM** | **Plan against this.** |

These commonly differ by several GiB. Reporting only the first is the single most misleading thing
this document can do.

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

Typical blockers worth checking for: CPU-only framework wheels on an accelerated machine;
accelerated runtime absent; virtualization disabled in firmware; insufficient disk for the model
class; an architecture the installed engine does not support.

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
- **Infer a dtype ranking.** Measure it.
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
*Collected via <commands>. <Notes on unavailable commands.>*
```

Tables over prose for anything enumerable. Callouts (`>`) for facts that change decisions. Fenced
blocks for verbatim command output — quoting real output is far more convincing, and more useful,
than paraphrasing it.
