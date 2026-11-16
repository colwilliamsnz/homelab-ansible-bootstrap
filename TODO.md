# TODO

Outstanding work for the bootstrap. Background, decisions and conventions are in `CLAUDE.md`; this file only tracks
what's still to do. Tick items off (or delete them) as they're done.

## Now: branch `access-only-bootstrap` (plan step 2)
- [x] Commit and push the branch.
- [x] Test on `lab-cfgm-01.wattle.nz`, at its console:
  - `sudo LC_ALL=C.UTF-8 ansible-pull -U https://github.com/colwilliamsnz/homelab-ansible-bootstrap.git -C access-only-bootstrap -i localhost, --check`, then without `--check`.
  - Timer installed but off: `systemctl status ansible-bootstrap-pull.timer`. SSH rules present: `sudo ufw show added | grep bootstrap-ssh`.
  - Optional timer test: add `-e bootstrap_schedule_enabled=true -e bootstrap_repo_branch=access-only-bootstrap`, then
    `sudo systemctl start ansible-bootstrap-pull.service` and `journalctl -u ansible-bootstrap-pull`.
  - From the controller: `sudo -u svc_ansible sh -c 'ansible lab-cfgm-01.wattle.nz -m ansible.builtin.ping'`.
- [x] Merge into `main` (`--ff-only`), push, delete the branch.
- [x] Controller repo follow-ups: `group_vars/all.yml` header, `CLAUDE.md`, and the `base` role's comments and README
  (branch `base-docs-access-split` in host-inf-cfgm-01).
- [x] **README here is out of date on the schedule:** `main` now has `bootstrap_schedule_enabled: true` and
  `bootstrap_schedule_only_if_changed: false` (re-apply every hour), but the README still describes the timer as
  off by default and runs with `--only-if-changed`. Update the variable table, the "Hourly pull" section, and the
  rollout / kill switch wording.

## Decisions pending
- [ ] **`grant_sudo`:** drop it (proposed: turning it off cuts off the controller's `become`, and `base` has no such
  switch) or keep it.
- [ ] **Signed commits** before the timer goes on everywhere (user investigating). Hourly pull means a bad push
  reaches every host within the hour. Check whether `ansible-pull --verify-commit` accepts SSH signatures or only GPG.
  Interim: branch protection plus restricted push on `main`.
- [ ] **Firewall tech** (user investigating): UFW (fits the current SSH-rule design), nftables (needs a shared
  include file between the two repos), or the Proxmox per-guest firewall (this repo would drop firewall handling entirely).

## Later (plan steps 3–5)
- [ ] **Step 3, roll out the timer:** ~~set `bootstrap_schedule_enabled: true`~~ (done). Run the bootstrap by hand
  once per host: ~~`lab-cfgm-01`~~ (done), then LXCs, VMs, PVE nodes.
- [ ] **Step 4 (controller repo):** `base` drops its access tasks and `base_ansible_*` / `base_sudoers_*` vars.
  - Then drop the shared values table from `CLAUDE.md` here.
  - Point the controller's `roles/base/README.md` "Rotating the controller key" at this README.
- [ ] **Step 5 (controller repo):** firewall role(s), once the firewall tech is chosen.
