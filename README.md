# System Recon - Linux Enumeration for Security Assessment

## Project Objective
The goal of this exercise was to perform system reconnaissance, the process of gathering key information about a Linux system to understand its configuration and identify potential security risks. This is a foundational skill in both system administration and cybersecurity (blue team and red team alike).

**Information targeted:**
- System identity (hostname, OS, kernel)
- Active users and their permissions
- Security groups and access controls
- Open ports and listening services
- Running processes and services
- Installed packages and updates

---

## Thought Process

I started by looking up commands and running them, but quickly realized I had no context for what the output *meant*. I didn't understand the purpose, the threat implications, or the importance of what I was seeing. So I slowed down and focused on understanding each category before moving on.

The learning progression was:
1. **Identity → Networking → Processes →Users** (structured top-down)
2. Run commands → understand output → document what it reveals
3. Ask: *What would an attacker learn from this? What would a defender watch for?*

---

## Part 1: System Identity
**Purpose:** Understand what machine you're on - OS, kernel version, name , and uptime.

| Command | What It Does | Why It Matters |
| ---| --- | --- |
| **hostnamectl** | Full system summary (OS, kernel, hostname) | Reveals OS version, kernel and machine identity |
| **hostname** | Quick machine name only | Fast identification |
| **uptime** | How long the system has been running | Long uptime = possibly unpatched; recent reboot = possible incident response or compromise |

**Key concepts learned:**
- **Hostname** = the name of the system
- **Kernel** = communicates directly with hardware; has the highest privilege level on the system
- **Codename** = determines what package versions are available (e.g., 'jammy', 'bookworm')
- **Uptime** = tells you how long since the last reboot - critical for patch assessment

### Security Relevance
A system with very long uptime liekly hasn't applied kernel patches. Kernel version info can reveal known CVEs. OS/codename can tell an attacker what exploits to target.

---

## Part 2: Networking
**Purpose:** Understand how the system is connected - interfaces, IPs, routing, open ports, DNS.

| Command | What It Does | Why It Matters | 
| --- | --- | --- |
| **ip a** | Shows network interfaces and IP addresses | Reveals all IPs the machine is reachable on |
| **ip r** | Shows the routing table | Shows how traffic flows in/out |
| **/etc/resolv.conf** | What DNS server is in use | Shows upstream DNS configuration |
| **resolvectl status** | Upstream DNS server details | Reveals stub resolver + upstream chain |
| **ss -tulnp** | Lists listening ports (sockets) | Reveals what services are exposed |

**Key 'ss -tulnp' output breakdown:**

| Field | Meaning |
| --- | --- |
| **Local addr** | Which interface/IP the service is available on |
| **Peer addr** | Who is allowed / currently connected |
| **Port** | Which service it is |
| **Netid** | Protocol used (TCP or UDP) |

### Security Relevance
Open ports = attack surface. A port in 'LISTEN' state means something is accepting connections. Unexpected open ports can indicate malware, backdoors, or misconfigured services. DNS config can reveal if DNS is being intercepted or spoofed.

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
