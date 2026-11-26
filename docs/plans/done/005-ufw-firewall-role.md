# 005: UFW firewall role in the controller repo (plan step 5)

- Status: Done
- Blocked by: none

The role turns UFW on and owns the default policy and service rules. This repo's SSH rules stay as they are, and the role follows the ownership rules in the README's "Firewall behaviour". The work is in host-inf-cfgm-01.

## Done when

- [x] Controller repo: the UFW firewall role exists and is applied.
- [x] Here: the README's "Firewall behaviour" no longer describes the role as planned.
