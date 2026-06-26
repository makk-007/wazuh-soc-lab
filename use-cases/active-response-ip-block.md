# Use Case 5: Active Response - Auto-Block IP After Repeated Failed Logins

**Date:** June 26, 2026
**Platform:** Wazuh 4.14.5 (single-node Docker deployment)
**Host:** Ubuntu 24.04 LTS (ASUS ROG Zephyrus G15)
**Attack Tool:** Hydra (local simulation)
**Monitored Agent:** Linux host agent (`Mawuse-ROG-Zephyrus-G15`)

---

## Objective

Configure Wazuh's Active Response system to automatically block the source IP of an SSH brute-force attacker the moment the brute-force threshold is crossed, then simulate an attack with Hydra and confirm that the block is applied at the firewall level without manual intervention. Confirm the automatic unblock fires after the configured timeout. The goal is to understand how a SIEM can move beyond passive alerting into automated threat containment.

---

## Concepts Covered

| Concept | Description |
|---------|-------------|
| Active Response | A Wazuh capability that executes a predefined defensive script on a monitored agent automatically when a specific rule fires |
| `firewall-drop` | A built-in Wazuh Active Response script that adds a DROP rule to the host firewall (`iptables`) to block all traffic from a specified source IP |
| `iptables` | Linux's built-in packet filtering framework. Used by `firewall-drop` to insert and remove blocking rules at the kernel network layer |
| Trigger rule | The Wazuh rule whose firing initiates the Active Response. In this use case rule `5712` (SSH brute-force, non-existent user) serves as the trigger |
| Timeout | The duration in seconds after which Wazuh automatically reverses the Active Response action, removing the firewall block |
| `<location>local</location>` | An Active Response directive specifying that the defensive action should be executed on the same agent that generated the triggering alert |
| Active Response log | A log file maintained by the Wazuh agent at `/var/ossec/logs/active-responses.log` recording every Active Response action executed, including the command, IP address, and timestamp |

---

## Environment

```
Host Machine (Ubuntu 24.04 LTS)
├── SSH Server (openssh-server) listening on port 22
├── Wazuh Agent (Linux) reporting to manager
├── iptables managing host firewall rules
├── Hydra (attacker) targeting 192.168.100.11
└── Docker
    └── single-node-wazuh.manager-1
        └── ossec.conf with Active Response block targeting rule 5712
```

---

## Approach

### 1. Active Response Configuration in the Manager

Active Response is configured on the manager side. The manager container was accessed:

```bash
docker exec -it single-node-wazuh.manager-1 /bin/bash
```

The built-in `firewall-drop` command block was confirmed present in the manager's `ossec.conf`:

```bash
grep -A 5 "firewall-drop" /var/ossec/etc/ossec.conf
```

The end of the file was inspected to identify the insertion point:

```bash
tail -30 /var/ossec/etc/ossec.conf
```

The `<active-response>` block was inserted before the closing `</ossec_config>` tag using a Python one-liner, since no text editors are available inside the container:

```bash
python3 -c "
content = open('/var/ossec/etc/ossec.conf').read()
block = '''
  <active-response>
    <command>firewall-drop</command>
    <location>local</location>
    <rules_id>5712</rules_id>
    <timeout>180</timeout>
  </active-response>
'''
content = content.replace('</ossec_config>', block + '</ossec_config>')
open('/var/ossec/etc/ossec.conf', 'w').write(content)
print('Done')
"
```

**Configuration breakdown:**

| Element | Value | Meaning |
|---------|-------|---------|
| `<command>` | `firewall-drop` | Built-in script that inserts a DROP rule into `iptables` |
| `<location>` | `local` | Execute the response on the agent that generated the alert |
| `<rules_id>` | `5712` | Only trigger when the SSH brute-force rule fires |
| `<timeout>` | `180` | Automatically remove the block after 180 seconds |

The insertion was verified:

```bash
grep -A 6 "active-response" /var/ossec/etc/ossec.conf
```

The manager was restarted to apply the change:

```bash
/var/ossec/bin/wazuh-control restart
```

The container was exited.

### 2. Pre-Attack Firewall State

The current `iptables` INPUT chain was captured before the attack to establish a baseline:

```bash
sudo iptables -L INPUT -n
```

No DROP rule for `192.168.100.11` was present at this point.

### 3. Agent Restart

The Wazuh agent was restarted on the host to ensure it was operating with the latest manager configuration:

```bash
sudo systemctl restart wazuh-agent
sudo systemctl status wazuh-agent
```

### 4. Hydra Brute-Force Attack

The same Hydra command used in Use Case 1 was rerun from the wordlist directory:

```bash
cd ~/wazuh-soc-lab/use-case-1-ssh-bruteforce
hydra -l testuser -P passwords.txt ssh://192.168.100.11
```

Hydra attempted all ten passwords against the non-existent `testuser` account, generating the same burst of parallel failed authentication attempts that triggered rule `5712` in Use Case 1.

### 5. Post-Attack Firewall State

Within seconds of Hydra completing, `iptables` was checked again:

