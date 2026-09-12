---
name: workstation-scan
description: >-
  Scan the local machine and produce or refresh `Workstation.md` — a hardware, accelerator and
  toolchain profile for agents and humans, so an agent immediately knows which device to target,
  the fastest dtype, how much memory it can actually allocate, which frameworks genuinely work
  versus are merely installed, and which approaches are dead ends here. Works on Windows and macOS
  (Apple Silicon and Intel), with Linux fallback commands. Creates it on first run; updates in
  place after. Use whenever the user says "use the workstation-scan skill", "scan this
  workstation", "create/update the Workstation.md", "profile this machine", "what are my specs",
  or "can this machine run X", or asks about GPU/VRAM/CPU/RAM/unified memory — and proactively
  before any hardware-dependent decision: choosing model size or quantisation, picking CUDA vs
  ROCm vs XPU vs Metal/MPS vs MLX vs CPU, sizing batch/thread counts, or diagnosing why a GPU
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

This SKILL.md governs **how to scan and the create-vs-update mechanics**, and is platform-neutral.
The platform reference chosen in Step 2 governs **which commands to run and which platform gotchas
apply**. The bundled grounding governs **what goes into the document**. Read the grounding before
writing anything.

## Step 0 — Read the standard (required)

Read `references/Workstation-grounding.md` in full before scanning or writing. It defines the
document structure, the frontmatter schema, the evidence-labelling contract, the severity taxonomy,
and the quality gates. It is authoritative on *what goes in*; if this file appears to conflict with
it on content, the grounding wins.

Do **not** copy the grounding or the platform references into the target directory. Only
`Workstation.md` is written out.

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

**If the existing `Workstation.md` was written on a different platform** (its frontmatter
`platform` does not match the machine you are on — e.g. a repo cloned from a Windows workstation to
a Mac), do not "update" it: the two machines are different subjects. Say so, and either write a new
profile with a platform-suffixed name (`Workstation.macos.md`) or, if the user confirms the file
should describe *this* machine, replace it in Create mode while carrying forward any case studies
under a heading that names the original machine.

## Step 2 — Detect the platform, then load its reference

Detect the OS **first**, from the environment, before running any inventory command:

| Evidence | Platform | Load |
|---|---|---|
| `$env:OS` = `Windows_NT`, PowerShell available, `sys.platform == "win32"` | **Windows** | `references/platform-windows.md` |
| `uname -s` = `Darwin`, `sw_vers` exists, `sys.platform == "darwin"` | **macOS** | `references/platform-macos.md` |
| `uname -s` = `Linux`, `/etc/os-release` exists | **Linux** | Fallback table below (no dedicated reference in this version) |

Read the platform reference **in full** before running commands. It carries the inventory commands,
the memory-figure sources, the vendor/runtime tools, the framework probes, the platform's specific
"expensive lie", and the frontmatter values that platform fills in. **Never assume a command exists
— check, and record the failure if it does not.**

Record the detected platform and the reference used in the document's closing line
(*Collected via …*), so a reader knows which procedure produced it.

### Linux fallback commands

Linux has no dedicated reference in this version. Use these, and apply the shared Steps 3–6:

| Target | Linux |
|---|---|
| OS / build | `/etc/os-release`, `uname -a` |
| CPU | `lscpu`, `/proc/cpuinfo` |
| Memory | `/proc/meminfo`, `dmidecode -t memory` |
| GPU | `lspci -nnk \| grep -A3 -i vga`, `glxinfo` |
| Accelerators | `lspci`, `/sys/class/` |
| Storage | `lsblk`, `df -h` |
| Board / BIOS | `dmidecode -t baseboard -t bios` |
| Power | `tlp-stat`, `/sys/class/power_supply/` |

Vendor tools: `nvidia-smi`, `rocm-smi` / `rocminfo`, `sycl-ls`, `vulkaninfo --summary`, `clinfo`.
Framework probes: the torch / onnxruntime snippets in `platform-windows.md` §W5 apply unchanged.

### Do not trust a single source for memory

GPU memory is reported differently by every layer and the numbers genuinely disagree. Record each
distinctly (see grounding §4):

- **OS/driver reported** — an accounting figure, usually the largest and least useful.
- **Runtime reported** (CUDA / Level Zero / Metal / ROCm `total_memory` or working-set budget) —
  the allocator's budget.
- **Actually allocatable** — what a probe successfully allocates before failure.

**Which of the last two governs decisions depends on the memory model**, and the platform reference
says which. On a discrete GPU the measured allocation ceiling is the plan-against number. On a
**unified-memory** machine (Apple Silicon, integrated GPUs sharing host RAM) the allocator may grant
more than the physical pool and the OS silently swaps — there, the runtime budget is the
plan-against number and the OOM ceiling is recorded as evidence, not as a budget. Where the figures
differ, **say so and name which one to plan against.**

## Step 3 — Probe the accelerator reality (the core of this skill)

**This step is what makes the artifact worth having. Never skip it.** Presence of hardware and
presence of a library prove nothing about whether they work together.

Run the vendor / runtime tools and the framework probes listed in the platform reference, and
record exact output. Absence of a tool is itself a finding — write "not found", not nothing.

### The failure this catches

Every platform has a version of the same expensive lie: **everything looks installed and nothing
warns you that the accelerator is not being used.**

- On Windows/Linux it is the **`+cpu` wheel**: a healthy GPU, a current driver, a working vendor
  runtime, `torch` installed — and every workload runs on the CPU because the installed wheel has
  no device kernels. `torch.cuda` / `torch.xpu` exist as an API surface and report **zero devices**.
