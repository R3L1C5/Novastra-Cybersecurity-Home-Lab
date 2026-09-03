# Network Intrusion Detection with Suricata

## 1. Chapter Objective

This chapter documents the deployment and integration design of **Suricata** as the Network Intrusion Detection System (NIDS) for the Novastra Technologies cybersecurity home laboratory.
The completed chapter covers:

- Internal network visibility
- Suricata sensor placement
- IDS rule configuration
- Custom Suricata signatures
- EVE JSON logging
- Wazuh integration
- Wazuh XML correlation rules
- Controlled attacker-to-endpoint detection
- SOC investigation workflow

No screenshots are required for this chapter. Configuration artifacts and packet-capture observations are used as the primary documentation.

> **Evidence note:** Packet-capture observations in this chapter are based on the supplied laboratory captures. Suricata/Wazuh runtime alert generation is documented as the intended implementation/validation procedure rather than represented as an independently observed alert where no corresponding alert record was supplied.

---

# 2. Laboratory Topology

The Novastra laboratory uses an isolated internal network behind the firewall.

| System | Hostname | Operating System | Role | Address |
|---|---|---|---|---|
| Firewall | NT-FW01 | OPNsense 26.7 | Firewall / Router | `192.168.10.1` |
| SOC | NT-SOC01 | Ubuntu Server 24.04.4 | Wazuh Manager / SOC | `192.168.10.20`* |
| Attacker | NT-KALI01 | Kali Linux 2026.2 | Attack Simulation | `192.168.10.20`* |
| Employee | NT-EMP01 | Ubuntu 24.04.4 Desktop | Employee Endpoint | `192.168.10.30` |

\*The supplied packet captures identify `192.168.10.20` as the source/test host communicating with `192.168.10.30`. The final IP assignment should therefore be treated according to the current VM configuration.

### VirtualBox adapters

The laboratory VMs use:

- **Adapter 1:** NAT
- **Adapter 2:** Internal Network
- **Internal Network name:** `intnet`

Logical traffic path:

```text
Internet
   |
VirtualBox NAT
   |
NT-FW01 / OPNsense
192.168.10.1
   |
LAN / em1
   |
Internal Network: intnet
   |
   +----------------------+
   |                      |
NT-KALI01              NT-EMP01
192.168.10.20           192.168.10.30
Attacker                Employee
   |
   +------ SOC monitoring ------+
                                |
                           NT-SOC01
                              Wazuh
```

---

# 3. Suricata Sensor Placement

Suricata must monitor the interface carrying the traffic that needs inspection.
The supplied OPNsense packet captures identify:

```text
Interface: LAN
Interface: em1
Network: 192.168.10.0/24
Gateway: 192.168.10.1
```

Therefore, the LAN-facing traffic path is the relevant monitoring point for the internal IDS exercise.
The objective is to inspect traffic between laboratory systems rather than merely monitoring the VirtualBox NAT connection.

---

# 4. Network Visibility Validation

The packet captures supplied for this chapter provide direct evidence that the internal hosts can communicate.
The primary observed communication was:

```text
192.168.10.20  <---->  192.168.10.30
```

with TCP port:

```text
3000
```

The capture contains a complete TCP connection sequence.

### TCP handshake

Client:

```text
192.168.10.20:43368 > 192.168.10.30:3000
Flags [S]
```

Server:

```text
192.168.10.30:3000 > 192.168.10.20:43368
Flags [S.]
```

Acknowledgement:

```text
192.168.10.20:43368 > 192.168.10.30:3000
Flags [.]
```

This demonstrates successful TCP connection establishment.

---

# 5. Bidirectional Data Transfer

The capture subsequently shows application data flowing in both directions.
Example:

```text
192.168.10.20:43368 > 192.168.10.30:3000
Flags [P.]
```

followed by:

```text
192.168.10.30:3000 > 192.168.10.20:43368
```

The endpoint returned substantial TCP payloads, including multiple large segments.
This establishes that the observed connection was not simply a failed SYN attempt.
The observed sequence therefore provides:

1. ARP resolution
2. TCP SYN
3. TCP SYN/ACK
4. TCP ACK
5. Bidirectional data transfer
6. TCP FIN
7. Session closure

