# Use Case 1: SSH Brute-Force Detection

**Date:** June 22, 2026
**Platform:** Wazuh 4.14.5 (single-node Docker deployment)
**Host:** Ubuntu 24.04 LTS (ASUS ROG Zephyrus G15)
**Attack Tool:** Hydra (local simulation)
**Monitored Agent:** Linux host agent (`Mawuse-ROG-Zephyrus-G15`)

---

## Objective

Simulate a dictionary-based SSH brute-force attack against the local Linux host using Hydra, then confirm that Wazuh detects, correlates, and alerts on the repeated authentication failure pattern. The goal is to understand how a SIEM translates raw authentication logs into actionable threat intelligence.

---

## Concepts Covered

| Concept | Description |
|---------|-------------|
| SSH brute-force attack | A method where an attacker systematically tries many username and password combinations against an SSH service to gain unauthorized access |
| Hydra | A parallelized login cracker used in penetration testing to automate credential guessing against network services |
| `/var/log/auth.log` | Ubuntu's system authentication log file, which records all login attempts, sudo usage, and PAM events |
| Wazuh rule correlation | Wazuh's ability to group multiple related log events within a time window and trigger a higher-severity alert when a threshold is crossed |
| PAM (Pluggable Authentication Modules) | Linux's authentication framework that sits between applications like SSH and the underlying system, generating its own log entries alongside sshd |
| Rule ID | A unique identifier assigned to each detection rule in Wazuh's ruleset, used to categorize and describe the type of event detected |

---

## Environment

```
Host Machine (Ubuntu 24.04 LTS)
├── SSH Server (openssh-server) listening on port 22
├── Wazuh Agent (Linux) reporting to manager
└── Hydra (attacker) running locally, targeting 192.168.91.241
```

The attack was simulated locally: Hydra ran on the same Ubuntu host it was targeting. This mirrors a scenario where an attacker on the same network segment as the target launches a credential-stuffing campaign.

---

## Approach

### 1. SSH Server Installation and Verification

Running `sudo systemctl status ssh` returned `Unit ssh.service could not be found`, confirming that `openssh-server` was not installed on the host. It was installed before proceeding:

```bash
sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable ssh
sudo systemctl start ssh
```

The service was then verified:

```bash
sudo systemctl status ssh
```

Output confirmed the service was `active (running)` and listening on port 22.

### 2. Target IP Identification

Retrieved the host's local IP address:

```bash
ip addr show | grep "inet "
```

Target IP identified: `192.168.91.241`

### 3. Hydra Installation

```bash
sudo apt update
sudo apt install -y hydra
```

Verified with `hydra -h`.

### 4. Wordlist Creation

A minimal wordlist was created for the simulation. A single fake username (`testuser`, not a real account on the system) was used alongside a short list of incorrect passwords to ensure all attempts would fail and generate authentication failure logs.

```
usernames.txt  →  testuser
passwords.txt  →  10 incorrect password entries
```

### 5. Attack Execution

```bash
hydra -l testuser -P passwords.txt ssh://192.168.91.241
```

Hydra attempted each password in sequence against the SSH service. Since `testuser` does not exist as a system account, every attempt was rejected at the authentication stage before any password verification occurred.

### 6. Auth Log Verification

Before checking the dashboard, the raw source log was inspected to confirm the attack had generated entries:

```bash
sudo tail -n 30 /var/log/auth.log
```

Multiple entries appeared in quick succession, each recording an invalid user attempt from `192.168.91.241` across different ephemeral ports, confirming Hydra's parallel connection behavior.

### 7. Wazuh Dashboard Analysis

Alerts were located under **Threat Intelligence > Threat Hunting > Events** tab (not under Security Events as initially expected). Four rule IDs were triggered by the attack:

| Rule ID | Description | Significance |
|---------|-------------|--------------|
| `5710` | sshd: Attempt to login using a non-existent user | Fired on each individual failed attempt against `testuser` |
| `5503` | PAM: User login failed | PAM layer recording the same failures from its own perspective |
| `5712` | sshd: Brute force trying to get access to the system. Non-existent user | Wazuh's correlation rule, triggered when multiple `5710` events were detected within a short window |
| `5551` | PAM: Multiple failed logins in a small period of time | PAM's equivalent correlation alert, mirroring `5712` at the PAM layer |

