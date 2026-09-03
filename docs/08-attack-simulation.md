# Attack Simulation

## 1. Overview

Phase 2 of the Novastra Technologies Cybersecurity Home Lab focused on controlled attack simulation against the intentionally vulnerable OWASP Juice Shop application hosted on the employee endpoint.

The objective was to generate realistic attacker activity from the Kali Linux attacker VM, validate network exposure, perform web reconnaissance, test authentication controls, and generate security telemetry that can later be used for detection engineering and incident investigation.

All activities were performed inside the isolated lab network.

### Lab Attack Path

```text
NT-KALI01
Kali Linux
192.168.10.20
     |
     | Reconnaissance / Web attacks
     v
NT-EMP01
Ubuntu Desktop + OWASP Juice Shop
192.168.10.30:3000
     |
     | Telemetry
     v
NT-SOC01
Wazuh SOC
192.168.10.10
```

## 2. Attack Simulation Scope

The following activities were performed during Phase 2:

| Activity                         | Tool       | Target             | Status   |
| -------------------------------- | ---------- | ------------------ | -------- |
| Port/service reconnaissance      | Nmap       | 192.168.10.30      | Complete |
| Web vulnerability reconnaissance | Nikto      | Juice Shop :3000   | Complete |
| Directory enumeration            | Gobuster   | Juice Shop :3000   | Complete |
| HTTP request inspection          | Burp Suite | Juice Shop :3000   | Complete |
| SQL injection testing            | SQLMap     | Product search API | Complete |
| Authentication testing           | cURL       | `/rest/user/login` | Complete |
| Authentication wordlist testing  | FFUF       | `/rest/user/login` | Complete |
| SSH authentication activity      | SSH        | NT-EMP01           | Complete |

The phase was intentionally limited to the isolated laboratory environment.

---

## 3. Target Application

OWASP Juice Shop was deployed on NT-EMP01 as the intentionally vulnerable web application used for attack simulation.

The application was successfully started with:

```bash
cd ~/juice-shop
npm start
```

The application reported:

```text
Server listening on port 3000
```

The service was verified locally on NT-EMP01 with:

```bash
sudo ss -lntp | grep ':3000'
```

The application was then accessed remotely from NT-KALI01.

The target was:

```text
http://192.168.10.30:3000
```

The application is intentionally vulnerable and was selected specifically to provide realistic web-security attack telemetry without exposing external systems.

---

## 4. Network Reconnaissance

### 4.1 Nmap Service Discovery

The first reconnaissance activity was a targeted service/version scan against TCP port 3000:

```bash
nmap -Pn -sV -p 3000 192.168.10.30
```

The result showed:

```text
3000/tcp open  ppp?
```

Nmap did not identify the application using its standard service database, but the returned HTTP fingerprint clearly demonstrated that the service was an HTTP application.

The response included:

```text
HTTP/1.1 200 OK
Content-Type: text/html; charset=UTF-8
X-Recruiting: /#/jobs
```

The HTTP response also contained the OWASP Juice Shop application HTML.

### Finding

The reconnaissance established that:

* NT-EMP01 was reachable from NT-KALI01.
* TCP/3000 was externally accessible within the lab network.
* An HTTP web application was running on the port.
* The target was confirmed as OWASP Juice Shop.

### Evidence

`/screenshots/attacks/phase-2-reconnaissance-nmap.png`

---

## 5. Web Reconnaissance with Nikto

Nikto was used to perform automated web-server reconnaissance:

```bash
nikto -h http://192.168.10.30:3000
```

The scan identified several interesting characteristics.

### HTTP Security Headers

Nikto reported missing recommended headers including:

```text
Permissions-Policy
Strict-Transport-Security
Referrer-Policy
Content-Security-Policy
```

The application also exposed:

```text
Access-Control-Allow-Origin: *
```

and:

```text
X-Recruiting: /#/jobs
```

### Interesting Resources

Nikto identified:

```text
/ftp/
/public/
/.htpasswd
/.bash_history
/.sh_history
/robots.txt
```

These findings were treated as reconnaissance results rather than automatically being considered exploitable vulnerabilities.

The Juice Shop application intentionally exposes several challenge-related resources.

### Evidence

`/screenshots/attacks/phase-2-web-reconnaissance-nikto.png`

