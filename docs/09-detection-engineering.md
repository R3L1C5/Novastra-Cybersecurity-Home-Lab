# Detection Engineering

## 1. Overview

This document describes the detection engineering implemented for the Novastra Technologies Cybersecurity Home Lab.

The objective is to transform raw endpoint telemetry into actionable SOC alerts using Wazuh detection rules and event analysis.

The detection engineering work currently covers:

* SSH brute-force detection
* SSH authentication telemetry
* Sudo and privilege-related activity
* Suspicious command activity
* Event correlation
* Alert severity classification
* MITRE ATT&CK mapping where applicable
* SOC evidence collection and validation

The detection workflow is:

```text
                    ATTACKER
                 NT-KALI01
                192.168.10.20
                       |
                       | Controlled attack activity
                       v
                  NT-EMP01
                 192.168.10.30
                       |
                       | systemd-journald telemetry
                       v
                  Wazuh Agent
                       |
                       v
                  NT-SOC01
                 192.168.10.10
                       |
                       v
               Wazuh Detection Engine
                       |
          +------------+-------------+
          |            |             |
          v            v             v
       SSH Brute     Sudo       Suspicious
        Force       Activity     Command
          |            |             |
          +------------+-------------+
                       |
                       v
                 SOC Alerts
                       |
                       v
                 SOC Analyst
```

---

# 2. Detection Engineering Objectives

The detection engineering phase was designed to demonstrate the ability to move from raw security telemetry to actionable detections.

The primary objectives are:

1. Collect security-relevant endpoint telemetry.
2. Identify meaningful authentication and privilege-related events.
3. Distinguish individual events from suspicious patterns.
4. Develop custom Wazuh detection logic where appropriate.
5. Assign appropriate alert severity.
6. Map detections to MITRE ATT&CK techniques where applicable.
7. Validate detections using controlled activity.
8. Preserve screenshots and other evidence for portfolio documentation.
9. Establish a foundation for future network-based detections using Suricata.

---

# 3. Detection Architecture

The current endpoint detection architecture is:

```text
                  NT-KALI01
               192.168.10.20
                     |
                     | Controlled activity
                     v
                  NT-EMP01
               192.168.10.30
                     |
          +----------+----------+
          |                     |
          v                     v
   SSH authentication      Local commands /
       activity             privilege activity
          |                     |
          +----------+----------+
                     |
                     v
             systemd-journald
                     |
                     v
                Wazuh Agent
                     |
                     | TCP 1514
                     v
                NT-SOC01
             192.168.10.10
                     |
                     v
              Wazuh Manager
                     |
                     v
             Detection Engine
                     |
                     v
             Wazuh Indexer
                     |
                     v
             Wazuh Dashboard
                     |
                     v
                SOC Analyst
```

This architecture provides centralized visibility into security activity generated on NT-EMP01.

---

# 4. Data Source

The primary data source for the current endpoint detections is:

```text
NT-EMP01
192.168.10.30
```

The endpoint uses:

```text
systemd-journald
```

The Wazuh Agent collects relevant authentication and system activity and forwards the telemetry to NT-SOC01.

The telemetry pipeline is:

```text
Endpoint Activity
       |
       v
systemd-journald
       |
       v
Wazuh Agent
       |
       v
Wazuh Manager
       |
       v
Analysis Engine
       |
       v
Wazuh Alert
       |
       v
Wazuh Dashboard
```

This provides the SOC with centralized security-event visibility.

---

# 5. SSH Brute-Force Detection

## 5.1 Detection Objective

The first custom detection use case is repeated SSH authentication failure activity.

The objective is to detect a potential SSH brute-force attack rather than treating every failed authentication as an isolated event.

Detection requirements:

* Monitor SSH authentication failures.
* Identify repeated authentication failures.
* Correlate five failures within 60 seconds.
* Generate a higher-severity alert.
* Map the detection to MITRE ATT&CK.
* Preserve source IP and username information for investigation.

---

## 5.2 Underlying SSH Telemetry

The detection uses SSH authentication telemetry collected from NT-EMP01.

An example underlying event is:

```text
Failed password for employee from 192.168.10.20 port 33990 ssh2
```

The source system is:

```text
NT-KALI01
192.168.10.20
```

The monitored endpoint is:

```text
NT-EMP01
192.168.10.30
```

---

## 5.3 Base Wazuh Detection

Wazuh's built-in SSH authentication detection rule:

```text
Rule: 5760
Level: 5
Description: sshd: authentication failed.
```

provides the foundation for the custom correlation rule.

Individual failed authentication events therefore become inputs to the higher-level brute-force detection.