Rule `5712` was selected for detailed inspection as the primary brute-force correlation alert.

### 8. Expanded Alert Detail (Rule 5712)

| Field | Value |
|-------|-------|
| Rule ID | `5712` |
| Description | sshd: Brute force trying to get access to the system. Non-existent user |
| Source IP | `192.168.91.241` |
| Username | `testuser` |

Raw log snippet captured in the alert:

```
Jun 22 10:47:06 Mawuse-ROG-Zephyrus-G15 sshd[18625]: Invalid user testuser from 192.168.91.241 port 52300
Jun 22 10:47:06 Mawuse-ROG-Zephyrus-G15 sshd[18629]: Invalid user testuser from 192.168.91.241 port 52294
Jun 22 10:47:06 Mawuse-ROG-Zephyrus-G15 sshd[18627]: Invalid user testuser from 192.168.91.241 port 52272
Jun 22 10:47:06 Mawuse-ROG-Zephyrus-G15 sshd[18632]: Invalid user testuser from 192.168.91.241 port 52328
Jun 22 10:47:06 Mawuse-ROG-Zephyrus-G15 sshd[18623]: Invalid user testuser from 192.168.91.241 port 52268
Jun 22 10:47:06 Mawuse-ROG-Zephyrus-G15 sshd[18611]: Disconnected from invalid user testuser 192.168.91.241 port 52250 [preauth]
Jun 22 10:47:06 Mawuse-ROG-Zephyrus-G15 sshd[18630]: Invalid user testuser from 192.168.91.241 port 52314
```

Each line represents a separate parallel connection opened by Hydra, all within the same second. The varying port numbers on the source side confirm Hydra's concurrent connection behavior. The `[preauth]` tag on the disconnection entry indicates the session was terminated before the authentication handshake completed.

---

## Screenshots

| File | Description |
|------|-------------|
| `01-hydra-attack-output.png` | Hydra terminal output showing attempted credentials and failure responses |
| `02-auth-log-entries.png` | Raw `/var/log/auth.log` entries generated by the attack |
| `03-dashboard-alert-list.png` | Wazuh dashboard showing all four triggered rule IDs |
| `04-expanded-5712-alert-part-01.png` | Expanded detail view of the first half of rule 5712 brute-force correlation alert |
| `05-expanded-5712-alert-part-02.png` | Expanded detail view of the second half of rule 5712 brute-force correlation alert |

---

## Key Takeaway

A single failed SSH login is noise. What Wazuh demonstrates here is the value of log correlation: individual `5710` events are low-signal, but when Wazuh detects several of them from the same source within a narrow time window, it escalates to `5712`, a rule that carries explicit brute-force intent. The dual-layer alerting from both sshd (`5710`, `5712`) and PAM (`5503`, `5551`) also shows how the same event surfaces through multiple subsystems, each contributing a different perspective on what happened. A SOC analyst seeing `5712` and `5551` fire together has high confidence the event is genuine rather than a false positive.

---

## Reflection

The navigation path to alerts differed from documentation: events were found under **Threat Intelligence > Threat Hunting > Events** rather than **Security Events**. This is a minor UI difference in Wazuh 4.14.5 and worth noting for anyone following this lab on the same version. The raw log inspection step before opening the dashboard proved valuable: seeing Hydra's parallel connections across multiple ports in the same second made the mechanics of the attack immediately clear in a way the dashboard alone would not have.

---

## References

- [Wazuh built-in ruleset documentation](https://documentation.wazuh.com/current/user-manual/ruleset/index.html)
- [Hydra project page](https://github.com/vanhauser-thc/thc-hydra)
- [Ubuntu openssh-server documentation](https://ubuntu.com/server/docs/openssh-server)
- [Wazuh rule 5712 source](https://github.com/wazuh/wazuh-ruleset/blob/master/rules/0095-sshd_rules.xml)