# End-to-End SOC Investigation: Reconnaissance to Detection

**AttackBox:** Kali Linux VM `192.168.56.10`  
**DefenseBox:** Windows 11 `192.168.56.20`  
**Completed:** 2026-05-19  
**Author:** Malakh Fuller

> **Privacy note:** Internal lab IP addresses, hostnames, usernames, and other system-specific details have been anonymized in this writeup and related screenshots. The original testing was performed only on my own local home lab network.

## Objective

During this home SOC lab exercise, I set out to simulate and investigate a small incident chain involving three phases:

1. Network reconnaissance
2. Repeated failed SMB authentication attempts
3. Suspicious PowerShell execution

The activity was intentionally generated in a controlled lab environment and reviewed using Kali Linux, Nmap, Wireshark, Windows Security logs, PowerShell, and Sysmon.

The main goal was to move beyond isolated labs and start practicing the kind of correlation a SOC analyst has to do: looking at network activity, authentication logs, endpoint telemetry, and command-line evidence together instead of treating each one as a separate event.

## Tools and Technologies

- **AttackBox:** Kali Linux VM
- **DefenseBox:** Windows 11
- **Tools:** Nmap, `smbclient`, PowerShell, Sysmon, Windows Event Viewer, Wireshark
- **Technologies:** Windows Security logging, Sysmon Operational logs, SMB, TCP/IP, PowerShell command-line activity
- **Concepts:** Network reconnaissance, SMB authentication, failed login analysis, process creation, encoded PowerShell, evidence correlation, incident timeline development
- **Prior Knowledge:** CompTIA A+, Network+, Security+, CySA+ study, TryHackMe SOC-related rooms, and previous home SOC lab projects

## Lab Addressing

For privacy, the IP addresses below are anonymized:

- **AttackBox:** `192.168.56.10`
- **DefenseBox:** `192.168.56.20`
- **Lab Network:** `192.168.56.0/24`

## Summary

This project was my first attempt to pull several previous home lab skills together into one end-to-end SOC-style investigation.

The original plan was to simulate a simple suspicious activity chain. First, the Kali AttackBox would perform reconnaissance against the Windows DefenseBox. Then, it would attempt repeated SMB logins with incorrect credentials. Finally, suspicious PowerShell activity would be generated and reviewed through Sysmon.

The lab did not behave perfectly at first. Nmap showed the target as filtered/no-response, and the first SMB attempts failed with `NT_STATUS_HOST_UNREACHABLE` instead of reaching authentication. Since this same workflow had worked in Project 2, I kept troubleshooting instead of moving on too quickly.

After rebooting the AttackBox and resetting the VirtualBox network adapter, SMB started working again. At that point, I was able to generate the failed SMB login attempts and confirm Windows Security Event ID 4625 on the DefenseBox.

In the end, the project produced usable evidence for all three phases:

1. Reconnaissance against common Windows and remote-access ports
2. Repeated failed SMB authentication attempts
3. Suspicious PowerShell execution captured by Sysmon

## Methodology

### 1. Reconnaissance

On the DefenseBox, I started Wireshark so I could watch for traffic involving the AttackBox. The goal was to preserve defender-side evidence if the scan traffic was visible.

From the AttackBox, I ran an Nmap scan against the DefenseBox to check a small group of common Windows and remote-access ports:

```bash
nmap -Pn -p 22,80,135,139,443,445,3389 192.168.56.20
```

The first result was unexpected:

```text
Nmap done: 1 IP address (0 hosts up)
```

That meant Nmap did not detect the DefenseBox as up during that scan, so it did not actually report the port states.

Before continuing, I needed to figure out what was going on. Some possible causes were:

- The DefenseBox IP address had changed.
- Windows Firewall was blocking the probes differently this time.
- Kali had lost bridged-network access.
- There was a typo or connectivity issue.

I checked the DefenseBox IP address from PowerShell:

```powershell
ipconfig
```

The DefenseBox IP had not changed.

I also checked Kali’s network configuration and confirmed the AttackBox was still on the same subnet as the DefenseBox. That told me the issue was not VirtualBox simply falling back to NAT.

