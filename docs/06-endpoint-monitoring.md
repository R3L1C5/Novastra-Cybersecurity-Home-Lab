# 6. Endpoint Monitoring

## 6.1 Overview

NT-EMP01 represents a monitored employee workstation within the Novastra Technologies Cybersecurity Home Lab.

The endpoint runs:

**ubuntu desktop 24.04.4**

IP address:

```text
192.168.10.30
````

The endpoint is monitored centrally by the Wazuh SOC running on NT-SOC01.

---

## 6.2 Endpoint Architecture

```text
                 NT-EMP01
            Employee Endpoint
             192.168.10.30
                    |
                    |
              Wazuh Agent
                    |
                    | Security telemetry
                    v
                 NT-SOC01
              192.168.10.10
                    |
                    v
              Wazuh Manager
                    |
                    v
               Wazuh Indexer
                    |
                    v
             Wazuh Dashboard
```

This architecture provides centralized visibility into endpoint activity without requiring the SOC analyst to work directly on the employee workstation.

---

## 6.3 Wazuh Agent

The Wazuh Agent was installed on NT-EMP01 and configured to communicate with the central Wazuh Manager.

Manager address:

```text
192.168.10.10
```

The registered agent is:

| Property   | Value         |
| ---------- | ------------- |
| Agent ID   | 001           |
| Agent Name | NT-EMP01      |
| IP Address | 192.168.10.30 |
| Manager    | 192.168.10.10 |
| Status     | Active        |

---

## 6.4 Agent Service

The Wazuh Agent was configured to start automatically when NT-EMP01 boots.

Validation:

```bash
sudo systemctl is-active wazuh-agent
sudo systemctl is-enabled wazuh-agent
```

Expected result:

```text
active
enabled
```

This ensures endpoint monitoring resumes automatically after a system restart.

---

## 6.5 Endpoint Telemetry

The Wazuh Agent collects security-relevant activity from the endpoint and forwards it to the central SOC.

Observed telemetry included events such as:

```text
Successful sudo to ROOT executed.
PAM: Login session opened.
PAM: Login session closed.
Wazuh agent started.
Wazuh agent stopped.
```

These events demonstrate successful endpoint-to-SOC telemetry flow.

---

## 6.6 systemd-journald Collection

ubuntu desktopSecurity OS uses:

```text
systemd-journald
```

for system logging.

Wazuh Logcollector was configured to monitor the system journal.

Configuration validation:

```bash
sudo /var/ossec/bin/wazuh-logcollector -t
```

The validation confirmed the journald configuration was recognized.

The Wazuh Agent log subsequently reported:

```text
Monitoring journal entries.
```

This confirmed that the endpoint's system journal was actively being monitored.

---

## 6.7 Endpoint Monitoring Validation

Endpoint monitoring was validated by checking:

### Agent service

```bash
sudo systemctl is-active wazuh-agent
```

Result:

```text
active
```

### Agent startup

```bash
sudo systemctl is-enabled wazuh-agent
```

Result:

```text
enabled
```

### Journald monitoring

```bash
sudo grep -i "Monitoring journal entries" \
/var/ossec/logs/ossec.log | tail -5
```

The log contained:

```text
Monitoring journal entries.
```

### FIM subsystem

The endpoint also reported successful initialization of the File Integrity Monitoring subsystem, including:

```text
FIM sync module started.
Real-time file integrity monitoring started.
```

FIM is documented separately in:

```text
07-fim.md
```

---

## 6.8 Endpoint Availability

The endpoint was rebooted during Phase validation to confirm that required services could recover automatically.

After startup:

```text
Wazuh Agent       ACTIVE
Journald          AVAILABLE
FIM               INITIALIZED
```

This confirms that NT-EMP01 can function as a persistent monitored endpoint rather than requiring manual monitoring startup.

---

## 6.9 Monitoring Model

The endpoint monitoring model is:

```text
       NT-EMP01
           |
           | System activity
           v
   systemd-journald
           |
           v
      Wazuh Agent
           |
           | TCP 1514
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

This provides a basic endpoint detection and monitoring pipeline.

---

## 6.10 Security Benefits

Centralized endpoint monitoring provides the SOC with visibility into:

* Authentication activity
* Privilege-related activity
* System events
* Agent status
* File integrity changes
* Other security-relevant endpoint telemetry

Centralization also allows future security events to be correlated with activity from other components of the laboratory.

---

## 6.11 Phase 1 Outcome

NT-EMP01 was successfully integrated into the centralized Wazuh SOC.

The completed endpoint monitoring implementation provides:

* Wazuh Agent deployment
* Automatic agent startup
* Centralized telemetry
* systemd-journald collection
* Endpoint security event visibility
* FIM subsystem integration
* Persistent monitoring after reboot

The endpoint is ready to serve as the monitored workstation for subsequent cybersecurity Phases.