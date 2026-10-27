# SSH Brute Force Attack Simulation & Log Analysis

## Objective
Simulate a real-world SSH brute force attack in a home lab and manually
analyze the resulting authentication logs (`/var/log/auth.log`) to identify
the attack pattern and key detection indicators, mapped to MITRE ATT&CK.

## Lab Setup
- **Target:** Ubuntu VM (hostname: soc-lab-wazuh) running OpenSSH server
- **Attacker:** Kali Linux VM running Hydra
- **Network:** Bridged mode (both VMs on the same local network)
- **Test account:** `testuser` created specifically for this lab, with a deliberately weak password (`password123`) to allow the attack to succeed
- **Note:** A Wazuh agent was configured on the target as part of a broader home SOC lab, but SIEM-level alert verification (dashboard) was not completed for this specific exercise — detection here is based on direct log analysis. Wazuh-based correlation (rule 5710/5712) is a planned follow-up.

## Attack Simulation

A small password dictionary (`passwords.txt`) was created with common weak
passwords. Hydra was then used to perform a dictionary-based SSH brute
force attack against the target:

```
hydra -l testuser -P passwords.txt ssh://192.168.1.34
```

**Result:**
```
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak
[DATA] max 10 tasks per 1 server, overall 10 tasks, 10 login tries (l:1/p:10), ~1 try per task
[DATA] attacking ssh://192.168.1.34:22/
[22][ssh] host: 192.168.1.34   login: testuser   password: password123
1 of 1 target successfully completed, 1 valid password found
```

See `screenshots/03_hydra_success_output.png`.

## Detection — Log Analysis

### Baseline (normal login behavior)
Before the attack, a baseline login was captured to establish what normal
SSH activity looks like in `/var/log/auth.log`:

```
sshd-session[25865]: Accepted password for testuser from 192.168.1.45 port 57562 ssh2
sshd-session[25865]: pam_unix(sshd:session): session opened for user testuser(uid=1002) by testuser(uid=0)
```

See `screenshots/01_baseline_auth_log.png`.

### Attack in progress
During the brute force attempt, `auth.log` was monitored live (`tail -f`).
Multiple rapid, near-simultaneous connection attempts were logged from the
same source IP within a fraction of a second — a pattern no human could
produce manually:

```
sshd-session[33012]: Connection closed by authenticating user testuser 192.168.1.45 port 47860 [preauth]
sshd[21476]: srclimit_penalise: 192.168.1.45/32: activating ipv4 penalty of 18 seconds for penalty: failed authentication
sshd-session[33010]: Connection closed by authenticating user testuser 192.168.1.45 port 47842 [preauth]
sshd-session[33020]: Connection closed by authenticating user testuser 192.168.1.45 port 47884 [preauth]
sshd-session[33023]: Connection closed by authenticating user testuser 192.168.1.45 port 47888 [preauth]
sshd-session[33025]: Connection closed by authenticating user testuser 192.168.1.45 port 47898 [preauth]
sshd-session[33027]: Connection closed by authenticating user testuser 192.168.1.45 port 47904 [preauth]
```

See `screenshots/02_live_attack_auth_log.png`.

### Key log indicators identified
| Indicator | Meaning |
|---|---|
| `Accepted password` | Successful login |
| `Failed password` | A failed login attempt |
| Multiple connections from one IP within milliseconds | Automated/scripted attack, not manual typing |
| `srclimit_penalise` | SSH's own built-in rate-limiting (prevention), activated after repeated failures from the same source IP |

## MITRE ATT&CK Mapping
- **T1110 — Brute Force**
- **T1110.001 — Password Guessing**

## Key Finding
Despite OpenSSH's built-in per-source rate-limiting (`srclimit_penalise`),
the attack succeeded because the password list was short and the correct
password was tried early. This highlights that rate-limiting alone is not
a sufficient control — it is a *prevention* mechanism, not a *detection*
one. A SOC analyst still needs SIEM-level alerting (e.g. Wazuh rule 5710/5712
for repeated authentication failures, a planned next step for this lab) and
strong account lockout / MFA policies as complementary controls.

## Recommendations
- Enforce strong password policy / disable password-based SSH auth in favor of key-based auth
- Configure SIEM alerting on repeated `Failed password` events from a single source within a short window
- Consider fail2ban or equivalent for automatic IP blocking after N failed attempts

## Screenshots
- `screenshots/01_baseline_auth_log.png` — normal login baseline
- `screenshots/02_live_attack_auth_log.png` — live auth.log during the attack
- `screenshots/03_hydra_success_output.png` — Hydra attack result
