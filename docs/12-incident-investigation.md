# 12. Incident Investigation

## 12.1 Overview

This chapter documents the investigation of a controlled security incident within the Novastra Technologies Cybersecurity Home Lab.

The investigation follows a SOC workflow from network detection through SIEM correlation, event analysis, timeline reconstruction, and final incident classification.

The primary activity investigated is traffic from **NT-KALI01 (`192.168.10.20`)** toward **NT-EMP01 (`192.168.10.30`)** over **TCP/3000**.

The investigation combines:

- Suricata network detection telemetry
- Wazuh correlation telemetry
- Network packet-capture evidence
- Endpoint telemetry from NT-EMP01
- Custom Suricata and Wazuh detection rules
- Incident timeline and SOC analysis

---

## 12.2 Incident Summary

| Field | Value |
|---|---|
| Incident ID | INC-001 |
| Incident Type | Suspicious internal network activity |
| Detection Source | Suricata |
| SIEM / Correlation | Wazuh |
| Source Host | NT-KALI01 |
| Source IP | `192.168.10.20` |
| Destination Host | NT-EMP01 |
| Destination IP | `192.168.10.30` |
| Destination Port | TCP/3000 |
| Protocol | TCP |
| Suricata Signature ID | `1000001` |
| Wazuh Correlation Rule | `100102` |
| Wazuh Alert Level | 9 |
| Suricata Severity | 1 |
| Action | Allowed |
| Monitoring Interface | OPNsense LAN / `em1` |
| Classification | Controlled Security Testing Activity |

The event represents authorized attack-simulation activity originating from the laboratory attacker system and directed toward the employee endpoint.

---

## 12.3 Detection Architecture

The incident detection path is:

```text
NT-KALI01
192.168.10.20
      |
      | TCP/3000
      v
NT-FW01 / OPNsense
192.168.10.1
      |
      | Suricata inspection
      v
Suricata Detection
SID 1000001
      |
      | EVE JSON telemetry
      v
Wazuh
Rule 100102
Level 9
      |
      v
SOC Investigation
```

Suricata provides network-level inspection at the firewall. EVE JSON provides structured telemetry containing the network addresses, ports, protocol, detection signature, severity, and flow information.

Wazuh provides the SIEM and SOC correlation layer used to process the Suricata event and raise its investigation priority.

---

## 12.4 Detection Rule

The primary Suricata detection rule is:

```text
alert tcp 192.168.10.20 any -> 192.168.10.30 3000
(msg:"NOVASTRA LAB Kali to EMP01 TCP/3000 activity";
flags:S;
sid:1000001;
)
```

### Rule purpose

The rule detects TCP SYN connection attempts originating from the laboratory attacker host and targeting TCP/3000 on the employee endpoint.

The rule is intentionally scoped to the laboratory topology so that the designated attack-simulation traffic can be identified precisely.

### Detection parameters

| Parameter | Value |
|---|---|
| Protocol | TCP |
| Source | `192.168.10.20` |
| Source Port | Any |
| Destination | `192.168.10.30` |
| Destination Port | `3000` |
| TCP Flag | SYN |
| Signature ID | `1000001` |
| Category | Novastra Laboratory Detection |
| Severity | 1 |

---

## 12.5 Suricata Incident Telemetry

The Suricata event records:

```json
{
  "timestamp": "2026-09-01T19:15:42.183921+0530",
  "flow_id": 1000001000001,
  "in_iface": "em1",
  "event_type": "alert",
  "src_ip": "192.168.10.20",
  "src_port": 43368,
  "dest_ip": "192.168.10.30",
  "dest_port": 3000,
  "proto": "TCP",
  "pkt_src": "wire",
  "ip_v": 4,
  "direction": "to_server",
  "flow": {
    "pkts_toserver": 1,
    "pkts_toclient": 0,
    "bytes_toserver": 60,
    "bytes_toclient": 0
  },
  "alert": {
    "action": "allowed",
    "gid": 1,
    "signature_id": 1000001,
    "rev": 1,
    "signature": "NOVASTRA LAB Kali to EMP01 TCP/3000 activity",
    "category": "Novastra Laboratory Detection",
    "severity": 1
  }
}
```

### Observed values

| Field | Observed Value |
|---|---|
| Timestamp | `2026-09-01 19:15:42.183921 +0530` |
| Interface | `em1` |
| Event Type | `alert` |
| Source | `192.168.10.20:43368` |
| Destination | `192.168.10.30:3000` |
| Protocol | TCP |
| Direction | `to_server` |
| Signature | `NOVASTRA LAB Kali to EMP01 TCP/3000 activity` |
| Signature ID | `1000001` |
| Severity | 1 |
| Action | Allowed |
| Packets to Server | 1 |
| Bytes to Server | 60 |

The `action: allowed` value indicates that the detected traffic was permitted rather than blocked. Detection and prevention are therefore separate controls in this event.

---

## 12.6 Wazuh Correlation

The corresponding Wazuh telemetry is:

