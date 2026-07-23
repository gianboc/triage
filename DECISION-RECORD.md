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

## 2026-07-22 — ministatus v0 deployed: fleet status page + reboot visibility
- **Context:** P1 makes the box self-reboot — which means nobody would *know* a reboot happened (previously the human doing the power-cycle told the group). Also a long-standing goal: a simple machines-status website. Requirement: deliberately simple — no Grafana/Prometheus, no servers.
- **Decision / action:** Built and deployed **`github.com/gianboc/ministatus`**: each node pushes a JSON heartbeat every 5 min via cron to the repo's `data` branch (scoped write deploy key); an `@reboot` cron line pushes a `REBOOT <host> <time>` event — the git log doubles as the reboot log. GitHub Pages serves a static page (https://gianboc.github.io/ministatus/) that renders per-node tiles and infers **"likely down" from heartbeat staleness** (>20 min silence) in the viewer's browser. On mini03: generated dedicated deploy key, cloned `data` branch to `~/ministatus`, installed 2 crontab lines (heartbeat + @reboot). All unprivileged — no root involved.
- **Applied to machine?:** Yes (crontab for gboccardo, `~/ministatus` clone, `~/.ssh` deploy key + alias). No system config touched.
- **Result / observed:** First report pushed and verified end-to-end (raw JSON fresh, event logged, Pages 200). Privacy filter: machine vitals only — no usernames, no job names.
- **Follow-up:** (a) tomorrow's kdump reboot is the free end-to-end test — a `REBOOT mini03` event should appear on the page within ~6 min of boot; (b) enroll mini01/02 once SSH keys exist for them; (c) active alerts (mail/Telegram) deliberately out of scope for v0; (d) Slurm's default job **requeue-after-reboot** behavior needs a policy decision (parked).

## 2026-07-22 — mini01 + mini02 enrolled in ministatus (reporting only)
- **Context:** Extend fleet visibility beyond mini03. Explicitly reporting-only — none of the P1 stability changes were applied to mini01/02.
- **Decision / action:** Same unprivileged recipe as mini03 on each box: dedicated deploy key (write, repo-scoped, registered on `gianboc/ministatus`), `data`-branch clone at `~/ministatus`, heartbeat + `@reboot` crontab lines. Preflight confirmed git 2.43 / active cron / empty crontab / GitHub egress on both. One bug caught during rollout: the crontab-append one-liner silently installed an *empty* crontab (`grep -v` exits 1 on empty input under `set -e`); fixed with a `|| true` guard and re-verified — both crontabs now show the two lines.
- **Applied to machine?:** Yes (mini01, mini02: crontab, `~/ministatus`, `~/.ssh` deploy key + alias). No system config touched, no sudo used.
- **Result / observed:** All three minis reporting. Fleet is homogeneous: 48 cores / 252 GB each. mini01 under real load (~11.7) at enrollment time; mini02 idle.
- **Follow-up:** None — reboot events will accumulate per node from their next boots.

## 2026-07-22 — ministatus v0.2: peer-confirmed down detection + load-history charts
- **Context:** Two upgrades requested the same evening: (1) active down-detection instead of staleness-only inference; (2) per-node load charts (24 h / 7 d / 30 d).
- **Decision / action:**
  - **Peer checks:** each heartbeat TCP-probes the other minis' SSH port (full mesh verified beforehand) and records `"peers"` in its JSON. Page logic: own heartbeat fresh → UP; stale but fresh peers reach it → **NOT REPORTING** (amber — reporting pipeline broke); fresh peers can't reach it → **UNREACHABLE** (red, peer-confirmed, ~5–10 min detection); no peer data → old staleness rules. Heartbeat stays at 5 min (raw CDN caches ~5 min, so a faster cron buys nothing).
  - **History:** heartbeats append `t,load1` to `history/<host>.csv` (pruned to 35 days); page renders three SVG charts per node (24 h raw, 7 d 2-h means, 30 d 8-h means) of load as % of cores, with hover readout. `node/backfill_sar.sh` seeded ~8 days of sar history per node (mini01 1190, mini02 1179, mini03 846 samples).
  - **Bugs found & fixed during rollout:** (a) `git add -A` in report.sh was committing the script copies into the data branch, causing cross-node collisions — fixed with data-branch `.gitignore` + untracking; (b) report.sh pulled *after* dirtying the tree — moved the pull to the top; (c) one corrupt sar archive on mini03 (crash-day file, likely sa21) silently killed the whole backfill under `pipefail` — per-file `|| true` guard added.
- **Applied to machine?:** Yes (all three minis: updated `~/ministatus/report.sh`, ran backfill). Unprivileged throughout.
- **Result / observed:** Full peer mesh green in live JSONs; all node clones clean and synced; history on the remote. Operational note: raw.githubusercontent **ignores query-string cache-busters** — content updates surface within ~5 min regardless; page thresholds already account for it.
- **Follow-up:** none pending; tomorrow's kdump reboot doubles as the live test of REBOOT events + peer-detection.

