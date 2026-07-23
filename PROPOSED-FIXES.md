# mini03 — Proposed fixes

Prioritized plan. **Nothing here is applied until it's logged in [DECISION-RECORD.md](DECISION-RECORD.md)** (what/why/result). Problems referenced (P-A … P-F) are defined in [CURRENT-STATE.md §3](CURRENT-STATE.md). Concepts explained in [CONCEPTS.md](CONCEPTS.md).

---

## P1 — Make hangs self-recover & capture the cause  *(low-risk, do first — addresses P-A, P-E)*

Goal: even before we know the root cause, the box should reboot itself and record *why* next time.

- [x] ~~**Enable the hardware watchdog**~~ **BLOCKED BY BIOS (2026-07-22):** module loads but `iTCO_wdt: unable to reset NO_REBOOT flag, device disabled by hardware/BIOS` → no `/dev/watchdog`. Board is an **HP Z8 G4**. Config left in place (`/etc/modules-load.d/watchdog.conf` + `/etc/systemd/system.conf.d/10-watchdog.conf` with `RuntimeWatchdogSec=30` / `RebootWatchdogSec=5min`) — inert, arms automatically if a device appears. Alternatives, in order: try **`mei_wdt`** (Intel ME watchdog — `/dev/mei0` exists on this box; `sudo modprobe mei_wdt`, then check `wdctl`; if it works, add to modules-load + re-run `systemctl daemon-reexec`); else look for a watchdog/TCO enable in BIOS setup on the next physical visit.
- [x] **Auto-reboot on panic/oops** — ✅ applied 2026-07-22, live + persistent (`/etc/sysctl.d/99-hang-recovery.conf`):
  ```
  kernel.panic = 10
  kernel.panic_on_oops = 1
  kernel.hung_task_panic = 1
  kernel.hardlockup_panic = 1   # addition: NMI hardlockup detector was warn-only; now panics → reboots
  ```
- [x] **Capture the cause of the next freeze** — ✅ **kdump ARMED 2026-07-22 evening** (`linux-crashdump` installed via scoped sudo; gboccardo rebooted; verified `kexec_crash_loaded=1` post-boot; crashkernel reservation 4096M on this 256 GB box; panic sysctls confirmed persistent across the reboot). Total reboot downtime 1 m 47 s. Next freeze that panics will leave a crash dump in `/var/crash`. *(netconsole remains an option if kdump proves insufficient.)*
- [x] **Install rasdaemon** — ✅ applied 2026-07-22: installed, enabled, active (`ras-mc-ctl: drivers are loaded`).

**Risk:** very low. Watchdog + panic-reboot only act when the box is *already* broken. Only caveat: once armed, a future hang will reboot on its own — make sure that's acceptable mid-job (it is; a hung box loses the jobs anyway).

---

## P2 — Investigate the real suspect: hard lockup  *(addresses P-F, P-D, storage clue)*

- [ ] **Test the GA 6.8 kernel** vs current HWE 6.14. Install it, select at the GRUB menu, run for a while. If hangs stop → likely a 6.14 regression or a 6.14↔NVIDIA interaction. Reversible (pick either kernel at boot).
- [ ] **Update the NVIDIA driver** (currently 580.65.06) and check for known lockup fixes.
- [ ] **Drop the desktop on the compute node** — removes the NVIDIA+Xorg lockup vector, frees the GPU for compute:
  ```bash
  sudo systemctl set-default multi-user.target    # reboot to apply; revert with graphical.target
  ```
- [ ] Keep the **NVMe / storage / writeback** path in view (Jul 15 clue) — watch `dmesg` for `nvme` timeouts, check firmware.

---

## P3 — Arm Slurm per-job memory enforcement  *(addresses P-B; do on a maintenance window)*

> ⚠️ **Attempted 2026-07-23 — REVERTED (unsolved).** Plugins loaded and CPU/device confinement worked, but RAM `memory.max` never got set (cgroup v2, Slurm 23.11.4) despite a verified-healthy environment and no slurmd error. Rolled back to keep the box usable for the group. Root cause not isolated. Full detail + leading hypothesis in [DECISION-RECORD.md](DECISION-RECORD.md) (2026-07-23 entry). Don't retry on a shared window.

