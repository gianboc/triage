# mini03 — Current state & problems identified

Findings as of **2026-07-22**. This is the diagnosis; proposed changes live in [PROPOSED-FIXES.md](PROPOSED-FIXES.md).

---

## 1. System facts

| Item | Value |
|------|-------|
| Hostname | `mini03` |
| CPU | 2× Intel Xeon Gold 6248R — 48 cores total (2×24, **no** hyperthreading) |
| NUMA | 2 nodes, ~128 GB each |
| RAM | 256 GiB (251 GiB usable), **ECC** |
| Swap | 30 GiB on `/dev/nvme0n1p4` |
| GPU | NVIDIA Quadro RTX 4000 (8 GB), driver 580.65.06 |
| OS | Ubuntu 24.04.1 LTS |
| Kernel | **6.14 HWE** (`linux-image-generic-hwe-24.04`); GA would be 6.8 |
| Disks | `/` 234 GB (21% used), `/home` 1.6 TB (59% used) — not full |
| Desktop | **gdm3 + Xorg + gnome-shell running** (graphical.target) |
| BMC / IPMI | **None detected** (workstation-class board) → no out-of-band reset |
| Scheduler | Slurm 23.11.4 (slurm-wlm) |

---

## 2. The hang is NOT memory exhaustion (evidence)

`sysstat`/`sar` keeps 10-minute history. Replaying the run-up to three separate hangs — the last sample before each freeze:

| Hang (freeze time) | RAM used | Swap used | kbavail | Load (of 48) | Blocked (D-state) |
|--------------------|----------|-----------|---------|--------------|-------------------|
| **Jul 20 23:40** | 10 %  | **0**       | 234 GB | 42 | 0 |
| **Jul 18 15:15** | 10.6 %| ~0          | 231 GB | 42 | 0 |
| **Jul 15 04:52** | 3 % (≈223 GB was reclaimable page cache) | ~0 | 250 GB | 36 | 0 |

In every case: free memory abundant, swap empty, nothing stuck on I/O. **No memory pressure at the moment of failure** — which is exactly why there's no OOM signature in the logs: the OOM killer had no reason to fire. (`journalctl -k | grep -i oom` → empty.)

**Failure shape = hard lockup.** On Jul 20 the journal's last line is a routine `sysstat-collect` at 23:40:24, then it **stops mid-stream** — no shutdown sequence — until a physical power-cycle at 08:17 next morning. sar's next sample was never written. That's a CPU wedged with interrupts disabled, not a gradual/OOM death.

**Clean on the usual hardware suspects:**
- **ECC errors = 0** on all 4 memory controllers (EDAC).
- **No MCE / machine-check** events.
- **No NVIDIA Xid** errors.
- **No thermal throttling** ever (consistent with the 1-hour `stress` test that hit ~90 °C without crashing).

> One clue: the Jul 15 freeze was preceded by heavy disk **writeback** (multi-GB dirty pages, ~223 GB page cache), so the **NVMe / storage path** stays on the suspect list — but `kbavail` was still ~250 GB, so it was not a memory shortage.

---

## 3. Problems identified

### P-A — No recovery path from a hang  *(biggest operational pain)*
`kernel.panic=0`, `kernel.panic_on_oops=0`, **no `/dev/watchdog`** enabled (the chipset's iTCO watchdog module isn't loaded), and **no BMC**. So a hard lockup has *no automatic recovery* — it stays dead until a human physically resets it. See [CONCEPTS.md §1](CONCEPTS.md).