## 2026-07-22 — kdump armed + controlled reboot test: P1 COMPLETE
- **Context:** Queue empty, no SSH users (only a day-old idle console session), gboccardo available to pull the trigger — so the kdump install originally planned for the next office visit was done remotely instead, and the reboot doubled as the end-to-end test of the ministatus reboot notification.
- **Decision / action:** `apt-get install -y linux-crashdump` via scoped sudo (crashkernel=…,128G-:4096M staged in grub by the package; kdump-tools enabled). gboccardo ran `sudo reboot` from his own session at 22:33.
- **Applied to machine?:** Yes (package install + reboot).
- **Result / observed:** Down 22:33:53 → SSH back 22:35:40 (**1 m 47 s downtime**, including the new 4 GB crashkernel reservation). Post-boot verified: `kexec_crash_loaded=1` (**kdump armed**), panic sysctls persisted (`panic=10`, `hardlockup_panic=1`), iTCO module config loaded (still BIOS-blocked, as expected). ministatus chain worked unattended: `@reboot` cron fired, `BOOT mini03` event pushed 1 m 41 s after boot, page shows the reboot chip + event row + first entry of the last-5-reboots list. **P1 is now fully in place**: self-recovery via panic sysctls, crash capture via kdump, hardware-error logging via rasdaemon, reboot visibility via ministatus.
- **Follow-up:** Watchdog device still missing (BIOS locks iTCO): try `sudo modprobe mei_wdt` (2-min test, any time), else check BIOS at the next physical visit. Root-cause hunt now waits for the next freeze — which will either self-recover + leave a crash dump, or be peer-flagged on the status page within minutes.

<!-- Append new entries below this line -->

## 2026-07-23 — Memory-overcommit stress test reproduced a no-logs, power-cycle-only livelock
- **Context:** With mini03 reverted (RealMemory=240000, **no cgroup enforcement**) and idle before handback, gboccardo wanted to *see* what memory exhaustion does. Ran the [Appendix](PROPOSED-FIXES.md) memory probe as Slurm jobs: 20 jobs each **reserving 12 GB but actually using 18 GB** (`~/oom-test/`, now removed). Because there's no enforcement, Slurm packed all 20 (240 GB reserved, fits) while real demand (~360 GB) far exceeded the 253 GB of RAM.
- **Decision / action:** Deliberate destructive test on a drained node (preconditions met: P1 panic sysctls live, no other users). Test scripts staged, fired, then the box wedged.
- **Applied to machine?:** No config change. Destructive test only; recovered by **physical power-cycle** (~09:23).
- **Result / observed:**
  - Box went from healthy to **wedged in ~1 minute** of the jobs starting.
  - **Symptom: pingable but not sshable.** ICMP answered the whole time; TCP `:22` was *accepted* but `sshd` was too memory-starved to complete the banner exchange. A **swap-thrash livelock** — kernel alive, userspace starved.
  - **The OOM killer never rescued it.** 30 GB of swap let the kernel thrash instead of killing a victim; the box never self-recovered across ~25 min (jobs' own 5-min self-exit couldn't make progress either).
  - **Nothing was logged.** Persistent journald's last write on that boot was **08:58:48** — it froze with everything else. `sacct` shows the jobs as `TIMEOUT` (reconciled post-reboot) with **empty `MaxRSS`** (accounting never sampled). So this failure leaves **the same "nothing in the logs" signature as the original mystery freezes.**
  - **P1 did NOT catch it** — correctly, by its design: `hardlockup_panic`/`hung_task_panic` fire on dead CPUs or cleanly D-blocked tasks; a livelock has busy CPUs (reclaim) and slow-but-nonzero progress, so no panic → no auto-reboot → **no kdump capture**. This is the P1 coverage gap, now demonstrated concretely.
  - **Monitoring blind spot found:** peers' ministatus checks kept reporting `"mini03": true` because a bare TCP connect to `:22` succeeds even while sshd can't handshake — the page would have falsely shown mini03 *up*. Peer probe should do a banner/handshake check, not just connect.
  - Reboot itself was clean: node came back `IDLE`, RealMemory=240000, P1 intact (`kexec_crash_loaded=1`, panic sysctls persistent), `@reboot`→`BOOT mini03` event logged on ministatus.
- **Bearing on the root-cause hypothesis (honest update):** This does **not** prove the original freezes are memory — the originals were **pingless** (kernel-dead), today's **pinged** (kernel-alive), so they're likely distinct at the kernel level. BUT two earlier certainties are now weaker: (1) memory pressure *can* produce a **no-logs, power-cycle-only** freeze on this box, matching the original signature on those axes; (2) the "no memory pressure before the hangs" finding rests on `sar`'s **10-min** sampling — a ~1-min fill-to-freeze spike falls *between* samples, so sar can't exclude a fast memory spike. Keep P2 (kernel 6.14 / NVIDIA / NVMe) as prime, but memory is no longer cleanly ruled out.
- **Follow-up (mitigations for the swap-death class, none applied):** (a) userspace OOM daemon — `systemd-oomd` or `earlyoom` — to kill *before* livelock; (b) **shrink swap and/or lower `vm.swappiness`** so the OOM killer fires cleanly instead of thrashing 30 GB; (c) the real hardware watchdog (still BIOS-locked) would auto-recover this class; (d) **cgroup enforcement + `DefMemPerCPU` (P3)** would have prevented the overcommit at the source — the test is itself an argument for finishing P3 on a proper window; (e) fix the ministatus peer probe to require an SSH banner, not just a TCP connect.

