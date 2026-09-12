# 🖥️ workstation-scan

**An Agent Skill that teaches Claude what your machine can actually do.**

`workstation-scan` profiles the computer it runs on and writes **`Workstation.md`** — a hardware,
accelerator and toolchain report structured for an AI agent to parse in seconds and for a human to
read in full. Drop it in a repo and any agent working there stops guessing about your hardware.

**V2 runs on Windows and macOS** (Apple Silicon and Intel Macs) with one shared procedure and one
document format, so a `Workstation.md` from either platform has the same keys and the same
structure.

Its governing principle is one line long:

> **Reported capability and actual working capability routinely disagree. Measure, don't assume.**

---

## ❓ Why this exists

Coding agents make hardware-dependent decisions constantly — which device to target, which dtype to
use, how big a model will fit, how many threads to spawn — and they usually make them from
guesswork, a half-remembered `nvidia-smi`, or an assumption that the machine looks like the average
machine in their training data.

That guesswork fails in a specific and expensive way. Consider a real case this skill was built
from:

A laptop with a healthy Intel Arc GPU, a current driver, a working oneAPI runtime, and PyTorch
installed. Everything *looked* correct. But the installed wheel was `torch==2.14.0+cpu`, which
ships no GPU kernels. `torch.xpu` existed as an API surface and reported **zero devices**. Nothing
warned, nothing errored — every workload simply ran on the CPU at **1/39th** of the machine's
measured bf16 throughput.

No amount of reading spec sheets would catch that. Running one command would.

The Mac version of the same lie, found while building V2: on a 16 GB Apple Silicon MacBook,
PyTorch's MPS allocator **granted 19 GiB** before refusing. It did not fail — macOS paged
everything else out to the SSD. An agent that "measured" the allocation ceiling and planned against
it would have been planning against swap. The real budget was the Metal working set: **11.8 GiB**.

`workstation-scan` catches exactly this class of problem, then writes the answer down so it is
found once rather than rediscovered.

### What it prevents

- Agents reaching for CUDA on a machine that has none
- "The GPU isn't being used" mysteries caused by a CPU-only wheel
- Planning around VRAM figures the allocator will never grant
- Assuming `threads = 2 × cores` on a CPU without SMT
- Choosing fp32 where a matrix engine makes bf16 several times faster — or assuming bf16 is
  several times faster on a GPU where it measured 8 % faster
- Planning against a unified-memory "allocation ceiling" that is really swap
- Probing the system Python when the project runs in an arm64 venv (or the reverse, under Rosetta)
- Re-attempting an approach that already failed on this hardware last week

---

## 📄 What it produces

A single `Workstation.md`, ordered **decisive-first** so both audiences are served.

**1. Machine-readable frontmatter** — fixed keys, so an agent gets the operative facts without
parsing prose:

```yaml
---
accelerator: Intel Arc 140V
accelerator_vendor: intel
torch_device: xpu
unified_memory: true
vram_dedicated_gib: 0
vram_allocatable_gib: 16.0        # MEASURED ceiling — plan against this
recommended_dtype: bfloat16       # measured fastest, not assumed
cpu_cores: 8
cpu_threads: 8                    # == cores, so no SMT
frameworks:
  torch: { version: 2.14.0+xpu, device: xpu, working: true }
  onnxruntime: { version: 1.29.0, providers: [CPUExecutionProvider], working: false }
forbidden: [cuda, bitsandbytes, nvidia-smi]
---
```

**2. Agent Directives** — imperative, positioned so an agent that reads *only* the frontmatter and
this section still chooses correctly:

> - Target **`xpu`**. This machine has **no CUDA**; `.cuda()`, `bitsandbytes` and `nvidia-smi` will
>   fail.
> - Prefer **`bfloat16`** — measured **~8× faster than fp32** on this GPU.
> - Plan against **16 GiB** of GPU memory, shared with system RAM; only ~13 GiB is free at idle.
> - Use **8** worker threads, never 16.

**3. Twelve evidence sections** — platform, CPU, accelerator, secondary accelerators (NPU/ANE),
memory & storage, OS & compute runtimes, toolchain, **framework reality check**, measured
performance, constraints, strengths, and ranked next steps.

