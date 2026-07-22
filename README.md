# mini03 — Stability Triage

**Machine:** `mini03`  ·  **Maintainer:** gboccardo (acting sysadmin)  ·  **Opened:** 2026-07-22
**Status:** 🔴 Root cause not yet confirmed — investigation ongoing, no fixes applied yet.

---

## The problem

`mini03` periodically **hard-freezes**: black screen, no ping, no SSH — but the box is still powered on. The only way back is a **physical power-cycle**, which in practice happens the *next morning* when someone walks over. For a machine used remotely by the whole group, that means hours of downtime and lost jobs each time it happens.

## The situation

- `mini03` is a shared research node used **remotely by the whole group**, in two roles at once:
  - an **interactive / GUI login node** — people SSH in and run OpenFOAM, ML, and mesoscopic work directly in the console;
  - a **Slurm compute node** — batch HPC jobs (mainly CFD).
- It is one of several "mini" nodes (mini01/02/03…); lessons here should carry over to the others.
- The original working theory was **memory exhaustion** (the box "runs out of RAM and locks up"). Investigation on 2026-07-22 did **not** support that — see [CURRENT-STATE.md](CURRENT-STATE.md).

## Why we're doing this

1. **Stop the bleeding:** even before we know the root cause, the machine should *recover itself* from a hang instead of waiting for a human — today it has no automatic recovery path at all.
2. **Find the root cause:** the freezes look like hard lockups with nothing logged; we need to capture the next one properly instead of guessing.
3. **Fix latent risks & learn:** tighten the Slurm/monitoring/config gaps found along the way, and — since gboccardo is learning the sysadmin role — understand *why* each change matters (see [CONCEPTS.md](CONCEPTS.md)).

## The documents

| File | What's in it |
|------|--------------|
| **[README.md](README.md)** | This file — the problem, the situation, why we're doing this. |
| **[CURRENT-STATE.md](CURRENT-STATE.md)** | What we found: the evidence against the memory theory, and every problem identified (Slurm, recovery, monitoring, exposure). |
| **[PROPOSED-FIXES.md](PROPOSED-FIXES.md)** | The prioritized fix plan (P1–P4) and the memory stress-test procedure. Nothing here is applied until it's logged in the decision record. |
| **[DECISION-RECORD.md](DECISION-RECORD.md)** | The running log: what we decided, what we actually changed, and what happened afterward. |
| **[CONCEPTS.md](CONCEPTS.md)** | Plain-language explanations of the sysadmin concepts referenced across these docs. |

> **Workflow:** diagnose in `CURRENT-STATE`, plan in `PROPOSED-FIXES`, and every time we *touch the machine*, append an entry to `DECISION-RECORD` (what/why/result). Keep this README stable; it's the orientation page.
