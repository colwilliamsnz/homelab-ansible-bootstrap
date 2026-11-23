# 003: Roll out the hourly timer (plan step 3)

- Status: Done
- Blocked by: 002 before the timer goes on everywhere

A host only gets the timer when the bootstrap is run on it by hand once (README, "Hourly pull", Rollout). The full plan is in host-inf-cfgm-01's `CLAUDE.md`.

## Done when

- [x] `bootstrap_schedule_enabled: true` on `main`.
- [x] Run by hand on `lab-cfgm-01`.
- [x] Run by hand on the LXCs.
- [x] Run by hand on the VMs.
- [x] Run by hand on the PVE nodes.
