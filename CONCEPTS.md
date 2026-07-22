# Sysadmin concepts — rundown for mini03 triage

Plain-language explanations of the terms used in [README.md](README.md). Aimed at someone new to the sysadmin side.

---

## 1) Hang recovery: `kernel.panic`, watchdogs, and BMC

**What a "panic" is.** When the Linux kernel hits a fatal internal error it can't safely continue from, it *panics* — the OS equivalent of a hard stop. `kernel.panic` is a tunable (a "sysctl") that says **what to do after a panic**, measured in seconds:

- `kernel.panic = 0` → **wait forever.** The machine sits frozen until a human power-cycles it. This is the default, and it's why your box stays dead until someone walks over in the morning.
- `kernel.panic = 10` → **reboot automatically after 10 seconds.** The machine recovers itself and comes back online for remote users. For an unattended, remotely-used node this is almost always what you want.

`kernel.panic_on_oops = 1` is a companion: an "oops" is a *non-fatal* kernel error that often leaves the system half-broken. Turning this on promotes an oops into a full panic, so (combined with `panic=10`) it also triggers the auto-reboot instead of limping along in a corrupted state.

**Why that isn't enough by itself.** A panic is the kernel *noticing* it's broken. But the worst hangs are when the kernel **can't notice** — a *hard lockup*, where a CPU is wedged with interrupts disabled. No panic runs, nothing gets logged (this matches mini03: logs just stop mid-sentence). For that you need something *outside* the kernel to catch it.

