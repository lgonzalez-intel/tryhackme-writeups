# TryHackMe write-ups

Write-ups of the TryHackMe boxes I solve while moving into cybersecurity (blue team / SOC).
Each one documents the methodology I followed — recon, enumeration, access, privilege escalation — and what I learned, plus a short blue-team view on how each weakness could be mitigated.

> No flags or passwords are published, to keep the challenges intact for others.

## 📂 Write-ups

| # | Box | Platform | Difficulty | Topics |
|---|---|---|---|---|
| 01 | [Pickle Rick](01-pickle-rick/) | TryHackMe | Easy | Web enumeration · Command injection · Privilege escalation (sudo) |

## 🧭 Methodology

Every write-up follows the same structure:

- **Recon** — scanning the target (Nmap) to find open ports and services
- **Enumeration** — investigating each service (web source, robots.txt, hidden pages…)
- **Access** — getting an initial foothold
- **Privilege escalation** — becoming root
- **What I learned** — commands and concepts
- **Mitigation (blue team)** — how the weaknesses could be fixed

## 👤 About

Lorena González — intelligence/OSINT analyst moving into cybersecurity, focused on blue team, SOC and threat intelligence. Studying ASIR (IT systems & networks).
