# 001: Keep `grant_sudo`

- Status: Accepted
- Date: 2026-11-17

## Context

`grant_sudo` controls whether the play writes the sudoers drop-in (README, "Sudo access"). Dropping the variable was proposed: setting it to `false` cuts off the controller's `become`, and the controller's `base` role has no such switch.

## Decision

Keep `grant_sudo` as it is. No change to `vars/main.yml`, `local.yml` or the README.

## Consequences

The README's warning stays: don't set `grant_sudo: false` on hosts the controller manages.