---

## 6. Directory Enumeration with Gobuster

Gobuster was used to enumerate accessible application paths:

```bash
gobuster dir -u http://192.168.10.30:3000 \
-w /usr/share/wordlists/dirb/common.txt \
-t 20 \
--exclude-length 9608
```

The exclusion was necessary because the Juice Shop application returned the same 9608-byte frontend response for nonexistent paths.

The enumeration identified several interesting endpoints:

```text
api
apis
assets
ftp
media
profile
promotion
redirect
rest
restaurants
restored
restore
restricted
robots.txt
video
```

Several API-related paths returned HTTP 500 responses, while resources such as `/ftp/`, `/promotion/`, `/robots.txt`, and `/video` returned successful responses.

### Finding

The enumeration demonstrated that automated directory discovery can identify application functionality and attack surface beyond the main landing page.

### Evidence

The Gobuster results were recorded as part of the Phase 2 attack documentation.

---

## 7. HTTP Inspection with Burp Suite

Burp Suite was used to inspect HTTP traffic between the attacker and the Juice Shop application.

A representative request to the application was captured:

```http
GET / HTTP/1.1
Host: 192.168.10.30:3000
User-Agent: Mozilla/5.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Cookie: language=en; cookieconsent_status=dismiss
Connection: keep-alive
```

The HTTP history confirmed that requests from the Kali browser were reaching the Juice Shop server.

Burp Suite was used primarily for traffic visibility and request inspection rather than as the primary exploitation tool.

### Evidence

`/screenshots/attacks/phase2-burp-http-history.png`

---

## 8. SQL Injection Testing

The Juice Shop product-search API was identified as:

```text
/rest/products/search?q=test
```

The endpoint was first verified using:

```bash
curl -s "http://192.168.10.30:3000/rest/products/search?q=test"
```

The server returned product data successfully.

SQLMap was then used to perform controlled SQL injection testing:

```bash
sqlmap -u "http://192.168.10.30:3000/rest/products/search?q=test" \
--batch \
--level=1 \
--risk=1
```

SQLMap tested the `q` parameter using several basic SQL injection techniques.

The final result was:

```text
GET parameter 'q' does not seem to be injectable
```

SQLMap also recorded multiple HTTP 500 responses during testing.

### Finding

No SQL injection vulnerability was confirmed on the tested `q` parameter using the selected low-risk SQLMap configuration.

This is an important negative finding and should not be documented as a successful SQL injection.

### Evidence

* `screenshots/attacks/phase2-sqlmap-products-search_P1.png`
* `screenshots/attacks/phase2-sqlmap-products-search_P2.png`

---

## 9. Authentication Endpoint Testing

The Juice Shop login endpoint was identified as:

```text
POST /rest/user/login
```

A baseline authentication request was generated with cURL:

```bash
curl -i -X POST http://192.168.10.30:3000/rest/user/login \
-H 'Content-Type: application/json' \
-d '{"email":"test@example.com","password":"wrongpassword"}'
```

The server returned:

```text
HTTP/1.1 401 Unauthorized
```

with:

```text
Invalid email or password.
```

This established the normal failed-authentication response.

---

## 10. Authentication Wordlist Testing

FFUF was then used to test a controlled password list against the known Juice Shop administrative account.

The test wordlist contained:

```text
wrong123
password123
admin123
test123
qwerty123
```

The command used was:

```bash
ffuf -u http://192.168.10.30:3000/rest/user/login \
-X POST \
-H 'Content-Type: application/json' \
-d '{"email":"admin@juice-sh.op","password":"FUZZ"}' \
-w /tmp/juice-passwords.txt \
-mc all \
-fr 'Invalid email or password'
```

FFUF identified:

```text
admin123
```

with:

```text
Status: 200
Size: 784
```

This demonstrated successful authentication using the intentionally vulnerable Juice Shop administrator credentials.

### Security Significance

This simulation demonstrated the risk of weak/default credentials and automated authentication testing.

It also generated realistic authentication activity suitable for subsequent SOC detection and investigation exercises.

### Evidence

`screenshots/attacks/phase2-authentication-testing-ffuf.png`

---

## 11. SSH Authentication Activity

SSH authentication activity was also generated against NT-EMP01 as part of the broader attack simulation.

