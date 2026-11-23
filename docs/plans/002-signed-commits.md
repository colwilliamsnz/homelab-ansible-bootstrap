# 002: Signed commits and verified pulls

- Status: Open, Colin investigating
- Blocked by: none

Push access to `main` is root on every host that pulls, and the hourly pull means a bad push reaches every host within the hour (README, "Security model"). Commits are to be signed and the pull run with `--verify-commit`. This is wanted before the timer goes on everywhere. Interim: branch protection plus restricted push on `main`.

## Done when

- [ ] Checked whether `ansible-pull --verify-commit` accepts SSH signatures or only GPG.
- [ ] Commits on `main` are signed.
- [ ] The signing key is on the targets and the pull runs with `--verify-commit`.
