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

## Finding 1 - Unnecessary Service Running as Root (CUPS)
**Severity: 🔴 Critical**

This is the highest-risk finding on the system. Two CUPS processes are actively running with no justification - this is a virtual machine with no printers attached.

Picture: (Processes)
Picture: (Ports)

**Why this is critical, not just a misconfiguration:**

'cupsd' runs as **root** (PID 898). The CUPS print system has a documented history of serious vulnerabilities - including remote code execution. CVE-2024-47176, disclosed in late 2024, allowed unauthenticated remote attackers to execute arbitrary commands via 'cups-browsed' on port 631. That exact binary is running on this system right now.

Even with 'cupsd' bound to loopback, 'cups-browsed' listens for mDNS advertisements on '0.0.0.0:5353' - broadcasting across the entire network segment. Any host on the '10.x.x.0/24' subnet can see this machine is running a print service. That's 254 potential pivot points if any other host on the network is compromised.

The attak chain: network-adjacent attacker → mDNS discovery → cups-browsed exploitation → root shell. All enabled by a service this machine has zero reason to run.

**Remediation:**
- sudo systemctl disable --now cups cups-browsed avahi-daemon

---

## Finding 2 - Unrestricted Sudo Configuration
**Severity: 🔴 Critical**

Picture: (sudo -l)

'(ALL : ALL) ALL' is the most permissive sudo policy possible. It means 'deshawn-test' can run any command, as any user, on any host - with only a password standing between a normal session and full root access.

**The threat model her is specific:** this isn't a theoretical risk. 'deshawn-test' is the only human account on the system besides root. If an attacker gains a foothold through any vulnerability - a phishing link opened in the browser, a malicious package, a vulnerabile web app - they land as 'deshawn-test'. From there, 'sudo su' or 'sudo bash' completes the privilege escalation. No exploit required. One password prompt.

The principle of least privilege exists percisely to prevent this. A properly scoped sudo policy would allow only the specific commands this user legitimately needs - not a blank check.

**Remediation:**
- sudo visudo
    - Replace: (ALL : ALL) ALL
    - With specific rules
 
---

## Finding 3 - Recon Intelligence Stored on Target
**Severity: 🔴 Critical**

Picture: (ls -l)

A directory named 'knowledge-base' exists in the home directory and contains notes about this system - the exact recon that was just performed. The permissions (drwxrwxr-x) allows any process running under the 'deshawn-test' group to modify its contents.

**Why this matters operationally:** Reconnaissance is the most time-consuming phase of an attack. An attacker who gains access to this account doesn't need to enumerate the system - the work is already done and documented. They have the network layout, the open ports, the user list, the privilege paths, and the sensitive file locations handed to them on arrival.

This is an operational security failure independent of any technical vulnerability. Sensitive documentation about a system should never live on the system.

**Remediation:**
- Tighten permissions immediately
    - chmod 750 ~/Knowledge-base
- Long term: move sensitive notes off the target entirely
- Store in an encrypted vault, separate machine, or offline

---

## Finding 4 - mDNS Network Exposure
**Severity: 🟡 Medium**

Picture: (Ports)

The Avahi mDNS daemon is advertising service discovery across the entire network segment. Any host on the subnet can query for and discover services running on this machine. In an isolated lab this is low consequence, but it represents unnecessary network visibility.

In a real environment, mDNS advertisements have been used as an initial reconnaissance vector - an attacker who lands anywhere on the network can passively map service exposure without ever touching the target machine. 

**Remediation:**
- Disable 'avahi-daemon' (covered in Finding 1 remediation above).

---

## Finding 5 - Knowledge-base Directory Write Permissions
**Severity: 🟡 Medium**

Pciture:(file)

The group write bit ('w' in position 5) means any process running as the 'deshawn-test' group can create, modify, or delete files inside this directory. In the context of a compromise, this enables an attacker to tamper with or plant files in a directory that appears to be a trusted personal notes store.

**Remediation:**
- chmod 750 ~/Knowledge-base

---

## Supporting Evidence - System Profile
### System Identity

Picture: (OS version)

Picture: (uptime)

| Field | Value | Security Relevance |
| --- | --- | --- |
| OS | Ubuntu 24.04.4 LTS | Narrows applicable CVE search to Noble Numbat package versions |
| Uptime | 54 minutes | Recently booted - kernel patches likely current |
| Hostname | 'deshawn-test-xxxxxxxxxxxxxxxxxxx | Leaks virtualization platform (QEMU/KVM i440FX chipset) |
| Load avg | 0.50 / 0.53 / 0.24 | Idle - no anomalous CPU activity |

### Network Configuration

Picture: (IP a)

Picture: (IP r)

| Component | Value | Notes |
| --- | --- | --- |
| Interface | 'ens18' (alt: 'enp0s18') | Single external-facing NIC |
| IP | '10.x.x.x/24' | Internal network - 254 hosts on segment |
| Gateway | '10.x.x.1' | pfSense router |
| IPv6 | Link-local only | Not globally routable |

Picture : (DNS)

- DNS flows from the local stub resolver through pfSense upstream.

### User & Privilege Inventory

Picture: (users)

Picture: (groups)

**Positive finding:** All service accounts use '/nologin' or '/bin/false'  - no unnecessary interactive login capability exists for system daemons. This is correct hardening practice and limits the blast radius of any single service compromise.

---

## Remediation Priority Matrix

| Priority | Action | Command | Impact |
| --- | --- | --- | --- |
| 1 - Immediate | Disable CUPS + Avahi | 'sudo systemctl disable --now cups cups-browsed avahi-daemon' | Eliminates root-level attack surface and network visibility |
| 2 - Immediate | Scope sudo policy | 'sudo visudo' → replace 'ALL' with specific commands | Breaks the one-step privilege escalation path |
| 3 - Immediate | Move Knowledge-base off target | Manual - relocate to encrypted vault or separate host | Eliminates recon intelligence exposure |
| 4 - Short-term | Fix directory permissions | 'chmod 750 ~/Knowledge-base' | Removes unauthorized write access |

---

## Key Takeaways
- The most significant lesson from this assessment is about **attack surface reduction**. The three critical findings share a common thread - they all represent things that exist with no operational justification. A print service on a VM with no printers. Unrestricted root access for an account that doesn't need it. Sensitive documentation stored in exactly the place an attacker would look.

- Security isn't only about finding vulnerabilities in software. It's about challenging whether each component of a system *earns its place*. Every unnecessary service, every overpermissioned account, every piece of sensitive data in the wrong location is a liability - regardles of whether a CVE has been published for it.

- The CUPS finding is a direct example of why this matters: CVE-2024-47176 was disclosed in September 2024 specifically targeting 'cups-browsed'. A system running this service not because it needs to, but because it was never explicitly disabled, is the exact scenario that vulnerability was written for.
