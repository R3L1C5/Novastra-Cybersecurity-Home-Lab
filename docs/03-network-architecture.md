# 3. Network Architecture

## 3.1 Overview

The Novastra Technologies Cybersecurity Home Lab uses a segmented virtual network designed to simulate a small enterprise environment.

The architecture provides:

- Controlled Internet access
- Centralized routing
- Firewall enforcement
- Internal VM communication
- Static IP addressing
- Network isolation
- A dedicated security monitoring environment

The internal laboratory network uses:

```text
192.168.10.0/24
````

The VirtualBox internal network is:

```text
intnet
```

---

## 3.2 Logical Architecture


								INTERNET
									|
							  VirtualBox NAT
									|
								 NT-FW01
								OPNsense
							 192.168.10.1/24
									|
								  intnet
							 192.168.10.0/24
									|
       +----------------------------+----------------------------+
       |                            |                            |
       |                            |          		             |
    NT-SOC01            		NT-KALI01           	     NT-EMP01
 Ubuntu Server             	   Kali Linux       	       Ubuntu Desktop
  192.168.10.10/24     		 192.168.10.20/24     	     192.168.10.30/24
       |                 		    |                  		     |
       |          		     Security Testing         		 Endpoint
   Wazuh SOC           		     Endpoint                   Monitoring
       |
   +---+--------+-----------+
   |            |           |
Indexer      Manager    Dashboard
 :9200        :1514       :443
                            |
                       SOC Analyst

					
---

## 3.3 Network Components

| Hostname  | Function                    | IP Address    |
| --------- | --------------------------- | ------------- |
| NT-FW01   | Firewall / Router / Gateway | 192.168.10.1  |
| NT-SOC01  | Central SOC                 | 192.168.10.10 |
| NT-KALI01 | Security Testing Endpoint   | 192.168.10.20 |
| NT-EMP01  | Employee Endpoint           | 192.168.10.30 |

Subnet:

```text
192.168.10.0/24
```

Default gateway:

```text
192.168.10.1
```

---

## 3.4 NT-FW01 — Network Security Boundary

NT-FW01 runs OPNsense 26.7 and acts as the network security boundary for the laboratory.

Its primary responsibilities are:

* Routing
* Firewall enforcement
* Network Address Translation (NAT)
* DNS services
* WAN connectivity
* LAN connectivity
* Controlled communication between the laboratory and external networks

The firewall separates the internal cybersecurity environment from the external network.

---

## 3.5 Internal Network

The laboratory systems communicate through the VirtualBox internal network:

```text
intnet
```

The internal network uses:

```text
192.168.10.0/24
```

Static addresses provide predictable system identification and simplify:

* Monitoring
* Troubleshooting
* Log analysis
* Incident investigation
* Security rule creation
* Documentation

---

## 3.6 Address Allocation

The primary address allocation is:

```text
192.168.10.1   NT-FW01
192.168.10.10  NT-SOC01
192.168.10.20  NT-KALI01
192.168.10.30  NT-EMP01
```

The addresses are intentionally spaced to make the environment easier to understand and maintain.

---

## 3.7 Traffic Flow

### Internet Access

Outbound traffic follows:

```text
Internal VM
    |
    v
NT-FW01
192.168.10.1
    |
    v
VirtualBox NAT
    |
    v
Internet
```

---

### Internal Communication

Systems on the internal network communicate directly through the `intnet` network.

For example:

```text
NT-EMP01
192.168.10.30
      |
      | Wazuh telemetry
      v
NT-SOC01
192.168.10.10
```

The security monitoring architecture therefore allows endpoint telemetry to reach the centralized SOC.

---

## 3.8 Wazuh Network Communication

NT-EMP01 communicates with NT-SOC01 for Wazuh monitoring.

Important Wazuh communication paths include:

```text
NT-EMP01
192.168.10.30
      |
      | TCP 1514
      v
NT-SOC01
192.168.10.10
```

Agent enrollment uses:

```text
TCP 1515
```

The Wazuh Dashboard is accessed through:

```text
https://192.168.10.10
```

using:

```text
TCP 443
```

---

## 3.9 Network Security Model

The network follows a simple layered security model:

```text
                    External Network
                           |
                           v
                    +-------------+
                    |   OPNsense  |
                    |   Firewall  |
                    +-------------+
                           |
                    192.168.10.0/24
                           |
          +----------------+----------------+
          |                |                |
          v                v                v
      Central SOC    Security Testing   Employee
      Environment       Endpoint        Endpoint
```

This provides a clear separation between:

1. Network security
2. Security monitoring
3. Security testing
4. Endpoint operations

---

## 3.10 Connectivity Validation

Network connectivity was validated between the primary systems.

Examples of validation included:

```bash
ping -c 3 192.168.10.1
ping -c 3 192.168.10.10
ping -c 3 192.168.10.20
ping -c 3 192.168.10.30
```

Service availability was also validated where appropriate.

For example, the SSH service on NT-SOC01 was identified during service discovery:

```text
22/tcp open ssh
OpenSSH 9.6p1 Ubuntu
```

This confirmed that the SOC server was reachable and exposing the expected SSH service.

---

## 3.11 Network Design Goals

The network was designed around five primary goals:

### 1. Isolation

Cybersecurity activities remain within a controlled virtual environment.

### 2. Centralization

Security monitoring is centralized on NT-SOC01.

### 3. Predictability

Static addressing makes systems easy to identify.

### 4. Expandability

Additional security technologies can be added without changing the basic network design.

### 5. Controlled Security Testing

The environment provides a dedicated endpoint for future controlled security testing while keeping the core SOC and employee endpoint separate.

---

## 3.12 Phase 1 Network Outcome

The network architecture was successfully implemented and validated.

The completed environment provides:

* A functioning firewall/router
* A dedicated internal network
* Static IP addressing
* Controlled Internet connectivity
* Internal VM communication
* Centralized security monitoring
* A dedicated employee endpoint
* A dedicated security testing endpoint
* A foundation for future security operations

```