The `cgroup.conf` is already written; this arms it. **Do this on a drained node and warn the group first** — see behavior-change note below.

1. [ ] Copy the prepared file and add swap constraint + drop the v1 leftover:
   ```bash
   sudo cp /home/gboccardo/git/mulmopro/sysAdmin/1-installation/slurmFiles/cgroup.conf /etc/slurm/cgroup.conf
   # edit /etc/slurm/cgroup.conf:
   #   add:     ConstrainSwapSpace=yes
   #   remove:  CgroupAutoMount=yes      (cgroup-v1 leftover; ignored on this v2 system)
   ```
2. [ ] In `/etc/slurm/slurm.conf`:
   ```
   TaskPlugin=task/affinity,task/cgroup
   ProctrackType=proctrack/cgroup          # <-- not in the original note; reliable cleanup, no orphans
   JobAcctGatherType=jobacct_gather/cgroup
   ```
3. [ ] Set a sane default so no-`--mem` jobs don't grab the whole node, e.g. on the `global` partition:
   ```
   DefMemPerCPU=4096      # tune to typical per-core footprint
   ```
4. [ ] Apply: `sudo scontrol reconfigure` (restart `slurmctld`/`slurmd` if plugins changed).
5. [ ] Verify: submit a job with `--mem=1G` that allocates 2 GB → it should be OOM-killed **inside its own cgroup**, box unaffected; check `sacct` / `dmesg`.

> **Behavior change (why it was deferred):** once `ConstrainRAMSpace=yes` is live, jobs that under-request `--mem` start getting killed. That's the goal, but it will surprise users whose jobs "always worked." Set `DefMemPerCPU`, announce it, and do it in a window. This protects the box from a **runaway Slurm job** — it does **not** stop non-Slurm interactive RAM use, and is **not** the hang fix.

Also, unrelated hardening: [ ] firewall slurm ports **6817/6818** off the public internet (P-C).

---

## P4 — Better forensics & fleet monitoring going forward  *(addresses P-E)*

- [ ] **Tighten local history:** drop `sar` interval to 1 min (`/etc/cron.d/sysstat` or the systemd timer), and install **atop** with 60 s logging — `atop -r <log>` replays which process held memory/CPU/GPU at crash time.
- [ ] **Fleet monitoring (replaces the cron+git usage script):** stand up **node_exporter + Prometheus + Grafana** across mini01/02/03… for central dashboards, retention, and alerts (disk >90 %, node down, sustained high load). Add `prometheus-slurm-exporter` for job/queue metrics. See [CONCEPTS.md §4](CONCEPTS.md).
- [ ] **Version-control `/etc/slurm/`** in git so config changes are tracked and "which file is real" is never ambiguous (P-B provenance).

---

## Appendix — Memory stress test (controlled probe)

⚠️ **Preconditions:** only after **P1 watchdog + `kernel.panic=10`** are in place, and when **no one else is on the machine** — otherwise a test-induced wedge means another physical reset.

Use **`stress-ng`** (not old `stress`). Plain `malloc` won't back pages under overcommit; you must *touch* them, which `--vm` does by writing.

```bash
# 1) Safe probe: fill ~90% and hold, watch pressure
stress-ng --vm 8 --vm-bytes 28G --vm-keep --timeout 600s --metrics &
watch -n2 'cat /proc/pressure/memory; free -g'

# 2) Destructive: force OOM, confirm the killer fires cleanly (expect it in dmesg)
stress-ng --vm 16 --vm-bytes 16G --vm-keep --timeout 300s
```

**Interpretation:** if the box survives OOM cleanly (OOM killer acts, box stays up), that **confirms memory is not the culprit** and steers the investigation to P2. Given the [evidence](CURRENT-STATE.md#2-the-hang-is-not-memory-exhaustion-evidence), this test is expected to *not* reproduce the real hang.
