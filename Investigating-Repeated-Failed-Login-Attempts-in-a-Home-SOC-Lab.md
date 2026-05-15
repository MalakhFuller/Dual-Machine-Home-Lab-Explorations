# Investigating Repeated Failed Login Attempts in a Home SOC Lab

**AttackBox:** Kali Linux VM  
**DefenseBox:** Windows 11  
**Completed:** 2026-05-15  
**Author:** Malakh Fuller

> **Privacy note:** Internal lab IP addresses have been anonymized in this writeup and related screenshots. The original testing was performed only on my own local home lab network.

## Objective

After getting my dual-machine attack/defense home lab set up and proving to myself that I could generate controlled network activity, observe it from the defender’s perspective, and document the results in a SOC-style format, I was ready to move into a more realistic alert scenario: repeated failed login attempts.

The goal of this project was to generate a small number of failed authentication attempts from the Kali AttackBox, then investigate the resulting Windows Security logs from the DefenseBox.

## Tools and Technologies

- **AttackBox:** Oracle VirtualBox, Kali Linux VM, `smbclient`
- **DefenseBox:** Windows 11, Windows Event Viewer, PowerShell
- **Concepts:** Authentication logging, Windows Security Events, SMB authentication, Event ID 4625, Logon Type 3, source IP tracking, failed login pattern recognition
- **Prior Knowledge:** CompTIA A+, Network+, Security+, CySA+ study, TryHackMe SOC-related rooms, and previous home lab reconnaissance project

## Lab Addressing

For privacy, the IP addresses below are anonymized:

- **AttackBox:** `192.168.56.10`
- **DefenseBox:** `192.168.56.20`
- **Lab Network:** `192.168.56.0/24`

## Summary

This project documents a controlled failed-login detection exercise in my home SOC lab.

I created a local test user on the Windows DefenseBox, then used the Kali AttackBox to attempt SMB authentication with the wrong password multiple times. After generating the failed attempts, I reviewed Windows Security logs in Event Viewer and used PowerShell to extract the relevant Event ID 4625 records into a cleaner table.

The purpose was not to perform a real brute-force attack or use a password list. The purpose was to safely generate a small, controlled pattern of failed authentication attempts and investigate them the way a SOC analyst might.

## Methodology

### 1. Preparing Windows Security Logging

I started by opening PowerShell as Administrator on the DefenseBox and running:

```powershell
auditpol /get /subcategory:"Logon"
```

This was done to confirm that Windows was configured to log both successful and failed logon attempts.

The output confirmed that Logon auditing was set to:

```text
Success and Failure
```

With failure logging enabled, I was ready to generate a controlled failed login event.

### 2. Creating a Safe Test User

Still on the DefenseBox, I created a local lab-only user account:

```powershell
net user soclab StrongLabPassword123! /add
```

This created a test account named `soclab`.

<img width="933" height="351" alt="01_powershell_user_creation_redacted" src="https://github.com/user-attachments/assets/776d50e4-2da5-43df-9a15-377e51a3db32" />

I used a dedicated lab account so that I would not be testing against my real Windows account or any personal account. The plan was to intentionally provide the wrong password from Kali and then review the failed authentication events in Windows.

### 3. Confirming SMB Authentication from Kali

On the Kali AttackBox, I opened Terminal and ran:

```bash
smbclient -L //192.168.56.20 -U soclab
```

When prompted for a password, I intentionally entered the wrong password:

```text
WrongPassword1!
```

Kali returned:

```text
NT_STATUS_LOGON_FAILURE
```

<img width="567" height="196" alt="02_nt_status_logon_failure_redacted" src="https://github.com/user-attachments/assets/97a81b34-0985-4fcf-8bee-5ed09790c29b" />

That was the expected result. It meant that the Windows DefenseBox rejected the login attempt, which should create a failed logon event in the Windows Security logs.

Next, I ran the same attempt in a cleaner one-line format:

```bash
smbclient -L //192.168.56.20 -U 'soclab%WrongPassword1!'
```

I repeated this five more times to create a clear and controlled pattern:

- Same source: Kali AttackBox
- Same target: Windows DefenseBox
- Same username: `soclab`
- Same result: failed logon
- Repeated attempts: 5

This gave me enough activity to investigate without using an automated brute-force tool or generating unnecessary noise.