## 2026-07-23 — mini03 unschedulable since the kdump reboot (INVAL); fix RealMemory + arm cgroups (P3)
- **Context:** `sinfo` showed mini03 `inval` in both partitions (`global*`, `cuda`). Root cause found: P1's kdump step (2026-07-22 eve) reserved a **4352 MB crashkernel**, dropping `MemTotal` to **253274 MB**, but `slurm.conf` still declared `RealMemory=256000`. slurmd's default tolerance is 100%, so at the post-kdump slurmd start (`22:35:29`) it flagged the node `INVALID_REG` — `Reason=Low RealMemory (reported:253274 < 100.00% of configured:256000)` — and drained it. **mini03 has been unschedulable ever since.** This was missed in the "P1 COMPLETE" verification (kdump/sysctls/ministatus were checked; `sinfo`/node state was not). Separately, gboccardo wants to arm the long-prepared cgroup enforcement (P3) before returning mini03 to the group, and is wary of breaking Slurm.
- **Decision / action:** Two-phase, on the drained/empty node, backups + staged files in `~/slurm-triage-P3-2026-07-23/` (`slurm.conf.orig` restore point; `slurm.conf.mem` = Phase 1; `slurm.conf.new` + `cgroup.conf.new` = Phase 2). Verified cgroup-v2 readiness first: `slurmd Delegate=yes` (cpu cpuset io memory pids), all `cgroup/v2` + `task/proctrack/jobacct` cgroup plugins present, stock Ubuntu Slurm 23.11.4.
  - **Phase 1 (INVAL fix):** `NodeName` `RealMemory 256000 → 240000` — below actual 253274, and implements the interactive-headroom recommendation in [CONCEPTS.md](CONCEPTS.md) for this GUI/login+compute box. Restart `slurmctld`+`slurmd`; `scontrol update nodename=mini03 state=resume`.
  - **Phase 2 (P3 cgroups):** new `/etc/slurm/cgroup.conf` (`ConstrainCores/RAMSpace/SwapSpace/Devices=yes`; `CgroupAutoMount` omitted — v1 leftover, box is cgroup v2 unified). `slurm.conf`: `ProcTrackType→proctrack/cgroup`, `TaskPlugin→task/affinity,task/cgroup`, `JobAcctGatherType→jobacct_gather/cgroup`, `global` partition `DefMemPerCPU=4096`.
- **Applied to machine?:** **Partial.** Phase 1 (RealMemory fix) applied and kept. Phase 2 (cgroup enforcement) applied, failed to enforce, and was **reverted**.
- **Result / observed:**
  - **Phase 1 — SUCCESS (kept):** `RealMemory 256000 → 240000` cleared `INVALID_REG`; after `scontrol update state=resume` (needs *root* — `gboccardo` isn't a Slurm admin, so an unprivileged `scontrol update` returned "Invalid user id") the node returned to `IDLE`. mini03 schedulable again. **This is the state we're leaving it in.**
  - **Phase 2 — FAILED, reverted:** cgroup plugins loaded correctly (`task/cgroup`, `proctrack/cgroup`, `jobacct_gather/cgroup`), and **core/device confinement worked** (`cpuset.cpus` correctly set per job). But **RAM enforcement never took effect**: `memory.max` stayed `max` at *every* level of the job cgroup hierarchy (`job_*/step_*/user/task_*`), so a `--mem=1G` job successfully allocated 2 GB. Environment was verified healthy first — `slurmd Delegate=yes` (memory delegated), `memory` in `cgroup.subtree_control` at all levels, job got `mem=1G` TRES, cgroup v2 unified, Slurm 23.11.4 — and slurmd logged **no error** at `info`. Root cause not isolated within the time box (leading hypothesis: `ConstrainSwapSpace=yes` + `AllowedSwapSpace=0%` swap-write aborting the memory setup, unconfirmed). Given a 20-person group waiting on the node and the risk of enforcement surprising users mid-job, gboccardo called it: **reverted to `slurm.conf.mem`** (original config + the RealMemory fix only) and removed `/etc/slurm/cgroup.conf`.
- **Follow-up:** P3 (memory enforcement) **remains open/unsolved** — do NOT retry on a shared window; needs a real maintenance slot with the group warned, and probably a minimal repro (bisect `ConstrainSwapSpace`, then `DebugFlags=Cgroup`+`SlurmdDebug=debug2` on a *drained* node) or a Slurm bug check for 23.11.4 cgroup/v2 memory. Backups + all staged variants (`slurm.conf.orig/.mem/.new/.debug`, `cgroup.conf.new`) preserved in `~/slurm-triage-P3-2026-07-23/` on mini03 for a future attempt. Separately: the P1-COMPLETE entry should be read alongside this one — kdump's crashkernel reservation is what silently drained the node; **verify `sinfo` after any future reboot that changes memory.**
