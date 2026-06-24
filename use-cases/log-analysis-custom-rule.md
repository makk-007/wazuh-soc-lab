# Use Case 3: Log Analysis with a Custom Decoder and Rule

**Date:** June 24, 2026
**Platform:** Wazuh 4.14.5 (single-node Docker deployment)
**Host:** Ubuntu 24.04 LTS (ASUS ROG Zephyrus G15)
**Monitored Agent:** Linux host agent (`Mawuse-ROG-Zephyrus-G15`)

---

## Objective

Onboard a fictional application log source into Wazuh by writing a custom decoder and detection rule from scratch. Inject log entries representing failed and successful login attempts and confirm that Wazuh fires alerts only on the failure pattern. The goal is to understand how Wazuh's two-stage log processing pipeline works and how a SOC engineer would extend the ruleset to cover a new application.

---

## Concepts Covered

| Concept | Description |
|---------|-------------|
| Decoder | A Wazuh component that parses a raw log line and extracts named fields such as username, source IP, and action for use by downstream rules |
| Parent decoder | A decoder that performs an initial match on a log line. Child decoders inherit its context and apply additional parsing on top of it |
| Rule | A Wazuh component that evaluates decoded fields and fires an alert when defined conditions are met |
| `<prematch>` | A lightweight string check used to quickly determine whether a decoder applies to a given log line before the more expensive regex runs |
| `<order>` | Maps each regex capture group to a named field in the sequence they appear. Field names here must not conflict with Wazuh's reserved static field names |
| Custom rule ID range | Wazuh reserves rule IDs below 100000 for its built-in ruleset. Custom rules must use IDs of 100000 or higher to avoid conflicts |
| MITRE ATT&CK mapping | Tagging a rule with a MITRE technique ID enriches the alert with standardised threat intelligence context usable across security tools |
| `wazuh-logtest` | A built-in Wazuh utility for testing log lines against the live decoder and rule pipeline without injecting entries into a real log file |
| `<localfile>` | An agent configuration block that tells the Wazuh agent which log files on the host to monitor and ship to the manager |

---

## Environment

```
Host Machine (Ubuntu 24.04 LTS)
├── Wazuh Agent (Linux) reporting to manager
├── /var/log/corp-portal/app.log    (custom log file monitored by agent)
└── Docker
    └── single-node-wazuh.manager-1
        ├── /var/ossec/etc/decoders/corp-portal-decoder.xml
        └── /var/ossec/etc/rules/corp-portal-rules.xml
```

The decoder and rule files reside inside the manager Docker container, not on the host filesystem. All file creation inside the container was performed using `cat` heredoc syntax since no text editors are available in the container image.

---

## Approach

### 1. Custom Log File Creation

A log file was created on the host for the fictional `corp-portal` application:

```bash
sudo mkdir -p /var/log/corp-portal
sudo touch /var/log/corp-portal/app.log
sudo chmod 644 /var/log/corp-portal/app.log
```

The target log format designed for this exercise:

```
2026-06-24 09:15:32 corp-portal ERROR user=jdoe action=login status=failed src_ip=10.0.0.55
```

### 2. Agent Configuration Update

The agent's `ossec.conf` on the host was updated to monitor the new log file:

```bash
sudo nvim /var/ossec/etc/ossec.conf
```

Added inside `<ossec_config>`:

```xml
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/corp-portal/app.log</location>
</localfile>
```

### 3. Container Access

All decoder and rule files were created inside the manager container:

```bash
docker exec -it single-node-wazuh.manager-1 /bin/bash
```

### 4. Custom Decoder

The initial decoder used a standalone structure with `<prematch>` targeting the `corp-portal` string. During testing with `wazuh-logtest`, the decoder failed to match because Wazuh's built-in `windows-date-format` decoder was intercepting the log line first due to the timestamp at the start (`2026-06-24 09:15:32`). The fix was restructuring the decoder as a child of `windows-date-format`, which handles the timestamp and passes the remainder of the line to the child for further parsing:

```bash
cat > /var/ossec/etc/decoders/corp-portal-decoder.xml << 'EOF'
<decoder name="corp-portal">
  <parent>windows-date-format</parent>
  <prematch>corp-portal</prematch>
  <regex>corp-portal \S+ user=(\S+) action=(\S+) status=(\S+) src_ip=(\S+)</regex>
  <order>dstuser, action, login_status, src_ip</order>
</decoder>
EOF
```

**Key decisions in this decoder:**

| Element | Decision | Reason |
|---------|----------|--------|
| `<parent>windows-date-format</parent>` | Added | The built-in `windows-date-format` decoder matched the timestamp at the start of the log line first. Making `corp-portal` a child of it ensures the log line reaches this decoder after the timestamp is consumed |
| `<order>` uses `dstuser` | `user` replaced with `dstuser` | During `wazuh-logtest` the extracted field for the username was reported as `dstuser`, which is the standard Wazuh field name for destination username. Using a custom name here would have broken the dynamic reference in the rule description |
| `<order>` uses `login_status` | `status` replaced with `login_status` | `status` is a reserved static field name in Wazuh. Using it in `<order>` caused a rule loading error: `Field 'status' is static` |

### 5. Custom Rule

```bash
cat > /var/ossec/etc/rules/corp-portal-rules.xml << 'EOF'
<group name="corp-portal,authentication_failed,">

  <rule id="100005" level="6">
    <decoded_as>windows-date-format</decoded_as>
    <field name="login_status">failed</field>
    <description>Corp Portal: Failed login attempt by user $(dstuser) from $(src_ip)</description>
    <mitre>
      <id>T1110</id>
    </mitre>
  </rule>

</group>
EOF
```