### 4. Reviewing Failed Login Events in Event Viewer

On the DefenseBox, I opened Windows Event Viewer and went to:

```text
Windows Logs → Security
```

I then filtered the current log for:

```text
Event ID: 4625
```

<img width="939" height="248" alt="03_event_4625_redacted" src="https://github.com/user-attachments/assets/39d93313-45f3-4441-a33c-75671f3ba3c0" />

Event ID 4625 means:

```text
An account failed to log on
```

Windows successfully captured the initial failed login attempt and the repeated attempts that followed.

For this lab, the key fields I wanted to confirm were:

- **Account Name:** `soclab`
- **Logon Type:** `3`
- **Source Network Address:** `192.168.56.10`

<img width="726" height="531" alt="04_event_4625_general_redacted" src="https://github.com/user-attachments/assets/3e346504-ca83-4202-9a76-4d262ae3963d" />

Logon Type 3 was important because it indicates a network logon. That matched the activity I generated, since the failed login attempt came from Kali over the network via SMB.

I also reviewed the failure details to better understand why the login failed. The logs showed a failed authentication result consistent with bad credentials.

<img width="932" height="698" alt="05_event_4625_details_redacted" src="https://github.com/user-attachments/assets/f1b01403-efcc-4991-bf0b-75c09d65a139" />

The relevant status values were:

- `0xC000006D` - general logon failure
- `0xC000006A` - user name is correct, but the password is wrong

That matched the lab scenario because I intentionally used the correct test username with the wrong password.

### 5. Using PowerShell to Pull the Evidence

Event Viewer was useful for confirming that the failed logons were being recorded, but PowerShell produced cleaner evidence for screenshots and documentation.

On the DefenseBox, I opened PowerShell as Administrator and ran:

```powershell
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625} -MaxEvents 10 |
Select-Object TimeCreated, Id, ProviderName, Message
```

<img width="914" height="229" alt="06_powershell_view_01_redacted" src="https://github.com/user-attachments/assets/eea0656a-97db-4568-8e89-9257da72d3ef" />

This showed the recent failed logon events, but the output was longer than I wanted for a clean investigation view.

Since I am still learning, I wanted to see if there was a better way to extract only the most useful fields. I used the following PowerShell command to parse the event XML and create a cleaner table:

```powershell
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625} -MaxEvents 10 |
ForEach-Object {
   [xml]$xml = $_.ToXml()
   [PSCustomObject]@{
       TimeCreated = $_.TimeCreated
       TargetUser = ($xml.Event.EventData.Data | Where-Object {$_.Name -eq 'TargetUserName'}).'#text'
       LogonType = ($xml.Event.EventData.Data | Where-Object {$_.Name -eq 'LogonType'}).'#text'
       IpAddress = ($xml.Event.EventData.Data | Where-Object {$_.Name -eq 'IpAddress'}).'#text'
       Status = ($xml.Event.EventData.Data | Where-Object {$_.Name -eq 'Status'}).'#text'
       SubStatus = ($xml.Event.EventData.Data | Where-Object {$_.Name -eq 'SubStatus'}).'#text'
   }
} | Format-Table -AutoSize
```

This produced a much cleaner table showing the failed login pattern in one place. 

The most useful fields were:

- **TimeCreated:** when the failed attempt occurred
- **TargetUser:** the account that was targeted
- **LogonType:** whether the login was local, network-based, or another type
- **IpAddress:** the source of the failed login attempt [Redacted from the public potfolio image]
- **Status/SubStatus:** why the attempt failed

<img width="935" height="398" alt="07_powershell_view_02_redacted" src="https://github.com/user-attachments/assets/99070c2d-d199-466a-834f-8680927d74d7" />

This was the best evidence view for the project because it showed the repeated pattern clearly and made the data easier to explain.

## Key Findings

- Successfully generated controlled failed login attempts from a Kali Linux AttackBox against a Windows 11 DefenseBox.
- Confirmed Windows was logging failed logon activity through Logon audit policy.
- Created a dedicated local test user named `soclab` for safe lab activity.
- Used `smbclient` from Kali to attempt SMB authentication with an intentionally incorrect password.
- Observed the expected `NT_STATUS_LOGON_FAILURE` response from Kali.
- Confirmed Windows recorded the failed login attempts as Event ID 4625.
- Identified Logon Type 3, indicating the attempts were network logons.
- Confirmed the source IP address matched the Kali AttackBox.
- Used PowerShell to extract and format the relevant failed logon event fields.
- Verified that the status and substatus values were consistent with incorrect credentials.