---

# 6. Custom SSH Brute-Force Rule

Custom rule `100100` was created for the Novastra SOC.

Repository location:

```text
detection-rules/wazuh/100100-ssh-bruteforce.xml
```

Production Wazuh location:

```text
/var/ossec/etc/rules/local_rules.xml
```

The rule is:

```xml
<group name="local,syslog,sshd,">

  <rule id="100100" level="10" frequency="5" timeframe="60">
    <if_matched_sid>5760</if_matched_sid>
    <description>Novastra SOC: Possible SSH brute-force attack detected - 5 failed authentication attempts within 60 seconds.</description>
    <mitre>
      <id>T1110</id>
    </mitre>
    <group>authentication_failed,brute_force,mitre_t1110,</group>
  </rule>

</group>
```

---

## 6.1 Rule Parameters

| Parameter    |                   Value | Purpose                       |
| ------------ | ----------------------: | ----------------------------- |
| Rule ID      |                `100100` | Novastra custom detection     |
| Level        |                    `10` | High-severity alert           |
| Frequency    |                     `5` | Five matching events          |
| Timeframe    |            `60 seconds` | Correlation window            |
| Parent rule  |                  `5760` | SSH authentication failure    |
| MITRE ATT&CK |                 `T1110` | Brute Force                   |
| Group        | `authentication_failed` | Authentication classification |
| Group        |           `brute_force` | Brute-force classification    |
| Group        |           `mitre_t1110` | MITRE classification          |

---

# 7. SSH Detection Validation

Before deployment, the Wazuh analysis engine configuration was validated using:

```bash
sudo /var/ossec/bin/wazuh-analysisd -t
```

The configuration completed without errors.

The Wazuh Manager was then restarted:

```bash
sudo systemctl restart wazuh-manager
```

Service status was verified:

```bash
sudo systemctl is-active wazuh-manager
```

Result:

```text
active
```

This confirmed that the custom detection rule was successfully loaded.

---

# 8. Controlled SSH Attack Simulation

The detection was tested using NT-KALI01 against NT-EMP01.

Kali generated repeated failed SSH authentication attempts:

```bash
for i in {1..5}; do
  ssh employee@192.168.10.30
done
```

An incorrect password was supplied for each attempt.

The resulting authentication events were collected by the Wazuh Agent and forwarded to NT-SOC01.

---

# 9. SSH Brute-Force Detection Result

Wazuh successfully correlated the authentication failures and generated custom rule `100100`.

Observed alert:

```text
Rule: 100100
Level: 10

Novastra SOC: Possible SSH brute-force attack detected -
5 failed authentication attempts within 60 seconds.

Src IP: 192.168.10.20
User: employee
```

The correlated activity included multiple failed authentication events:

```text
Failed password for employee from 192.168.10.20
Failed password for employee from 192.168.10.20
Failed password for employee from 192.168.10.20
Failed password for employee from 192.168.10.20
Failed password for employee from 192.168.10.20
```

This demonstrates event correlation in which multiple lower-level authentication events are converted into a single higher-severity SOC detection.

---

# 10. MITRE ATT&CK Mapping — SSH Brute Force

The SSH brute-force detection is mapped to:

```text
MITRE ATT&CK T1110 — Brute Force
```

The mapping is appropriate because the simulated activity consists of repeated authentication attempts against an SSH service.

The current detection identifies repeated authentication failures. Future detection engineering can distinguish more specific authentication attack patterns such as:

* Password spraying
* Credential stuffing
* Repeated brute-force attempts
* Distributed authentication attacks

---

# 11. Sudo and Privilege-Related Detection

Additional endpoint detection work was performed around sudo and privilege-related activity.

The purpose of this detection use case is to provide SOC visibility into commands executed through elevated privileges.

This is important because privilege-related activity can provide useful context during an investigation.

For example, an authentication sequence may contain:

```text
User authentication
       |
       v
Session established
       |
       v
sudo execution
       |
       v
Privileged command
```

The Wazuh Agent collects the underlying endpoint telemetry and makes the resulting events available to the SOC.

---

## 11.1 Detection Objective

The sudo detection use case is intended to identify privilege-related activity on NT-EMP01.

The SOC should be able to determine:

* Which user performed the action.
* When the action occurred.
* That sudo was used.
* What privileged activity was associated with the event.
* Which endpoint generated the event.

This provides important context for determining whether privileged activity is expected administrative behavior or potentially suspicious activity.

---

## 11.2 Observed Telemetry

The endpoint generated privilege-related telemetry such as:

```text
Successful sudo to ROOT executed.
```

