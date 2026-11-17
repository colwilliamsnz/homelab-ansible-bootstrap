# Ansible Bootstrap

Wattle Labs ansible-pull bootstrap. It gets a new host to the point where the Ansible controller
(`inf-cfgm-01`) can connect and manage it, and keeps it there. **Access only:** everything else is configured
by the controller repo, [host-inf-cfgm-01](https://github.com/colwilliamsnz/host-inf-cfgm-01).

Run it once by hand on a new host. After that, an hourly systemd timer (installed by the play) pulls this repo and
re-applies it every hour. A controller IP or key change is just a commit here, not a visit to every host, and
changes made on a host by hand are corrected within the hour.

Supported targets: Debian and Ubuntu, including Proxmox VE (access only, no firewall). The play asserts this and
fails on anything else.

## What it does

1. **Checks first, before any change:** Debian family, at least one well-formed key in `controller_pubkeys`, and (except on PVE)
   at least one network in `ssh_allowed_cidrs`.
2. **Detects Proxmox VE** (`/etc/pve`). PVE nodes get access only: no `ufw` package and no firewall rules.
   See [Proxmox VE nodes](#proxmox-ve-nodes).
3. **Installs only what access and the hourly pull need** (`bootstrap_packages`): `openssh-server`, `sudo`,
   `python3`, `python3-apt`, `git`, `ansible-core`, `ca-certificates`, `nano`, plus `ufw` on non-PVE hosts. Installed
   if missing, never upgraded.
4. **Creates the `svc_ansible` service account** with a locked password, and writes the controller's public
   key to its `authorized_keys`. The file is authoritative: any other key is removed.
5. **Grants `svc_ansible` passwordless sudo** via a drop-in in `/etc/sudoers.d/`. See [Sudo access](#sudo-access).
6. **Manages the UFW rules that let SSH in** from `ssh_allowed_cidrs` (not on PVE). It does **not** turn UFW
   on. See [Firewall behaviour](#firewall-behaviour).
7. **Makes sure the SSH service is enabled and running.**
8. **Installs the hourly pull** (`ansible-bootstrap-pull.service` and `.timer`), and turns the timer on or off
   from `bootstrap_schedule_enabled`. See [Hourly pull](#hourly-pull).

## Out of scope

The controller repo configures everything else after the bootstrap, including:

- base packages, timezone and locale (its `base` role);
- the firewall's default policy, turning UFW on, and service rules (planned firewall role);
- NTP, automatic security updates, MOTD, shell prompt;
- SSH daemon hardening (password auth, root login), and package upgrades.

## Public repo

This repo is public, and hosts pull it over HTTPS with no credentials. It contains no secrets: the controller
public key is public by nature, and everything else is configuration. The one thing it does reveal is internal
network layout (`ssh_allowed_cidrs`, `controller_from`), which is accepted.

Being public controls who can read, not who can write. Repo integrity is what matters, all the more because
hosts pull it every hour. See [Security model](#security-model).

## Why a separate repo

The bootstrap is kept apart from the controller repo on purpose:

- ansible-pull clones the whole repo onto every target. Merged, every host would get the controller's inventory and roles.
- Push access to this repo means root on every host that pulls it (see [Security model](#security-model)). A small repo with one purpose is easier to lock down.

## Prerequisites (target host)

```bash
sudo apt update && sudo apt install git ansible-core
```

The host also needs outbound HTTPS to `github.com`.

## Configure

All variables live in `vars/main.yml`. Review them before the first run. In particular, confirm
`ssh_allowed_cidrs` covers the network you're connecting from, or you'll lock yourself out once UFW is on.

| Variable | Purpose |
| --- | --- |
| `ansible_svc_user`, `ansible_svc_home`, `ansible_svc_shell` | The service account the controller connects as |
| `controller_pubkeys` | The controller's SSH public keys, one full line each (type, key, comment). Normally one; two during a rotation |
| `controller_from` | Source IP restriction for the key (`from=`). Comma-separated for more than one. Empty means none |
| `controller_key_options` | Options prepended to the key. Default `restrict`. Empty means none |
| `grant_sudo`, `sudoers_dropin_path` | Passwordless sudo for the service account, and where the drop-in goes |
| `ssh_service_name` | SSH service unit name (`ssh` on Debian/Ubuntu) |
| `ssh_port`, `ssh_allowed_cidrs` | Port and networks for the UFW SSH rules. Authoritative: other allow/limit rules for `ssh_port` are removed |
| `bootstrap_packages` | Packages access and the pull need (plus `ufw` on non-PVE hosts) |
| `bootstrap_schedule_enabled` | Hourly timer on (`true`, the current setting) or off (`false`, the [kill switch](#hourly-pull)) |
| `bootstrap_schedule_oncalendar`, `bootstrap_schedule_random_delay` | When the timer fires (`hourly`, plus up to `15min`) |
| `bootstrap_schedule_only_if_changed` | `false` (current): re-apply every hour. `true`: only when there's a new commit |
| `bootstrap_repo_url`, `bootstrap_repo_branch`, `bootstrap_checkout_dir` | What the timer pulls, and where it keeps the checkout |

`restrict` disables port forwarding, agent forwarding, X11 forwarding and pty allocation for the key. Ansible
needs none of these: it runs modules over the SSH exec channel, and `become` uses passwordless sudo. Clear
`controller_key_options` if you need an interactive shell as the service account.

## First run (by hand)

```bash
sudo LC_ALL=C.UTF-8 ansible-pull \
    -U https://github.com/colwilliamsnz/homelab-ansible-bootstrap.git \
    -C main \
    -d /var/lib/ansible-pull/bootstrap \
    -i localhost, \
    --clean \
    local.yml
```

- `LC_ALL=C.UTF-8`: ansible-core refuses to run without a UTF-8 locale, and a fresh minimal image may not have
  one set. `C.UTF-8` is built into Debian and Ubuntu. The timer sets the same thing for its runs.
- `-d /var/lib/ansible-pull/bootstrap`: the same checkout the timer uses, so the timer updates it rather than
  cloning a second copy.
- With `bootstrap_schedule_enabled: true` (the current setting), this run also starts the hourly timer.
- HTTPS verifies GitHub through the system CA bundle (`ca-certificates`), so no `--accept-host-key` or
  `known_hosts` entry is needed.
- The play is safe to re-run. A re-run reports no changes if nothing has drifted.
- Add `--check` for a dry run. It skips the `ufw` commands, so it doesn't preview firewall rule changes. On a
  host that doesn't have the timer yet, the timer step's error is ignored in a dry run.
- To test a branch, use `-C <branch>`.

## After the first run: trust the host key on the controller

Add the target's host key to the controller's `known_hosts`, so later automated SSH connections can verify it.
Playbooks on the controller run as `svc_ansible`, so the key goes in that account's `known_hosts`. Run these on
the controller, from the root of the [host-inf-cfgm-01](https://github.com/colwilliamsnz/host-inf-cfgm-01) repo,
after adding the host to `inventory/hosts.yml` there.

Single host:

```bash
ssh-keyscan -H -t ed25519 target.wattle.nz | sudo -u svc_ansible tee -a /home/svc_ansible/.ssh/known_hosts
```

Every host in the inventory:

```bash
sudo -u svc_ansible sh -c 'ansible all --list-hosts' | tail -n +2 | xargs -n1 ssh-keyscan -H -t ed25519 2>/dev/null \
  | sudo -u svc_ansible tee -a /home/svc_ansible/.ssh/known_hosts >/dev/null
```

`ssh-keyscan` trusts whatever key answers. Compare the fingerprint against the target's console
(`ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub`) before relying on it.

Then apply the controller's baseline:

```bash
sudo -u svc_ansible sh -c 'ansible-playbook site.yml --limit <host>'
```

## Hourly pull

The play always installs two systemd units:

- **`ansible-bootstrap-pull.service`:** a oneshot, so runs never overlap. It runs:
  ```
  ansible-pull -U <bootstrap_repo_url> -C main -d /var/lib/ansible-pull/bootstrap -i localhost, --clean local.yml
  ```
  with `LC_ALL=C.UTF-8` and `HOME=/root`. `--clean` discards any local edits to the checkout.
- **`ansible-bootstrap-pull.timer`:** `OnCalendar=hourly`, `RandomizedDelaySec=15min` (so hosts don't all hit
  GitHub at once), and `Persistent=true` (a run missed while the host was off happens at the next boot).

**Every hour, not just on new commits.** With `bootstrap_schedule_only_if_changed: false` (the current setting),
every run applies the play, whether or not the repo has changed. So changes made on the host itself (e.g. someone
editing `authorized_keys` or deleting the SSH rule) are put back within the hour. For an access-only play, the cost
is small. Setting it to `true` adds `--only-if-changed`: runs then only apply the play when there's a new commit.
To re-apply on one host straight away, run the [first-run command](#first-run-by-hand) again.

**Until the controller's `base` role stops managing access** (plan step 4 in host-inf-cfgm-01), both repos write
`authorized_keys` and sudoers. If their values differ, each run of one overwrites the other, now every hour.
Keep them identical (see [Changing controller access](#changing-controller-access)).

**Rollout:** the timer is on in this repo (`bootstrap_schedule_enabled: true`). A host only gets it when the
bootstrap is run by hand there once, which installs and starts the timer. A host that has never run this version
never pulls, so it can't pick up the timer by itself. Order: the test host (`lab-cfgm-01`), then LXCs, VMs and
Proxmox VE nodes.

**Kill switch:** set `bootstrap_schedule_enabled: false` and push. On its next hourly run, every host with a running
timer pulls the change and turns its own timer off.

**Checking a host:**

```bash
systemctl list-timers ansible-bootstrap-pull.timer   # next and last run
journalctl -u ansible-bootstrap-pull -n 50           # output of recent runs
systemctl is-failed ansible-bootstrap-pull.service   # "failed" if the last run failed
```

A failed run leaves the service failed, so systemd reports the host as `degraded`. The controller's MOTD shows
that at login, with the unit name.

## Changing controller access

Do these as an overlap, so the controller never loses access while hosts catch up.

**Controller IP change:**
1. Add the new IP alongside the old one: `controller_from: "10.10.20.42,<new>"`. Make sure its network is in
   `ssh_allowed_cidrs`. Merge.
2. Wait about 75 minutes (hourly, plus up to 15 minutes of random delay), and confirm the controller still reaches
   every host.
3. Move the controller.
4. Remove the old IP and merge.

**Controller key change (rotation):**
1. Add the new public key to `controller_pubkeys`, keeping the old one. Merge.
2. Wait about 75 minutes, and confirm every host has both keys
   (`sudo -u svc_ansible sh -c 'ansible all -m ansible.builtin.command -a "wc -l .ssh/authorized_keys"'` should show 2).
3. Point the controller at the new private key (`private_key_file` in host-inf-cfgm-01's `ansible.cfg`), and confirm a run works.
4. Remove the old key from `controller_pubkeys` and merge.

**Until the controller's `base` role stops managing access** (plan step 4 in host-inf-cfgm-01), it still writes
`authorized_keys` and sudoers from that repo's `inventory/group_vars/all.yml` (`base_ansible_from`,
`base_ansible_pubkeys`). At each step above, change the same value there too and run `--tags base`. Otherwise the
two repos undo each other every hour. host-inf-cfgm-01's `roles/base/README.md` has the same procedure from that side.

## Proxmox VE nodes

PVE nodes are detected by `/etc/pve` and get access only: the service account, key, sudo, sshd and the hourly pull.
They get no `ufw` package and no firewall rules. Default-deny UFW would block the web UI (8006), corosync,
migration and SNMP. PVE has its own firewall for that.

## Sudo access

The controller connects as `svc_ansible`, an unprivileged account. Almost everything a playbook does (install
packages, write to `/etc`, manage services) needs root, which Ansible gets with `become: true`. On Debian/Ubuntu,
`become` runs `sudo` under the hood, and there's no one at the keyboard to type a password. The sudoers drop-in is
what makes that work.

The play writes one line to `/etc/sudoers.d/99-ansible`:

```
svc_ansible ALL=(ALL) NOPASSWD: ALL
```

Read left to right: on any host, `svc_ansible` may run any command, as any user, without a password.

Why a drop-in rather than editing `/etc/sudoers`:

- `/etc/sudoers` is owned by the distro package. Editing it invites merge conflicts on upgrade and makes the change hard to find later.
- A drop-in is a single file with one owner and one purpose. Removing it (or setting `grant_sudo: false`) removes the grant cleanly.
- The `99-` prefix sorts it after the distro's own drop-ins, so it's applied last.

Why it's safe to write with Ansible:

- The file is passed through `visudo -cf` before it's put in place. A syntax error in sudoers can lock everyone out of `sudo` on the host; `visudo -cf` rejects the file and the task fails instead.
- Mode is `0440`, owner `root:root`. `sudo` ignores drop-ins with looser permissions.

Why `NOPASSWD: ALL` and not a narrower command list:

- Ansible modules run as arbitrary Python scripts in temp paths, so a command allow-list can't be expressed meaningfully. Any attempt to restrict it would either break Ansible or be trivially bypassed.
- The real boundary is who can authenticate as `svc_ansible`. That's controlled by the `authorized_keys` file (only the listed keys, `restrict`, `from=`) and the firewall. See [Security model](#security-model).

`grant_sudo: false` removes the drop-in. Don't use it on hosts the controller manages: the controller's playbooks need `become`.

## Firewall behaviour

This play owns **only the UFW rules for SSH** (`ssh_port`), on non-PVE hosts. The firewall's default policy,
turning UFW on, and rules for other services belong to the controller repo (a planned firewall role).

- **It doesn't turn UFW on.** UFW stores rules even while it's off. When the controller's firewall role turns
  UFW on with default-deny, SSH from `ssh_allowed_cidrs` is already allowed. Hosts where UFW is already on keep it on.
- **Until that role exists, a newly bootstrapped host has no active firewall.** That's accepted.
- **The controller's firewall role uses UFW as well** (decided Nov 2026), not nftables or the Proxmox per-guest firewall: UFW is already on every bootstrapped host, and the tagged SSH rules here already work with it. The role never runs `ufw reset` and only deletes rules it tagged itself, so the role and the hourly pull don't undo each other.
- **Managed rules** take the form `allow from <cidr> to any port <ssh_port> proto tcp`, with the comment
  `bootstrap-ssh`. New rules are added before old ones are removed, so SSH from a permitted network is never
  cut off. Networks removed from `ssh_allowed_cidrs` have their rule deleted.
- **Any other `allow` or `limit` rule for `ssh_port` is deleted.** This covers `allow 22/tcp`, `limit 22/tcp`,
  `allow OpenSSH`, `allow ssh`, `allow from any to any port 22`, and interface-scoped forms such as
  `allow in on eth0 to any port 22`. Rules for other ports are left alone. Extra SSH sources (e.g. an admin
  network) go in `ssh_allowed_cidrs`, not in the firewall role.
- **Lockout risk:** once UFW is on, SSH from a network not in `ssh_allowed_cidrs` is refused. The controller's IP
  (and `controller_from`) must fall inside a permitted network.
- Rules are IPv4 only when the CIDRs are IPv4. Inbound IPv6 SSH is then denied by the default policy.

## Security model

The controller public key isn't a secret. Repo integrity is what matters. Anyone who can push to `main` decides
which key is authorised, and so gets root on every host that pulls. **With the hourly timer, that's within the hour.**
The repo being public doesn't change that: visibility controls who can read, not who can write.

- Protect `main`, and restrict push access.
- Sign commits, and pull with `--verify-commit` once a signing key is in place on targets (planned; needed before
  the timer goes on everywhere).
- Set `controller_from` to the controller's IP.
- Keep `controller_key_options` at `restrict` unless there's a reason not to.
- The [kill switch](#hourly-pull) stops every running timer on the next pull, but it also arrives as a commit, so
  it relies on the same repo integrity.

## Working on this repo

### Scope

- Access only. Anything listed under [Out of scope](#out-of-scope) goes in the controller repo, not here.
- No secrets. The repo stays public and pulled over HTTPS: no private-repo or deploy-key variant.
- The play never turns UFW on, sets its defaults, or touches rules for ports other than `ssh_port`.
- `bootstrap_schedule_only_if_changed` stays `false`, so every hourly run re-applies the play.

### The play

- It runs hourly on every host, so it must stay safe to re-run.
- Asserts come before any change. Validate before writing (`visudo -cf`).
- Anything that can lock out the controller needs a way to dry-run it with `--check`.
- `ansible.builtin` modules only. A fresh host has only `ansible-core`, so no collections.
- Loops are expanded before `when` is checked. Tasks skipped on PVE need `| default([])` on registered results in their loops.
- The `authorized_keys` output must stay byte-identical to the controller's `base` role output for the same values.

### Values shared with the controller repo

Until plan step 4, both repos write `authorized_keys` and sudoers. Change these in both repos in the same session, following [Changing controller access](#changing-controller-access). Never replace an old value in a single commit.

| `vars/main.yml` (here) | `inventory/group_vars/all.yml` (host-inf-cfgm-01) |
| --- | --- |
| `controller_pubkeys` | `base_ansible_pubkeys` |
| `controller_from` | `base_ansible_from` |
| `controller_key_options` | `base_ansible_key_options` |
| `ansible_svc_user`, `ansible_svc_home`, `ansible_svc_shell` | `base_ansible_user`, `base_ansible_home`, `base_ansible_shell` |
| `sudoers_dropin_path` | `base_sudoers_dropin_path` |

`controller_from` must fall inside `ssh_allowed_cidrs`.

### Testing and merging

- Ansible isn't installed where this repo is edited, so nothing is run or linted locally.
- Test from a branch, at the console of `lab-cfgm-01.wattle.nz`, with `--check` first and then without it. Test on a PVE node only after that.
  ```bash
  sudo LC_ALL=C.UTF-8 ansible-pull -U https://github.com/colwilliamsnz/homelab-ansible-bootstrap.git -C <branch> -i localhost, --check
  ```
- Flow: branch, commit, push the branch, test on `lab-cfgm-01`, `git merge --ff-only` into `main`, push, delete the branch.
- Hosts pull `main`, so anything merged reaches every host within the hour.
