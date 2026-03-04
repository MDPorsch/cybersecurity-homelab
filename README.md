# Exercise 01 — Linux CLI Fundamentals
### Cybersecurity Homelab Series | Building in Public

---

##  Overview

**Date:** March 4, 2026
**Environment:** Ubuntu VM (VirtualBox on MacBook M4)
**Difficulty:** Beginner
**Time Taken:** ~45 minutes

This is the first exercise in my cybersecurity homelab series. Before you can hack anything or defend anything, you need to be comfortable in the Linux terminal, because almost everything in security happens there. This exercise covers the fundamentals: navigating the filesystem, managing files, understanding permissions, managing users, reading processes, and automating tasks with cron.

---

##  Objectives

- Navigate the Linux filesystem confidently
- Create, read and manage files from the terminal
- Understand and modify file permissions
- Create and manage user accounts
- Read and interpret running system processes
- Set up an automated cron job

---

##  Tools & Environment

| Item | Detail |
|------|--------|
| Host Machine | MacBook M4 |
| VM Platform | VirtualBox |
| Operating System | Ubuntu (Linux) |
| Terminal | Bash |
| User | mdubuntu |

---

##  Step-by-Step Walkthrough

### Step 1 — Orienting Yourself

The first thing you do when you land on any Linux system — whether you're a sysadmin or an attacker — is figure out where you are and who you are.

```bash
pwd && whoami
```

This returned `/tmp` and `mdubuntu`, meaning my terminal had opened in the `/tmp` directory. I navigated home immediately:

```bash
cd ~ && pwd
```

Output: `/home/mdubuntu` — that's my home directory, my base of operations.



---

### Step 2 — Listing Files & Understanding the Home Directory

```bash
ls -la
```

The `-l` flag gives a detailed list and `-a` shows hidden files (files starting with `.`). Running this revealed some interesting things already present on my system:

- A **Diamorphine** directory — a rootkit from a previous exercise
- **Wazuh agent** `.deb` files — a SIEM tool I had previously installed
- A `.ssh` directory — SSH keys already configured
- Hidden config files like `.bashrc`, `.bash_history`, `.profile`



> **Why this matters:** When a blue teamer investigates a compromised system, `ls -la` in the home directory is one of the first commands they run. Hidden files and unexpected directories are red flags. Attackers love hiding malicious files as dotfiles (e.g. `.malware`) because basic `ls` won't show them — you need `ls -la`.

---

### Step 3 — Creating Files & Writing to Them

I created a working directory for this exercise and my first file:

```bash
mkdir -p ~/lab/ex01 && cd ~/lab/ex01 && pwd
touch notes.txt
echo "Hello Homelab" > notes.txt
cat notes.txt
```

Output: `Hello Homelab`

Breaking this down:
- **`mkdir -p`** — creates the directory and any parent directories needed
- **`touch`** — creates an empty file
- **`echo "text" > file`** — writes text into a file (`>` overwrites, `>>` appends)
- **`cat`** — reads and displays file contents



---

### Step 4 — Understanding & Changing File Permissions

This is one of the most important concepts in Linux security.

```bash
ls -l notes.txt
```

Output:
```
-rw-rw-r-- 1 mdubuntu mdubuntu 14 Mar 4 12:56 notes.txt
```

Here's how to read the permission string:

```
-  rw-  rw-  r--
|   |    |    |
|   |    |    └── Everyone else — READ only
|   |    └─────── Group — READ & WRITE
|   └──────────── Owner — READ & WRITE
└──────────────── File type (- = regular file)
```

I then locked the file down to owner-only access:

```bash
chmod 600 notes.txt && ls -l notes.txt
```

Output:
```
-rw------- 1 mdubuntu mdubuntu 14 Mar 4 12:56 notes.txt
```



Then opened it up with execute permissions:

```bash
chmod 755 notes.txt && ls -l notes.txt
```

Output:
```
-rwxr-xr-x 1 mdubuntu mdubuntu 14 Mar 4 12:56 notes.txt
```



**chmod number reference:**

| Number | Permissions |
|--------|-------------|
| 7 | Read + Write + Execute |
| 6 | Read + Write |
| 5 | Read + Execute |
| 4 | Read only |
| 0 | No permissions |

> **Why this matters:** Misconfigured permissions are one of the most common privilege escalation vectors. SSH private keys MUST be `600` — Linux will refuse to use them otherwise. Attackers scan for world-writable files (`777`) as easy targets to plant malicious code.

---

### Step 5 — User Management

First I checked my own identity and privileges:

```bash
id
```

Output:
```
uid=1000(mdubuntu) gid=1000(mdubuntu) groups=1000(mdubuntu),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),100(users),116(lpadmin)
```



Key observation — I'm in the **sudo group (27)**. This means I can run commands as root. In a privilege escalation attack, getting added to the sudo group is often the end goal.