---

# 6. ARP Validation

The capture also contains successful ARP resolution.
The source host requested:

```text
who-has 192.168.10.30 tell 192.168.10.20
```

The destination responded:

```text
192.168.10.30 is-at 08:00:27:90:13:af
```

Reverse resolution was also observed:

```text
who-has 192.168.10.20 tell 192.168.10.30
```

followed by:

```text
192.168.10.20 is-at 08:00:27:ec:80:0d
```

This confirms Layer-2 reachability between the two laboratory hosts.

---

# 7. Additional HTTPS Traffic

A separate supplied capture demonstrates LAN traffic between:

```text
192.168.10.1
```

and:

```text
192.168.10.30
```

using:

```text
TCP/443
```

Observed traffic includes:

```text
192.168.10.1:443 > 192.168.10.30:<ephemeral-port>
```

and corresponding traffic in the reverse direction.
This demonstrates that the LAN interface carries additional TCP services and is therefore suitable for network security monitoring.

---

# 8. Suricata Configuration Model

The Suricata implementation consists of:

```text
/etc/suricata/
├── suricata.yaml
└── rules/
    └── novastra_suricata.rules

/var/log/suricata/
└── eve.json
```

The major configuration requirements are:

- Define the internal laboratory network as `HOME_NET`.
- Select the LAN monitoring interface.
- Enable IDS inspection.
- Load the custom rules.
- Enable EVE JSON output.

Recommended HOME_NET:

```text
HOME_NET: "[192.168.10.0/24]"
```

---

# 9. EVE JSON Logging

EVE JSON provides structured IDS telemetry suitable for Wazuh collection.
Recommended configuration fragment:

```yaml
outputs:
  - eve-log:
      enabled: yes
      filetype: regular
      filename: eve.json
      types:
        - alert
        - flow
        - dns
        - http
        - tls
        - stats
```

Typical output location:

```text
/var/log/suricata/eve.json
```

The exact location can vary according to the installation.
EVE JSON events can contain:

- Timestamp
- Event type
- Source IP
- Source port
- Destination IP
- Destination port
- Protocol
- Signature
- Signature ID
- Severity
- Flow metadata

---

# 10. Custom Suricata Rules

Suricata native detection signatures are stored in `.rules` files.
The Novastra custom rules are designed around the actual laboratory addressing and the controlled attack-simulation workflow.

## Rule 1 — Kali to EMP01 TCP/3000

```text
alert tcp 192.168.10.20 any -> 192.168.10.30 3000 (msg:"NOVASTRA LAB Kali to EMP01 TCP/3000 activity"; flags:S; sid:1000001; rev:1;)
```

**Purpose:** Detect TCP connection attempts from the laboratory attacker/test host to TCP port 3000 on the employee endpoint.

---

## Rule 2 — Kali to EMP01 TCP Connection

```text
alert tcp 192.168.10.20 any -> 192.168.10.30 any (msg:"NOVASTRA LAB Kali TCP connection to EMP01"; flags:S; sid:1000002; rev:1;)
```

**Purpose:** Detect TCP connection attempts from the attacker/test system to the employee endpoint.

---

## Rule 3 — ICMP Echo Detection

```text
alert icmp 192.168.10.20 any -> 192.168.10.30 any (msg:"NOVASTRA LAB ICMP from Kali to EMP01"; itype:8; sid:1000003; rev:1;)
```

**Purpose:** Detect ICMP echo requests from the attacker/test system.
This provides a simple reconnaissance-visibility rule.

---

## Rule 4 — HTTP GET Detection

```text
alert tcp 192.168.10.20 any -> 192.168.10.30 any (msg:"NOVASTRA LAB Possible HTTP exploit-style request from Kali"; flow:to_server,established; content:"GET"; http_method; sid:1000004; rev:1;)
```

**Purpose:** Detect HTTP GET activity from the controlled attacker/test host toward the employee endpoint.
This is a laboratory detection signature and is not intended to classify every HTTP GET request as malicious.

---

# 11. Suricata Rule File

The complete native rule file is:

```text
/etc/suricata/rules/novastra_suricata.rules
```

Contents:

```text
alert tcp 192.168.10.20 any -> 192.168.10.30 3000 (msg:"NOVASTRA LAB Kali to EMP01 TCP/3000 activity"; flags:S; sid:1000001; rev:1;)

alert tcp 192.168.10.20 any -> 192.168.10.30 any (msg:"NOVASTRA LAB Kali TCP connection to EMP01"; flags:S; sid:1000002; rev:1;)

alert icmp 192.168.10.20 any -> 192.168.10.30 any (msg:"NOVASTRA LAB ICMP from Kali to EMP01"; itype:8; sid:1000003; rev:1;)

alert tcp 192.168.10.20 any -> 192.168.10.30 any (msg:"NOVASTRA LAB Possible HTTP exploit-style request from Kali"; flow:to_server,established; content:"GET"; http_method; sid:1000004; rev:1;)
```

---

# 12. Wazuh Integration

The intended telemetry pipeline is:

```text
NT-KALI01
    |
    | Controlled network activity
    v
Internal LAN
    |
    v
Suricata
    |
    | Detection
    v
EVE JSON
    |
    v
Wazuh
    |
    | Correlation
    v
SOC Alert
```

Suricata provides network-level detection while Wazuh provides centralized event collection, correlation, and SOC visibility.

---

# 13. Wazuh XML Correlation Rules

Wazuh correlation rules use XML.
Suggested file:

```text
/var/ossec/etc/rules/novastra_suricata_wazuh_rules.xml
```

Example rule set:

```xml
<group name="suricata,novastra,">

  <rule id="100100" level="5">
    <decoded_as>json</decoded_as>
    <field name="event_type">alert</field>
    <description>Suricata generated an alert</description>
  </rule>

  <rule id="100101" level="8">
    <if_sid>100100</if_sid>
    <field name="src_ip">192.168.10.20</field>
    <field name="dest_ip">192.168.10.30</field>
    <description>Novastra: Suricata alert from attacker/test host to employee endpoint</description>
  </rule>

  <rule id="100102" level="9">
    <if_sid>100100</if_sid>
    <field name="dest_port">3000</field>
    <description>Novastra: Suricata detected traffic to TCP/3000</description>
  </rule>

</group>
```

These are **Wazuh XML rules**, not Suricata rules.
Suricata signatures remain in `.rules` format.

---

# 14. Detection Scenario

The principal controlled detection scenario is:

```text
NT-KALI01
192.168.10.20
       |
       | TCP/3000
       v
NT-EMP01
192.168.10.30
```

The attacker/test host initiates a connection.
Suricata inspects the traffic and evaluates it against the configured signatures.
When a signature matches:

```text
Packet
  |
  v
Suricata Signature
  |
  v
EVE JSON Alert
  |
  v
Wazuh Collection
  |
  v
Wazuh Correlation Rule
  |
  v
SOC Event
```

The resulting event should expose source/destination information and the corresponding Suricata signature metadata.

---

# 15. Expected EVE JSON Structure

A Suricata alert event follows the general structured model:

```json
{
  "event_type": "alert",
  "src_ip": "192.168.10.20",
  "src_port": 43368,
  "dest_ip": "192.168.10.30",
  "dest_port": 3000,
  "proto": "TCP",
  "alert": {
    "signature": "NOVASTRA LAB Kali to EMP01 TCP/3000 activity",
    "signature_id": 1000001,
    "severity": 1
  }
}
```

This is an **illustrative event structure**, not a claim that this exact JSON event was captured.

---

# 16. SOC Interpretation

When the TCP/3000 signature matches, the analyst interpretation is:

> A controlled connection attempt originating from the laboratory attacker/test host `192.168.10.20` was directed toward the employee endpoint `192.168.10.30` on TCP port `3000`. Suricata is configured to identify this activity and forward structured telemetry for Wazuh correlation.

Because this is a controlled cybersecurity laboratory, the activity is expected attack-simulation traffic.

---

# 17. Suricata and Wazuh Roles

| Capability | Suricata | Wazuh |
|---|---|---|
| Packet inspection | Yes | No |
| Network IDS | Yes | No |
| Signature detection | Yes | Correlation |
| EVE JSON | Yes | Consumes |
| Endpoint monitoring | No | Yes |
| Centralized alerts | Limited | Yes |
| Rule format | `.rules` | `.xml` |
| SOC correlation | Limited | Yes |