**Key decisions in this rule:**

| Element | Decision | Reason |
|---------|----------|--------|
| `<decoded_as>windows-date-format</decoded_as>` | Changed from `corp-portal` | Because the parent decoder processes the log line, Wazuh reports the event as decoded by `windows-date-format`. Using `corp-portal` here caused the rule to never fire |
| `$(dstuser)` | Changed from `$(user)` | Matched the actual extracted field name reported by `wazuh-logtest` |
| `rule id="100005"` | Changed from `100001` | Adjusted to avoid any potential conflict with other custom rules |
| `level="6"` | Retained | Level 6 represents a medium-severity security event on Wazuh's 0-15 scale |

### 6. Manager Restart and Logtest Validation

The manager was restarted inside the container to load the new decoder and rule:

```bash
/var/ossec/bin/wazuh-control restart
```

`wazuh-logtest` was then used to validate the pipeline before injecting real log entries:

```bash
/var/ossec/bin/wazuh-logtest
```

Test line used:

```
2026-06-24 09:15:32 corp-portal ERROR user=jdoe action=login status=failed src_ip=10.0.0.55
```

Output confirmed:
- Decoder: `windows-date-format` with child `corp-portal`
- Fields extracted: `dstuser=jdoe`, `action=login`, `login_status=failed`, `src_ip=10.0.0.55`
- Rule ID `100005` fired with the expected description

### 7. Agent Restart

After exiting the container, the Wazuh agent was restarted on the host to apply the `<localfile>` configuration change:

```bash
sudo systemctl restart wazuh-agent
sudo systemctl status wazuh-agent
```

### 8. Log Entry Injection

Three log entries were injected to test both the positive and negative cases:

```bash
# Should trigger rule 100005
echo "2026-06-24 09:15:32 corp-portal ERROR user=jdoe action=login status=failed src_ip=10.0.0.55" | sudo tee -a /var/log/corp-portal/app.log

# Should trigger rule 100005
echo "2026-06-24 09:16:10 corp-portal ERROR user=asmith action=login status=failed src_ip=10.0.0.87" | sudo tee -a /var/log/corp-portal/app.log

# Should produce no alert (status=success does not match the rule condition)
echo "2026-06-24 09:17:45 corp-portal INFO user=makk007 action=login status=success src_ip=192.168.1.10" | sudo tee -a /var/log/corp-portal/app.log
```

### 9. Dashboard Verification

Alerts were located under **Threat Intelligence > Threat Hunting > Events** tab filtered by the Linux host agent:

- Rule `100005` fired twice, once for `jdoe` and once for `asmith`
- No alert was generated for the `makk007` success entry, confirming the rule condition correctly filters on `login_status=failed` only
- Expanded alert detail confirmed the dynamic description rendered correctly with the decoded username and source IP values

---

## Screenshots

| File | Description |
|------|-------------|
| `01-corp-portal-decoder.png` | Terminal output of `cat` confirming the decoder file contents inside the container |
| `02-corp-portal-rule.png` | Terminal output of `cat` confirming the rule file contents inside the container |
| `03-logtest-output.png` | Wazuh-logtest output confirming decoder match, field extraction, and rule 100005 firing |
| `04-log-injection-terminal.png` | Terminal showing the three echo commands injecting log entries into `app.log` |
| `05-dashboard-alerts.png` | Dashboard showing rule 100005 alerts for jdoe and asmith with no alert for the success entry |
| `06-expanded-alert-detail.png` | Expanded alert showing the dynamic description and all decoded fields |

---

## Key Takeaway

Writing a custom decoder and rule exposes the full mechanics of how Wazuh processes logs. The decoder extracts meaning from raw text and the rule acts on that meaning, and the two components are loosely coupled enough that either can be modified independently. The most instructive moment in this exercise was discovering that the built-in `windows-date-format` decoder was intercepting the log line before the custom decoder could run. Resolving this by establishing a parent-child decoder relationship is the correct Wazuh pattern for any log format that begins with a timestamp, and understanding it means the same technique can be applied to any future custom log source onboarding task.

---

## Reflection

Five deviations from the initial plan were encountered and resolved during this use case. The most significant was the `windows-date-format` decoder conflict, which required restructuring the custom decoder as a child rather than a standalone component. A secondary conflict arose from `status` being a reserved static field name in Wazuh, resolved by renaming it to `login_status` in both the decoder and rule. The `<decoded_as>` value and dynamic field reference in the rule description also required correction based on actual `wazuh-logtest` output rather than assumed values. All decoder and rule files were created inside the Docker manager container using `cat` heredoc syntax due to the absence of text editors in the container image. Together these deviations produced a more complete understanding of Wazuh's internals than a clean run would have.

---

## References

- [Wazuh custom decoder documentation](https://documentation.wazuh.com/current/user-manual/ruleset/custom-decoder.html)
- [Wazuh custom rules documentation](https://documentation.wazuh.com/current/user-manual/ruleset/custom-rules.html)
- [Wazuh decoder syntax reference](https://documentation.wazuh.com/current/user-manual/ruleset/ruleset-xml-syntax/decoders.html)
- [MITRE ATT&CK T1110: Brute Force](https://attack.mitre.org/techniques/T1110/)
- [Wazuh logtest documentation](https://documentation.wazuh.com/current/user-manual/ruleset/testing.html)