I then ran the scan again with the `--reason` option:

```bash
nmap -sT -Pn -p 22,80,135,139,443,445,3389 --reason 192.168.56.20
```

This produced a more useful result:

```text
Host is up, received user-set.
All tested ports: filtered / no-response
```

The scan checked:

- `22/tcp` - SSH
- `80/tcp` - HTTP
- `135/tcp` - MSRPC
- `139/tcp` - NetBIOS
- `443/tcp` - HTTPS
- `445/tcp` - SMB
- `3389/tcp` - RDP

All tested ports returned as `filtered` with a reason of `no-response`.

<img width="581" height="270" alt="01_nmap-no-response" src="https://github.com/user-attachments/assets/c5febb66-9305-4f03-a211-d2d4481ed52b" />

From an analyst perspective, the important part was not that open services were found. They were not. The important part was that the AttackBox attempted targeted service discovery against common Windows and remote-access ports.

In a production environment, this would be worth investigating if the source host was not an approved vulnerability scanner, administrator workstation, or known management system.

### 2. SMB Authentication Attempts

The next phase was to generate repeated failed SMB login attempts.

From the AttackBox, I ran:

```bash
smbclient -L //192.168.56.20 -U soclab
```

At first, Kali returned:

```text
do_connect: Connection to 192.168.56.20 failed (Error NT_STATUS_HOST_UNREACHABLE)
```

That meant Kali was unable to connect to the DefenseBox over SMB.

I was stuck at this point, but I was not ready to give up because this same workflow had worked during Project 2. I moved back to the DefenseBox and started troubleshooting.

First, I checked whether the SMB server service was running:

```powershell
Get-Service LanmanServer
```

`LanmanServer` was running, which was a good sign.

Next, I checked whether port `445` was reachable from Windows itself:

```powershell
Test-NetConnection 127.0.0.1 -Port 445
```

That returned `True`.

Then I checked the DefenseBox’s own IP address on port `445`:

```powershell
Test-NetConnection 192.168.56.20 -Port 445
```

That also returned `True`.

Both results indicated that SMB was listening locally on the DefenseBox.

Next, I confirmed the network profile:

```powershell
Get-NetConnectionProfile
```

The network category was set to:

```text
Private
```

That was expected for a home lab network.

I also enabled the File and Printer Sharing firewall rules for the private profile:

```powershell
Set-NetFirewallRule -DisplayGroup "File and Printer Sharing" -Profile Private -Enabled True
```

Then I created a narrow firewall rule intended to allow SMB traffic only from the AttackBox:

```powershell
New-NetFirewallRule `
  -DisplayName "SOC Lab Allow SMB from AttackBox" `
  -Direction Inbound `
  -Protocol TCP `
  -LocalPort 445 `
  -RemoteAddress 192.168.56.10 `
  -Action Allow `
  -Profile Private
