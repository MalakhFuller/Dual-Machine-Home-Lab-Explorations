# Investigating Suspicious PowerShell Activity with Sysmon

**DefenseBox:** Windows 11  
**Completed:** 2026-05-18  
**Author:** Malakh Fuller

> **Privacy note:** Internal lab identifiers, usernames, hostnames, and other system-specific details have been anonymized in this writeup and related screenshots. The original testing was performed only on my own local home lab system.

## Objective

The goal of this project was to run benign PowerShell commands on my Windows DefenseBox that would still look suspicious from a defender’s perspective, then identify how that behavior appears in Windows logs and Sysmon.

This is a useful defensive skill to develop because attackers often abuse PowerShell for discovery, encoded commands, script execution, and other post-compromise activity. In this lab, nothing malicious was executed. The purpose was to safely generate activity that resembles behavior a SOC analyst might need to investigate.

## Tools and Technologies

- **DefenseBox:** Windows 11
- **Tools:** PowerShell, Sysmon, Windows Event Viewer
- **Technologies:** Windows Security Logging, Sysmon Operational Logs, PowerShell command-line activity
- **Concepts:** Process creation, command-line analysis, encoded PowerShell, endpoint logging, MITRE ATT&CK mapping
- **Prior Knowledge:** CompTIA A+, Network+, Security+, CySA+ study, TryHackMe SOC-related rooms, and previous home SOC lab projects

## Summary

This project documents my first attempt to use Sysmon to investigate suspicious-looking PowerShell activity in a home SOC lab.

I installed Sysmon, configured it to capture process creation events, generated several benign PowerShell commands that overlap with common attacker discovery behavior, and then reviewed the resulting Sysmon Event ID 1 logs.

The most useful evidence came from a PowerShell command that launched a second PowerShell process using the   `-EncodedCommand` parameter. The encoded command itself was harmless and only printed a test message, but the command-line pattern is important because encoded PowerShell is commonly reviewed by defenders during investigations.

## Methodology

### 1. Installing Sysmon

I started by downloading Sysmon from Microsoft Sysinternals and placing it in:

```text
C:\Tools\Sysmon
```

I then created a basic Sysmon configuration file called:

```text
sysmon-basic.xml
```

My first version of the configuration looked like this:

```xml
<Sysmon schemaversion="4.90">
  <EventFiltering>
    <!-- Log process creation -->
    <ProcessCreate onmatch="include" />

    <!-- Log network connections -->
    <NetworkConnect onmatch="include" />

    <!-- Log file creation time changes -->
    <FileCreateTime onmatch="include" />

    <!-- Log process termination -->
    <ProcessTerminate onmatch="include" />
  </EventFiltering>
</Sysmon>
```

The purpose of this file was to tell Sysmon what types of activity I wanted it to log. In this case, I was mainly interested in process creation events because I wanted to see PowerShell execution details.

I then opened PowerShell as Administrator and installed Sysmon using:

```powershell
cd C:\Tools\Sysmon
.\Sysmon64.exe -accepteula -i .\sysmon-basic.xml
```

The `-accepteula` option accepts the Sysinternals license agreement, and the `-i` option installs Sysmon using the configuration file I provided.

To confirm Sysmon was running, I used:

```powershell
Get-Service Sysmon64
```

The output showed Sysmon in a `Running` state, which confirmed that the service was installed and active.

### 2. Confirming Sysmon Logs Were Being Written

Next, I opened Windows Event Viewer and navigated to:

```text
Applications and Services Logs
→ Microsoft
→ Windows
→ Sysmon
→ Operational
```

I expected to find:

```text
Event ID 1 - Process Create
```

However, I only saw Event IDs `4` and `16`.

That told me Sysmon itself was running, but it was not yet logging the process creation events I needed for this lab. At that point, I knew I likely had either a permissions issue, a configuration issue, or both.

To troubleshoot from PowerShell, I opened PowerShell as Administrator and ran:

```powershell
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 10 |
Select-Object TimeCreated, Id, ProviderName, Message
```

This command pulls recent events directly from the Sysmon Operational log. It was useful because it let me check the log without relying only on the Event Viewer interface.

