# Penetration Test & Vulnerability Assessment Report

## 1. Executive Summary

The recent security assessment of the internal network segment (`10.0.2.0/24`) revealed a critical vulnerability stemming from inadequate access controls and deficient authenticator management. An adversary successfully compromised a core Ubuntu 24.04 LTS host (`10.0.2.10`) in under 60 seconds by exploiting a weak password policy on an externally exposed Secure Shell (SSH) interface. This unauthorized access grants a malicious actor full administrative control, presenting an immediate risk of data exfiltration, systemic ransomware deployment, and lateral movement across the corporate domain.

From a business risk perspective, this vulnerability represents a severe breakdown in fundamental cybersecurity hygiene. The absence of rate-limiting and the reliance on rudimentary, easily guessable credentials leave the organization exposed to automated exploitation campaigns. If left unmitigated, this vector severely threatens the confidentiality, integrity, and availability of corporate assets, and exposes the organization to significant compliance penalties under frameworks such as ISO 27001 and PCI DSS. Immediate remediation is required to halt potential persistent threats.

## 2. Assessment Details
- **Target System:** Ubuntu 24.04 LTS (IP: `10.0.2.10`)
- **Attacking Node:** Kali Linux (IP: `10.0.2.15`)
- **Vulnerable Service:** OpenSSH 9.6p1 (Port 22/TCP)
- **Time to Compromise:** < 60 seconds

## 3. Vulnerability Analysis & Risk Scoring

### 3.1. Vulnerability: Insecure Remote Access via Weak Authentication
The target host permitted password-based authentication over SSH without implementing protective measures against entropy-based credential exhaustion attacks.

- **CVSS v3.1 Base Score:** **9.8 (Critical)** 
- **Vector String:** `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H`
- **Justification:**
  - **Ease of Exploit (AV:N/AC:L/PR:N/UI:N):** The vulnerability is network-accessible. The attack complexity is trivially low, requiring no prior privileges or user interaction. Off-the-shelf automated exploitation tools (e.g., THC Hydra) can independently execute this attack.
  - **Impact on Confidentiality, Integrity, & Availability (C:H/I:H/A:H):** Complete loss of confidentiality. The compromised account (`john`) provides an interactive shell, enabling unauthorized access to the filesystem, potential privilege escalation, and access to sensitive data stores. Full system compromise enables the attacker to install persistence mechanisms, tamper with log integrity, or trigger denial-of-service conditions.

## 4. Technical Attack Narrative (Kill Chain)

**Phase 1: Reconnaissance & Enumeration**
Network scanning identified an active OpenSSH 9.6p1 listener on Port 22. Service enumeration confirmed that the host accepted password-based authentication, exposing a viable attack surface for automated password spraying and brute-forcing.

![Nmap SSH Discovery](./screenshots/nmap_ssh_discovery.jpeg)

**Phase 2: Exploitation & Credential Exhaustion**
An aggressive, entropy-based credential exhaustion attack was executed using dictionary payloads against the target. Due to the absence of rate-limiting or lockout mechanisms, the system sustained high-velocity authentication requests, yielding the valid credential pair (`john`:`12345`) in under a minute.

![Hydra Brute-Force Exploit](./screenshots/hydra_bruteforce_exploit.jpeg)

**Phase 3: Post-Exploitation & Impact**
Using the compromised credentials, an interactive remote session was established. This unauthorized lateral movement grants the attacker a permanent foothold. The level of access obtained acts as a staging ground for deploying persistence mechanisms (e.g., cron jobs, authorized_keys manipulation), privilege escalation vectors, and potential ransomware payloads.

![SSH Successful Login](./screenshots/ssh_successful_login.jpeg)

**Phase 4: Defensive Evasion & Log Integrity Verification**
Forensic analysis of `/var/log/auth.log` revealed clear indicators of compromise (IoCs), transitioning from sustained "Failed password" entries from `10.0.2.15` to an "Accepted password" entry. However, an attacker with this level of access possesses the capability to alter or systematically purge syslog events, thereby erasing the audit trail and severely hindering incident response capabilities.

![Forensic Auth Log Analysis](./screenshots/forensic_auth_log_analysis.jpeg)

## 5. Framework Alignment

### 5.1. MITRE ATT&CK Mapping
- **T1110.001 - Brute Force: Password Guessing:** Adversaries iteratively guessed passwords to obtain valid credentials.
- **T1078.003 - Valid Accounts: Local Accounts:** Utilization of a compromised local account (`john`) to establish a persistent session.
- **T1021.004 - Remote Services: SSH:** Exploitation of the SSH protocol to execute unauthorized lateral movement.
- **T1070.001 - Indicator Removal on Host: Clear Windows Event Logs / Syslog:** The achieved access level allows the adversary to perform log integrity tampering.

### 5.2. NIST 800-53 Security Controls
- **IA-5 (Authenticator Management):** The system fails to enforce sufficient password complexity and entropy requirements.
- **AC-7 (Unsuccessful Logon Attempts):** The system lacks enforced limits on consecutive invalid logon attempts, permitting unbounded brute-force attacks.
- **AU-6 (Audit Review, Analysis, and Reporting):** The lack of automated real-time alerting on sustained authentication failures demonstrates a gap in the defensive posture.

## 6. GRC Context & Compliance Impact

The exploitation of this vulnerability constitutes a material breach of several industry-standard compliance and regulatory frameworks:

- **ISO/IEC 27001:**
  - **A.9.2.1 (User Registration and De-registration):** Fails to secure credential lifecycle management.
  - **A.9.4.3 (Password Management System):** Violates the requirement for an interactive system that enforces quality passwords.
  - **A.12.4.1 (Event Logging):** While logs exist, the lack of automated detection and response capabilities fails to adequately protect against ongoing attacks.

- **PCI DSS v4.0:**
  - **Requirement 8.3.4:** Mandates that invalid authentication attempts must be limited (typically blocking access after no more than 10 attempts). The unchecked brute-force attack is a direct violation.
  - **Requirement 8.3.6:** Mandates strong cryptography and minimum complexity for passwords. The credential (`12345`) fails to meet fundamental complexity requirements, risking significant financial penalties or loss of merchant processing abilities.

## 7. Remediation & Hardening Directives

To immediately mitigate these severe risks, the following corrective actions must be deployed:

1. **Enforce Asymmetric Cryptography (Immediate):**
   - Disable password-based authentication universally by configuring `PasswordAuthentication no` and `PermitRootLogin prohibit-password` within `/etc/ssh/sshd_config`. Mandate ED25519 or RSA-4096 key-based authentication.

2. **Implement Intrusion Prevention Systems (Immediate):**
   - Deploy rate-limiting solutions such as `Fail2Ban` or `CrowdSec` to automatically drop network traffic (via `iptables`/`nftables`) from sources exhibiting abnormal authentication failure velocity.

3. **Centralized Log Aggregation (Short-term):**
   - Forward all authentication logs (`/var/log/auth.log`) to an isolated, tamper-proof SIEM environment to ensure log integrity verification and to establish real-time SOC alerting for anomalous access patterns.

4. **Credential Auditing (Short-term):**
   - Conduct a comprehensive audit of all local and domain accounts to purge weak, default, or compromised credentials.