```

After that, I tried the `smbclient` command again from Kali, but I still received:

```text
NT_STATUS_HOST_UNREACHABLE
```

At this stage, I had confirmed:

- The DefenseBox network profile was set to Private.
- `LanmanServer` was running.
- SMB was reachable locally from the DefenseBox.
- A firewall rule had been created to allow SMB from the AttackBox.
- Kali was on the same subnet.
- Nmap reported `445/tcp` as `filtered/no-response`.
- `smbclient` returned `NT_STATUS_HOST_UNREACHABLE`.

At first, I thought I might have to document the SMB phase as blocked before authentication. That would still have been useful because it shows the difference between a blocked network connection and a failed authentication attempt.

However, it was still bothering me that this worked in Project 2 and would not work now. I decided to close everything out, reboot the AttackBox, and reset the VirtualBox network adapter by switching it from Bridged Adapter to Host-Only Adapter and then back to Bridged Adapter.

After rebooting, Kali received a new IP address on the lab network. I am not completely sure whether the reboot, the adapter reset, the new DHCP lease, or some combination of those fixed the issue. What mattered for the investigation was that SMB connectivity was restored.

When I reran:

```bash
smbclient -L //192.168.56.20 -U soclab
```

Kali finally prompted me for the `soclab` password.

At that point, things seemed like they were back to how they should be. I entered the wrong password multiple times to generate a controlled failed authentication pattern.

On the DefenseBox, I opened PowerShell as Administrator and ran:

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

This finally returned the failed login evidence I had been trying to generate.

<img width="852" height="324" alt="07_everything is working" src="https://github.com/user-attachments/assets/11b7d0de-aac8-4800-99a6-34750b6eb99a" />

The most important fields were:

```text
TargetUser: soclab
LogonType: 3
IpAddress: 192.168.56.10
Status: 0xC000006D
SubStatus: 0xC000006A
```

From a SOC perspective:

- `TargetUser: soclab` showed the lab account being targeted.
- `LogonType: 3` confirmed this was a network logon attempt, which matched SMB activity.
- `IpAddress` matched the AttackBox.
- `0xC000006D` indicated a general logon failure.
- `0xC000006A` indicated a bad password for an existing account.

This gave me the missing middle phase of the investigation.

### 3. Suspicious PowerShell Activity

After confirming the failed SMB logon events, I moved into the endpoint activity phase.

On the DefenseBox, I generated a safe encoded PowerShell command:

```powershell
$command = 'Write-Output "SOC lab encoded command test"'
$bytes = [System.Text.Encoding]::Unicode.GetBytes($command)
$encoded = [Convert]::ToBase64String($bytes)
powershell.exe -EncodedCommand $encoded
```

This command was benign. It only printed:

```text
SOC lab encoded command test
```

However, it launched PowerShell with the `-EncodedCommand` parameter, which is a command-line pattern defenders often investigate.

Next, I pulled the Sysmon evidence with PowerShell:

```powershell
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 100 |
Where-Object {$_.Id -eq 1} |
ForEach-Object {
    [xml]$xml = $_.ToXml()
    [PSCustomObject]@{
        TimeCreated = $_.TimeCreated
        Image = ($xml.Event.EventData.Data | Where-Object {$_.Name -eq 'Image'}).'#text'
        CommandLine = ($xml.Event.EventData.Data | Where-Object {$_.Name -eq 'CommandLine'}).'#text'
        ParentImage = ($xml.Event.EventData.Data | Where-Object {$_.Name -eq 'ParentImage'}).'#text'
    }
} |
Where-Object {$_.CommandLine -like "*EncodedCommand*"} |
Format-Table -Wrap
```

Sysmon captured a process creation event showing:

```text
powershell.exe -EncodedCommand
```

<img width="1323" height="888" alt="06_Final_Image" src="https://github.com/user-attachments/assets/638217ad-65ab-46d6-9475-6134f40b3bc6" />

In this lab, the encoded command was harmless. From a SOC perspective, the command-line pattern is still worth investigating because encoded PowerShell can be used to obscure command content during malicious activity.

At this point, Project 5 had usable evidence for:

1. Reconnaissance: Nmap scan against common Windows and remote-access ports.
2. Authentication activity: repeated failed SMB login attempts from the AttackBox.
3. Endpoint activity: Sysmon captured PowerShell launched with `-EncodedCommand`.

## Investigation Timeline

| Time | Event | Evidence | Analyst Note |
|---|---|---|---|
| T0 | AttackBox performed targeted Nmap scan against DefenseBox | Nmap output | Common Windows and remote-access ports were tested. |
| T1 | Nmap reported all tested ports as filtered/no-response | Nmap `--reason` output | No services were confirmed open from the scanner perspective. |
| T2 | Initial SMB attempt failed as host unreachable | Kali `smbclient` output | Suggested a network, adapter, or firewall reachability issue. |
| T3 | AttackBox was rebooted and the network adapter was reset | Lab notes | Bridged network state refreshed and Kali received a new IP. |
| T4 | SMB connection reached authentication stage | Kali password prompt / failed authentication output | Confirmed SMB path was restored. |
| T5 | AttackBox generated repeated failed SMB login attempts | Kali terminal / Windows Security logs | Controlled failed authentication pattern. |
| T6 | DefenseBox logged Event ID 4625 failures | PowerShell Security log query | Logon Type 3 confirmed network-based logon attempts. |
| T7 | Failed logons showed bad password status codes | Event ID 4625 fields | `0xC000006D` / `0xC000006A` indicated incorrect credentials. |
| T8 | Suspicious PowerShell activity occurred on DefenseBox | Sysmon Event ID 1 | PowerShell launched with `-EncodedCommand`. |
| T9 | Analyst correlated network, authentication, and endpoint evidence | Final report | Multi-stage suspicious activity pattern confirmed in lab. |

## Key Findings

### Reconnaissance Was Attempted

The AttackBox attempted targeted service discovery against common Windows and remote-access ports. Nmap initially reported confusing results, but the `--reason` scan showed the DefenseBox as up with all tested ports returning `filtered/no-response`.

### Troubleshooting Was Part of the Investigation

The SMB phase did not work at first, even though the same workflow had worked in Project 2. I had to validate the DefenseBox IP, confirm Kali was still on the same subnet, check SMB locally, confirm the Windows network profile, and review firewall behavior.

This was a useful reminder that lab environments can change state, and that an analyst has to separate the expected result from the actual evidence.

### SMB Connectivity Was Eventually Restored

After rebooting the AttackBox and resetting the VirtualBox network adapter, Kali received a new IP address and SMB connectivity started working again. I am not completely sure which specific action fixed the issue, but the result was clear: the SMB connection finally reached the authentication stage.

### Failed SMB Logins Were Captured by Windows Security Logs

Once SMB reached authentication, the DefenseBox logged Event ID 4625 failed logon events. The failed attempts showed `TargetUser: soclab`, `LogonType: 3`, and the AttackBox as the source IP.

The status and substatus values showed a bad password for an existing account, which matched the activity I generated from Kali.

### Sysmon Captured Suspicious-Looking PowerShell Activity

Sysmon Event ID 1 captured a PowerShell process launched with the `-EncodedCommand` parameter. The command was benign in this lab, but the pattern is suspicious enough to review in a real SOC environment.

### The Evidence Supported a Multi-Stage Pattern

By the end of the lab, I had evidence for reconnaissance, repeated failed network logons, and suspicious-looking PowerShell execution. In a real environment, those events would deserve correlation and follow-up investigation.

## MITRE ATT&CK Mapping

I kept the MITRE ATT&CK mapping focused on the behaviors I actually generated and observed.

- **T1046 - Network Service Discovery**  
  The AttackBox scanned common Windows and remote-access ports using Nmap.

- **T1110 - Brute Force**  
  The repeated failed SMB authentication attempts resembled basic password-guessing behavior. In this lab, the attempt count was intentionally small and controlled, so I am not overstating it as a real brute-force attack at scale.

- **T1059.001 - Command and Scripting Interpreter: PowerShell**  
  PowerShell was used to execute a benign command using the `-EncodedCommand` parameter.

The commands and activity in this lab were authorized and benign, but they overlap with behaviors that may appear during real investigations.

## Key Competencies Demonstrated

- Nmap-based service discovery
- Wireshark-based traffic observation
- Basic network troubleshooting
- SMB reachability testing
- Windows Firewall and network profile review
- Failed authentication analysis
- Windows Security Event ID 4625 review
- PowerShell-based event log querying
- Sysmon Event ID 1 process creation review
- PowerShell command-line analysis
- MITRE ATT&CK mapping
- Evidence correlation across multiple tools
- Building an incident timeline
- Distinguishing between failed connectivity and failed authentication
- Adjusting an investigation based on evidence
- SOC-style documentation and final verdict writing

## Employer-Relevant Skills

This project demonstrates more than just running individual tools. It shows the ability to investigate when the lab does not behave the way I expected.

In a SOC environment, evidence does not always line up cleanly. A scanner may report filtered ports. A connection attempt may fail before authentication. A service may be listening locally but unreachable from another host. The analyst’s job is to work through the evidence, document what happened, avoid overclaiming, and continue the investigation when the original plan changes.

This project also demonstrates basic correlation across different evidence sources: Nmap output, SMB client behavior, Windows Security logs, firewall/network checks, and Sysmon process creation logs.

## SOC Relevance

From a SOC perspective, this activity would be treated as suspicious and worth investigating.

The Nmap scan suggests possible reconnaissance. The repeated SMB failures suggest password guessing or attempted access against a Windows service. The encoded PowerShell event is worth reviewing because encoded commands can be used to hide what PowerShell is doing.

In this lab, everything was authorized and benign. In a production environment, the same pattern would require follow-up questions:

- Is the scanning host authorized?
- Is the target system expected to expose SMB?
- Were the failed logons isolated or repeated across multiple accounts?
- Did any authentication attempts succeed after the failures?
- Was the PowerShell command run by an expected user?
- What was the parent process?
- What did the encoded command decode to?
- Was there any related file creation, network activity, persistence, or privilege escalation?

For this lab, the final outcome was:

- **Verdict:** Authorized lab simulation / multi-stage suspicious activity pattern
- **Severity:** Low in lab context; potentially medium or high in production depending on source host, user context, scale, and follow-on activity
- **Confidence:** High for observed events
- **Recommended Action:** Validate the source host, confirm whether scanning was expected, review the targeted account, check for successful logons after the failures, decode and review encoded PowerShell content, and search for related process, network, file, and authentication activity.

## HUMINT to SOC Translation

My previous work in competitive intelligence and HUMINT-style research involved collecting fragments of information, validating what could be confirmed, identifying what could not be confirmed, and writing conclusions without overstating the evidence.

This lab felt similar.

The original plan was to produce a clean incident chain. At first, the evidence did not fully support that because SMB was blocked before authentication. Instead of pretending the failed logon happened, I had to separate what I observed from what I expected.

After troubleshooting, SMB started working again and the Event ID 4625 evidence was generated. That changed the case. The final version of the investigation had stronger evidence, but only because I kept digging instead of stopping at the first unexpected result.

That is very similar to intelligence work: source A may suggest one thing, source B may not confirm it yet, and the final report has to explain the gap clearly. If new evidence appears later, the assessment should be updated.

For this investigation, the evidence supported reconnaissance, repeated failed SMB authentication attempts, and encoded PowerShell execution. That distinction matters, and learning to document those differences is exactly why I am building these labs.

## What I Learned

This project helped me understand that an investigation does not have to go perfectly to be useful.

The SMB phase did not behave the way it did in Project 2 at first, but that forced me to think more carefully about what the evidence actually showed. I learned that a blocked port or unreachable service may prevent Windows from generating authentication logs because the login attempt never reaches that layer.

The biggest takeaway was that troubleshooting is part of the investigation. Rebooting the AttackBox and resetting the VirtualBox network adapter restored SMB connectivity, even though I cannot say with certainty which specific action fixed the issue. 

Had I not kept pushing to figure out what was actually going on, the Project outcome would have been far less optimal. Not being able to reach DefenseBox would have felt more like a failure. This outcome, though not the one originally envisioned, still felt like a success because I worked through the issue, restored connectivity, generated the missing evidence, and documented what changed.

## Next Steps

For a future version of this lab, I want to:

- Troubleshoot exactly why SMB from Kali became filtered during the first part of the run.
- Compare blocked SMB attempts against successful failed-authentication attempts.
- Add Windows Firewall logs to the investigation.
- Forward Sysmon and Security logs into a SIEM.
- Create a simple detection rule for repeated Event ID 4625 failures.
- Create a simple detection rule for PowerShell `-EncodedCommand`.
- Build a more complete timeline using synchronized timestamps from each tool.
- Add a dashboard or alert view to make the investigation more SOC-like.

## Author

- GitHub: [Malakh Fuller](https://github.com/MalakhFuller)
- LinkedIn: [Malakh Fuller](https://www.linkedin.com/in/malakhfuller/)