Since Event ID 1 was still not appearing, I updated the Sysmon configuration.

I opened the configuration file with:

```powershell
notepad C:\Tools\Sysmon\sysmon-basic.xml
```

Then I replaced the contents with:

```xml
<Sysmon schemaversion="4.90">
  <HashAlgorithms>*</HashAlgorithms>
  <EventFiltering>
    <ProcessCreate onmatch="exclude" />
    <NetworkConnect onmatch="exclude" />
  </EventFiltering>
</Sysmon>
```

This version was better for the lab because it logs process creation and network connection events unless they are specifically excluded. Since I did not list any exclusions, it gave me the process creation visibility I needed.

After saving the file, I updated Sysmon with:

```powershell
cd C:\Tools\Sysmon
.\Sysmon64.exe -c .\sysmon-basic.xml
```

The `-c` option updates the existing Sysmon configuration. After running this, Sysmon confirmed that the configuration was updated.

Once this was complete, Event ID 1 process creation events began appearing in the Sysmon Operational log.

### 3. Running Safe Suspicious-Looking PowerShell Commands

With Sysmon logging correctly, I generated a series of benign commands that would still be relevant from a defender’s point of view.

The goal was not to run anything malicious. The goal was to create safe activity that resembles the types of commands analysts may see during investigation work.

#### Create a Lab Folder

First, I created a dedicated project folder:

```powershell
mkdir C:\SOC-Lab\Project3
cd C:\SOC-Lab\Project3
```

This kept the lab activity organized and made the command history easier to explain.

#### Basic Host Discovery

I then ran several basic host discovery commands:

```powershell
whoami
hostname
ipconfig /all
```

These commands are not malicious by themselves. However, they are commonly seen during discovery because they help identify the current user, system name, and network configuration.

#### User and Group Discovery

Next, I ran:

```powershell
net user
net localgroup
```

These commands can be used to identify local users and groups on a system. In a real investigation, this type of activity could matter if it appears after suspicious access or from an unusual process.

#### Process and Service Discovery

I also ran:

```powershell
Get-Process
```

Process discovery can help an attacker understand what is running on a system, including security tools, business applications, browsers, remote access tools, or other useful targets.

Then I ran:

```powershell
Get-Service
```

Service discovery can reveal installed services, security agents, backup tools, remote access services, and other software that may be useful during post-compromise activity.

#### Encoded PowerShell Command

Next, I ran a harmless encoded PowerShell command:

```powershell
$command = 'Write-Output "SOC lab encoded command test"'
$bytes = [System.Text.Encoding]::Unicode.GetBytes($command)
$encoded = [Convert]::ToBase64String($bytes)
powershell.exe -EncodedCommand $encoded
```

This command only printed:

```text
SOC lab encoded command test
```

However, it did so by launching PowerShell with the `-EncodedCommand` parameter.

That is important because encoded PowerShell can be used to make command content less readable in logs or command history. It is not automatically malicious, but it is suspicious enough that a SOC analyst would usually want to review it.

#### Local Download-Style Simulation

Sometimes attackers use PowerShell to download or stage files. I did not want to download anything from the internet for this lab, so I created a harmless local test file instead:

```powershell
Set-Content -Path C:\SOC-Lab\Project3\payload-test.txt -Value "This is a harmless local test file."
```

Then I confirmed the file contents with:

```powershell
Get-Content C:\SOC-Lab\Project3\payload-test.txt
```

This produced the expected result and gave me another example of file-related PowerShell activity without introducing any unsafe downloads.

<img width="622" height="68" alt="harmless local test file" src="https://github.com/user-attachments/assets/a16441aa-0e8d-457a-adde-f4acca43ccb7" />

### 4. Finding the Sysmon Events

After generating the activity, I returned to Event Viewer and navigated back to:

```text
Applications and Services Logs
→ Microsoft
→ Windows
→ Sysmon
→ Operational
```

This time, I filtered for:

```text
Event ID: 1
```

Event ID 1 is Sysmon’s process creation event. It records information about newly created processes, including fields such as:

- Image
- CommandLine
- CurrentDirectory
- User
- ParentImage
- ParentCommandLine
- ProcessId
- ParentProcessId
- Hashes

For this project, the most important event was the one showing:

```text
C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
```

with a command line containing:

```text
-EncodedCommand
```

That confirmed Sysmon captured the suspicious-looking PowerShell execution.

<img width="935" height="823" alt="02_processcreate_opsec_redacted" src="https://github.com/user-attachments/assets/f9bceda5-47a9-42c5-a97f-38a1be80a22e" />

### 5. Pulling Cleaner Evidence with PowerShell

Event Viewer showed the evidence, but I also wanted a cleaner view that would be easier to screenshot and explain.

I opened PowerShell as Administrator and ran:

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
Where-Object {$_.Image -like "*powershell.exe*"} |
Format-Table -Wrap
```

This command searched recent Sysmon Event ID 1 logs, parsed the XML event data, and extracted the fields I cared about most:

- `TimeCreated`
- `Image`
- `CommandLine`
- `ParentImage`

This produced clear evidence of a PowerShell process launched with `-EncodedCommand` at approximately `10:58 AM` on `05/18/2026`.

<img width="1898" height="829" alt="image 3 edit" src="https://github.com/user-attachments/assets/00976352-e900-42a2-bb8f-bed09ac311b1" />

For me, this was one of the most useful parts of the lab. Event Viewer gave me the full event, but PowerShell helped turn the logs into a cleaner investigation view.

### 6. MITRE ATT&CK Mapping

I am still learning MITRE ATT&CK mapping, so I kept this section focused on the behaviors I actually generated rather than trying to force too much into the writeup.

The activity in this lab maps most closely to:

- **T1059.001 - Command and Scripting Interpreter: PowerShell**
- **T1082 - System Information Discovery**
- **T1087 - Account Discovery**
- **T1057 - Process Discovery**
- **T1007 - System Service Discovery**

The commands I ran are not malicious by themselves. The important point is that they overlap with behaviors commonly seen during post-compromise discovery.

In a real SOC environment, the context would matter. PowerShell launched by an administrator during normal maintenance may be expected. PowerShell running encoded commands from an unusual parent process, at an unusual time, or followed by network activity would be more suspicious.

## Key Findings

### Sysmon Captured Process Creation Activity

After updating the Sysmon configuration, Event ID 1 process creation events were successfully captured. This was important because process creation logs show what executed, when it executed, and what command-line arguments were used.

### PowerShell Command Lines Are Valuable Evidence

The most important evidence in this lab was the command line showing PowerShell launched with the `-EncodedCommand` parameter. Even though the command was harmless, this parameter is commonly investigated because it can obscure what a command is doing.

### Configuration Matters

Sysmon was installed and running before the useful process creation events appeared. The issue was not that Sysmon was broken. The issue was that the first configuration was not giving me the logging visibility I needed. Updating the configuration was what made Event ID 1 process creation events visible.

### Context Determines Severity

None of the commands in this lab were malicious by themselves. The same commands could be normal administrator activity or part of suspicious post-compromise behavior depending on the user, host, parent process, timing, and follow-on activity.

### PowerShell and Event Viewer Work Well Together

Event Viewer was useful for reviewing the full Sysmon event, while PowerShell was better for pulling specific fields into a cleaner investigation view. Using both gave me a better understanding of the evidence.

## Key Competencies Demonstrated

- Sysmon installation and configuration
- Windows Event Viewer navigation
- Sysmon Operational log review
- Event ID 1 process creation analysis
- PowerShell command-line analysis
- Encoded PowerShell detection
- PowerShell-based log querying with `Get-WinEvent`
- Basic XML event parsing in PowerShell
- Parent process review
- MITRE ATT&CK mapping
- SOC-style documentation of suspicious endpoint activity

## Employer-Relevant Skills

This project demonstrates the ability to generate controlled endpoint activity, collect the resulting logs, and explain why the behavior matters from a defensive perspective.

For entry-level SOC work, this is relevant because analysts often investigate alerts involving PowerShell, suspicious command-line activity, discovery commands, or process creation events. Being able to identify the process, review the command line, understand the parent process, and explain the behavior clearly is an important analyst skill.

This project also shows that I can troubleshoot logging issues instead of assuming the tool is working just because the service is running. Sysmon was active, but I still had to validate whether the right events were being captured.

## SOC Relevance

From a SOC perspective, PowerShell activity is common and cannot be treated as automatically malicious. Administrators use PowerShell constantly. However, PowerShell is also frequently abused because it is built into Windows, powerful, scriptable, and often available by default.

In this lab, the activity was authorized and benign. The encoded command only printed a harmless test message. However, the behavior was still useful to investigate because `-EncodedCommand` can be used to obscure command content.

In a production environment, this type of event would require additional context. An analyst would want to know:

- Which user ran the command?
- What host did it run on?
- What was the parent process?
- Was the command encoded?
- Can the encoded content be decoded and reviewed?
- Was there related network activity?
- Was it followed by file creation, credential access, lateral movement, or persistence activity?

For this lab, the final outcome was:

- **Verdict:** Authorized lab activity / confirmed suspicious-looking PowerShell behavior
- **Severity:** Low in lab context; potentially medium or high in a production environment depending on parent process, user context, encoded content, and follow-on activity
- **Recommended Action:** Review the full command line, decode the encoded content, validate the user and host, check the parent process, and search for related process, network, file, or authentication activity.

## Basic Incident Timeline

| Time | Event |
|---|---|
| T0 | Installed Sysmon on the Windows DefenseBox. |
| T1 | Confirmed the Sysmon service was running. |
| T2 | Checked Event Viewer and found Sysmon Event IDs 4 and 16, but not Event ID 1. |
| T3 | Updated the Sysmon configuration to capture process creation events. |
| T4 | Confirmed Sysmon Event ID 1 process creation events were being written. |
| T5 | Ran benign discovery commands using PowerShell. |
| T6 | Ran a harmless encoded PowerShell command using `-EncodedCommand`. |
| T7 | Located the PowerShell process creation event in Event Viewer. |
| T8 | Used PowerShell and `Get-WinEvent` to extract cleaner evidence from Sysmon logs. |
| T9 | Mapped the observed behavior to relevant MITRE ATT&CK techniques. |

## HUMINT to SOC Translation

My previous work in competitive intelligence and HUMINT-style research involved reviewing incomplete information, validating sources, identifying patterns, and turning messy evidence into clear written findings.

This lab followed a similar process in a technical setting.

Instead of reviewing interview notes or human-source reporting, I reviewed process creation logs. Instead of validating information across conversations or research sources, I compared the activity I intentionally generated in PowerShell against the evidence recorded by Sysmon.

The process felt familiar: generate or identify activity, locate the supporting evidence, validate what happened, understand why it matters, and document the finding clearly enough that another person could follow the logic.

## What I Learned

This lab helped me understand that simply installing a security tool is not enough. Sysmon was running, but I still had to confirm that it was collecting the right events.

I also learned why command-line logging is so valuable. The process name alone only tells part of the story. Seeing the full command line is what made the `-EncodedCommand` behavior visible.

The biggest takeaway was that suspicious behavior is often about context. PowerShell is not automatically bad, and discovery commands are not automatically malicious. But when certain behaviors appear together, especially encoded commands or unusual parent-child process relationships, they become worth investigating.

## Next Steps

For a future version of this lab, I want to expand the investigation by:

- Decoding the Base64 command and documenting the decoded output.
- Comparing normal PowerShell activity against suspicious PowerShell activity.
- Reviewing parent-child process relationships more closely.
- Creating a simple detection rule for `-EncodedCommand`.
- Searching for related network or file activity after the PowerShell event.
- Forwarding Sysmon logs into a SIEM for alerting and dashboarding.
- Testing the same behavior with different parent processes to see how the evidence changes.

## Author

- GitHub: [Malakh Fuller](https://github.com/MalakhFuller)
- LinkedIn: [Malakh Fuller](https://www.linkedin.com/in/malakhfuller/)
