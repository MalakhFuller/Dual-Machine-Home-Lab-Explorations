# Home SOC Lab Portfolio by Malakh Fuller

This repository documents my hands-on journey building a home Security Operations Center (SOC) lab from the ground up.

The purpose of this lab is to develop practical SOC analyst skills by safely generating attacker-like activity, collecting logs and network traffic, investigating events, and writing professional case studies that demonstrate real defensive thinking.

My background in HUMINT-driven Competitive Intelligence helps shape my approach to cybersecurity: collecting signals, validating sources, correlating activity, and building clear, defensible narratives from incomplete data.

This repository will grow one project at a time as I build, investigate, document, and refine my home lab.

---

## Repository Goal

The goal of this repository is to build a SOC analyst home-lab portfolio through practical, beginner-friendly case studies.

Each project is designed to teach and demonstrate:

- How to generate safe attacker-like activity
- How to collect logs and network traffic
- How to investigate suspicious behavior
- How to document findings like a SOC analyst
- How to turn lab work into resume, LinkedIn, and GitHub portfolio artifacts

This lab is not focused on offensive exploitation. It is focused on detection, investigation, documentation, and analyst-style reporting.

---

## What This Repository Demonstrates

This repository demonstrates hands-on SOC preparation in areas such as:

- Network reconnaissance detection
- Authentication log analysis
- Failed login and brute-force investigation
- Windows Event Log review
- Sysmon-based investigation
- Suspicious PowerShell analysis
- Phishing email triage
- IOC extraction
- MITRE ATT&CK mapping
- Incident timeline creation
- SOC-style case study writing

---

## Planned Home Lab Projects

### Project 1: Detecting Network Reconnaissance Activity in a Dual-Machine Home Lab

This project focused on detecting basic network scanning activity in a controlled home lab environment.

A Kali Linux machine was used to scan a defensive machine, and the resulting traffic was reviewed using Wireshark.

**Skills demonstrated:**

- Basic reconnaissance detection
- Network traffic review
- Source and destination IP analysis
- Port scan behavior identification
- SOC-style documentation

**Status:** [Completed.](https://github.com/MalakhFuller/Dual-Machine-Home-Lab-Explorations/blob/main/Detecting-Basic-Network-Reconnaissance-in-a-Dual-Machine-Home-Lab.md)

---

### Project 2: Investigating Repeated Failed Login Attempts in a Home SOC Lab

This project focused on generating and investigating repeated failed login attempts.

The goal was to understand how brute-force or password-spraying behavior may appear in Windows or Linux authentication logs.

**Skills demonstrated:**

- Authentication log review
- Failed login pattern recognition
- Source IP tracking
- Alert severity assessment
- Basic incident timeline creation

**Status:** [Completed.](https://github.com/MalakhFuller/Dual-Machine-Home-Lab-Explorations/blob/main/Investigating-Repeated-Failed-Login-Attempts-in-a-Home-SOC-Lab.md)

---

### Project 3: Investigating Suspicious PowerShell Activity with Sysmon

This project focuses on running safe PowerShell commands that may appear suspicious from a defender’s perspective.

Windows Event Logs and Sysmon will be used to collect and review activity, with findings mapped to relevant defensive concepts.

**Skills demonstrated:**

- Windows event investigation
- Sysmon basics
- Command-line analysis
- Suspicious process review
- MITRE ATT&CK mapping
- Detection logic development

**Status:** [Completed](https://github.com/MalakhFuller/Dual-Machine-Home-Lab-Explorations/blob/main/Investigating-Suspicious-PowerShell-Activity-with-Sysmon.md)

---

### Project 4: SOC-Style Phishing Email Triage Report

This project focuses on analyzing a safe phishing email sample or mock phishing email.

The investigation will review sender details, headers, links, domains, and indicators of compromise to reach a final analyst verdict.

**Skills demonstrated:**

- Email header review
- Sender analysis
- URL and domain inspection
- IOC extraction
- Phishing verdict writing
- SOC-style triage reporting

**Status:** Planned

---

### Project 5: End-to-End SOC Investigation: Reconnaissance to Detection

This project will combine multiple investigation skills into a mini incident report.

The final case study will include endpoint logs, network traffic, a timeline of activity, evidence, findings, and a clear conclusion.

**Skills demonstrated:**

- Full investigation workflow
- Endpoint and network evidence review
- Timeline creation
- Evidence handling
- Detection and response thinking
- Executive-style reporting

**Status:** Planned

---

## Lab Environment

The home lab will be built gradually using a small, controlled environment.

Planned components may include:

- VirtualBox or VMware
- Kali Linux
- Windows workstation or Windows Server
- Ubuntu or other Linux defensive machine
- Wireshark
- Zeek
- Sysmon
- Windows Event Viewer
- Splunk, ELK, or another SIEM-style platform

The lab will focus on safe, legal, isolated testing only.

---

## Case Study Format

Each project is planned to follow a consistent SOC-style format:

1. Objective
2. Lab setup
3. Activity generated
4. Data collected
5. Investigation steps
6. Key findings
7. Timeline of activity
8. Indicators of compromise, if applicable
9. MITRE ATT&CK mapping, if applicable
10. Analyst conclusion
11. Lessons learned

This structure is designed to show not only what happened, but also how the investigation was conducted.

---

## Current Focus

Project 4: SOC-Style Phishing Email Triage Report



---

## Professional Development Goals

This repository supports my transition into SOC analyst work by demonstrating:

- Practical security investigation experience
- Clear technical documentation
- Defensive security mindset
- Evidence-based analysis
- Familiarity with common SOC tools and workflows
- Ability to communicate findings professionally

Long term, this lab will also support my growth toward detection engineering by helping me understand how alerts are created, validated, and improved.

---

## Ethical and Legal Notice

All activity in this repository is performed in a controlled home lab environment using systems that I own or am authorized to test.

This repository is for educational and professional development purposes only.

No unauthorized scanning, exploitation, credential attacks, or malicious activity is supported or encouraged.

---

## About Me

I am transitioning into cybersecurity with a focus on SOC analysis, detection engineering, and threat-informed defense.

My previous experience in HUMINT-driven Competitive Intelligence gives me a strong foundation in research, source validation, pattern recognition, and analytical reporting. This repository documents how I am applying those strengths to hands-on cybersecurity investigations.

---

## Connect

- GitHub: [Malakh Fuller](https://github.com/MalakhFuller)
- LinkedIn: [Malakh Fuller](https://www.linkedin.com/in/malakhfuller/)
