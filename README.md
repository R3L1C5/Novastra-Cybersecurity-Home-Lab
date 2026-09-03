# Novastra Technologies — Cybersecurity Home Lab

A VirtualBox-based cybersecurity home lab designed to demonstrate practical SOC, network security, endpoint monitoring, detection engineering, vulnerability management, and incident investigation workflows.

## Project Overview

The Novastra Technologies Cybersecurity Home Lab models a small segmented enterprise environment in which security controls are deployed, monitored, tested, and documented.

The lab combines:

- OPNsense firewall/router
- Kali Linux attack-simulation host
- Ubuntu employee endpoint
- Ubuntu Server SOC/Wazuh manager
- Suricata IDS telemetry
- Wazuh SIEM/XDR-style monitoring and correlation
- Custom detection rules
- Vulnerability-management workflow
- Incident-investigation workflow
- Packet-level and host-level evidence

The objective is not to reproduce a production enterprise network, but to demonstrate the security-engineering lifecycle from network architecture and hardening through detection, vulnerability assessment, and incident investigation.

---

## Architecture

### Logical Topology

```text
                         Internet / VirtualBox NAT
                                  |
                                  |
                           [ NT-FW01 ]
                         OPNsense 26.7
                          192.168.10.1
                                  |
                         LAN / em1
                                  |
                         Internal Network
                             "intnet"
                                  |
              +-------------------+-------------------+
              |                   |                   |
              |                   |                   |
        [ NT-KALI01 ]       [ NT-EMP01 ]       [ NT-SOC01 ]
        Kali Linux           Ubuntu Desktop      Ubuntu Server
        Attacker             Employee Endpoint   SOC / Wazuh
        192.168.10.20        192.168.10.30        Wazuh Manager
```

### Core Components

| Host | Operating System / Platform | Role | Address |
|---|---|---|---|
| NT-FW01 | OPNsense 26.7.3_8 | Firewall / Router | 192.168.10.1 |
| NT-KALI01 | Kali Linux 2026.2 | Attack Simulation | 192.168.10.20 |
| NT-EMP01 | Ubuntu 24.04.4 Desktop | Employee Endpoint | 192.168.10.30 |
| NT-SOC01 | Ubuntu Server 24.04.4 | SOC / Wazuh Manager | SOC management host |

> IP assignments documented here reflect the lab configuration used during validation. The SOC host is treated as the monitoring/management component even where packet-level testing focused on traffic between the attacker and employee endpoint.

---

## Security Monitoring Flow

```text
Attack / Network Activity
          |
          v
     OPNsense / LAN
          |
          v
       Suricata
          |
          | EVE JSON
          v
   Wazuh Monitoring
          |
          v
   Detection / Correlation
          |
          v
    SOC Investigation
          |
          v
 Assessment / Response / Documentation
```

The lab therefore demonstrates both network-level and endpoint-level telemetry.

---

# Documentation Chapters

## 01 — Lab Architecture

Defines the overall Novastra Technologies environment, virtualization model, network topology, host roles, addressing, and security boundaries.

**Primary outcome:** A reproducible understanding of how the lab is structured.

---

## 02 — Virtualization & Host Deployment

Documents the VirtualBox-based deployment of the security infrastructure and endpoint systems.

**Primary outcome:** Establishes the virtual infrastructure on which the security controls operate.

---

## 03 — Network Configuration

Documents interface configuration, addressing, internal networking, routing, and communication paths between the lab components.

**Primary outcome:** Provides the network foundation required for controlled security testing.

---

## 04 — Firewall & Gateway Security

Documents the OPNsense firewall/router configuration, rule structure, interface exposure, stateful filtering, and gateway security controls.

**Primary outcome:** Demonstrates enforcement of network security boundaries.

---

## 05 — Endpoint Security

Documents the employee endpoint security configuration and the security-relevant telemetry available from the Ubuntu workstation.

**Primary outcome:** Demonstrates host-level security visibility.

---

## 06 — Wazuh Deployment

Documents the Wazuh SOC architecture, manager configuration, endpoint-agent deployment, log collection, and security monitoring.

**Primary outcome:** Establishes centralized security monitoring and event analysis.

---

## 07 — Suricata IDS

Documents Suricata deployment, inspection configuration, custom detection signatures, EVE JSON telemetry, and packet-level detection.