- On macOS it is the **x86_64 Python under Rosetta**: `torch` imports, `pip` works, and
  `torch.backends.mps.is_available()` is `False` with no explanation; MLX will not install at all.

Nothing errors; throughput is simply 10–40× lower than it should be.

**Therefore:**
- A framework is "working" only if it reports **≥1 device** and completes a real operation.
- Record the **build variant** verbatim: the `+cpu` / `+cu124` / `+xpu` / `+rocm` wheel suffix on
  Windows/Linux, the **interpreter architecture** (`arm64` vs `x86_64`) on macOS. `2.14.0+cpu` and
  `2.14.0+xpu` are not the same fact, and neither are an arm64 and an x86_64 `2.10.0`.
- **Probe the interpreter a task will actually use.** The system `python3` is rarely it. Find
  project venvs and conda environments and probe those; record which one the profile describes.
- If a device is unavailable, diagnose **why** (wrong wheel / wrong arch / missing runtime / driver
  / disabled in firmware) and record the specific remedy.

Also check for a dependency file that would silently undo a working setup — e.g. a
`requirements.txt` pinning `torch>=2.x` with no accelerator index URL, which reinstalls the CPU
wheel on the next clean install. Record it as a constraint.

## Step 4 — Verify capability with real operations

Capability claims must be executed, not inferred. Keep this tier fast (seconds). The platform
reference has the synchronisation idiom and the platform-specific caveats for each item.

1. **Op coverage.** Run the operations real workloads need — for ML: conv2d, layer/group norm,
   scaled-dot-product attention across dtypes, softmax, embedding, autocast + backward, and a
   host↔device round trip. Report an *n/total passed* count and name every failure. A device that
   loads models but lacks a kernel is worse than one that fails loudly.
2. **Dtype ranking.** Time a small matmul in each supported dtype (fp32, bf16, fp16). **Do not
   assume** which is fastest — matrix-engine paths can make one dtype several times faster than
   another, vendor capability flags sometimes contradict measured behaviour, and on GPUs without a
   separate matrix engine the margin may be negligible. Record the measured order **and the
   margin**, and state whether the recommended dtype is chosen for throughput or for footprint.
3. **Allocation ceiling.** Allocate increasing buffers until failure to find the real limit — then
   interpret it per the platform's memory model (Step 2).

### Heavier benchmarking is opt-in

Sustained throughput benchmarks (full model runs, long token-generation tests) are **not** part of
a default scan — they take minutes and generate heat. Run them only when the user asks, or when a
specific decision depends on them. If run, obey the benchmarking rules in grounding §7 — above all:
**baseline and variant must be measured in the same session**, because thermal drift on laptops
(and especially passively-cooled ones) can move prompt-processing figures by 1.5× on identical
inputs.

## Step 5 — Write `Workstation.md`

Compose per the grounding. Non-negotiable requirements:

- **Machine-readable frontmatter first** (grounding §2), so an agent gets the decisive facts
  without parsing prose. Fill the platform-specific keys from the platform reference; set keys that
  do not apply to this platform to `null`, never omit them.
- **Agent Directives near the top** (grounding §3) — the imperative do/don't list. An agent that
  reads only the frontmatter and that section must still make correct choices.
- **Every claim labelled `[measured]` or `[vendor spec]`** (grounding §5). No unlabelled assertions.
- **A constraint table with severities** (grounding §6), blockers first.
- **Strengths stated as well as limits.** A profile listing only problems leads agents to
  underestimate the machine and refuse feasible work.
- **No secrets or identifiers.** Some inventory commands print serial numbers and hardware UUIDs;
  strip them (grounding §1).

## Step 6 — Quality gates

Do not report completion until all pass:

- [ ] Every numeric claim is labelled `[measured]` or `[vendor spec]`.
- [ ] **Nothing is asserted that was not actually checked.** Power source, thermal state, link
      speed and driver dates are all easy to guess wrong — run the command or omit the claim.
      Anything that needed elevated privileges you did not have is recorded as "not run".
- [ ] Accelerator availability was probed by execution, not inferred from installed packages.
- [ ] Framework build variants recorded verbatim (wheel suffix, or interpreter architecture), and
      the probed interpreter's path is named.
- [ ] The three memory figures are distinguished, and the plan-against number is named **and
      justified by the platform's memory model**.
- [ ] Recommended dtype is backed by a measurement, with its margin, not by reputation.
- [ ] Constraints carry severities; blockers state a concrete remedy.
- [ ] Frontmatter agrees with the body (no stale values after an update); platform-inapplicable
      keys are `null`, not missing.
- [ ] In Update mode: changes since the previous profile are stated, and prior case-study /
      session-log sections are preserved.
- [ ] Internal consistency: figures repeated in several sections match, or the discrepancy is
      explained.
- [ ] No serial numbers, UUIDs, credentials or private hostnames/IPs.

## Correcting the record

If a later measurement contradicts something the document already says, **fix the claim and mark
the correction inline** — do not quietly overwrite. The artifact's value rests on being trusted, so
a visible correction is worth more than a silent edit. Mark superseded constants clearly and point
to the section that replaced them.

## Reporting to the user

Summarise in a few lines: the machine, the accelerator and its working status, the single most
consequential finding (usually a blocker or a corrected assumption), and where the file was
written. Lead with anything that changes what they should do next.