```json
{
  "timestamp": "2026-09-01T19:15:42.221000+0530",
  "rule": {
    "level": 9,
    "id": "100102",
    "description": "Novastra: Suricata detected traffic to TCP/3000"
  },
  "location": "suricata",
  "decoder": {
    "name": "json"
  },
  "data": {
    "event_type": "alert",
    "src_ip": "192.168.10.20",
    "src_port": "43368",
    "dest_ip": "192.168.10.30",
    "dest_port": "3000",
    "proto": "TCP",
    "alert": {
      "signature": "NOVASTRA LAB Kali to EMP01 TCP/3000 activity",
      "signature_id": "1000001",
      "severity": "1",
      "category": "Novastra Laboratory Detection",
      "action": "allowed"
    }
  }
}
```

### Correlation result

| Field | Value |
|---|---|
| Wazuh Rule ID | `100102` |
| Wazuh Level | 9 |
| Description | Novastra: Suricata detected traffic to TCP/3000 |
| Source IP | `192.168.10.20` |
| Source Port | `43368` |
| Destination IP | `192.168.10.30` |
| Destination Port | `3000` |
| Protocol | TCP |
| Suricata SID | `1000001` |
| Action | Allowed |

The Wazuh correlation rule increases the SOC significance of traffic directed at the monitored TCP/3000 service.

---

## 12.7 Detection Timeline

The timestamps establish the following sequence:

| Time (IST) | Event |
|---|---|
| 19:15:42.183921 | Suricata records the TCP/3000 detection on `em1`. |
| 19:15:42.183921 | `192.168.10.20:43368` targets `192.168.10.30:3000`. |
| 19:15:42.221000 | Wazuh records the correlated Suricata event. |
| ~37 ms later | Wazuh raises rule `100102` at Level 9. |

The Wazuh event occurs approximately **37 milliseconds** after the Suricata event, demonstrating a rapid detection-to-correlation sequence.

---

## 12.8 Network Evidence

Network traffic previously captured from the OPNsense LAN interface established communication between:

```text
Source:      192.168.10.20
Destination: 192.168.10.30
Protocol:    TCP
Port:        3000
Interface:   em1
```

The packet capture showed the expected TCP session sequence:

```text
ARP resolution
      |
TCP SYN
      |
TCP SYN/ACK
      |
TCP ACK
      |
Application payload
      |
TCP FIN
      |
Session closure
```

Application payload packets of approximately 1448 bytes were also observed during the session.

This evidence supports the investigation by demonstrating actual TCP communication between the two laboratory hosts.

---

## 12.9 Incident Analysis

The source address `192.168.10.20` corresponds to NT-KALI01, the designated laboratory attack-simulation host.

The destination address `192.168.10.30` corresponds to NT-EMP01, the employee endpoint.

The activity therefore represents traffic moving from the designated attacker environment toward an employee workstation.

TCP/3000 was specifically selected for monitoring through the custom Suricata rule.

The evidence establishes:

1. NT-KALI01 generated TCP traffic toward NT-EMP01.
2. The destination service was TCP/3000.
3. Suricata matched the traffic against signature `1000001`.
4. The traffic was observed on OPNsense interface `em1`.
5. Wazuh correlated the Suricata event through rule `100102`.
6. Wazuh raised the resulting event to Level 9.
7. Packet-capture evidence confirms TCP communication between the two hosts.
8. The available evidence does not establish successful exploitation or compromise of NT-EMP01.

The appropriate SOC conclusion is therefore **detected suspicious activity requiring investigation**, rather than confirmed endpoint compromise.

---

## 12.10 Endpoint Telemetry Context

Wazuh is also receiving endpoint telemetry from NT-EMP01.

Observed endpoint events include:

- Package-management activity
- PAM session activity
- Successful SSH authentication
- Privileged commands executed through `sudo`
- Administrative SSH connections to SOC01 and NT-FW01

These events provide additional context during incident triage.

A SOC analyst must distinguish legitimate administrative activity from the network security event involving NT-KALI01. This prevents normal administrative operations from being incorrectly classified as malicious behavior.

---

## 12.11 MITRE ATT&CK Mapping

The available evidence supports investigation against several ATT&CK areas, but does not justify asserting techniques that were not observed.

| ATT&CK Area | Assessment |
|---|---|
| Discovery | Network targeting activity may warrant investigation for service discovery. |
| Initial Access | Not established by the available evidence. |
| Exploitation | Not established. |
| Privilege Escalation | Not established as part of this incident. |
| Lateral Movement | Not established. |
| Command and Control | Not established. |
| Exfiltration | Not established. |

The distinction is important: a detection rule identifying network traffic is evidence of **activity**, not automatic evidence of successful exploitation or compromise.

---

## 12.12 SOC Investigation Procedure

### Step 1 — Identify the alert

Wazuh raises rule `100102` at Level 9 for detected TCP/3000 activity.

### Step 2 — Identify the source

The source address is:

```text
192.168.10.20
```

This maps to NT-KALI01.

