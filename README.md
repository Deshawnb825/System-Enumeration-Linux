# System Recon - Linux Enumeration for Security Assessment

## Project Objective
Perform a structured **System Reconnaissance** on my Linux VM to understand what information is exposed, what it means, and whether it poses a security risk. This mirrors what both an attacker and a defender would do as a first step.

**Six areas investigated:**

| # | Category | What I Was Looking For |
| --- | --- | --- |
| 1 | System Identity | OS, kernel, hostname, uptime |
| 2 | Networking | Interfaces, IPs, routing, DNS |
| 3 | Open Ports & Services | What's listening and why |
| 4 | Processes | What's actively running and who owns it |
| 5 | Users & Groups | Who can log in, what can they do |
| 6 | File Permissions | What files are accessible and how |

---

## Thought Process

My first instinct was to just run commands. That fell apart immediately - I had no context for what the ouput *meant*, what te threat was, or why any of it mattered.

So I slowed down and used this loop for every section:

1. Run the command
2. Understand what the output is actually telling me
3. Ask - what would an attacker do with this? What should a defender watch for?

Structure I followed: Identity > Networking > Ports/Services > Processes > Users > Files

---

## Part 1: System Identity
**Purpose:** Establish what machine this is, what it's running, and how long it's been up.

| Command | What It Does | Why It Matters |
| ---| --- | --- |
| **hostnamectl** | Full system summary (OS, kernel, hostname) | Reveals OS version, kernel and machine identity |
| **hostname** | Quick machine name only | Fast identification |
| cat /etc/os-release | Detailed OS metadata | Exact version, codename, distro family |
| **uptime** | How long the system has been running | Long uptime = possibly unpatched; recent reboot = possible incident response or compromise |

**Commands run & output:**
Pictures:

**What this tells me:**

| Field | Value | Significance |
| --- | --- | --- |
| OS | Ubuntu 24.04.4 LTS | Long-term support release - tells you exact patch window |
| Codename | noble | Determines available package versions |
| Uptime | 54 minutes | Recently booted - patches likely current |
| Load avg | 0.50 / 0.53 / 0.24 | System is idle, no unusual CPU pressure |

### Security Relevance
OS version + codename immediately narrows the CVE search space. An attacker knows exactly what known vulnerabilities to check for Ubuntu 24.04. Noble. The hostname also reveals this is a QUME/ KVM virtual machine - leaking the virtualization platform

---

## Part 2: Networking
**Purpose:** Map out every interface, IP, route, and DNS path on the system.

| Command | What It Does | Why It Matters | 
| --- | --- | --- |
| **ip a** | Shows network interfaces and IP addresses | Reveals all IPs the machine is reachable on |
| **ip r** | Shows the routing table | Shows how traffic flows in/out |
| **/etc/resolv.conf** | What DNS server is in use | Shows upstream DNS configuration |
| **resolvectl status** | Upstream DNS server details | Reveals stub resolver + upstream chain |

**Commands run & output:**
Pictures

**Network layout:**

| Component | Value | Notes |
| --- | --- | --- |
| Loopback (lo) | 127.0.0.1 | Internet only - not reachable externally |
| Primary interface | ens18("enp0s18") | Internet-facing NIC |
| System IP | 10.x.x.x | Internal network address |
| Subnet | /24 > 255.255.255.0 | 254 usable hosts on this segment |
| Gateway | 10.x.x.1 | pfSense router |

DNS

**DNS resolution chain:**

| Step | Address | Role |
| --- | --- | --- |
| 1 - App sends query | 127.0.0.53 | Local stub resolver | 
| 2 - Stub forwards to | 10.x.x.1 | pfSense router - upstream DNS |
| 3 - Router resolves | External DNS | Final answer returned back up the chain |

**Key 'ss -tulnp' output breakdown:**

| Field | Meaning |
| --- | --- |
| **Local addr** | Which interface/IP the service is available on |
| **Peer addr** | Who is allowed / currently connected |
| **Port** | Which service it is |
| **Netid** | Protocol used (TCP or UDP) |

### Security Relevance
The DNS domain leaks internal lab naming conventions. DNS config could also show if DNSSEC is unsupported - meaning DNS responses are not cryptographically verified, which leaves it open to DNS spoofing if the network were compromised.