The custom Suricata rules include laboratory signatures for:

- Kali → EMP01 TCP/3000 activity
- Kali → EMP01 TCP connection attempts
- ICMP traffic from Kali to EMP01
- HTTP GET activity associated with an exploit-style laboratory request

**Primary outcome:** Demonstrates network intrusion detection and custom signature engineering.

---

## 08 — SIEM & Detection Engineering

Documents the relationship between Suricata telemetry and Wazuh, including event parsing, custom Wazuh rules, alert levels, and SOC-oriented correlation.

Custom Wazuh correlation includes:

- Base Suricata alert handling
- Attacker-to-employee correlation
- Elevated severity for traffic targeting TCP/3000

**Primary outcome:** Demonstrates transformation of raw telemetry into actionable security detections.

---

## 09 — Security Validation & Attack Simulation

Documents controlled security-testing activity performed from the Kali attack-simulation host against the designated employee endpoint.

Validation focuses on:

- Network reachability
- TCP connection behavior
- ICMP activity
- Application-layer traffic
- Suricata detection
- Wazuh event visibility
- Evidence correlation

**Primary outcome:** Demonstrates that configured controls can observe and identify authorized laboratory activity.

---

## 10 — Security Monitoring & SOC Operations

Documents operational monitoring concepts, event review, alert interpretation, evidence handling, and the workflow used by the simulated SOC environment.

**Primary outcome:** Demonstrates practical SOC monitoring rather than isolated security-tool configuration.

---

## 11 — Vulnerability Management

Documents the vulnerability-management lifecycle used by the lab:

```text
Identify
   |
   v
Assess
   |
   v
Prioritize
   |
   v
Remediate
   |
   v
Validate
   |
   v
Document
```

The chapter covers vulnerability discovery, severity assessment, remediation planning, validation, and risk-based prioritization.

Where live repository/package verification was not available from the OPNsense environment, the documentation preserves the distinction between validated configuration evidence and assessment methodology rather than treating an unavailable package feed as a confirmed vulnerability.

**Primary outcome:** Demonstrates a structured vulnerability-management process.

---

## 12 — Incident Investigation

Documents the investigation workflow used to analyze a security event from available network and SIEM telemetry.

The investigation follows:

```text
Alert
  |
  v
Triage
  |
  v
Evidence Collection
  |
  v
Timeline Construction
  |
  v
Event Correlation
  |
  v
Impact Assessment
  |
  v
Conclusion
  |
  v
Lessons Learned
```

The documented investigation uses Suricata EVE JSON and Wazuh alert evidence to reconstruct an observed laboratory event involving:

- Source: `192.168.10.20`
- Destination: `192.168.10.30`
- Destination port: `3000/TCP`
- Suricata signature ID: `1000001`
- Wazuh correlation rule: `100102`
- Event classification: controlled laboratory security activity

**Primary outcome:** Demonstrates an end-to-end incident-investigation methodology using correlated network and SIEM evidence.

---

# Evidence Model

The project separates evidence into several layers.

### Network Evidence

Captured through OPNsense and packet-level observation.

Examples include:

- ARP resolution
- TCP SYN / SYN-ACK / ACK
- Application payloads
- Connection termination
- HTTPS traffic

### IDS Evidence

Generated through Suricata EVE JSON.

Important fields include:

```text
timestamp
event_type
in_iface
src_ip
src_port
dest_ip
dest_port
proto
alert.signature
alert.signature_id
alert.severity
alert.action
flow
```

### SIEM Evidence

Collected and correlated through Wazuh.

Important fields include:

```text
timestamp
rule.id
rule.level
rule.description
agent.name
agent.ip
data.src_ip / srcip
data.dest_ip
data.dest_port
full_log
decoder.name
location
```

### Host Evidence

Collected from monitored endpoints and SOC infrastructure.

Examples include:

- SSH authentication events
- PAM session events
- sudo activity
- package-management events
- system logs
- Wazuh agent telemetry

---

# Detection Engineering

The project uses two complementary detection layers.

## Suricata Rules

Suricata operates at the network-inspection layer.

Example laboratory signature:

```text
alert tcp 192.168.10.20 any -> 192.168.10.30 3000
(msg:"NOVASTRA LAB Kali to EMP01 TCP/3000 activity";
flags:S;
sid:1000001;)
```