### Step 3 — Identify the target

The destination is:

```text
192.168.10.30:3000
```

This maps to NT-EMP01.

### Step 4 — Validate the network detection

Suricata telemetry confirms that the event was detected on OPNsense LAN interface `em1`.

### Step 5 — Validate packet behavior

Packet-capture evidence confirms TCP session establishment, payload transfer, and session termination.

### Step 6 — Check endpoint context

Wazuh endpoint telemetry is reviewed for authentication, privilege escalation, package changes, process activity, and other indicators that could demonstrate successful compromise.

### Step 7 — Determine compromise status

No supplied evidence confirms successful exploitation, persistence, privilege escalation, or data exfiltration resulting from the TCP/3000 event.

### Step 8 — Classify the event

Because the source is the authorized laboratory attack-simulation host, the event is classified as:

```text
Controlled Security Testing Activity
```

### Step 9 — Preserve evidence

The Suricata event, Wazuh event, detection rules, packet-capture evidence, and relevant endpoint telemetry form the investigation evidence set.

---

## 12.13 Incident Classification

| Category | Assessment |
|---|---|
| Network Activity | Confirmed |
| Suspicious TCP/3000 Activity | Confirmed |
| Suricata Detection | Confirmed |
| Wazuh Correlation | Confirmed |
| TCP Session | Confirmed by packet evidence |
| Successful Exploitation | Not established |
| Endpoint Compromise | Not established |
| Data Exfiltration | Not established |
| Lab Authorization | Confirmed |
| Final Classification | Controlled Security Testing Activity |

---

## 12.14 Recommended SOC Response

For a production environment, the response to a comparable alert would normally include:

1. Validate the source and destination assets.
2. Determine whether communication between the assets is authorized.
3. Review surrounding network flows for additional suspicious activity.
4. Review endpoint telemetry for process execution, authentication, privilege escalation, and persistence.
5. Determine whether the destination service is expected to be exposed.
6. Contain the source or destination if malicious activity is confirmed.
7. Preserve relevant packet, IDS, SIEM, and endpoint evidence.
8. Escalate according to the organization's incident-response severity model.
9. Document the investigation and final disposition.

For the Novastra laboratory, containment is not required because the source system is the designated attack-simulation host and the activity is authorized.

---

## 12.15 Evidence Register

| Evidence ID | Evidence | Purpose |
|---|---|---|
| EV-001 | Suricata EVE JSON alert | Establishes network IDS detection |
| EV-002 | Wazuh alert JSON | Establishes SIEM correlation |
| EV-003 | Suricata rule `1000001` | Establishes detection logic |
| EV-004 | Wazuh rule `100102` | Establishes correlation logic |
| EV-005 | OPNsense LAN packet capture | Establishes network-session behavior |
| EV-006 | Wazuh endpoint telemetry | Provides endpoint investigation context |
| EV-007 | Laboratory topology | Establishes asset and IP relationships |

---

## 12.16 Key Findings

### FINDING-IR-001 — Controlled TCP/3000 Activity

NT-KALI01 (`192.168.10.20`) generated TCP traffic toward NT-EMP01 (`192.168.10.30`) on port 3000.

**Status:** Investigated

### FINDING-IR-002 — Suricata Detection

Suricata detected the traffic using custom signature `1000001`.

**Status:** Confirmed

### FINDING-IR-003 — Wazuh Correlation

Wazuh correlated the Suricata event using custom rule `100102` and raised the event to Level 9.

**Status:** Confirmed

### FINDING-IR-004 — No Confirmed Compromise

The available evidence does not establish successful exploitation or compromise of NT-EMP01.

**Status:** No compromise established

### FINDING-IR-005 — Authorized Laboratory Activity

The source system is the designated attack-simulation host in the Novastra laboratory.

**Status:** Authorized

---

## 12.17 Final Incident Assessment

The investigation demonstrates an end-to-end SOC incident-investigation workflow within the Novastra Technologies Cybersecurity Home Lab.

A controlled connection from NT-KALI01 (`192.168.10.20`) to NT-EMP01 (`192.168.10.30`) on TCP/3000 was identified by the custom Suricata signature `1000001`. The resulting telemetry was correlated by Wazuh rule `100102` and raised to Level 9.

Packet evidence confirms that the hosts established TCP communication and exchanged application payload data.

The available evidence does not establish successful exploitation or compromise of NT-EMP01. The activity is therefore classified as an **authorized controlled security-testing event**.

The completed investigation demonstrates the intended SOC workflow:

```text
Network Traffic
      ↓
OPNsense
      ↓
Suricata IDS
      ↓
EVE JSON
      ↓
Wazuh Correlation
      ↓
SOC Alert
      ↓
Incident Investigation
      ↓
Evidence-Based Classification
```

This chapter completes the incident-response and investigation component of the Novastra Technologies Cybersecurity Home Lab, demonstrating how network telemetry, IDS detections, SIEM correlation, packet evidence, and endpoint context can be combined into a structured SOC investigation.
