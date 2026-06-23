# Use Case 2: File Integrity Monitoring (FIM)

**Date:** June 23, 2026
**Platform:** Wazuh 4.14.5 (single-node Docker deployment)
**Host:** Ubuntu 24.04 LTS (ASUS ROG Zephyrus G15)
**Monitored Agent:** Linux host agent (`Mawuse-ROG-Zephyrus-G15`)

---

## Objective

Configure Wazuh's File Integrity Monitoring engine to watch a custom directory in real time, then trigger a series of file system events (creation, modification, permission change, deletion) and confirm that Wazuh detects and alerts on each one. The goal is to understand how a SIEM can serve as a tripwire against unauthorized or unexpected changes to files on a monitored system.

---

## Concepts Covered

| Concept | Description |
|---------|-------------|
| File Integrity Monitoring (FIM) | A security control that detects changes to files and directories by tracking attributes such as hash, size, permissions, and ownership |
| syscheck | Wazuh's built-in FIM engine, configured inside the agent's `ossec.conf` file |
| inotify | A Linux kernel subsystem that provides real-time filesystem event notifications, used by Wazuh's syscheck for real-time monitoring |
| Cryptographic hash | A fixed-length fingerprint of a file's contents. Any change to the file, however small, produces a completely different hash, making tampering detectable |
| Baseline scan | The initial scan syscheck performs when the agent starts, establishing a reference state for all monitored files against which future changes are compared |
| `report_changes` | A syscheck attribute that captures the actual content difference in a modified text file, not just that a change occurred |

---

## Environment

```
Host Machine (Ubuntu 24.04 LTS)
├── Wazuh Agent (Linux) reporting to manager
├── /monitored-lab/fim-test/    (custom directory added to syscheck)
└── Test file: sensitive-config.txt  (created, modified, permission-changed, deleted)
```

---

## Approach

### 1. Test Directory Creation

A dedicated directory was created to serve as the FIM target. Using a purpose-built directory rather than a sensitive system path keeps the lab contained and avoids interfering with real system files:

```bash
sudo mkdir -p /monitored-lab/fim-test
```

Verified with:

```bash
ls -la /monitored-lab/
```

### 2. Wazuh Agent Configuration

The agent configuration file was opened:

```bash
sudo nvim /var/ossec/etc/ossec.conf
```

Inside the `<syscheck>` block, an existing `<directories>` section was already present grouping other monitored paths. The custom directory was added there, keeping the configuration consistent with Wazuh's expected structure:

```xml
<directories realtime="yes" check_all="yes" report_changes="yes">/monitored-lab/fim-test</directories>
```

**Attribute breakdown:**

| Attribute | Meaning |
|-----------|---------|
| `realtime="yes"` | Monitors the directory continuously via Linux's inotify subsystem rather than waiting for the scheduled scan interval |
| `check_all="yes"` | Tracks changes to file permissions, ownership, size, timestamps, and cryptographic hash |
| `report_changes="yes"` | Captures and reports the actual content difference for modified text files |

### 3. Agent Restart

The agent was restarted to apply the configuration change:

```bash
sudo systemctl restart wazuh-agent
```

Verified the restart was clean:

```bash
sudo systemctl status wazuh-agent
```

Output confirmed `active (running)`. A 30-second wait was observed before making file changes, allowing syscheck to complete its initial baseline scan.

### 4. FIM Event Triggers

The following sequence of file system actions was performed inside the monitored directory, each intended to generate a distinct Wazuh alert:

```bash
# File creation
sudo touch /monitored-lab/fim-test/sensitive-config.txt

# Write initial content
echo "initial configuration value" | sudo tee /monitored-lab/fim-test/sensitive-config.txt

# Modify content
echo "tampered configuration value" | sudo tee /monitored-lab/fim-test/sensitive-config.txt

# Change permissions
sudo chmod 777 /monitored-lab/fim-test/sensitive-config.txt

# Delete the file
sudo rm /monitored-lab/fim-test/sensitive-config.txt
```

### 5. Wazuh Dashboard Analysis

Alerts were located under **Threat Intelligence > Threat Hunting > Events** tab, filtered by the Linux host agent. The following rule IDs were triggered:

| Rule ID | Description | Triggered By |
|---------|-------------|--------------|
| `554` | File added to the system | Creation of `sensitive-config.txt` |
| `550` | Integrity checksum changed | Content modification and permission change |
| `553` | File deleted from the system | Deletion of `sensitive-config.txt` |

Each alert was expanded to confirm the affected file path, the nature of the change detected, and the timestamp of detection.

---

## Screenshots

| File | Description |
|------|-------------|
| `01-ossec-conf-syscheck.png` | The syscheck block in `ossec.conf` showing the added directory entry |
| `02-fim-events-dashboard-part-01.png` | Events tab showing the first portion of FIM alerts |
| `03-fim-events-dashboard-part-02.png` | Events tab showing the remaining FIM alerts |
| `04-expanded-file-added-alert-part-01.png` | Expanded detail of the rule 554 file creation alert, first portion |
| `05-expanded-file-added-alert-part-01.png` | Expanded detail of the rule 554 file creation alert,second portion |
| `06-expanded-file-modified-alert-part-01.png` | Expanded detail of the rule 550 modification alert, first portion |
| `07-expanded-file-modified-alert-part-02.png` | Expanded detail of the rule 550 modification alert, second portion |
| `08-expanded-file-deleted-alert-part-01.png` | Expanded detail of the rule 553 file deletion alert, first portion |
| `09-expanded-file-deleted-alert-part-01.png` | Expanded detail of the rule 553 file deletion alert, first portion |

---

## Key Takeaway

FIM turns Wazuh into a tripwire. Without it, an attacker who modifies a configuration file or drops a malicious script onto a system may go unnoticed indefinitely. With syscheck enabled, every file system change inside a monitored directory generates an alert with a precise timestamp, the exact file affected, and a record of what changed, including content diffs for text files when `report_changes` is enabled. The separation of rule IDs by event type (554 for addition, 550 for modification, 553 for deletion) means a SOC analyst can immediately understand the nature of the change without reading raw logs.

---

## Reflection

The `ossec.conf` file already contained a `<directories>` section inside the syscheck block grouping other monitored paths. Rather than adding a standalone entry directly beneath the `<frequency>` line as initially planned, the custom directory entry was placed inside the existing section to maintain structural consistency with the rest of the configuration. The alert IDs and behavior matched expectations exactly, with all three event types (creation, modification, deletion) producing distinct, correctly labelled alerts in the dashboard.

---

## References

- [Wazuh FIM documentation](https://documentation.wazuh.com/current/user-manual/capabilities/file-integrity/index.html)
- [Wazuh syscheck configuration reference](https://documentation.wazuh.com/current/user-manual/reference/ossec-conf/syscheck.html)
- [Linux inotify man page](https://man7.org/linux/man-pages/man7/inotify.7.html)