---

## 🔬 What makes it different

Most system-info tools print what the OS claims. This one is built around the places that claim is
wrong.

### Capability is probed by execution, not inferred

A framework counts as working only if it reports **≥ 1 device** and completes a real operation.
Build-variant suffixes are recorded **verbatim**, because `2.14.0+cpu` and `2.14.0+xpu` are not the
same fact.

### Three memory figures, not one

GPU memory is reported differently by every layer, and the numbers genuinely disagree:

| Figure | Source | Use |
|---|---|---|
| OS/driver reported | WDDM total, registry, `system_profiler` | Accounting only — usually the largest |
| Runtime reported | CUDA / Level Zero / Metal / ROCm `total_memory` | The allocator's view |
| **Actually allocatable** | **Measured by allocating until OOM** | **Plan against this** |

On the machine above these read **18 GiB / 16.46 GiB / 16 GiB**. Reporting only the first is the
most misleading thing such a document can do.

Which of the last two is the plan-against figure depends on the memory model, and the skill says
which. On a discrete GPU the measured ceiling governs. On **unified memory** (Apple Silicon,
integrated GPUs) the runtime budget governs, because the ceiling includes swap — on the M4 MacBook
the three figures read **16 GiB / 11.84 GiB / 19 GiB**, and only the middle one is safe to plan
against.

### Every claim carries its provenance

Facts are labelled **`[measured]`** or **`[vendor spec]`**. An unlabelled number is a defect, and
the standard explicitly forbids asserting conditions that were never checked — power source,
thermal state, link width — because a confident unverified claim is worse than a missing one. It
will be trusted later.

### Benchmarks come with methodology

Same-session baselines are mandatory: thermal drift on a constrained chassis moved a compute-bound
figure by **1.54× on identical inputs** during development. The standard also requires reporting
variance, stating predictions before measuring, and admitting that two data points *fit* a
two-parameter model rather than validating it.

### Strengths are required, not optional

A deficit-only profile makes agents refuse work the machine can do. The standard mandates a
Strengths section for exactly this reason.

---

## 📦 Installation

Clone into your Claude Code skills directory:

```bash
# Personal skill (all projects)
git clone https://github.com/MushroomFleet/workstation-scan-skill \
  ~/.claude/skills/workstation-scan

# Or project-scoped
git clone https://github.com/MushroomFleet/workstation-scan-skill \
  .claude/skills/workstation-scan
```

On Windows:

```powershell
git clone https://github.com/MushroomFleet/workstation-scan-skill `
  "$env:USERPROFILE\.claude\skills\workstation-scan"
