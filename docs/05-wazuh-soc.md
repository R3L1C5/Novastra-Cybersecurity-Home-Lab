# 5. Wazuh Security Operations Center

## 5.1 Overview

NT-SOC01 provides the centralized Security Operations Center (SOC) for the Novastra Technologies Cybersecurity Home Lab.

The system runs:

**Ubuntu Server 24.04.4 LTS**

IP address:

```text
192.168.10.10
````

Wazuh was deployed as an all-in-one security monitoring platform.

The deployment provides centralized:

* Security event collection
* Endpoint monitoring
* Log analysis
* File Integrity Monitoring
* Security alerting
* Security investigation
* SOC visibility

---

## 5.2 SOC Architecture

The Wazuh deployment consists of:

```text
                    NT-SOC01
                 192.168.10.10
                       |
        +--------------+--------------+
        |              |              |
        v              v              v
 Wazuh Indexer    Wazuh Manager   Wazuh Dashboard
    :9200        :1514 / :1515        :443
                       |
                       v
                 Wazuh API
                    :55000
```

The deployment consolidates the primary Wazuh components onto a single SOC server, making it suitable for a resource-conscious home laboratory.

---

## 5.3 Wazuh Components

### Wazuh Indexer

The Wazuh Indexer stores and indexes security event data used by the SOC.

Primary port:

```text
9200/TCP
```

---

### Wazuh Manager

The Wazuh Manager receives and processes telemetry from Wazuh Agents.

Primary ports:

```text
1514/TCP
1515/TCP
```

Port `1514` is used for agent communication.

Port `1515` is used for agent enrollment.

---

### Wazuh Dashboard

The Wazuh Dashboard provides the analyst-facing interface for:

* Security alerts
* Endpoint information
* Event analysis
* File Integrity Monitoring
* Security monitoring

Dashboard:

```text
https://192.168.10.10
```

Primary port:

```text
443/TCP
```

---

### Wazuh API

The Wazuh API provides programmatic access to the Wazuh platform.

Primary port:

```text
55000/TCP
```

---

## 5.4 Wazuh Installation

Wazuh was installed using the official assisted installation method.

The installation completed successfully with:

```text
INFO: Installation finished.
```

The all-in-one deployment was selected to provide the complete SOC platform on NT-SOC01.

---

## 5.5 Service Configuration

The primary Wazuh services are:

```text
wazuh-indexer
wazuh-manager
wazuh-dashboard
```

The services were configured to start automatically when NT-SOC01 boots.

Configuration:

```bash
sudo systemctl enable wazuh-indexer wazuh-manager wazuh-dashboard
```

Validation:

```bash
systemctl is-enabled wazuh-indexer
systemctl is-enabled wazuh-manager
systemctl is-enabled wazuh-dashboard
```

Expected result:

```text
enabled
enabled
enabled
```

---

## 5.6 Service Validation

The running state of the Wazuh services was validated using:

```bash
systemctl is-active wazuh-indexer
systemctl is-active wazuh-manager
systemctl is-active wazuh-dashboard
```

The completed environment returned:

```text
active
active
active
```

The required listening services were also validated.

```bash
sudo ss -lntp | grep -E ':1514|:1515|:443|:55000|:9200'
```

The Wazuh components were confirmed to be listening on their required ports.

---

## 5.7 Wazuh Agent Architecture

NT-EMP01 is enrolled as a Wazuh Agent.

```text
NT-EMP01
192.168.10.30
      |
      | Wazuh telemetry
      |
      v
NT-SOC01
192.168.10.10
      |
      v
Wazuh Manager
      |
      v
Indexer
      |
      v
Dashboard
```

The agent was registered with:

```text
Agent ID: 001
Agent Name: NT-EMP01
IP Address: 192.168.10.30
```

The agent was verified as active.

---

## 5.8 Endpoint Telemetry

The Wazuh SOC successfully received telemetry from NT-EMP01.

Examples of observed events included:

```text
Successful sudo to ROOT executed.
PAM: Login session opened.
PAM: Login session closed.
Wazuh agent started.
Wazuh agent stopped.
```

This demonstrated that the SOC was receiving and processing endpoint security telemetry.

---

## 5.9 Journald Integration

NT-EMP01 uses:

```text
systemd-journald
```

rather than relying on traditional `/var/log/syslog` collection.

Wazuh Logcollector was configured to monitor the system journal.

Configuration validation was performed using:

```bash
sudo /var/ossec/bin/wazuh-logcollector -t
```

The configuration validation reported:

```text
INFO: Merge journald log configurations
```

After restarting the Wazuh Agent, the agent log confirmed:

```text
Monitoring journal entries.
```

This established journald as an active endpoint telemetry source.

---

## 5.10 SOC Dashboard

The Wazuh Dashboard provides the primary analyst interface.

The dashboard allows the analyst to:

* Monitor endpoint activity
* Search security events
* Review alerts
* Investigate suspicious activity
* Examine File Integrity Monitoring events
* Identify affected endpoints

Dashboard:

```text
https://192.168.10.10
```

The dashboard was successfully accessed and used to validate endpoint telemetry.

---

## 5.11 SOC Availability

The SOC environment was validated after system startup.

The following were confirmed:

```text
Wazuh Indexer       ACTIVE
Wazuh Manager       ACTIVE
Wazuh Dashboard     ACTIVE
Wazuh Agent         ACTIVE
```

Automatic startup was also confirmed for the primary Wazuh services.

This ensures the SOC is available after NT-SOC01 is rebooted.

---

## 5.12 Phase 1 Outcome

The Wazuh-based SOC was successfully deployed and validated.

The completed SOC provides:

* Centralized security monitoring
* Wazuh Indexer
* Wazuh Manager
* Wazuh Dashboard
* Wazuh API
* Endpoint telemetry
* Journald collection
* Automatic service startup
* File Integrity Monitoring integration
* A centralized platform for future detection and investigation

The SOC is now ready to support the subsequent cybersecurity Phases.
