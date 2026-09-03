# 7. File Integrity Monitoring

## 7.1 Overview

File Integrity Monitoring (FIM) was implemented on NT-EMP01 using the Wazuh Syscheck subsystem.

FIM provides the SOC with visibility into unauthorized or unexpected changes to monitored files and directories.

The implementation supports detection of:

- New files
- Modified files
- Deleted files
- Integrity checksum changes

---

## 7.2 Wazuh FIM Configuration

The Wazuh Agent on NT-EMP01 uses the Syscheck module for File Integrity Monitoring.

The configuration includes:

```xml
<syscheck>
    <disabled>no</disabled>

    <frequency>43200</frequency>

    <scan_on_start>yes</scan_on_start>

    <directories>/etc,/usr/bin,/usr/sbin</directories>
    <directories>/bin,/sbin,/boot</directories>

    <directories realtime="yes">/opt/novastra-fim</directories>
</syscheck>
````

FIM is enabled with:

```xml
<disabled>no</disabled>
```

---

## 7.3 Scheduled Monitoring

The configured scan frequency is:

```text
43200 seconds
```

which is equivalent to:

```text
12 hours
```

The agent log confirmed:

```text
File integrity monitoring scan frequency: 43200 seconds
```

FIM scans are also configured to begin when the Wazuh Agent starts:

```xml
<scan_on_start>yes</scan_on_start>
```

---

## 7.4 Real-Time Monitoring

A dedicated directory was configured for real-time FIM testing:

```text
/opt/novastra-fim
```

Configuration:

```xml
<directories realtime="yes">/opt/novastra-fim</directories>
```

This allows file changes within the directory to be detected without waiting for the next scheduled scan.

The Wazuh Agent confirmed:

```text
Real-time file integrity monitoring started.
```

---

## 7.5 Additional Configuration

The FIM configuration excludes selected file types that are not useful for the laboratory's monitoring objectives.

For example:

```xml
<ignore type="sregex">.log$|.swp$</ignore>
```

Private key contents are also protected from inclusion in FIM event differences:

```xml
<nodiff>/etc/ssl/private.key</nodiff>
```

These settings reduce unnecessary monitoring noise while protecting sensitive file contents.

---

## 7.6 FIM Validation

The Wazuh Agent logs confirmed successful initialization of the FIM subsystem.

Relevant events included:

```text
File integrity monitoring scan started.
File integrity monitoring scan ended.
FIM sync module started.
Real-time file integrity monitoring started.
```

This confirmed that FIM was operational on NT-EMP01.

---

## 7.7 Controlled FIM Test

A controlled file-integrity test was performed inside:

```text
/opt/novastra-fim
```

The test involved creating a file and subsequently modifying it.

The Wazuh Dashboard successfully detected the resulting changes.

### File Creation

The SOC generated:

```text
File added to the system.
```

Wazuh rule:

```text
554
```

Severity:

```text
Level 5
```

---

### File Modification

The SOC subsequently generated:

```text
Integrity checksum changed.
```

Wazuh rule:

```text
550
```

Severity:

```text
Level 7
```

Affected endpoint:

```text
NT-EMP01
```

---

## 7.8 Detection Flow

The completed FIM detection pipeline is:

```text
File created/modified
        |
        v
NT-EMP01
        |
        v
Wazuh Syscheck
        |
        v
Wazuh Agent
        |
        v
Wazuh Manager
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

This demonstrates an end-to-end endpoint integrity monitoring workflow.

---

## 7.9 Evidence

The final FIM detection evidence is stored in:

```text
screenshots/wazuh/FIM Test.png
```

The screenshot shows the Wazuh Dashboard detecting:

```text
NT-EMP01
File added to the system.
Level 5
Rule 554
```

and:

```text
NT-EMP01
Integrity checksum changed.
Level 7
Rule 550
```

This screenshot serves as the primary evidence for the FIM implementation.

---

## 7.10 Phase 1 Outcome

File Integrity Monitoring was successfully implemented and validated.

The completed implementation provides:

* Scheduled integrity scanning
* Real-time monitoring
* File creation detection
* File modification detection
* Wazuh rule-based alerting
* Centralized SOC visibility
* Endpoint-specific investigation capability

FIM is considered **complete for Phase 1** and does not require further testing unless a configuration change is introduced.