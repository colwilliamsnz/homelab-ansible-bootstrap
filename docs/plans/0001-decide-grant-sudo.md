# 0001: Decide whether to drop `grant_sudo`

- Status: Open, decision pending
- Blocked by: none

`grant_sudo` controls whether the play writes the sudoers drop-in (README, "Sudo access"). Proposed: drop it. Turning it off cuts off the controller's `become`, and the controller's `base` role has no such switch.

## Done when

- [ ] Decided: drop or keep.
- [ ] Decision recorded in `docs/decisions/`.
- [ ] If dropped: `grant_sudo` is gone from `vars/main.yml`, `local.yml` and the README.