This identifies a TCP SYN attempt from the designated attack-simulation host to TCP/3000 on the employee endpoint.

## Wazuh Rules

Wazuh operates at the event-analysis and correlation layer.

The custom rules raise the significance of relevant Suricata events and associate them with the lab's known attacker and employee hosts.

This creates a layered detection architecture:

```text
Packet
  |
  v
Suricata Signature
  |
  v
EVE JSON Event
  |
  v
Wazuh Decoder
  |
  v
Wazuh Rule
  |
  v
SOC Alert
  |
  v
Investigation
```

---

# Incident Investigation Model

A repeatable investigation should answer five questions:

1. **What happened?**
2. **When did it happen?**
3. **Which host initiated the activity?**
4. **Which host or service was targeted?**
5. **What evidence supports the conclusion?**

For the documented laboratory scenario:

| Investigation Field | Finding |
|---|---|
| Event | TCP activity targeting TCP/3000 |
| Source | 192.168.10.20 / NT-KALI01 |
| Destination | 192.168.10.30 / NT-EMP01 |
| Protocol | TCP |
| Destination Port | 3000 |
| Suricata Signature | 1000001 |
| Wazuh Rule | 100102 |
| Detection Severity | Wazuh level 9 |
| Network Interface | em1 |
| Classification | Controlled laboratory activity |

The investigation demonstrates how an analyst can correlate a network IDS event with centralized SIEM telemetry instead of treating an alert as an isolated indicator.

---

# Security Controls Demonstrated

| Security Domain | Control / Capability |
|---|---|
| Network Security | Segmented internal network |
| Perimeter Security | OPNsense firewall |
| Stateful Filtering | PF state table |
| Management Security | SSH access control |
| Network IDS | Suricata |
| SIEM | Wazuh |
| Endpoint Monitoring | Wazuh agent |
| Detection Engineering | Custom Suricata rules |
| Event Correlation | Custom Wazuh rules |
| Vulnerability Management | Risk-based assessment workflow |
| Incident Response | Evidence-driven investigation |
| Evidence Collection | Packet, IDS, SIEM, and host telemetry |

---

# Project Validation Philosophy

The lab follows a security-engineering approach rather than simply installing tools.

Each major control is documented through:

1. **Configuration**
2. **Deployment**
3. **Validation**
4. **Evidence**
5. **Interpretation**

This makes the project suitable for demonstrating practical understanding of security operations, network defense, monitoring, and investigation.

---

# Final Project Outcome

The completed Novastra Technologies Cybersecurity Home Lab demonstrates a complete defensive security workflow:

```text
Architecture
     |
     v
Network Segmentation
     |
     v
Firewall Enforcement
     |
     v
Endpoint Monitoring
     |
     v
Network Detection
     |
     v
SIEM Collection
     |
     v
Detection Engineering
     |
     v
Security Validation
     |
     v
Vulnerability Management
     |
     v
Incident Investigation
```

The project therefore functions as a cohesive SOC/security-engineering portfolio rather than a collection of unrelated tool demonstrations.

---

# Directory Structure

```text
Novastra-Technologies-Cybersecurity-Home-Lab/
│
├── README.md
│
├── 01-lab-architecture.md
├── 02-virtualization-and-host-deployment.md
├── 03-network-configuration.md
├── 04-firewall-and-gateway-security.md
├── 05-endpoint-security.md
├── 06-wazuh-deployment.md
├── 07-suricata-ids.md
├── 08-siem-and-detection-engineering.md
├── 09-security-validation-and-attack-simulation.md
├── 10-security-monitoring-and-soc-operations.md
├── 11-vulnerability-management.md
└── 12-incident-investigation.md
```

Additional evidence directories may contain configuration artifacts, detection rules, logs, packet captures, and supporting validation material.

---

# Conclusion

Novastra Technologies Cybersecurity Home Lab provides a controlled environment for demonstrating defensive cybersecurity capabilities across network security, endpoint monitoring, intrusion detection, SIEM operations, detection engineering, vulnerability management, and incident investigation.

The final twelve chapters document the environment from initial architecture through the investigation of security events, providing a coherent technical record of the lab's design, implementation, validation, and operational security workflow.