### P-B — Slurm does not enforce per-job memory  *(latent risk; deferred on purpose)*
The scheduler *accounts* memory (`SelectTypeParameters=CR_CORE_MEMORY`, so it won't over-book a node) but nothing *confines* a job at runtime. Current relevant settings:

| Directive (live) | Effect |
|---|---|
| `TaskPlugin=task/affinity` | CPU pinning only — no memory/device containment. |
| `ProctrackType=proctrack/linuxproc` | Tracks job processes by PID tree — fragile; daemonized/double-forked procs escape → orphans survive job end. |
| `JobAcctGatherType=jobacct_gather/linux`, `JobAcctGatherParams=(null)` | `/proc` sampling for stats; no `OverMemoryKill`, so no polling-based kill. |
| *(no `cgroup.conf`, no `task/cgroup`)* | **No runtime confinement** — a job requesting `--mem=4G` can grow to 250 GB and nothing kills it. |
| `global` partition has no `DefMemPerCPU` | A job submitted without `--mem` reserves the **entire node's** memory. |

**This was a deliberate, documented decision — not a mistake.** The live `/etc/slurm/slurm.conf` header says a correct `cgroup.conf` was already prepared and left the 3-step enable plan, choosing *"for now, use the linux accounting, which we know it works."* The prepared file is at `/home/gboccardo/git/mulmopro/sysAdmin/1-installation/slurmFiles/cgroup.conf` (has `ConstrainCores/RAMSpace/Devices=yes`; missing `ConstrainSwapSpace`; has a v1-era `CgroupAutoMount` line). Arming steps are in [PROPOSED-FIXES.md §P3](PROPOSED-FIXES.md). Reminder: enforcement protects the box from a **runaway Slurm job**, but does **not** stop non-Slurm interactive RAM use, and is **not** the hang fix.

### P-C — slurmctld is internet-exposed
Right before the Jul 20 hang, slurmctld logged `Insane message length` / `Zero Bytes` probes from `204.62.215.1`. Almost certainly coincidental to the hang, but ports **6817/6818 should not face the public internet**.

### P-D — GUI desktop on a compute node
`gdm3 + Xorg + gnome-shell` run on the same GPU used for compute. Adds a real hard-lockup vector (NVIDIA + Xorg) for no benefit on a headless-usable node.

### P-E — Coarse forensics
`sar` samples every 10 min (too coarse to catch a fast event's peak) and there's no per-process history recorder (no `atop`). Enough to establish trends and rule memory out, but not to pinpoint a culprit process. See [CONCEPTS.md §3](CONCEPTS.md).

### P-F — Kernel/driver as prime suspect (unconfirmed)
Running **HWE 6.14** (newer backported kernel) rather than GA 6.8. A silent hard lockup can be a kernel regression or a kernel↔NVIDIA-driver interaction. Untested so far.

---

## 4. Config provenance (so "which file is real" is never a question)

- **Authoritative, live config:** `/etc/slurm/slurm.conf` (owned by root, 2025-09-25). Confirmed via `scontrol show config` → `SLURM_CONF=/etc/slurm/slurm.conf`.
- **`/home/gboccardo/slurm.conf`** is a **stale personal draft** (2025-08-26). Identical to the live file **except one line**: it has `RealMemory=257626`; live has `RealMemory=256000`. Slurm ignores it. (Recommendation: put `/etc/slurm/` under git so this is unambiguous — see [PROPOSED-FIXES.md §P4](PROPOSED-FIXES.md).)

---

## 5. Quick-reference diagnostics used

```bash
# Replay memory/load before a hang (sar files: /var/log/sysstat/saDD, one per day-of-month)
LANG=C sar -r -f /var/log/sysstat/sa20 -s 22:00:00      # memory
LANG=C sar -S -f /var/log/sysstat/sa20 -s 22:00:00      # swap
LANG=C sar -q -f /var/log/sysstat/sa20 -s 22:00:00      # load / run-queue / blocked

journalctl --list-boots            # boot history — abrupt ends + morning restarts = hangs
journalctl -b -1 | tail            # last lines before previous reboot (hang = stops mid-stream)
journalctl -k | egrep -i 'oom|lockup|panic|mce|Xid|EDAC'
for m in /sys/devices/system/edac/mc/mc*; do echo "$m CE=$(cat $m/ce_count) UE=$(cat $m/ue_count)"; done
scontrol show config | egrep -i 'TaskPlugin|ProctrackType|JobAcctGather|SelectTypeParam'  # enforcement status
```