```bash
sudo iptables -L INPUT -n
```

A new DROP rule for `192.168.100.11` was present, confirming `firewall-drop` executed successfully on the agent in response to the rule `5712` alert.

### 6. Active Response Log Verification

The agent's Active Response log was inspected to confirm the action was recorded:

```bash
sudo cat /var/ossec/logs/active-responses.log
```

The log showed the `firewall-drop` command executed with the attacker IP `192.168.100.11` and a precise timestamp matching the Hydra attack window.

### 7. Automatic Unblock Verification

After the 180 second timeout elapsed, `iptables` was checked a final time:

```bash
sudo iptables -L INPUT -n
```

The DROP rule for `192.168.100.11` was no longer present, confirming the automatic unblock executed as configured.

### 8. Dashboard Verification

Alerts were located under **Threat Intelligence > Threat Hunting > Events** tab filtered by the Linux host agent. The following rule IDs fired:

| Rule ID | Description | Significance |
|---------|-------------|--------------|
| `5710` | sshd: Attempt to login using a non-existent user | Individual failed attempt per Hydra connection |
| `5503` | PAM: User login failed | PAM layer recording of the same failures |
| `5551` | PAM: Multiple failed logins in a small period of time | PAM brute-force correlation alert |
| `5712` | sshd: Brute force trying to get access to the system. Non-existent user | Primary brute-force correlation alert and Active Response trigger |
| `651` | Host blocked by firewall-drop Active Response | Confirmation that the automated block was applied |
| `652` | Host unblocked by firewall-drop Active Response | Confirmation that the automated block was removed after the timeout |

Rules `651` and `652` together capture the full Active Response lifecycle: automated block on threat detection and automatic release after the configured timeout, both recorded as first-class events in the SIEM.

The `651` alert was expanded to confirm the source IP, the triggering rule, and the timestamp of the block action.

---

## Screenshots

| File | Description |
|------|-------------|
| `01-manager-ossec-conf-active-response.png` | Manager `ossec.conf` showing the inserted active-response block |
| `02-hydra-attack-output.png` | Hydra terminal output showing the brute-force attempt |
| `03-iptables-before.png` | `iptables -L INPUT -n` before the attack showing no block rule |
| `04-iptables-after.png` | `iptables -L INPUT -n` after the attack showing the DROP rule for `192.168.100.11` |
| `05-active-response-log-part-01.png` | Active response log first portion showing firewall-drop execution |
| `06-active-response-log-part-02.png` | Active response log second portion showing the unblock action |
| `07-iptables-unblocked.png` | `iptables -L INPUT -n` after the 180 second timeout showing the rule removed |
| `08-dashboard-alerts-part-01.png` | Dashboard showing the first portion of alerts including rules 5710, 5503, 5551 |
| `09-dashboard-alerts-part-02.png` | Dashboard showing rule 5712 and rule 651 alerts |
| `10-dashboard-alerts-part-03.png` | Dashboard showing rule 652 unblock alert |
| `11-expanded-651-alert-part-01.png` | Expanded detail of the rule 651 active response alert, first portion |
| `12-expanded-651-alert-part-02.png` | Expanded detail of the rule 651 active response alert, second portion |
| `13-expanded-651-alert-part-03.png` | Expanded detail of the rule 651 active response alert, third portion |
| `14-expanded-651-alert-part-04.png` | Expanded detail of the rule 651 active response alert, fourth portion |
| `15-expanded-651-alert-part-05.png` | Expanded detail of the rule 651 active response alert, fifth portion |

---

## Key Takeaway

Active Response transforms Wazuh from a passive observer into an automated first responder. The moment rule `5712` fired, no human intervention was required to contain the threat: the attacker's IP was blocked at the kernel network layer within seconds. The presence of both rule `651` and rule `652` in the dashboard is significant because it shows the entire containment lifecycle as auditable SIEM events. A SOC analyst reviewing this timeline can see exactly when the block was applied, how long it lasted, and when it was lifted, which is precisely the kind of traceable, time-stamped response record that satisfies both operational and compliance requirements. The 180 second timeout also demonstrates a key principle in automated response design: blocks should be time-limited by default to avoid permanently locking out legitimate traffic based on a misidentified source.

---

## Reflection

This use case completed without deviations from the planned steps. The only additional detail observed beyond the expected alerts was the appearance of rule `652` in the dashboard, confirming that the automatic unblock after the 180 second timeout was also recorded as a discrete SIEM event. This was not anticipated in the initial plan but reflects Wazuh's design of treating both the application and removal of an Active Response as auditable events, which strengthens the forensic value of the alert timeline.

---

## References

- [Wazuh Active Response documentation](https://documentation.wazuh.com/current/user-manual/capabilities/active-response/index.html)
- [Wazuh firewall-drop script reference](https://documentation.wazuh.com/current/user-manual/capabilities/active-response/default-active-response-scripts.html)
- [iptables man page](https://linux.die.net/man/8/iptables)
- [MITRE ATT&CK T1110: Brute Force](https://attack.mitre.org/techniques/T1110/)