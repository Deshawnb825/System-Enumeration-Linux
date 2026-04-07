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
The system is running Ubuntu 24.04 LTS, which is a long-term support release that receives regular security updates.

The codename "noble" determines the repository sources used by the system, ensuring proper package compatibility and updates.

This system is generally secure if properly maintained, but outdated packages can still introduce vulnerabilities.

---

## Security Implications
-The OS version can be used by attackers to identify known vulnerabilities

-If updates are not applied regularly, the system may become exposed to exploits

-Misconfigured repositories may introduce insecure or outdated packages

---

##Next Step (Vulnerability Check)
To determine if this system is vulnerable:

1. Check Known vulnerabilities:

     -https://ubuntu.com/security/cves

     -Search: Ubuntu 24.04 noble

2. Check for missing updates:

-sudo apt update && sudo apt list --upgradable

3. Review outdated packages for potential risk exposure
