# Cascade (Medium)

An LDAP frequency-anomaly hunt reveals a legacy base64 password, a VNC config leaks another, and a custom audit tool's AES encryption is reverse engineered with dnSpy to recover a third account — finishing with AD "deleted objects" abuse.

## Enumeration

```bash
nmap -sC -sV -p- <target>
```

```
53/tcp    open domain        Microsoft DNS (Windows Server 2008 R2 SP1)
88/tcp    open kerberos-sec  (Domain: cascade.local)
135/tcp   open msrpc
139/tcp   open netbios-ssn
389/tcp   open ldap
445/tcp   open microsoft-ds?
636/tcp   open tcpwrapped
3268/tcp  open ldap
3269/tcp  open tcpwrapped
```

```bash
rpcclient -U '' -N <target>
rpcclient $> enumdomusers
```

## Foothold — LDAP anomaly hunting

A full LDAP dump was analyzed for rare/one-off attribute names, a useful technique when a dump is too large to read manually:

```bash
ldapsearch -x -H ldap://<target> -s sub -b 'DC=cascade,DC=local' > ldap.txt
cat ldap.txt | awk '{print $1}' | sort | uniq -c | sort -nr | grep ':'
```

This surfaced a custom attribute, `cascadeLegacyPwd`, base64-encoded. Decoded, it gave a working domain credential (redacted — `<REDACTED>`).

```bash
crackmapexec smb <target> -u r.thompson -p '<REDACTED>' -M spider_plus
```

## Share discovery — VNC password

The `spider_plus` module's share crawl surfaced a `VNC Install.reg` file containing a **hex-encoded TightVNC password**. TightVNC uses a fixed, publicly known DES key, so any captured hex blob is instantly reversible with existing tools/scripts ([frizb/PasswordDecrypts](https://github.com/frizb/PasswordDecrypts)) — recovered plaintext (redacted) matched a second domain account, `s.smith`.

## Share discovery — Reverse engineering a custom audit tool

`s.smith` had access to an `audit$` share containing `Audit.db` (SQLite) with an encrypted password column, plus the custom binary (`CascAudit.exe` / `CascCrypto.dll`) used to read it.

```bash
sudo mount -t cifs -o 'user=s.smith,password=<REDACTED>' //<target>/audit$ /mnt/audit
impacket-smbserver -smb2support cascade $(pwd)
```

The DLL was opened in **dnSpy** on a Windows analysis VM, which showed the custom AES routine and a hardcoded decryption key embedded in the binary (not reproduced here). A breakpoint was set in the decompiler just before the plaintext is produced, then the tool was run against `Audit.db` to recover the final account's plaintext password (the `ArkSvc` service account).

## Privilege Escalation — Deleted Objects (Tombstone) recovery

`ArkSvc`'s group membership granted rights to view AD's "deleted objects" container, which can retain security-sensitive attributes of previously-removed accounts:

```powershell
Get-ADObject -SearchBase "CN=Deleted Objects,DC=Cascade,DC=Local" -Filter {ObjectClass -eq "user"} -IncludeDeletedObjects -Properties *
```

This surfaced an administrative credential tied to a deleted account, used to complete the compromise.

## Takeaways

- `cat ldap.txt | awk '{print $1}' | sort | uniq -c | sort -nr` is a fast way to spot one-off custom LDAP attributes buried in a huge dump.
- TightVNC's registry password format uses a fixed, publicly documented key — never trust it for secret storage.
- "Security" tools that roll their own crypto with an embedded key are reversible by any analyst with dnSpy/ILSpy; don't rely on binary obfuscation as a security boundary.
- AD's Deleted Objects container can retain sensitive attributes long after an account is removed — restrict read access to it.
