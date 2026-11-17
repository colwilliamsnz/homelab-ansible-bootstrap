# 0004: Remove access from the controller's `base` role (plan step 4)

- Status: Not started
- Blocked by: 0003

Until this is done, both repos write `authorized_keys` and sudoers (README, "Values shared with the controller repo"). Afterwards this repo is the only source of controller access. Most of the work is in host-inf-cfgm-01.

## Done when

- [ ] Controller repo: `base` drops its access tasks and the `base_ansible_*` / `base_sudoers_*` vars.
- [ ] Here: the shared values table and the "until plan step 4" notes are removed from `README.md`.
- [ ] Controller repo: `roles/base/README.md` "Rotating the controller key" points at this README.