```

The skill is picked up on the next session. No dependencies beyond what it probes.

---

## 🚀 Usage

Just ask:

```
use the workstation-scan skill
```

It also triggers on `scan this workstation`, `profile this machine`, `what are my specs`,
`can this machine run X`, `create the Workstation.md`, `update the Workstation.md`, and on questions
about GPU, VRAM, CPU or RAM.

It is written to trigger **proactively** too — before choosing a model size or quantisation, picking
between CUDA / ROCm / XPU / Metal / CPU, sizing batch, thread or context counts, or diagnosing why a
GPU "isn't being used."

### Create and update

- **No `Workstation.md` present** → full scan, file written.
- **Already present** → re-scan, reconcile, rewrite in place, and **state what changed** since the
  last profile. Recorded case studies and session logs are preserved, never silently dropped.

### Heavy benchmarks are opt-in

A default scan stays fast: inventory plus a light device probe, op-coverage check, dtype ranking and
an allocation ceiling. Sustained throughput runs take minutes and generate heat, so they only happen
when you ask.

---

## 🗂️ Repository structure

```
workstation-scan/
├── SKILL.md                            # Shared procedure: mode, detect OS, probe, verify, write
├── references/
│   ├── Workstation-grounding.md        # The document standard (authoritative on content)
│   ├── platform-windows.md             # Windows commands, WDDM memory sources, +cpu-wheel probe
│   └── platform-macos.md               # macOS commands, unified-memory rules, Rosetta/MPS/MLX/ANE
└── README.md
```

The split is deliberate. `SKILL.md` governs *how to scan* and is platform-neutral, so it stays
short enough to load cheaply. It detects the OS and loads exactly one platform reference, which
carries that platform's commands and gotchas. The grounding governs *what goes in the document*
and is the same on every platform.

---

## 💻 Platform support

| Platform | Status | Reference |
|---|---|---|
| **Windows** | Full — `Get-CimInstance`, `Get-PnpDevice`, `powercfg`, registry; unchanged from V1 | `references/platform-windows.md` |
| **macOS — Apple Silicon** | Full — `system_profiler`, `sysctl`, `pmset`, `diskutil`, `ioreg`; MPS, MLX, Core ML, llama.cpp/Ollama Metal | `references/platform-macos.md` |
| **macOS — Intel** | Degraded path — inventory and Metal-on-AMD; no ANE, no MLX | `references/platform-macos.md` §M1 |
| **Linux** | Fallback command table in `SKILL.md`; shared probes apply | — |

Accelerators: **NVIDIA** (CUDA), **AMD** (ROCm), **Intel** (XPU / Level Zero / SYCL), **Apple**
(Metal / MPS / MLX), plus **Vulkan**, **OpenCL**, **DirectML**, **OpenVINO** and **Core ML**
runtimes, and NPU/ANE detection.

> **Windows note baked into the skill:** `wmic` is removed on Windows 11 builds ≈26100+. The skill
> uses `Get-CimInstance` and knows not to retry `wmic` variants when it fails — a failure that
> reads like a path error but isn't.

> **macOS notes baked into the skill:** the scan never needs `sudo` (thermal and ANE power figures
> from `powermetrics` are recorded as "not run"); `df -h /` reports the sealed system snapshot, so
> the Data volume is measured instead; `system_profiler` prints serial numbers and UUIDs that must
> be stripped; the system `python3` is a universal 3.9 with no ML packages, so the skill hunts for
> the project venv and records which interpreter it probed; and `float64` on MPS is a known dead
> end, recorded up front.

### What changed in V2

- `SKILL.md` is now platform-neutral. Windows procedure moved verbatim to
  `references/platform-windows.md`; macOS procedure is new in `references/platform-macos.md`.
- Frontmatter gained fixed keys: `cpu_arch`, `cpu_topology`, `gpu_cores`, `metal_version`,
  `rosetta_translated`, `virtualization_enabled`, `python_probed`, `python_arch`,
  `recommended_dtype_reason`. Keys that do not apply to a platform are `null`, never omitted.
- The memory rule now says which figure to plan against **per memory model** — measured ceiling on
  discrete GPUs, runtime budget on unified memory.
- Dtype ranking must report its margin, and `recommended_dtype_reason` says whether the dtype is
  chosen for throughput or for footprint.
- The "expensive lie" is documented per platform: the `+cpu` wheel on Windows, the x86_64
  interpreter under Rosetta on macOS.
- Update mode refuses to "update" a profile written on a different platform — a cloned repo's
  Windows `Workstation.md` is not the Mac it now sits on.

---

## 🔒 What it will not record

A hardware profile gets committed and shared, so the standard forbids writing secrets, tokens,
licence keys, full serial numbers, network credentials, or private host/IP inventories. A drive
model is recorded; its serial number is not.

---

## ⚖️ License

Licensed under the **Apache License 2.0**. See [`LICENSE`](LICENSE) for the full text.

---

## 📚 Citation

### Academic Citation

If you use this codebase in your research or project, please cite:

```bibtex
@software{workstation_scan_skill,
  title = {workstation-scan: an Agent Skill that profiles a machine's real, measured compute capability for AI agents},
  author = {Drift Johnson},
  year = {2026},
  url = {https://github.com/MushroomFleet/workstation-scan-skill},
  version = {2.0.0}
}
```

### Donate:

[![Ko-Fi](https://cdn.ko-fi.com/cdn/kofi3.png?v=3)](https://ko-fi.com/driftjohnson)
