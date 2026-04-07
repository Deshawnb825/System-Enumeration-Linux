# Kernel Information

## Objective
Identify the kernel version and architecture to evaluate patch level, privilege exposure, and potential vulnerability risk.

---

## Comands Used
-uname -a

-uname -r

---

## Findings
-Kernel Version: 6.17.0-19-generic

-Architecture: x84_64

---

## Analysis
The kernel is the core component of the operating system and operates with the highest level of privilege.

The system is running kernel version 6.17.0-19-generic, which is relatively modern. However, kernel security is dependent on patch status rather than version alone.

At this stage, it has not been verified whether this kernel version includes all relevant security patches, meaing potential vulnerabilities may still exist.

Kernel versions should be validated against knwon vulnerabilities (CVEs) to determine actual exposure.

---

## Security Implications
-Kernel vulnerabilities can lead to privilege escalation and full system compromise

-Attackers often target kernel-level exploits after gaining initial access

-An unpatched kernel significantly increases the risk of local exploitation

---

## Next Step (Vulnerability Check)
To determine if this kernel is vulnerable:

1. Check known kernel vulnerabilities:

  -https://nvd.nist.gov/
  -Search: "Linux kernel 6.17.0"

2. Check distribution-specific vulnerabilities:
   
  -https://ubuntu.com/security/cves
  
  -Search: kernel version or related packages

4. Verify patch/updates status:

  -uname -r

  -apt list --installed | grep linux-image

Goal: Confirms installed kernel version matched latest patched release

---

## Current Risk Assessment
At this stage, not direct kernel vulnerabilities have been confirmed. However, patch status has not yet been verified, meaning the system may still be exposed if updates are missing.
