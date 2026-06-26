# Wazuh SOC Lab

A home SIEM environment built on Wazuh 4.14.5, demonstrating core Security Operations Centre capabilities across five detection and response use cases. The lab runs on a single Ubuntu 24.04 LTS host using a Docker single-node deployment for the Wazuh stack, with a Windows 11 Pro VM acting as a second monitored endpoint.

---

## Environment

| Component | Details |
|-----------|---------|
| Host OS | Ubuntu 24.04 LTS |
| Hardware | ASUS ROG Zephyrus G15 (AMD Ryzen 6900HS, 16 GB RAM, Nvidia RTX 3060) |
| Wazuh Stack | 4.14.5 via Docker single-node deployment (manager, indexer, dashboard) |
| Linux Agent | Wazuh 4.14.5 installed on the Ubuntu host |
| Windows Agent | Wazuh 4.14.5 installed on a Windows 11 Pro VM (VirtualBox) |
| Virtualization | VirtualBox on Ubuntu 24.04 LTS |

### Architecture

```
Host Machine (Ubuntu 24.04 LTS)
├── Docker
│   ├── wazuh.manager      (Wazuh Manager)
│   ├── wazuh.indexer      (OpenSearch)
│   └── wazuh.dashboard    (Kibana-based UI)
├── Wazuh Agent (Linux)    installed on host, reports to manager
└── VirtualBox
    └── Windows 11 Pro VM
        └── Wazuh Agent (Windows)   reports to manager via local network
```

---

## Setup

Full environment setup documentation including Docker installation, VirtualBox configuration, Windows 11 VM creation, Wazuh stack deployment, and six infrastructure deviations encountered and resolved:

- [wazuh-docker-setup.md](setup/wazuh-docker-setup.md)

---

## Use Cases

| # | Use Case | Key Technique | Rule IDs |
|---|----------|---------------|----------|
| 1 | [SSH Brute-Force Detection](use-cases/ssh-brute-force.md) | Hydra dictionary attack against SSH, Wazuh log correlation | 5710, 5712, 5503, 5551 |
| 2 | [File Integrity Monitoring](use-cases/file-integrity-monitoring.md) | Real-time syscheck monitoring via inotify, file create/modify/delete detection | 554, 550, 553 |
| 3 | [Log Analysis with Custom Decoder and Rule](use-cases/log-analysis-custom-rule.md) | Custom decoder and rule for a fictional application log source | 100005 |
| 4 | [Malware Detection via EICAR Test File](use-cases/malware-detection-eicar.md) | ClamAV on Linux, Windows Defender on Windows 11, cross-platform SIEM ingestion | 52502, 62123, 62124 |
| 5 | [Active Response: Auto-Block IP](use-cases/active-response-ip-block.md) | Automated firewall-drop triggered by brute-force detection, timed unblock | 651, 652 |

---

## Use Case Summaries

### 1. SSH Brute-Force Detection
Simulated a dictionary-based SSH brute-force attack using Hydra against the local Linux host. Wazuh's built-in SSHD ruleset correlated the burst of parallel failed authentication attempts from `/var/log/auth.log` into a brute-force alert (rule 5712), with additional correlation alerts from the PAM layer (rule 5551). The dual-layer alerting from both sshd and PAM demonstrates how the same event surfaces through multiple subsystems, each contributing a different perspective on what happened.

### 2. File Integrity Monitoring
Configured Wazuh's syscheck engine to monitor a custom directory (`/monitored-lab/fim-test`) in real time using Linux's inotify subsystem. Triggered file creation, content modification, permission change, and deletion events and confirmed Wazuh generated distinct alerts for each event type. Demonstrates how Wazuh can serve as a tripwire against unauthorised changes to sensitive files and directories.

