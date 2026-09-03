# 2. Lab Requirements

## 2.1 Host Hardware

The Novastra Cybersecurity Home Lab runs on a Windows 11 host using Oracle VirtualBox.

|     Component     |    Specification    |
|-------------------|---------------------|
|      Host OS      |      Windows 11     |
|        CPU        | Intel Core i5-11400 |
|        RAM        |        32 GB        |
|   Boot Storage    |     ~250 GB SSD     |
| Secondary Storage |      ~1 TB HDD      |
|     Hypervisor    |  VirtualBox 7.2.14  |

The host provides sufficient resources to operate the laboratory while allowing multiple virtual machines to run simultaneously.

## 2.2 Virtualization Platform

The laboratory uses:

**Oracle VirtualBox 7.2.14**

Virtual machines are connected using a dedicated VirtualBox internal network:

```text
intnet
````

The internal network isolates the laboratory's virtual systems while allowing controlled communication between them.
Internet access is provided through the OPNsense firewall/router.

## 2.3 Virtual Machines

The laboratory consists of four primary virtual machines.

| Hostname  | Purpose                   | Operating System          | IP Address    |
| --------- | ------------------------- | ------------------------- | ------------- |
| NT-FW01   | Firewall / Router         | OPNsense 26.7             | 192.168.10.1  |
| NT-SOC01  | Central SOC               | Ubuntu Server 24.04.4 LTS | 192.168.10.10 |
| NT-KALI01 | Security Testing Endpoint | Kali Linux  2026.2        | 192.168.10.20 |
| NT-EMP01  | Employee Endpoint         | ubuntu Desktop 24.04.4    | 192.168.10.30 |


## 2.4 Network Requirements

The internal laboratory network uses:

```text
Network: 192.168.10.0/24
Gateway: 192.168.10.1
```

Static addresses are assigned to the primary systems to provide predictable communication and simplify security monitoring.

The logical network is:

```text
Internet
   |
VirtualBox NAT
   |
NT-FW01
192.168.10.1
   |
intnet
   |
+-------------------------------+
|               |               |
|               |               |
NT-SOC01     NT-KALI01       NT-EMP01
.10          .20             .30
```


## 2.5 Security Monitoring Requirements

The central SOC requires sufficient resources to operate the Wazuh platform.

### NT-SOC01

| Resource | Allocation                |
| -------- | ------------------------- |
| CPU      | 4 vCPU                    |
| RAM      | 8 GB                      |
| Disk     | ~97 GB                    |
| OS       | Ubuntu Server 24.04.4 LTS |

The Wazuh deployment includes:

* Wazuh Indexer
* Wazuh Manager
* Wazuh Dashboard
* Wazuh API


## 2.6 Endpoint Monitoring Requirements

NT-EMP01 acts as the monitored employee endpoint.

Requirements include:

* Wazuh Agent
* systemd-journald
* File Integrity Monitoring
* Network connectivity to the SOC
* Automatic Wazuh Agent startup

The endpoint must be able to communicate with the Wazuh Manager at:

```text
192.168.10.10
```


## 2.7 Internal PKI Requirements

The laboratory includes an internal Certificate Authority for establishing trusted certificates within the environment.

The Root CA uses:

```text
Algorithm: RSA
Key Size: 4096-bit
Hash: SHA-256
Validity: 10 years
```

The Root CA is trusted by the laboratory systems that require it.

Certificate deployment is automated using Bash scripts designed to be repeat-safe.


## 2.8 Required Network Services

The following Wazuh services and ports are required for the completed SOC environment:

| Port  | Protocol | Purpose                     |
| ----- | -------- | --------------------------- |
| 1514  | TCP      | Wazuh Agent communication   |
| 1515  | TCP      | Agent enrollment            |
| 1516  | TCP      | Wazuh cluster communication |
| 55000 | TCP      | Wazuh API                   |
| 9200  | TCP      | Wazuh Indexer               |
| 443   | TCP      | Wazuh Dashboard             |

The completed environment was validated to ensure the required Wazuh services were operational.


## 2.9 Software Requirements

Primary software used in Phase 1 includes:

* Oracle VirtualBox 7.2.14
* OPNsense 26.7
* Ubuntu Server 24.04.4 LTS
* Kali Linux
* Ubuntu desktop 24.04.4
* Wazuh 4.14.7
* OpenSSH
* systemd-journald
* OpenSSL / Linux certificate management utilities


## 2.10 Design Considerations

The laboratory was designed with the following considerations:

### Isolation

Cybersecurity activities are performed inside an isolated virtual environment.

### Resource Efficiency

Virtual machine resources are allocated according to their role to allow multiple systems to operate on a 32 GB host.

### Predictable Addressing

Static IP addresses simplify administration, monitoring, and investigation.

### Centralized Monitoring

Security telemetry is centralized on NT-SOC01 through Wazuh.

### Expandability

The architecture allows additional security capabilities to be added without redesigning the core infrastructure.

Planned future capabilities include:

* Attack simulation
* Detection engineering
* Network intrusion detection
* Vulnerability management
* Incident investigation


## 2.11 Phase 1 Requirement Completion

The infrastructure requirements for Phase 1 were successfully implemented.

The completed environment provides:

* Virtualized infrastructure
* Firewall and routing
* Internal networking
* Centralized security monitoring
* Endpoint telemetry
* File Integrity Monitoring
* Internal PKI
* Security testing infrastructure
* Automatic service startup
* A foundation for subsequent cybersecurity Phases