---

## Part 3: Processes & Services
**Purpose:** See what's actively running on the system, who's running it, and how it's managed.

| Command | What It Does | Why It Matters |
| --- | --- | --- |
| **ps** | Shows currently active processes | Snapshot of what's running |
| **ps aux** | All users ('a'), user-oriented format ('u'), no terminal ('x') | Full view of every process |
| **ps -ef** | Process hierarchy / parent-child tree | Understand how processes are spawned |
| **ps aux \| grep -E 'systemd-resolved\|cups' | Filter processes by name | Targeted search for specific services |
| **systemctl** | What the system manages as services | Lists all system-managed units |

**Pipe & grep breakdown:**
- '|' - takes output from the left command and feeds it to the right command
- 'grep' - searches for text patterns
- '-E' - extend patterns (OR conditions, more flexible matching)
- '-v' - invert match (exclude results)

**Process States:**
| State | Meaning |
| --- | --- |
| 'S' | Sleeping |
| 'R' | Running |
| 'Z' | Zombie (dead but not cleaned up) |
| 'Ss' | System process |

### Security Relevance
Malware often hides as a background process. Zombie processes can indicate crashed services. Processes running as 'root' that shouldn't  be are a **red flag**. Comparing **expected services** vs.  **what's running** helps detect intrusions.

---

## Part 4: Users & Permissions
**Purpose:** Understand who exists on the system, what groups they belong to, and what they can do.

| Command | What It Does | Why It Matters |
| --- | --- | --- |
| **whoami** | Current username | Confirms your active identity |
| **id** | Full user info (UID, GID, groups) | Shows your privilege level |
| **groups** | Groups you belong to | Shows what access you have |
| **cat /etc/passwd** | System's user database | Lists all users (including service accounts) |
| **cat /etc/group** | System's group database | Lists all group and members |
| **ls -l** | What files can be accessed | Shows file permissions |
| **sudo -l** | Commands you can run as root | Reveals privilege escalation paths |

**'/etc/passwd' entry breakdown:**

| root | x | 0 | 0 | root | /root | /bin/bash |
| --- | --- | --- | --- | --- | --- | --- |
| **user** | **pass** | **UID** | **GID** | **comment** | **homedir** | **shell** |

-'/bin/bash' = can log in
-'/usr/sbin/nologin' = **cannot** log in (service account)

**Key security groups:**
| Group | Access Level |
| --- | --- |
| **sudo** | Root access |
| **adm** | Log access |
| **docker** | Effectively root on some systems | 
| **wheel** | Root (on some systems) |

**Findings from the exercise:**

- Most users wer **service accounts** (not real humans)
- Only **2 real users**: 'root and 'deshaun-test'
- 'deshaun-test' has **privilege escalation** via 'sudo' - can get root access with a password
- Sensitive files: '/etc/shadow', '/etc/passwd', '/etc/sudoers', '/etc/group'

### Security Relevance
- Accounts with 'sudo' access are high-value targets
- Service accounts that can log in ('/bin/bash') are a risk - they should use '/nologin'
- 'docker' group = essentially root - a major privilege escalation vector
- '/etc/shadow' stores hashed passwords - if readable, passwords can be cracked offline
- Unexpected users = possible persistence mechanism after a breach

---

## Overall Security Risk Summary
| Category | Risk | Notes |
| --- | --- | --- |
| Kernel version exposed | Medium | Can be matched to known CVEs |
| Long uptime | Medium | May indicate unpatched system |
| Open ports (ss -tulnp) | High | 'deshaun-test' can escalate to root |
| docker group membership | Critical | Equivalent to root access |
| Readable /etc/passwd | Low-medium | Usernames enumerable; shadow files is the real risk |
| Processes running as root | Medium-High | Any compromise = full system access |
| Service accounts with login shells | Medium | Unnecessary login capability is a risk |

---

## What I Learned
1. You can *run* recon commands without understanding them - but that's useless without context
2. Every piece of system info has a **threat model**: who would want it, and what would they do with it?
3. The most dangerous things on a system are often obvious - a misconfigured group membership or an old kernel can be more dangerous than anything visible
4. Recon is the **first step** in both attack and defense - knowing your own system's exposure is essential 