The combination provides both network and endpoint visibility.

---

# 18. Validation Matrix

| Component | Documentation Status |
|---|---|
| VirtualBox Adapter 1 — NAT | Configured |
| VirtualBox Adapter 2 — Internal Network | Configured |
| Internal network `intnet` | Configured |
| OPNsense LAN / `em1` | Observed |
| LAN subnet `192.168.10.0/24` | Validated |
| ARP between `.20` and `.30` | Demonstrated in supplied capture |
| TCP/3000 SYN | Demonstrated |
| TCP/3000 SYN/ACK | Demonstrated |
| TCP/3000 bidirectional traffic | Demonstrated |
| TCP session termination | Demonstrated |
| TCP/443 LAN traffic | Demonstrated |
| Suricata architecture | Documented |
| EVE JSON configuration | Documented |
| Custom Suricata signatures | Prepared |
| Wazuh XML correlation rules | Prepared |
| SOC detection workflow | Documented |
| Live Suricata alert record | Not independently supplied |
| Live Wazuh correlated alert | Not independently supplied |

---

# 19. Evidence Strategy

This chapter intentionally uses configuration artifacts instead of screenshots.
The evidence package consists of:

```text
1. Suricata configuration
2. Suricata custom rule file
3. Wazuh XML correlation rules
4. EVE JSON schema/example
5. Packet-capture observations
6. Network topology documentation
7. Validation matrix
```

This is sufficient to document the architecture and implementation design without relying on screenshots.
The packet captures provide actual evidence of the underlying network traffic.
The rule files provide reproducible detection logic.
The XML file provides reproducible Wazuh-side correlation logic.

---

# 20. Recommended File Structure

```text
Novastra-Suricata/
│
├── README.md
│
├── suricata/
│   ├── suricata.yaml.fragment
│   └── novastra_suricata.rules
│
├── wazuh/
│   └── novastra_suricata_wazuh_rules.xml
│
└── evidence/
    └── packet_capture_observations.md
```

---

# 21. Implementation Workflow

### Step 1 — Network

Verify that the internal hosts communicate through the `intnet` network.

### Step 2 — Sensor

Configure Suricata against the appropriate LAN monitoring interface.

### Step 3 — Rules

Load:

```text
novastra_suricata.rules
```

### Step 4 — Logging

Enable EVE JSON.

### Step 5 — Wazuh

Configure Wazuh to consume the Suricata event stream.

### Step 6 — Correlation

Load:

```text
novastra_suricata_wazuh_rules.xml
```

### Step 7 — Controlled Test

Generate authorized traffic from the laboratory attacker system toward the employee endpoint.

### Step 8 — SOC Analysis

Review the resulting network detection and correlate it with endpoint telemetry.

---

# 22. Chapter Completion Statement

The Suricata chapter establishes a complete documented IDS architecture for the Novastra Technologies cybersecurity home laboratory.
The supplied network evidence demonstrates that the internal laboratory segment is functioning and that traffic between the attacker/test host and employee endpoint is observable on the LAN interface.
The chapter additionally defines the Suricata detection layer, custom signatures, EVE JSON telemetry, Wazuh integration model, and XML correlation rules.
The implementation is therefore documented as a reproducible SOC laboratory design, while observed packet-capture evidence and unobserved runtime alerts are kept technically distinct.

---

# 23. Final Architecture

```text
                    NOVASTRA SOC
                         |
                      WAZUH
                         |
                XML Correlation Rules
                         |
                     EVE JSON
                         |
                     SURICATA
                         |
                     LAN / em1
                         |
              Internal Network intnet
                  /               \
                 /                 \
        NT-KALI01                 NT-EMP01
        192.168.10.20             192.168.10.30
          Attacker                  Employee
                 \                 /
                  \               /
                    NT-FW01
                 OPNsense 26.7
                  192.168.10.1
                         |
                    VirtualBox NAT
                         |
                      Internet
```

**Chapter status:** Documentation complete. Configuration artifacts defined. Network traffic evidence documented. Runtime IDS/SIEM alerts are not represented as independently observed evidence unless corresponding logs are available.
