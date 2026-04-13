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
| **uptime** | How long the system has been running since last reboot | Long uptime = possibly unpatched; recent reboot = possible incident response or compromise |
