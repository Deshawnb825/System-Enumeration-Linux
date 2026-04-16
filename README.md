# 🔍System Recon - Linux Enumeration for Security Assessment

## 🎯Project Objective
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

## 🧠Thought Process

My first instinct was to just run commands. That fell apart immediately - I had no context for what the ouput *meant*, what te threat was, or why any of it mattered.

So I slowed down and used this loop for every section:

1. Run the command
2. Understand what the output is actually telling me
3. Ask - what would an attacker do with this? What should a defender watch for?

Structure I followed: Identity > Networking > Ports/Services > Processes > Users > Files

---

## 🖥️Part 1: System Identity
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

### ⚠️Security Relevance
OS version + codename immediately narrows the CVE search space. An attacker knows exactly what known vulnerabilities to check for Ubuntu 24.04. Noble. The hostname also reveals this is a QUME/ KVM virtual machine - leaking the virtualization platform

---

## 🌐Part 2: Networking
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

### ⚠️Security Relevance
The DNS domain leaks internal lab naming conventions. DNS config could also show if DNSSEC is unsupported - meaning DNS responses are not cryptographically verified, which leaves it open to DNS spoofing if the network were compromised.

---

## 🔌Part 3: Open Ports & Services
**Purpose:** Find what's listening on the network and assess whether it should be.

| Command | What It Does | Why It Matters |
| --- | --- | --- |
| **ss -tulnp** | Lists all listening sockets | Shows every open port and which process owns it |

**Key 'ss -tulnp' output breakdown:**

| Field | Meaning |
| --- | --- |
| **Local addr** | Which interface/IP the service is available on |
| **Peer addr** | Who is allowed / currently connected |
| **Port** | Which service it is |
| **Netid** | Protocol used (TCP or UDP) |

**Command run & output:**
Pictures

**Breaking down what's exposed:**

| Port | Protocol | Service | Exposure | Risk |
| --- | --- | --- | --- | --- |
| 53 (x2) | UDP + TCP | systemd-resolved (DNS) | Loopback only | Low - Internal DNS stub |
| 631 | TCP | cupsd (CUPS printer) | Loopback + IPv6 localhost | Medium - unnecessary on a VM |
| 5353 | UDP | mDNS / printer discovery | All interfaces 0.0.0.0 | Medium - broadcasts on network | 
| 55444 | UDP | Unknown/ephemeral | All interfaces | Worth investigating

### ⚠️Security Relevance
cupsd (CUPS) is running on a VM that has no printer. Port 631 being open and mDNS broadcasting printer discovery (5353) on all interfaces is an unnecessary attack surface. This is the highest-risk finding in the entire recon - a service with no business reason to exist is running and advertising itself on the network.

---

## ⚙️Part 4: Processes & Services
**Purpose:** See what's actually running, who owns it, and confirm findings from the ports section.

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

**Commands run & output:**
Pictures

**Process States:**
| State | Meaning |
| --- | --- |
| 'S' | Sleeping - waiting for input/event |
| 'R' | Running - actively using CPU |
| 'I' | Idle kernel thread |
| 'Ss' | Session leader, sleeping |
| 'R+' | Running in foreground |

**Key observations:**
- cupsd is owned by root - the print daemon runs with full system privilege
- cups-browsed (network printer discovery) runs as its own service user
- systemd-resolved is owned by systemd+ service user - normal
- All kernel threads ([kworker], [rcu_*], etc) are expected - these are normal Linux internals

### ⚠️Security Relevance
cupsd running as root means any exploits in the CUPS service would immediately grant root-level access. Combined with the fact this VM has no printers, this process has no reason to exist here.

---

## 👤Part 5: Users & Permissions
**Purpose:** Enumerate every account, map group membership, and find privilege escalation paths.

| Command | What It Does | Why It Matters |
| --- | --- | --- |
| **whoami** | Current username | Confirms your active identity |
| **id** | Full user info (UID, GID, groups) | Shows your privilege level |
| **groups** | Groups you belong to | Shows what access you have |
| **cat /etc/passwd** | System's user database | Lists all users (including service accounts) |
| **cat /etc/group** | System's group database | Lists all group and members |
| **ls -l** | What files can be accessed | Shows file permissions |
| **sudo -l** | Commands you can run as root | Reveals privilege escalation paths |

**Commands run & output:**
Picture (users)

**'/etc/passwd' entry breakdown:**

| deshawn-test: | x: | 1000: | 1000: | Deshawn_test: | /home/deshawn-test: | /bin/bash |
| --- | --- | --- | --- | --- | --- | --- |
| **user** | **pass** | **UID** | **GID** | **comment** | **homedir** | **shell** |
| Accoutnt name | Stored in /etc/shadow, not here | How the system identifies this user | Primary group ID | Display name / description | Landing directory on login | Shell launched on login - determines login capability |

**Shell = login capability:**
- /bin/bash → interactive login allowed✅
- /usr/sbin/nologin → service account, blocked🔒
- /bin/false → same, blocked🔒

**Real login-capable accounts found:** only *root* and *deshawn-test* - everything else correctly uses /nologin.

**Commands run & output:**
Pciture (groups)

**Key security groups:**
| Group | Access Level |
| --- | --- |
| **sudo** | ⚠️Root access |
| **adm** | ⚠️Log access |
| **docker** | Effectively root on some systems | 
| **wheel** | Root (on some systems) |

**Commands run & output:**
Picture (sudo access)

sudo -l **result**: (ALL : ALL) ALL - this means deshawn-test can run any command as any user with no restrictions. Most permissive sudo config possible.

**Commands run & output:**
Pictures (directory permissions)

**Home directory permissions breakdown:**

| Directory | Permissions | Risk |
| --- | --- | --- |
| Desktop | drwxr-xr-x | Others can read/enter - low risk |
| Downloads | drwxr-xr-x | Others can read/enter - low risk |
| Knowledge-base | drwxrwxr-x | ⚠️Group has write access - anyone in deshawn-test group can modify |
| snap | drwx------ | ✅Private - only owner can access |

### ⚠️Security Relevance
1. (ALL : ALL) ALL in sudo is the most permissive possible config. Compromise this account = instant root, no restrictions.
2. ~/Knowledge-base has group write permissions (rwxrwxr-x) - files inside could be modified by any process running as the deshawn-test group.

---

## 🛡️Security Risk Summary
| Finding | Risk | Recommendation |
| --- | --- | --- |
| cupsd running as root on a VM | 🔴High | sudo systemctl disable --now cups cups-browsed |
| mDNS broadcasting on all interfaces (5353) | 🔴High | Disable avahi-daemon if not needed |
| (ALL : ALL) ALL sudo | 🔴High | Scope sudo rules to only needed commands |
| ~/Knowledge-base has group write permissions | 🟡Medium | chmod 750 ~/Knowledge-base |
| Service accounts all use /nologin | ✅Good | No action needed |
| Only 2 human login accounts | ✅Good | Minimal attack surface |

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