Associated authentication/session activity can include:

```text
PAM: Login session opened.
PAM: Login session closed.
```

These events demonstrate that privilege-related activity is being collected by the Wazuh Agent and presented to the SOC.

---

## 11.3 Validation

The sudo detection was validated through controlled privilege-related activity on NT-EMP01.

The resulting Wazuh events were reviewed in the Wazuh Dashboard.

The validation demonstrated that:

```text
sudo activity
      |
      v
systemd-journald
      |
      v
Wazuh Agent
      |
      v
Wazuh Manager
      |
      v
Wazuh Dashboard
```

was operational.

---

## 11.4 Evidence

The sudo detection evidence is preserved in:

```text
screenshots/attacks/phase3-wazuh-sudo-detection P1.png
screenshots/attacks/phase3-wazuh-sudo-detection P2.png
screenshots/attacks/phase3-wazuh-sudo-detection P3.png
screenshots/attacks/phase3-wazuh-sudo-detection P4.png
```

These screenshots provide evidence of the observed sudo and privilege-related activity.

---

# 12. Suspicious Command Detection

A further detection use case was developed around suspicious command execution.

The purpose is to identify command activity that may warrant investigation rather than treating all command execution as malicious.

Command-line telemetry is particularly valuable in endpoint detection because it can provide context about what an attacker or user attempted to execute after obtaining access.

---

## 12.1 Detection Objective

The suspicious-command detection aims to provide visibility into potentially unusual command execution on NT-EMP01.

The detection workflow is:

```text
Command execution
       |
       v
systemd-journald / endpoint telemetry
       |
       v
Wazuh Agent
       |
       v
Wazuh Manager
       |
       v
Detection logic
       |
       v
SOC alert / event
       |
       v
Analyst investigation
```

The detection should be treated as an investigation signal rather than automatically assuming that every matching command represents malicious activity.

Context such as:

* User
* Parent process
* Source of execution
* Time
* Command arguments
* Related authentication activity
* Other alerts

should be considered during investigation.

---

## 12.2 Validation

The suspicious-command detection was validated using controlled command activity within the laboratory environment.

The resulting event was observed through the Wazuh monitoring pipeline.

The validation demonstrated that endpoint command activity could be collected and surfaced to the SOC for investigation.

---

## 12.3 Evidence

The suspicious-command detection evidence is preserved in:

```text
screenshots/attacks/phase3-wazuh-suspicious-command.png
```

This screenshot provides portfolio evidence of the suspicious-command monitoring workflow.

---

# 13. Detection Correlation Model

The current detection engineering model can be represented as:

```text
                    Endpoint Activity
                           |
          +----------------+----------------+
          |                |                |
          v                v                v
     SSH Failure       sudo Activity    Command Activity
          |                |                |
          v                v                v
       Rule 5760        Wazuh Events     Wazuh Events
          |
          v
     Rule 100100
          |
          v
     Level 10 Alert
          |
          +----------------+----------------+
                           |
                           v
                    SOC Investigation
```

This model demonstrates the difference between:

1. Raw endpoint telemetry.
2. Individual detection events.
3. Correlated high-confidence detections.
4. Analyst investigation.

---

# 14. Detection Matrix

The current detection coverage is:

| Detection Use Case         | Data Source        | Detection Method               | Severity   | MITRE ATT&CK            | Status   |
| -------------------------- | ------------------ | ------------------------------ | ---------- | ----------------------- | -------- |
| SSH authentication failure | SSH/journald       | Wazuh Rule 5760                | Level 5    | Authentication activity | Complete |
| SSH brute force            | SSH/journald       | Custom Rule 100100             | Level 10   | T1110                   | Complete |
| Sudo / privilege activity  | journald           | Wazuh endpoint telemetry       | Observed   | To be expanded          | Complete |
| Suspicious command         | Endpoint telemetry | Wazuh detection/event analysis | Observed   | To be expanded          | Complete |
| File integrity change      | Wazuh Syscheck     | Wazuh FIM                      | Levels 5/7 | To be expanded          | Complete |

The severity and MITRE classifications for future custom rules should be assigned only after the underlying telemetry and detection logic have been formally validated.

---

# 15. Evidence Repository

The detection-engineering evidence is maintained under:

```text
screenshots/attacks/
```

Current evidence includes:

```text
phase3-ssh-bruteforce-wazuh-detection.png

phase3-ssh-bruteforce P1.png
phase3-ssh-bruteforce P2.png

phase3-wazuh-ssh-bruteforce P1.png
phase3-wazuh-ssh-bruteforce P2.png

phase3-wazuh-sudo-detection P1.png
phase3-wazuh-sudo-detection P2.png
phase3-wazuh-sudo-detection P3.png
phase3-wazuh-sudo-detection P4.png

phase3-wazuh-suspicious-command.png
```

