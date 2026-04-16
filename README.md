# Linux System Security Assessment - Ubuntu 24.04 VM

> **Assessment Date:** April 7, 2026
>
> **Target:** Ubuntu 24.04.4 LTS (Noble Numbat) - QEMU/KVM Virtual Machine
>
> **Assessed By:** deshawn-test
>
> **Methodology:** Manual enumeration using native Linux tooling - no automated scanners

---

## Executive Summary
I performed a full manual security assessment of a Linux VM across six domains: system identity, network configuration, exposed services, running processes, user privileges, and file permissions. The goal was not just to enumerate the system, but to evaluate each finding through a threat lens - asking what an attacker could do with it and what the real-world impact would be.

**Three findings warrant immediate remediation:**

| Severity | Finding | Business Impact |
| --- | --- | --- |
| 🔴 Critical | 'cupsd' (CUPS) running as root with no business justification | Any CUPS exploit on this VM = instant root shell |
| 🔴 Critical | Unrestricted 'sudo' - '(ALL : ALL) ALL' | One compromised password = full system takeover |
| 🔴 Critical | Sensitive recon notes stored on the assessed target itself | Attacker gains access = reconnaissance already done for them |

**Two findings require hardening:**

| Severity | Finding |
| --- | --- |
| 🟡 Medium | mDNS broadcasting printer discovery on all interfaces |
| 🟡 Medium | '~/Knowledge-base' directory has group write permissions |

**What is working correctly:** Service accounts are properly locked with '/nologin'. The attack surface for human logins is minimal - only two accounts can authenticate interactively. DNS resolution chain is standard and expected for this lab environment.

---

## Methodology
Manual enumeration following a structured threat-modeling approach. For each domain, I identified what information was exposed, who could leverage it, and what the realistic attack path would be - not just whether something existed, but whether it *should* exist and what damage it enables.

Enumerate → Contextualize → Threat model → Prioritize by impact

Tools used: hostnamectl, cat /etc/os-release, uptime, ip a, ip r, cat /etc/resolv.conf, resolvectl status, ss -tulnp, ps aux | grep, cat /etc/passwd, cat /etc/group, sudo -l, ls -l

---


---
## 🛠️Remediation Steps:

### 1. Disable the CUPS print service entirely
- sudo systemctl disable --now cups cups-browsed
### 2. Disable mDNS printer broadcasting
- sudo systemctl disable --now avahi-daemon
### 3. Tighten Knowledge-base permissions (remove group write)
- chmod 750 ~/Knowledge-base
### 4. Verify /etc/shadow is only root-readable
- ls -l /etc/shadow
- SHould show: -rw-r----- root shadow
### 5. Audit and scope sudo rules (edit with visudo)
- sudo visudo
- Replace "(ALL : ALL) ALL" with specific allowed commands


## 📚What I Learned
> **"Running commands without understanding the output is useless - context is everything."**

1. The most dangerous finding wasn't a user or password - it was cupsd: a service with zero reason to exist on this VM, running as root, advertising itself on the network
2. My own ~/Knowledge-base is a liability. Storing recon notes on the target machine means an attacker who gains access skips the reconnaissance phase entirely - I've done it for them
3. (ALL : ALL) ALL in sudo is red flag even for personal machines. It's a one-password path to full root
4. Recon is the first step in both offense and defense. Learning to read your own system the way an attacker would is one of the most practical security skills you can build