The activity was observed by Wazuh on NT-EMP01 and generated authentication telemetry.

Relevant events included:

* Failed/successful authentication activity
* PAM login events
* Privilege escalation through `sudo`
* Session open/close events

The Wazuh telemetry demonstrated the complete path:

```text
Kali attack activity
       ↓
NT-EMP01 authentication subsystem
       ↓
Wazuh Agent
       ↓
NT-SOC01 Wazuh Manager
       ↓
SOC telemetry
```

### Evidence

* `screenshots/attacks/phase-2-ssh-authentication-test.png`
* `screenshots/attacks/phase-2-ssh-authentication-telemetry.png`

---

## 12. Phase 2 Findings

Phase 2 successfully demonstrated several attacker behaviors against the isolated lab environment.

### Confirmed Results

1. NT-EMP01 was reachable from NT-KALI01.
2. TCP/3000 exposed an HTTP application.
3. OWASP Juice Shop was successfully identified.
4. Nikto identified several interesting application resources and missing security headers.
5. Gobuster identified multiple accessible application paths.
6. Burp Suite successfully captured HTTP traffic.
7. SQLMap tested the product-search parameter but did not confirm SQL injection.
8. The login API returned a normal 401 response for invalid credentials.
9. FFUF successfully identified the intentionally weak `admin123` credential.
10. Wazuh successfully collected authentication-related endpoint telemetry.

---

## 13. Attack-to-Telemetry Chain

The most important outcome of Phase 2 was establishing the attack-to-telemetry pipeline required for the SOC project.

```text
                  ATTACKER
               NT-KALI01
              192.168.10.20
                    |
          +---------+---------+
          |         |         |
        Nmap     Nikto     Gobuster
          |         |         |
          +---------+---------+
                    |
                    v
             OWASP Juice Shop
                NT-EMP01
             192.168.10.30:3000
                    |
          +---------+---------+
          |                   |
      Web activity       Authentication
          |                   |
          +---------+---------+
                    |
                    v
              Wazuh Agent
                    |
                    v
                NT-SOC01
             192.168.10.10
                    |
                    v
             SOC Detection
              [Phase 3]
```

Phase 2 therefore provides the attack data required for the next stage: detection engineering.

---

## 14. Phase 2 Limitations

Several Juice Shop features reported configuration warnings during startup:

```text
ALCHEMY_API_KEY is not present
```

and:

```text
http://localhost:11434/v1 is not reachable
```

These affect specific Web3 and LLM-related Juice Shop challenges.

They were not required for the Phase 2 reconnaissance and authentication exercises and were therefore intentionally left unresolved.

The SQLMap test also did not identify an injectable parameter. This is recorded as a test result rather than a failure of the lab.

---

## 15. Evidence Summary

| Evidence                                   | Purpose                        |
| ------------------------------------------ | ------------------------------ |
| `phase-2-reconnaissance-nmap.png`          | Network/service reconnaissance |
| `phase-2-web-reconnaissance-nikto.png`     | Web reconnaissance             |
| `phase2-burp-http-history.png`             | HTTP traffic inspection        |
| `phase2-sqlmap-products-search_P1.png`     | SQLMap testing                 |
| `phase2-sqlmap-products-search_P2.png`     | SQLMap result                  |
| `phase2-authentication-testing-ffuf.png`   | Authentication testing         |
| `phase-2-ssh-authentication-test.png`      | SSH authentication activity    |
| `phase-2-ssh-authentication-telemetry.png` | Wazuh authentication telemetry |

---

## 16. Phase 2 Status

**Status: COMPLETE**

Phase 2 established the controlled attacker-to-target workflow required by the Novastra Technologies SOC lab.

The lab now has:

* A dedicated attacker VM.
* A vulnerable web application.
* Network reconnaissance capability.
* Web reconnaissance capability.
* HTTP inspection capability.
* Authentication attack capability.
* SQL injection testing capability.
* Endpoint authentication telemetry.
* Wazuh visibility into attack-related activity.
* Evidence screenshots for portfolio documentation.

The next phase is **Detection Engineering**, where the attack activity generated here will be converted into Wazuh detections, severity levels, MITRE ATT&CK mappings, and SOC investigation workflows.
