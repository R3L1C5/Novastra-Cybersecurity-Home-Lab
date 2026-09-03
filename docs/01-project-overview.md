# 1. Phase Overview

## Novastra Technologies Cybersecurity Home Lab

**Organization:** Novastra Technologies Pvt. Ltd.  
**Project:** Novastra Technologies Cybersecurity Home Lab  
**Environment:** VirtualBox-based cybersecurity home laboratory

---

## 1.1 Phase Purpose

The Novastra Technologies Cybersecurity Home Lab is an enterprise-style virtual cybersecurity environment designed to simulate the core infrastructure and security monitoring capabilities of a small organization's IT environment.

The Phase was built to develop practical experience in:

- Network security
- Firewall administration
- Security monitoring
- Endpoint monitoring
- Security event collection
- File Integrity Monitoring (FIM)
- Internal Public Key Infrastructure (PKI)
- SOC operations
- Security investigation
- Cybersecurity documentation

The environment is intentionally isolated within a controlled virtual network so that security technologies and future attack simulations can be tested safely.

---

## 1.2 Phase Objectives

The primary objectives of Phase 1 were to:

1. Build an isolated enterprise-style virtual network.
2. Deploy and configure a dedicated firewall/router.
3. Establish controlled Internet and internal network connectivity.
4. Deploy a centralized Security Operations Center (SOC).
5. Implement Wazuh for security monitoring and endpoint telemetry.
6. Deploy an employee endpoint with Wazuh Agent monitoring.
7. Implement systemd-journald log collection.
8. Implement File Integrity Monitoring.
9. Establish an internal Root Certificate Authority.
10. Deploy and trust internal certificates across the lab.
11. Validate that the complete environment is operational and ready for subsequent security Phases.

---

## 1.3 Lab Architecture

The completed environment consists of four primary virtual machines:

| Hostname  | Role                      | Operating System          | IP Address    |
|-----------|---------------------------|---------------------------|---------------|
| NT-FW01   | Firewall / Router         | OPNsense 26.7             | 192.168.10.1  |
| NT-SOC01  | Central SOC               | Ubuntu Server 24.04.4 LTS | 192.168.10.10 |
| NT-KALI01 | Security Testing Endpoint | Kali Linux 2026.2         | 192.168.10.20 |
| NT-EMP01  | Employee Endpoint         | ubuntu desktop 24.04.4    | 192.168.10.30 |

The internal network uses: 192.168.10.0/24 (intnet)

## 1.4 High-Level Architecture:

                         INTERNET
                            |
                      VirtualBox NAT
                            |
                         NT-FW01
                        OPNsense
                      192.168.10.1
                            |
                          intnet
                            |
       +--------------------+--------------------+
       |                    |                    |
       |                    |                    |
    NT-SOC01            NT-KALI01             NT-EMP01
 Ubuntu Server         Kali Linux          Ubuntu Desktop
     .10                  .20                  .30
       |                    |                    |
       |            Security Testing          Endpoint
       |              Endpoint                Monitoring
       |
   Wazuh SOC
       |
   +---+--------+-----------+
   |            |           |
Indexer      Manager    Dashboard
 :9200        :1514       :443
                            |
                      SOC Analyst
					
					
## 1.5 Security Components

**OPNsense**
OPNsense provides the network security boundary for the virtual environment.

Implemented functions include:
- WAN connectivity
- LAN configuration
- Routing
- Network Address Translation (NAT)
- DNS
- Firewall policy
- Internal network connectivity

**Wazuh**
Wazuh provides centralized security monitoring for the environment.

The Wazuh deployment includes:
- Wazuh Indexer
- Wazuh Manager
- Wazuh Dashboard
- Wazuh API
- Wazuh Agent on NT-EMP01

The SOC provides centralized visibility into endpoint security events and file integrity activity.

**Endpoint Monitoring**

NT-EMP01 acts as the simulated employee workstation. 
The Wazuh Agent collects endpoint telemetry including system and authentication-related events.
Because ubuntu desktopSecurity OS uses systemd-journald, journald collection was explicitly configured for the Wazuh Agent.

**File Integrity Monitoring**

Wazuh Syscheck was configured to monitor important system locations and a dedicated real-time test directory.
FIM successfully detected:
*File creation
*File modification
*Integrity checksum changes

The successful FIM detection is preserved as Phase evidence in:
screenshots/wazuh/FIM Test.png

**Internal PKI**

An internal Root Certificate Authority named Novastra Root CA was created for the laboratory.
The Root CA uses:
-RSA 4096-bit key
-SHA-256
-10-year validity

The CA trust was deployed to the laboratory systems and automated using repeat-safe Bash scripts.

## 1.6 Phase Validation

Phase 1 was considered operational after validating the following components:

-Virtual network connectivity
-OPNsense firewall and routing
-Internet access
-Internal VM communication
-Internal PKI trust
-Wazuh Indexer
-Wazuh Manager
-Wazuh Dashboard
-Wazuh API
-Wazuh Agent
-Journald log collection
-File Integrity Monitoring
-Security testing endpoint

The Wazuh services were configured to start automatically with the SOC server.
The employee endpoint's Wazuh Agent was configured for automatic startup and successfully demonstrated endpoint telemetry and FIM functionality.

## 1.7 Phase Boundary

Phase 1 focuses on building and validating the cybersecurity infrastructure.
It does not include the attack-and-detection workflow.
The subsequent Phase will use this completed environment to perform controlled security testing, generate security events, develop detections, and investigate the resulting activity.
The separation is intentional:
Project: Novastra Technologies Cybersecurity Home Lab
					|
					v
				Phase 1
		Cybersecurity Infrastructure
					|
					v
				Phase 2
		Attack Simulation + Detection
					|
					v
				Phase 3
		Advanced Security Operations

## 1.8 Outcome

Phase 1 established a functioning enterprise-style cybersecurity laboratory consisting of a firewall, segmented virtual network, internal PKI, centralized Wazuh SOC, monitored employee endpoint, and File Integrity Monitoring.
The environment is now ready to serve as the foundation for subsequent attack simulation, detection engineering, network intrusion detection, vulnerability assessment, and incident investigation work.