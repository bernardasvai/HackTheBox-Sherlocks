# Windows Machines

Write-ups for retired HTB Windows boxes, mostly Active Directory environments, split by difficulty.

> Credentials, hashes, tickets and private keys found during these boxes are redacted. Flags are never included. See the [root README](../README.md) for details.

## Easy

| Box | Key techniques |
|---|---|
| [Active](Easy/Active.md) | Anonymous SMB read, GPP cpassword decryption, Kerberoasting |
| [Forest](Easy/Forest.md) | AS-REP roasting, BloodHound, Exchange Windows Permissions → DCSync |
| [Return](Easy/Return.md) | Web-based printer admin panel credential leak, service binary hijack |
| [Sauna](Easy/Sauna.md) | Username generation + AS-REP roasting, DCSync via BloodHound |
| [Support](Easy/Support.md) | SMB share recon, LDAP credential leak over the wire, RBCD abuse |
| [Timelapse](Easy/Timelapse.md) | Credential in zipped PFX, LAPS password dumping |

## Medium

| Box | Key techniques |
|---|---|
| [Cascade](Medium/Cascade.md) | LDAP anomaly hunting, custom binary reverse engineering (AES), deleted-object recovery |
| [Escape](Medium/Escape.md) | MSSQL creds on a public share, NTLM capture via `xp_dirtree`, ADCS ESC1 |
| [Hospital](Medium/Hospital.md) | Webshell upload, PHP config credential leak, GhostScript CVE, service hijack |
| [Intelligence](Medium/Intelligence.md) | PDF metadata/content harvesting, ADIDNS record abuse + NTLM relay, gMSA secrets |
| [Manager](Medium/Manager.md) | Password spraying, MSSQL `xp_dirtree` web root dump, ADCS officer abuse |
| [Monteverde](Medium/Monteverde.md) | `rpcclient` enum, password spray, Azure AD Connect credential decryption |
| [Resolute](Medium/Resolute.md) | LDAP description field leak, PowerShell transcript credential leak, DnsAdmins DLL abuse |
| [Scrambled](Medium/Scrambled.md) | Kerberoasting over Kerberos auth, silver ticket forging, reverse-engineered secret decryption |

## Hard

| Box | Key techniques |
|---|---|
| [Blackfield](Hard/Blackfield.md) | AS-REP roasting, RPC password reset, LSASS dump analysis, `SeBackupPrivilege` abuse |