## Key Competencies Demonstrated

- Windows Security log review
- Event Viewer filtering
- PowerShell-based log extraction
- Authentication failure analysis
- SMB authentication testing
- Source IP tracking
- Pattern recognition across repeated events
- Basic incident timeline development
- SOC-style documentation of failed login activity
- Safe lab execution without using real accounts or automated brute-force tooling

## Employer-Relevant Skills

This project demonstrates the ability to generate, detect, and investigate a common SOC alert type: repeated failed login attempts.

In a real SOC environment, analysts often review failed authentication alerts to determine whether the activity is benign, misconfigured, user error, password spraying, brute-force activity, or part of a larger intrusion attempt. This lab gave me practice identifying the target account, source IP address, logon type, failure reason, and repeated pattern of activity.

It also shows that I can use both graphical tools like Event Viewer and command-line tools like PowerShell to review Windows authentication evidence.

## SOC Relevance

From a SOC perspective, repeated failed login attempts can indicate several different things depending on context.

In this lab, the activity was authorized and expected. I intentionally generated a small number of failed SMB authentication attempts using a lab account and an incorrect password.

In a production environment, similar activity would require additional investigation. A few failed attempts from a known user may simply be a mistyped password. Multiple failed attempts from the same source, repeated attempts against the same account, or attempts from an unusual host could indicate brute-force activity, password spraying, credential misuse, or internal reconnaissance.

For this lab, the final outcome was:

- **Verdict:** Authorized lab activity / confirmed failed authentication pattern
- **Severity:** Low in lab context; potentially medium in a production environment if unauthorized or repeated at scale
- **Recommended Action:** Validate whether the source host is known, confirm whether the activity was expected, review the targeted account, check for additional failed attempts against other users, and monitor for follow-on successful logons or SMB activity.

## Basic Incident Timeline

| Time | Event |
|---|---|
| T0 | Confirmed Windows Logon auditing was enabled for Success and Failure. |
| T1 | Created local lab user `soclab` on the Windows DefenseBox. |
| T2 | Attempted SMB authentication from Kali using an incorrect password. |
| T3 | Kali returned `NT_STATUS_LOGON_FAILURE`. |
| T4 | Repeated the failed login attempt five additional times to create a controlled pattern. |
| T5 | Reviewed Windows Security logs for Event ID 4625. |
| T6 | Confirmed the failed attempts showed Logon Type 3 and originated from the AttackBox. |
| T7 | Used PowerShell to extract the failed logon events into a cleaner table. |

## HUMINT to SOC Translation

My previous work in competitive intelligence and HUMINT-style research involved looking at incomplete information, validating sources, identifying patterns, and turning messy evidence into clear written findings.

This lab used a similar workflow in a technical environment.

Instead of reviewing human-source reporting or interview data, I reviewed authentication logs. Instead of validating information across conversations or sources, I compared the activity I generated from Kali against the evidence recorded in Windows Security logs.

The process felt familiar: establish the source, confirm the target, review the evidence, identify the pattern, assess the significance, and document the finding clearly enough for someone else to understand.

## What I Learned

This lab helped me better understand how failed login activity appears from both the attacker and defender sides.

From the Kali side, the failed attempt was simply an `NT_STATUS_LOGON_FAILURE` message. From the Windows side, the same activity created structured security events with useful fields like account name, logon type, source IP address, status, and substatus.

The most important takeaway was that the Windows logs provided enough detail to separate a generic failed login from a network-based failed login attempt against a specific account from a specific source.

## Next Steps

For a future version of this lab, I want to expand the investigation by:

- Comparing failed login events against successful login events.
- Reviewing Event ID 4624 for successful logons.
- Testing account lockout behavior in a controlled way.
- Creating a simple detection rule or alert threshold.
- Building a timeline that includes both Windows logs and network traffic.
- Exploring how the same activity appears in a SIEM.

## Author

- GitHub: [Malakh Fuller](https://github.com/MalakhFuller)
- LinkedIn: [Malakh Fuller](https://www.linkedin.com/in/malakhfuller/)