### 3. Log Analysis with Custom Decoder and Rule
Onboarded a fictional application log source (`corp-portal`) into Wazuh by writing a custom decoder and detection rule from scratch. Discovered that Wazuh's built-in `windows-date-format` decoder intercepts timestamp-prefixed log lines before custom decoders run, requiring the custom decoder to be structured as a child of `windows-date-format`. Also resolved a reserved field name conflict (`status` replaced with `login_status`). Validated the full pipeline using `wazuh-logtest` before injecting live log entries.

### 4. Malware Detection via EICAR Test File
Deployed EICAR test files on both the Linux host and Windows 11 VM and confirmed detection through each platform's antivirus engine. On Linux, discovered that `clamscan` produces output with no Wazuh decoder while `clamdscan` routes results through the `clamd` daemon log that Wazuh's built-in rules target. On Windows, discovered that the Windows Defender Operational event channel is absent from the default agent configuration and must be added manually. Both detections surfaced as distinct alerts in the Wazuh dashboard from their respective agents.

### 5. Active Response: Auto-Block IP
Configured Wazuh's Active Response system to execute `firewall-drop` automatically when rule 5712 fires, inserting a DROP rule into `iptables` to block the attacker's source IP. Simulated a second Hydra brute-force attack and confirmed the block was applied at the kernel network layer within seconds of the alert firing, with no manual intervention. Confirmed the automatic unblock after the 180 second timeout. Both the block (rule 651) and the unblock (rule 652) were recorded as auditable events in the dashboard, capturing the full containment lifecycle.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Wazuh 4.14.5 | SIEM: log ingestion, rule correlation, alerting, active response |
| Docker | Single-node Wazuh stack deployment |
| VirtualBox | Windows 11 Pro VM hosting the second monitored endpoint |
| Hydra | SSH brute-force simulation (Use Cases 1 and 5) |
| ClamAV (`clamdscan`) | Linux malware detection (Use Case 4) |
| Windows Defender | Windows malware detection (Use Case 4) |
| iptables | Firewall rule enforcement for Active Response (Use Case 5) |
| `wazuh-logtest` | Decoder and rule pipeline validation (Use Case 3) |

---

## Key Learnings

- `clamscan` and `clamdscan` are not interchangeable in a Wazuh context. Only `clamdscan` routes results through the `clamd` daemon log that Wazuh's built-in decoders target.
- The `windows-date-format` decoder intercepts any log line beginning with a timestamp, requiring custom decoders for timestamp-prefixed formats to be structured as child decoders.
- `status` is a reserved static field name in Wazuh and cannot be used in decoder `<order>` tags.
- The Windows Defender Operational event channel (`Microsoft-Windows-Windows Defender/Operational`) is not included in the default Windows agent configuration and must be added manually.
- Active Response records both the block (rule 651) and the unblock (rule 652) as discrete SIEM events, providing a complete auditable containment timeline.
- All decoder and rule files in a Docker-based Wazuh deployment must be created inside the manager container. No text editors are available in the container image; file creation relies on `cat` heredoc syntax.

---

## Repository Structure

```
wazuh-soc-lab/
├── README.md
├── setup/
│   └── wazuh-docker-setup.md
├── use-cases/
│   ├── ssh-brute-force.md
│   ├── file-integrity-monitoring.md
│   ├── log-analysis-custom-rule.md
│   ├── malware-detection-eicar.md
│   └── active-response-ip-block.md
└── screenshots/
    ├── use-case-1/
    ├── use-case-2/
    ├── use-case-3/
    ├── use-case-4/
    └── use-case-5/
```

---

## References

- [Wazuh official documentation](https://documentation.wazuh.com/current/index.html)
- [Wazuh Docker deployment guide](https://documentation.wazuh.com/current/deployment-options/docker/index.html)
- [Wazuh built-in ruleset](https://documentation.wazuh.com/current/user-manual/ruleset/index.html)
- [ClamAV documentation](https://docs.clamav.net/)
- [MITRE ATT&CK T1110: Brute Force](https://attack.mitre.org/techniques/T1110/)