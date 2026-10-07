<p align="center">
  <img src="https://capsule-render.vercel.app/api?type=waving&color=0:141d2b,100:9fef00&height=220&section=header&text=HackTheBox%20Sherlocks&fontSize=45&fontColor=9fef00&animation=fadeIn&fontAlignY=38&desc=DFIR%20%7C%20Malware%20Analysis%20%7C%20Threat%20Hunting&descAlignY=55&descSize=18" width="100%" />
</p>

<p align="center">
  <img src="https://media.giphy.com/media/AjbFYUVVpFaTu/giphy.gif" width="250" alt="Matrix Rain" />
</p>

---

# Welcome

This is my HackTheBox repository where I put solved challenges explained the way I understand them.

Each write-up is written as if I am documenting a real incident — with findings, timelines, IOCs, and a detection rule at the end.

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
| VMware Workstation | Virtualization for isolated analysis VMs |
| [FLARE-VM](https://github.com/mandiant/flare-vm) | Windows malware analysis & forensics VM |

### Forensic Analysis — [Eric Zimmerman's Tools](https://ericzimmerman.github.io/)

| Tool | Purpose | Used in |
|------|---------|---------|
| ShellBags Explorer | Parse ShellBags (folders browsed, network shares, zip contents) from `UsrClass.dat` | Baggage |
| Registry Explorer | Browse registry hives (BAM, UserAssist, RecentDocs, TypedPaths) | Baggage, CAMouflage |
| PECmd | Parse Prefetch files (program execution, run times, files loaded) | CAMouflage |
| EvtxECmd | Parse Windows event logs (`.evtx`) to CSV | CAMouflage |
| MFTECmd | Parse `$MFT` and `$J` (USN Journal: file creation, renames, deletions) | CAMouflage |
| Timeline Explorer | View and filter the CSV output of the tools above | CAMouflage |

### Other

| Tool | Purpose | Used in |
|------|---------|---------|
| PowerShell | Running CLI tools, file hashing (`Get-FileHash`), file signature checks (`Format-Hex`) | CAMouflage |
| Notepad++ | Safe viewing and manual deobfuscation of malicious scripts | CAMouflage |