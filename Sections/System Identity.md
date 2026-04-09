# OS Information

## Objective
Identify the operating system, version, and distribution details to assess system relevance, support status, and potential vulnerabilities.

## Commands Used
-cat /etc/os-release

-hostnamectl

---

## Findings
-OS: Ubuntu 24.04 LTS

-Distribution: Ubuntu

-Codename: noble

---

## Analysis:
The system is running Ubuntu 24.04 LTS, which is actively supported and receives regular security updates.

However, system security depends on patch status rather than OS version alone. At this stage, update status has not yet been verified, meaning the system may still be exposed to known vulnerabilities if patches are missing.

Further validation is required to determine whether the system is fully up to date.

---

## Security Implications
-OS fingerprinting allows attackers to map the system to known CVEs

-If patching is inconsistent, publicly known exploits may be applicable

-Attackers may target default services and packages commonly associated with this OS version

---

##Next Step (Vulnerability Check)
To determine if this system is vulnerable:

1. Check Known vulnerabilities:

     -https://ubuntu.com/security/cves

     -Search: Ubuntu 24.04 noble

     -Goal: Identify CVEs affecting this OS version

3. Check for missing updates:

     -sudo apt update
   
     -sudo apt list --upgradable

4. Review outdated packages for potential risk exposure
