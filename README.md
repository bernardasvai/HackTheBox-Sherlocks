<p align="center">
  <img src="https://capsule-render.vercel.app/api?type=waving&color=0:141d2b,100:9fef00&height=220&section=header&text=HackTheBox&fontSize=45&fontColor=9fef00&animation=fadeIn&fontAlignY=38&desc=CTF%20%7C%20DFIR%20%7C%20Malware%20Analysis%20%7C%20Threat%20Hunting&descAlignY=55&descSize=16" width="100%" />
</p>

<p align="center">
  <img src="https://media.giphy.com/media/AjbFYUVVpFaTu/giphy.gif" width="250" alt="Matrix Rain" />
</p>

---

# Welcome

This is my HackTheBox repository where I put solved challenges explained the way I understand them.

- **CTF** write-ups are documented as a real pentest engagement — enumeration, foothold, privilege escalation, and a lessons-learned recap.
- **Sherlocks** write-ups are documented as if I am handling a real incident — findings, timeline, IOCs, and a detection rule at the end.

> ⚠️ Per HTB's rules, flag values are never published. Lab-specific secrets (passwords, hashes, tickets, private keys) found during CTF boxes are redacted — only the technique and commands used to obtain/abuse them are kept.

---

## HTB CTF

Active Directory / network pentesting boxes, organized by OS and difficulty. Every write-up covers:

- Enumeration
- Foothold
- Privilege escalation
- Tools & techniques used, with a short takeaway on the underlying weakness

### Windows

| Difficulty | Boxes |
|---|---|
| Easy | [Active](HTB-CTF/Windows/Easy/Active.md) · [Forest](HTB-CTF/Windows/Easy/Forest.md) · [Return](HTB-CTF/Windows/Easy/Return.md) · [Sauna](HTB-CTF/Windows/Easy/Sauna.md) · [Support](HTB-CTF/Windows/Easy/Support.md) · [Timelapse](HTB-CTF/Windows/Easy/Timelapse.md) |
| Medium | [Cascade](HTB-CTF/Windows/Medium/Cascade.md) · [Escape](HTB-CTF/Windows/Medium/Escape.md) · [Hospital](HTB-CTF/Windows/Medium/Hospital.md) · [Intelligence](HTB-CTF/Windows/Medium/Intelligence.md) · [Manager](HTB-CTF/Windows/Medium/Manager.md) · [Monteverde](HTB-CTF/Windows/Medium/Monteverde.md) · [Resolute](HTB-CTF/Windows/Medium/Resolute.md) · [Scrambled](HTB-CTF/Windows/Medium/Scrambled.md) |
| Hard | [Blackfield](HTB-CTF/Windows/Hard/Blackfield.md) |

See [HTB-CTF/Windows/README.md](HTB-CTF/Windows/README.md) for a table of key techniques per box.

---

## HTB Sherlocks

DFIR and threat hunting challenges. Every solved Sherlock includes:

- Full investigation walkthrough
- Attack timeline
- Indicators of Compromise (IOCs)
- **YARA rule** to detect the malware/artifact
- **Sigma rule** for SIEM detection (Splunk, Elastic, etc.)
- Hardening and prevention recommendations

| Challenge | Difficulty | Category | Date |
|-----------|------------|----------|------|
| [Baggage](HTB-Sherlocks/Baggage/Baggage.md) | Very Easy | DFIR | 2026-09-24 |

---

## Skills Demonstrated

- Active Directory enumeration & exploitation (Kerberoasting, AS-REP roasting, GPP/LAPS abuse, ACL/BloodHound-driven attack paths, AD CS misconfigurations, delegation abuse)
- Web & database exploitation (SQLi, NTLM capture via MSSQL, credential/config leakage)
- Digital Forensics & Incident Response (DFIR)
- Malware Analysis
- Threat Hunting
- YARA / Sigma rule writing
- MITRE ATT&CK mapping
- Log analysis (Windows Event Logs, Sysmon, network captures)

## Tools

### Environment

| Tool | Purpose |
|------|---------|
| Windows 11 Pro 25H2 | Host OS |
| Kali Linux | Attack/pentesting VM |
| VMware Workstation | Virtualization for isolated analysis VMs |
| [FLARE-VM](https://github.com/mandiant/flare-vm) | Windows malware analysis & forensics VM |

### Active Directory / CTF

| Tool | Purpose |
|------|---------|
| `nmap` | Service & version enumeration |
| `crackmapexec` / `netexec` | SMB/LDAP/MSSQL auth testing, share & password spraying |
| `impacket` suite | `GetNPUsers.py`, `GetUserSPNs.py`, `secretsdump.py`, `psexec.py`, `mssqlclient.py`, `ticketConverter.py`, `getTGT.py`, `getST.py` |
| `BloodHound` / `bloodhound-python` | AD attack-path mapping |
| `Rubeus`, `Certify` / `certipy-ad` | Kerberos ticket abuse, AD CS enumeration & exploitation |
| `PowerView` / `PowerSploit`, `Powermad` | AD recon & object manipulation |
| `evil-winrm` | Remote Windows shell over WinRM |
| `hashcat`, `john` | Offline hash/ticket cracking |
| `kerbrute` | Kerberos pre-auth user enumeration |
| `Responder` | NTLM hash capture |

### Forensic Analysis — [Eric Zimmerman's Tools](https://ericzimmerman.github.io/)

| Tool | Purpose | Used in |
|------|---------|---------|
| ShellBags Explorer | Parse ShellBags (folders browsed, network shares, zip contents) from `UsrClass.dat` | Baggage |
| Registry Explorer | Browse registry hives (BAM, UserAssist, RecentDocs, TypedPaths) | Baggage, CAMouflage |
| PECmd | Parse Prefetch files (program execution, run times, files loaded) | CAMouflage |
| EvtxECmd | Parse Windows event logs (`.evtx`) to CSV | CAMouflage |
| MFTECmd | Parse `$MFT` and `$J` (USN Journal: file creation, renames, deletions) | CAMouflage |
| Timeline Explorer | View and filter the CSV output of the tools above | CAMouflage |