These screenshots provide visual evidence of the detection and validation activities performed during the phase.

---

# 16. Detection Rule Repository

Custom Wazuh detection rules are maintained separately from the production Wazuh configuration so that they can be version-controlled and included in the project repository.

Current repository structure:

```text
detection-rules/
└── wazuh/
    └── 100100-ssh-bruteforce.xml
```

The production copy of the SSH brute-force rule is loaded into:

```text
/var/ossec/etc/rules/local_rules.xml
```

Future custom rules should follow the same repository structure.

---

# 17. Validation Summary

| Validation                       | Result |
| -------------------------------- | ------ |
| SSH server available on NT-EMP01 | PASS   |
| Kali → NT-EMP01 connectivity     | PASS   |
| SSH authentication telemetry     | PASS   |
| Wazuh Agent collection           | PASS   |
| Built-in Rule 5760               | PASS   |
| Custom Rule 100100               | PASS   |
| Five-event correlation           | PASS   |
| 60-second timeframe              | PASS   |
| Level 10 SSH brute-force alert   | PASS   |
| MITRE T1110 mapping              | PASS   |
| Sudo telemetry                   | PASS   |
| Sudo detection evidence          | PASS   |
| Suspicious-command telemetry     | PASS   |
| Suspicious-command evidence      | PASS   |

---

# 18. Current Detection Coverage

The Novastra SOC currently demonstrates visibility across several endpoint-security categories:

```text
                    ENDPOINT SECURITY
                           |
          +----------------+----------------+
          |                |                |
          v                v                v
   Authentication      Privilege         Command
      Activity          Activity         Activity
          |                |                |
          v                v                v
    SSH Failures          sudo       Suspicious Commands
          |
          v
    Brute-Force
    Correlation
          |
          v
     Rule 100100
          |
          v
      Level 10
          |
          v
   MITRE T1110
```

Combined with the previously implemented File Integrity Monitoring capability, the SOC now has multiple endpoint telemetry sources suitable for investigation.

---

# 19. Detection Engineering Methodology

The Novastra SOC detection-development process follows:

```text
Attack / User Activity
        |
        v
Telemetry Generation
        |
        v
Telemetry Collection
        |
        v
Event Analysis
        |
        v
Detection Logic
        |
        v
Wazuh Rule / Detection
        |
        v
Alert Validation
        |
        v
Severity Assignment
        |
        v
MITRE ATT&CK Mapping
        |
        v
Evidence Collection
        |
        v
SOC Investigation
```

This methodology ensures that detections are supported by observed telemetry and controlled validation rather than being created solely as theoretical rules.

---

# 20. Future Detection Engineering

Additional detections will be developed during subsequent phases of the Novastra SOC project.

Planned areas include:

* Network reconnaissance detection.
* Nmap scanning detection.
* Web attack detection.
* Authentication anomaly detection.
* Suricata-based network detections.
* Endpoint suspicious-command detections.
* File-integrity-based detections.
* Vulnerability-related detections.
* Cross-source alert correlation.
* Incident investigation.

The next major detection capability is network-based detection using Suricata.

The intended architecture is:

```text
                 NT-KALI01
              192.168.10.20
                     |
              Attack Traffic
                     |
                     v
                  NT-FW01
                 OPNsense
                     |
                  Suricata
                     |
              Network Alerts
                     |
                     v
                 NT-SOC01
                Wazuh SOC
                     |
                     v
              Wazuh Dashboard
                     |
                     v
                SOC Analyst
```

This will extend the existing endpoint-focused detection capability into network security monitoring.

---

# 21. Phase Status

## Completed Detection Engineering

The following detection capabilities have been validated:

* SSH authentication monitoring
* SSH brute-force correlation
* Custom Wazuh Rule `100100`
* Level 10 SSH brute-force alert
* MITRE ATT&CK T1110 mapping
* Sudo/privilege activity monitoring
* Suspicious-command monitoring
* Detection evidence collection
* Endpoint telemetry analysis

## Current Status

**Endpoint Detection Engineering: COMPLETE**

The Novastra SOC has successfully demonstrated the ability to collect endpoint telemetry, identify security-relevant activity, correlate repeated authentication failures, generate custom Wazuh alerts, map detections to MITRE ATT&CK, and preserve investigation evidence.

The next stage is:

**Network Intrusion Detection with Suricata**

```
```
