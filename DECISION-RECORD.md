# mini03 — Decision & action record

Running log of **decisions made** and **changes actually applied to the machine**, with results. Append a new entry every time we touch the box or change direction. Newest at the bottom.

**How to use this file**
- One entry per decision or action. Don't rewrite history — if something is reversed, add a *new* entry that says so.
- Keep proposals in [PROPOSED-FIXES.md](PROPOSED-FIXES.md); this file records what we *committed to* and what *happened*.

**Entry template**
```
## YYYY-MM-DD — <short title>
- **Context:** why this came up
- **Decision / action:** what we decided or did (exact commands/files if applied)
- **Applied to machine?:** yes / no (investigation only)
- **Result / observed:** what happened after (fill in later if pending)
- **Follow-up:** next step, if any
```

---

## 2026-07-22 — Initial triage; reframed root-cause hypothesis
- **Context:** `mini03` periodically hard-freezes (black screen, no ping/SSH, box on); recovered only by physical power-cycle. Working theory was memory exhaustion.
- **Decision / action:** Reviewed hardware, Slurm config, journald/kernel logs, EDAC/MCE, and replayed `sar` history for three hangs (Jul 15 / 18 / 20). Concluded the hangs show **no memory pressure** (free RAM, empty swap, no OOM logged) and match a **hard lockup**. Reprioritized: root cause likely kernel/driver/hardware, not memory. Full evidence in [CURRENT-STATE.md](CURRENT-STATE.md).
- **Applied to machine?:** **No — investigation only.** No configuration changed.
- **Result / observed:** Memory theory not supported by data. Prime suspects now: HWE 6.14 kernel / NVIDIA driver / storage path. Also identified latent issues: no hang-recovery path (P-A), Slurm memory not enforced (P-B), slurmctld internet-exposed (P-C), GUI on compute node (P-D), coarse forensics (P-E).
- **Follow-up:** Apply P1 safety-net (watchdog + panic sysctls + rasdaemon) — see [PROPOSED-FIXES.md](PROPOSED-FIXES.md).

## 2026-07-22 — Clarified the Slurm memory-enforcement situation
- **Context:** Question of whether Slurm memory limits were misconfigured; maintainer recalled setting up cgroups.
- **Decision / action:** Verified live config (`scontrol show config`, checked for `cgroup.conf`, inspected `/sys/fs/cgroup`). Found enforcement is **off by deliberate, documented choice** — the live `slurm.conf` header records that a correct `cgroup.conf` was prepared and enforcement was *postponed* in favor of "linux accounting, which we know works." Not a misconfiguration.
- **Applied to machine?:** **No — investigation only.**
- **Result / observed:** Confirmed: scheduler accounts memory (`CR_CORE_MEMORY`) but no runtime confinement. Prepared file lives at `/home/gboccardo/git/mulmopro/sysAdmin/1-installation/slurmFiles/cgroup.conf`. Arming steps captured in [PROPOSED-FIXES.md §P3](PROPOSED-FIXES.md).
- **Follow-up:** Arm cgroup enforcement on a maintenance window, after announcing the behavior change to the group.

<!-- Append new entries below this line -->