**Watchdog.** A watchdog is a **dead-man's switch**. There's a hardware timer (on Intel server chipsets it's called the *iTCO watchdog*) that counts down, e.g. from 30 s. A healthy system "pets" (resets) the timer every few seconds via a small daemon. If the system hard-locks, nobody pets it, the timer hits zero, and the **hardware forces a reboot** — no working kernel required. `systemd` has this built in: set `RuntimeWatchdogSec=30` and systemd both arms the hardware timer and does the petting. This is the one mechanism that recovers a *true* hard lockup. On mini03 the iTCO watchdog exists but the module isn't loaded, so it's doing nothing.

**BMC (Baseboard Management Controller).** A separate tiny computer on server-grade motherboards (Dell iDRAC, HPE iLO, Supermicro IPMI). It runs even when the main OS is dead and lets you **remotely power-cycle, see the console, read sensors** over the network. mini03 appears to be a *workstation*-class board (it has onboard audio and a DVD drive) with **no BMC** — so you have no out-of-band reset. That's exactly why the watchdog matters so much here: it's your only automatic remote-recovery path.

> **Takeaway:** `panic=10` recovers *noticed* crashes; the **watchdog** recovers *unnoticed* hard locks; a BMC would let *you* recover manually from anywhere — but you don't have one.

---

## 2) Slurm memory: accounting vs. enforcement

Two separate things, easily confused (the online Slurm configurator sets up the first, not the second):

**Accounting (scheduling).** `SelectTypeParameters=CR_CORE_MEMORY` makes the scheduler treat memory as a bookable resource, like cores. It won't hand out 300 GB of jobs on a 256 GB node. **This is configured and working.** ✅

**Enforcement (confinement).** Actually *stopping* a single job from using more RAM than it reserved. A job can ask for 4 GB and then leak to 250 GB; accounting doesn't stop that — only enforcement does. Enforcement needs **Linux cgroups**:

- `TaskPlugin=...,task/cgroup` — put each job in a control group.
- a `cgroup.conf` with `ConstrainRAMSpace=yes` — set that group's `memory.max` so the kernel kills the job if it exceeds its reservation.

You have `task/affinity` only, no `cgroup.conf` → **enforcement is off.** ❌ (A legacy alternative, `JobAcctGatherParams=OverMemoryKill`, also off — yours reads `(null)`.)

**cgroups** ("control groups") are a Linux kernel feature to cap and isolate a group of processes' CPU, memory, and I/O. They're the same technology containers/Docker use. Slurm leans on them to box each job in.

**Two more knobs from the README:**
- `RealMemory=256000` tells Slurm the node has ~250 GiB *to give away*. But this box is *also* an interactive + GUI login node — people SSH in and run OpenFOAM directly, which Slurm knows nothing about. So Slurm may book the whole box while interactive users are also using it. Leaving headroom (e.g. `240000`) is safer.
- `DefMemPerCPU` = the **default** memory a job gets if the user doesn't pass `--mem`. Your `global` partition has none set, which under `CR_*_MEMORY` means a job with no `--mem` **reserves the entire node's memory** — blocking everyone else. Always set a sensible default.

> **Takeaway:** you have the scheduler *booking* memory correctly, but nothing *confining* a runaway job. It's a latent risk worth fixing, but it is **not** what caused the three hangs (they had free memory).

---

## 3) `sar` / sysstat, and "won't a fast OOM slip between samples?"

**`sar`** (System Activity Reporter, part of the `sysstat` package) is a built-in tool that **records system metrics to disk on a timer** — CPU, memory, swap, load, disk, network — so you can look *back in time*. On Ubuntu a systemd timer runs a collector every 10 minutes into `/var/log/sysstat/saDD` (one file per day of month). That history is how we replayed the state before each hang and ruled out memory. `df` / `free` / `top` show you *now*; `sar` shows you *then*.

**Your question — "an OOM happens in seconds, so isn't 10-minute (or even 1-second) sar useless for it?"** Great instinct, and mostly correct:

- For a **sudden spike**, yes — sampling every N seconds can miss the peak that happened between two samples. Cranking sar to 1–2 s just to catch spikes is the wrong tool (it also bloats logs and adds overhead).
- But you **don't need sar to diagnose an OOM at all.** The kernel's **OOM-killer logs the event itself, at the instant it happens**, to `dmesg`/journal — with the exact culprit process, its memory use, and a full memory breakdown. That's the authoritative, event-driven record. Check it with:
  ```bash
  journalctl -k | grep -i 'killed process\|out of memory\|oom'
  ```
  On mini03 that came back **empty** → no OOM ever fired → memory wasn't the trigger. (An event that logs itself beats any polling interval.)
- So what *is* `sar` good for? **Trends and "was the system under pressure," not instantaneous spikes.** Capacity planning, "were we swapping / thrashing / CPU-saturated in the hour before it got slow," and confirming/excluding whole categories of cause — which is exactly how it earned its keep here.
- **`atop`** is the middle ground worth adding: it snapshots **per-process** CPU/mem/GPU/IO on an interval and — crucially — also captures **short-lived processes that died between samples** (via kernel process accounting). After a reboot, `atop -r <logfile>` replays the machine like a video. Still interval-limited for the exact peak, but far better than sar for "*which process* was growing."

> **Rule of thumb:** event-driven problems (OOM, segfault, oops) → the kernel log records them itself. Gradual/aggregate problems (creeping usage, saturation, thrash) → `sar`/`atop` history. Use the right one; don't try to make sar sample fast enough to catch instantaneous events.

---

## 4) Your cron + git usage-logging script — is there a "real" tool?

What you built (cron every 5 min → write cores/disk stats → `git push` to GitHub) is genuinely reinventing **monitoring**. It works, but there are standard tools:

**For a single machine's history:** `sysstat`/`sar` (already installed and running) already records CPU, load, memory, swap, disk I/O, and network every 10 min. You can:
- change the interval (edit the systemd timer / `/etc/cron.d/sysstat`),
- add filesystem-usage logging (`sar -F` if enabled), and
- query any past day: `sar -u` (CPU), `sar -r` (mem), `sar -q` (load), `sadf` to export CSV/JSON for plotting.

That replaces the *data-collection* half of your script for one host, with no git repo needed.

**For a fleet of mini-clusters with central dashboards** (this is what you actually want across mini01/02/03…), the industry-standard, free, self-hosted stack is:
- **`node_exporter`** — a tiny agent on each node that exposes cores, memory, disk, load, temps as metrics.
- **Prometheus** — a central server that *scrapes* every node every ~15 s and stores the time series.
- **Grafana** — dashboards and graphs over that data; per-node and fleet-wide, with alerting ("disk >90%", "node down", "load >48 for 10 min").

For a research group with a handful of nodes this is very reasonable to run (all on one small VM/host) and gives you exactly the "usage statistics across all mini clusters" you were building by hand — with real graphs, retention, and alerts, and no GitHub-push hack. **Slurm-specific:** `prometheus-slurm-exporter` adds job/queue/utilization metrics on top.

> **Takeaway:** keep your script if you like it, but for one host `sar` already has the data; for the fleet, `node_exporter + Prometheus + Grafana` is the standard, low-effort answer.

---

## 5) Ubuntu kernels: GA vs. HWE

Ubuntu LTS (24.04 here) ships **two kernel tracks**, and mini03 is on the newer one:

- **GA — "General Availability"** kernel. The kernel the LTS *launched* with; for 24.04 that's **6.8**. It's frozen at that version and gets security/bug fixes for the LTS's whole life. **Conservative and stable** — fewer surprises.
- **HWE — "Hardware Enablement"** kernel. A *newer* kernel backported from later Ubuntu releases onto the same LTS, so recent hardware (new CPUs/GPUs/NICs) works. 24.04's HWE is currently **6.14** — which is what mini03 runs (`linux-image-generic-hwe-24.04`). Desktop installs of 24.04.1+ get HWE by default. **Newer hardware support, but newer code = more chance of a regression/bug.**

**Why it's in the triage:** a *hard lockup with nothing logged* can be a **kernel regression**. Newer ≠ more stable. Since 6.14 is much newer code than 6.8, a clean experiment is to **boot the GA 6.8 kernel and see if the hangs stop.** If they do, you've likely found a 6.14 regression (or a 6.14 + NVIDIA-driver interaction). You can install both and pick at the GRUB boot menu — switching kernels is low-risk and reversible.

> **Takeaway:** GA = original/stable, HWE = newer/for-new-hardware. You're on HWE 6.14; testing GA 6.8 is a cheap way to rule a kernel regression in or out.

---

*See [README.md](README.md) for the triage findings and the prioritized action plan.*
