# 4. Network Security and Internal PKI

## 4.1 Overview

Phase 1 established two important security foundations for the Novastra Technologies Cybersecurity Home Lab:

1. A network security boundary using OPNsense.
2. An internal Public Key Infrastructure (PKI) using a private Root Certificate Authority.

Together, these components provide controlled network connectivity and internal certificate trust for the laboratory environment.

---

# Part A — Network Security

## 4.2 OPNsense Firewall

NT-FW01 runs **OPNsense 26.7** and serves as the network security boundary and default gateway for the laboratory.

Primary responsibilities include:

- Firewall enforcement
- Network routing
- Network Address Translation (NAT)
- DNS services
- WAN connectivity
- LAN connectivity
- Controlled Internet access

```text
                         INTERNET
                            |
                     VirtualBox NAT
                            |
                            v
                    +---------------+
                    |    NT-FW01    |
                    | OPNsense 26.7 |
                    | 192.168.10.1  |
                    +-------+-------+
                            |
                          intnet
                            |
                    192.168.10.0/24
                            |
          +-----------------+------------------+
          |                 |                  |
          v                 v                  v
      NT-SOC01          NT-KALI01          NT-EMP01
      .10               .20                .30
````

---

## 4.3 WAN and LAN

The firewall uses separate network interfaces for external and internal connectivity.

### WAN

The WAN interface connects toward the VirtualBox NAT network and provides controlled external connectivity.

```text
NT-FW01 WAN
     |
VirtualBox NAT
     |
Internet
```

### LAN

The LAN interface connects to the isolated VirtualBox internal network:

```text
intnet
```

The LAN address is:

```text
192.168.10.1/24
```

This address is used as the default gateway for the internal systems.

---

## 4.4 Internal Network

The protected laboratory network uses:

```text
Network: 192.168.10.0/24
Gateway: 192.168.10.1
```

Primary systems:

| Hostname  | Role                      | IP Address    |
| --------- | ------------------------- | ------------- |
| NT-FW01   | Firewall / Router         | 192.168.10.1  |
| NT-SOC01  | Central SOC               | 192.168.10.10 |
| NT-KALI01 | Security Testing Endpoint | 192.168.10.20 |
| NT-EMP01  | Employee Endpoint         | 192.168.10.30 |

Static addressing provides predictable system identification for administration, monitoring, and security investigation.

---

## 4.5 Routing

NT-FW01 acts as the default gateway for the internal systems.

External traffic follows:

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

Communication between systems on the same internal subnet remains within the laboratory network.

---

## 4.6 Network Address Translation

NAT allows internal laboratory systems to access external networks through NT-FW01 without requiring directly routable external addresses.

```text
192.168.10.x
     |
     v
NT-FW01
     |
     v
NAT
     |
     v
External Network
```

This provides a controlled path between the isolated laboratory and the Internet.

---

## 4.7 DNS

DNS functionality was configured as part of the network infrastructure.

DNS resolution is required for:

* Operating-system updates
* Package installation
* Security-tool updates
* External service resolution
* Certificate-related operations

---

## 4.8 Network Security Validation

The completed network security infrastructure was validated through:

* Gateway connectivity
* Internal VM communication
* Routing
* NAT
* DNS resolution
* Internet connectivity
* Firewall operation

Typical Linux validation commands included:

```bash
ip addr
ip route
ping -c 3 192.168.10.1
ping -c 3 192.168.10.10
ping -c 3 192.168.10.20
ping -c 3 192.168.10.30
```

These checks confirmed that the internal systems could communicate through the completed network architecture.

---

# Part B — Internal PKI

## 4.9 Internal Public Key Infrastructure

An internal PKI was implemented to establish certificate trust within the laboratory environment.

The laboratory Root Certificate Authority is:

```text
Novastra Root CA
```

The CA provides the foundation for issuing and trusting certificates within the private laboratory environment.

---

## 4.10 Root CA Configuration

The Novastra Root CA was created using:

| Property       | Configuration              |
| -------------- | -------------------------- |
| Key Algorithm  | RSA                        |
| Key Size       | 4096-bit                   |
| Hash Algorithm | SHA-256                    |
| Validity       | 10 years                   |
| CA Type        | Root Certificate Authority |

The configuration provides a strong cryptographic foundation appropriate for an isolated enterprise-style laboratory.

---

## 4.11 Certificate Trust Deployment

The Novastra Root CA was deployed and trusted on the laboratory systems that require internal certificate validation.

Trusted systems include:

```text
NT-SOC01
NT-KALI01
NT-EMP01
```

The certificates were installed into the appropriate Linux trust stores and the operating systems' certificate databases were updated.

---

## 4.12 PKI Automation

Certificate deployment was automated using Bash scripts.

The automation performs tasks such as:

1. Detecting the certificate.
2. Converting the certificate filename from `.pem` to `.crt` where required.
3. Copying the certificate into the Linux trust store.
4. Updating the system certificate database.
5. Completing the operation safely when the certificate is already installed.

The scripts were designed to be **idempotent and repeat-safe**.

This allows certificate deployment to be repeated without unnecessarily duplicating or corrupting the trust configuration.

---

## 4.13 PKI Validation

The Root CA trust configuration was validated on the required laboratory systems.

The validation confirmed that:

* The Root CA was present.
* The certificate was installed in the appropriate trust store.
* The operating system certificate database was updated.
* Internal certificate trust could be established.

---

## 4.14 Security Architecture

The completed security foundation can be represented as:

```text
                    External Network
                           |
                           v
                    +-------------+
                    |   OPNsense  |
                    |   NT-FW01   |
                    +------+------+
                           |
                    192.168.10.0/24
                           |
          +----------------+----------------+
          |                |                |
          v                v                v
      NT-SOC01         NT-KALI01         NT-EMP01
          |                |                |
          +----------------+----------------+
                           |
                    Novastra Root CA
                    Internal Trust
```

OPNsense provides the **network security boundary**, while the Novastra Root CA provides the **internal certificate trust foundation**.

---

## 4.15 Phase 1 Outcome

The network security and internal PKI foundation was successfully implemented.

### Network Security

The laboratory has:

* OPNsense firewall
* WAN connectivity
* LAN connectivity
* Routing
* NAT
* DNS
* Controlled Internet access
* Isolated internal networking

### Internal PKI

The laboratory has:

* Novastra Root CA
* RSA 4096-bit cryptography
* SHA-256
* 10-year certificate validity
* Trusted CA deployment
* Automated certificate deployment
* Repeat-safe PKI scripts

These components provide the network and trust infrastructure required for the subsequent cybersecurity Phases.