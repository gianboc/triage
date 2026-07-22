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

## 2026-07-22 — Remote-ops setup: home-side Claude, SSH keys, scoped sudoers
- **Context:** Operations moved to a home-machine session driving `mini03` over SSH (a session on the box can't observe its own reboot). The home machine had no SSH config or keys for the minis — access had always been password-based from Windows.
- **Decision / action:** Created a dedicated ed25519 keypair + `~/.ssh/config` aliases (`mini01/02/03` → `*.polito.it`, user gboccardo) on the home machine. gboccardo installed the public key (`ssh-copy-id`) and a **scoped NOPASSWD sudoers drop-in** on `mini03` — enumerated P1 commands only (modprobe iTCO_wdt, tee into `modules-load.d`/`sysctl.d`/`system.conf.d`, daemon-reexec, sysctl --system, apt-get install, wdctl). Never `NOPASSWD: ALL`.
- **Applied to machine?:** Yes (authorized_keys + sudoers drop-in, installed manually by gboccardo).
- **Result / observed:** Key-based SSH from home works; `sudo -n -l` confirms the enumerated list. Caveat noted for fleet reuse: sudoers `*` wildcards also match spaces, so `tee /etc/sysctl.d/*` is wider than it looks.
- **Follow-up:** Replicate key + sudoers pattern on mini01/02 if this becomes routine.

## 2026-07-22 — P1 applied: panic sysctls are the recovery path; iTCO watchdog is BIOS-locked
- **Context:** Implementing P1 (self-recovery) per [PROPOSED-FIXES.md](PROPOSED-FIXES.md). Queue empty, one interactive user, load ~0.
- **Decision / action:** Applied over SSH via the scoped sudo:
  1. `/etc/modules-load.d/watchdog.conf` (iTCO_wdt) + `modprobe iTCO_wdt` — module loads but reports **`unable to reset NO_REBOOT flag, device disabled by hardware/BIOS`** → **no `/dev/watchdog`**. Board identified as **HP Z8 G4** (board 81C7): the chipset TCO watchdog is locked off in firmware.
  2. `/etc/systemd/system.conf.d/10-watchdog.conf` (`RuntimeWatchdogSec=30`, `RebootWatchdogSec=5min`) + `daemon-reexec` — accepted (`RuntimeWatchdogUSec=30s`), **inert until a watchdog device exists**, arms automatically once one does.
  3. `/etc/sysctl.d/99-hang-recovery.conf`: `kernel.panic=10`, `kernel.panic_on_oops=1`, `kernel.hung_task_panic=1`, **plus `kernel.hardlockup_panic=1`** (addition to the original plan: the NMI hardlockup detector was already running warn-only — it detects exactly the observed failure shape, so it now panics → reboots). Verified live and persistent.
  4. `apt-get install -y rasdaemon` — enabled + active, drivers loaded.
- **Applied to machine?:** Yes (items 1–4).
- **Result / observed:** First-ever automatic recovery path is live: any oops, 2-min hung task, or NMI-detectable hard lockup now reboots the box in ~10 s. **Coverage gap:** a wedge the NMI detector can't see (SMM/firmware/hardware-level) still has no recovery — that needs a real hardware watchdog.
- **Follow-up:** (a) try **`mei_wdt`** — Intel ME watchdog; `/dev/mei0` present and `mei_me` loaded, so the Z8's ME may expose a usable `/dev/watchdog` (needs a sudoers extension or a manual `modprobe mei_wdt`); (b) check BIOS for a TCO/watchdog enable on the next physical visit; (c) kdump (`linux-crashdump` + reboot) still pending — needs a maintenance window.

<!-- Append new entries below this line -->