I then created a new unprivileged user:

```bash
sudo adduser labuser
```



After creation I verified the user in `/etc/passwd`:

```bash
cat /etc/passwd | grep labuser
```

Output:
```
labuser:x:1001:1001:,,,:/home/labuser:/bin/bash
```

Breaking this down:
```
labuser : x    : 1001 : 1001 :     : /home/labuser : /bin/bash
   |       |      |      |            |                  |
username  pwd   uid   gid        home dir           default shell
```

The `x` in the password field means the actual password hash is stored in `/etc/shadow` — readable only by root.

I switched into the new user and checked their privileges:

```bash
su - labuser
id
```

Output:
```
uid=1001(labuser) gid=1001(labuser) groups=1001(labuser),100(users)
```



Notice **no sudo group** — labuser is completely unprivileged. Then switched back:

```bash
exit
whoami
```

---

### Step 6 — Reading Running Processes

```bash
ps aux
```


This lists every process running on the system. Key columns:

| Column | Meaning |
|--------|---------|
| USER | Who owns the process |
| PID | Process ID (unique number) |
| %CPU | CPU usage |
| %MEM | Memory usage |
| COMMAND | What's actually running |

Interesting processes I spotted on my system:

```bash
root  2529  /usr/bin/suricata        # Network IDS — already installed!
root  2536  /var/ossec/bin/wazuh-execd   # Wazuh SIEM agent running
wazuh 2544  /var/ossec/bin/wazuh-agentd
mdubuntu 3051  [sh] <defunct>        # A zombie process
```

> **Why this matters:** `ps aux` is one of the first commands run after compromising a system — attackers use it to identify security tools to avoid, or vulnerable services to target. Blue teamers hunt for unexpected or suspicious processes. Zombie processes (`<defunct>`) can sometimes indicate something unusual.

---

### Step 7 — Automating with Cron

Cron is Linux's task scheduler. I opened the cron editor:

```bash
crontab -e
```

And added this job at the bottom:

```
* * * * * echo "Cron works! $(date)" >> ~/lab/ex01/cronlog.txt
```

The cron time format:
```
* * * * * command
| | | | |
| | | | └── Day of week (0-7)
| | | └──── Month (1-12)
| | └────── Day of month (1-31)
| └──────── Hour (0-23)
└────────── Minute (0-59)
```

After waiting 2 minutes:

```bash
cat ~/lab/ex01/cronlog.txt
```

Output:
```
Cron works! Wed Mar  4 13:35:01 WAT 2026
Cron works! Wed Mar  4 13:36:01 WAT 2026
Cron works! Wed Mar  4 13:37:01 WAT 2026
```


Firing exactly every 60 seconds. ✅

> **Why this matters:** Cron jobs are used legitimately for backups, log rotation and scheduled scans. But they're also heavily abused by attackers for **persistence** — adding a malicious cron job means their code keeps running after every reboot. Investigating cron jobs (`crontab -l` and `/etc/cron*`) is standard practice during incident response.

---

##  Mistakes & Fixes

### Mistake 1 — Weak Password Rejected
**What happened:** When creating `labuser` I first tried a password shorter than 8 characters. Ubuntu rejected it with:
```
BAD PASSWORD: The password is shorter than 8 characters
```
**Fix:** Used a longer password.

### Mistake 2 — Dictionary Password Rejected
**What happened:** Tried `password123` — Ubuntu's PAM library rejected it with:
```
BAD PASSWORD: The password fails the dictionary check - it is based on a dictionary word
```
**Fix:** Used a complex password with mixed characters (`L@buser123`).

**Lesson:** This is exactly why password complexity policies exist. Tools like Hashcat can crack simple dictionary-based passwords in seconds.

---

##  Key Takeaways

1. **`ls -la` reveals everything** — hidden files, permissions, ownership. Always run it when investigating a system.
2. **Permissions are security** — `600` for private keys, `755` for scripts, never `777` in production.
3. **`/etc/passwd` maps users, `/etc/shadow` stores hashes** — both are targets for attackers.
4. **Sudo group membership = game over** — it's the difference between a limited user and full system control.
5. **`ps aux` tells the full story** — running processes reveal what's on a system, what security tools are active, and what's suspicious.
6. **Cron jobs persist** — a favourite attacker technique for maintaining access after a breach.

---

##  Real-World Relevance

| Skill Practiced | Real-World Application |
|----------------|----------------------|
| File permissions | Hardening servers, investigating privilege escalation |
| User management | Sysadmin tasks, detecting backdoor accounts post-breach |
| Process monitoring | SOC analysis, incident response, threat hunting |
| Cron jobs | Persistence detection during forensic investigations |
| Password policies | Understanding why complexity requirements exist |

Every SOC analyst, pentester and security engineer lives in the Linux terminal. This exercise is the foundation everything else is built on.

---
