# SOC-Style Phishing Email Triage Report

**DefenseBox:** Windows 11  
**Completed:** 2026-05-19  
**Author:** Malakh Fuller

> **Privacy note:** This project uses a mock phishing email for safe analysis. Any email addresses, domains, URLs, and indicators shown here are intentionally defanged or fictionalized for portfolio use.

## Objective

The goal of this project was to analyze a safe mock phishing email, review the sender and link details, identify suspicious indicators, extract IOCs, and write a final SOC-style verdict.

This is a useful skill to practice because phishing triage is a common task for junior SOC analysts. Even when an email looks simple on the surface, the analyst still needs to slow down, review the sender, inspect the link safely, identify the social engineering tactics, and make a clear recommendation.

## Tools and Technologies

- **DefenseBox:** Windows 11
- **Tools:** Browser-based document review, screenshot tools, and image editing tools.
- **Technologies:** Microsoft Word, Google Docs, MS Paint
- **Concepts:** Email triage, sender analysis, URL inspection, domain review, IOC extraction, phishing indicators, final verdict writing
- **Prior Knowledge:** CompTIA A+, Network+, Security+, CySA+ study, TryHackMe SOC-related rooms, and previous home SOC lab projects

## Summary

For this project, I did not attack anything, scan anything, or interact with a live malicious website. Instead, I used a mock phishing email to practice the basic workflow of a SOC-style phishing investigation.

The email pretended to come from Microsoft Security Team and warned that the recipient’s password was about to expire. It used urgency, brand impersonation, a lookalike sender domain, and a suspicious credential-verification link. My job was to review the message safely, identify the suspicious indicators, extract the useful IOCs, and write a final triage verdict.

## Email Sample

The mock email used for this lab was:

```text
From: Microsoft Security Team <security-alert@micros0ft-support[.]com>
To: Employee User <employee@example[.]com>
Subject: Urgent: Password Expiration Notice

Your Microsoft 365 password is scheduled to expire today.

To avoid losing access to your email, Teams, OneDrive, and company files, please verify your account immediately.

Verify your account here: hxxps://microsoft-security-check[.]com/login

Failure to complete verification within 2 hours may result in temporary account suspension.

Thank you,
Microsoft Security Team
```

## Methodology

### 1. Initial Email Review

I started by treating the email like a SOC analyst receiving a possible phishing submission.

Some basic triage questions I wanted to answer were:

- Who sent it?
- Who received it?
- What is the subject?
- Is there urgency or pressure?
- Is the sender domain expected?
- Does the link match the claimed sender?
- Is the message asking for credentials?
- Are there attachments?
- Is the user being pushed to act quickly?

The initial review already showed several suspicious signs. The message claimed to be from Microsoft Security Team, but the sender domain did not look like a legitimate Microsoft domain. The email also used urgency by saying the password would expire today and that the user could lose access within two hours.

There were no attachments in this sample, so the main focus was the sender, link, and social engineering language.

### 2. Sender Analysis

The sender display name claimed to be:

```text
Microsoft Security Team
```

However, the actual sender address was:

```text
security-alert@micros0ft-support[.]com
```

The domain `micros0ft-support[.]com` is suspicious because it uses a zero in place of the letter `o` in “Microsoft.” This is a lookalike technique intended to resemble the legitimate Microsoft brand while still being a completely different domain.

A normal user might quickly read `micros0ft-support[.]com` as “microsoft-support,” especially if they were moving fast or worried about losing access to their account.

From a SOC perspective, the mismatch between the display name and actual sender domain is a strong phishing indicator.

### 3. URL Inspection

The email included the following defanged URL:

```text
hxxps://microsoft-security-check[.]com/login
```

The URL is suspicious because it uses Microsoft-themed language but does not point to a known Microsoft domain. The words “microsoft,” “security,” and “check” are meant to make the link appear legitimate, but the actual domain is separate from Microsoft.

The message also frames the link as a required account verification step. That is a common credential-harvesting lure: the user is pressured into visiting a fake login page and entering credentials.

For the writeup, I kept the URL defanged as:

```text
hxxps://microsoft-security-check[.]com/login
```

This defanging process prevents it from becoming a clickable link in the public report.

### 4. Social Engineering Indicators

This email used several common phishing tactics:

- **Brand impersonation:** The message claims to come from Microsoft Security Team.
- **Lookalike sender domain:** `micros0ft-support[.]com` uses `0` instead of `o`.
- **Urgency:** The password is supposedly expiring today.
- **Impact threat:** The email warns about possible account suspension.
- **Credential harvesting lure:** The user is asked to verify the account.
- **Suspicious login URL:** `microsoft-security-check[.]com` uses Microsoft-themed wording but is not a Microsoft domain.
- **Time pressure:** The user is told to act within two hours.

<img width="1724" height="912" alt="edit email infographic" src="https://github.com/user-attachments/assets/06d7ffc8-597e-4920-889c-751af979fdcb" />

None of these indicators alone would require a dramatic conclusion, but together they create a clear phishing pattern.

### 5. IOC Extraction

After reviewing the email, I extracted the indicators that would be useful for detection, blocking, or follow-up investigation.

| Indicator Type | Value | Notes |
|---|---|---|
| Sender Email | `security-alert@micros0ft-support[.]com` | Lookalike Microsoft-themed sender |
| Sender Domain | `micros0ft-support[.]com` | Uses `0` instead of `o` |
| URL | `hxxps://microsoft-security-check[.]com/login` | Credential verification lure |
| URL Domain | `microsoft-security-check[.]com` | Microsoft-themed but not a known Microsoft domain |
| Subject | `Urgent: Password Expiration Notice` | Uses urgency and account-access pressure |
| Requested Action | Verify account credentials | Possible credential harvesting attempt |
| Attachment | None observed | Link-based phishing attempt |

### 6. Final Verdict

**Verdict:** Phishing / credential harvesting attempt  
**Severity:** Medium  
**Confidence:** High  

I assessed this email as a likely credential-harvesting phishing attempt. The message impersonates Microsoft, uses a lookalike sender domain, pressures the user to act quickly, and directs the user to a Microsoft-themed login URL that does not appear to belong to Microsoft.

I rated the severity as **Medium** because the email is designed to steal credentials, but there is no evidence in this mock scenario that a user clicked the link or submitted credentials. If a user had entered credentials, or if this had been sent broadly across an organization, the severity would increase.

**Recommended Action:**

- Do not click the link.
- Report the email to the security team.
- Block or monitor the sender domain and URL domain.
- Add the sender/domain/URL to detection or blocklists if confirmed malicious.
- Search mailboxes for additional recipients.
- If a user clicked the link, review browser history, authentication logs, and sign-in activity.
- If credentials were submitted, reset the user’s password and revoke active sessions.

## Key Findings

### The Sender Domain Was a Strong Indicator

The sender display name claimed to be Microsoft Security Team, but the actual sender domain was `micros0ft-support[.]com`. The use of `0` instead of `o` was one of the clearest signs that the sender was attempting to impersonate Microsoft.

### The URL Did Not Match the Claimed Sender

The link used Microsoft-themed wording, but it did not point to a known Microsoft domain. This mismatch between the claimed sender and the actual URL increased suspicion.

### The Email Used Pressure to Drive Action

The message created urgency by claiming the password would expire today and that access could be suspended within two hours. This is a common social engineering tactic because it pushes the user to react before thinking carefully.

### The Email Appeared Designed for Credential Harvesting

The requested action was to “verify” the account through a login-style link. Based on the sender, URL, and message content, the likely goal was credential collection.

### No Attachment Was Present

This sample did not include an attachment. That helped narrow the investigation toward sender analysis, URL inspection, and credential-harvesting indicators rather than attachment-based malware delivery.

## Key Competencies Demonstrated

- Phishing email triage
- Sender and display-name review
- Lookalike domain identification
- Safe URL inspection and defanging
- Social engineering analysis
- IOC extraction
- Severity and confidence assessment
- Final verdict writing
- SOC-style documentation
- Clear communication of risk and recommended actions

## Employer-Relevant Skills

This project demonstrates a basic but important SOC workflow: reviewing a suspicious email and determining whether it is likely phishing.

For entry-level SOC work, phishing triage is especially relevant because analysts are often asked to review user-submitted emails, identify suspicious senders or links, extract IOCs, and recommend next steps. This project shows that I can slow down, review the message carefully, identify the suspicious pieces, and explain the verdict in plain language.

It also demonstrates safe handling habits, including defanging URLs and avoiding interaction with live suspicious links.

## SOC Relevance

From a SOC perspective, this email would be treated as a likely phishing attempt because it combines brand impersonation, urgency, a lookalike sender domain, and a suspicious credential-verification link.

In a real environment, the next step would be to determine scope and impact. An analyst would want to know whether the email reached one user or many users, whether anyone clicked the link, whether credentials were submitted, and whether there were any suspicious sign-ins afterward.

For this lab, the final outcome was:

- **Verdict:** Phishing / credential harvesting attempt
- **Severity:** Medium
- **Confidence:** High
- **Recommended Action:** Report and block the sender/domain/URL, search for additional recipients, and investigate any user interaction with the link.

## Basic Triage Timeline

| Step | Action |
|---|---|
| T0 | Received or reviewed mock phishing email sample. |
| T1 | Reviewed sender display name and sender email address. |
| T2 | Identified lookalike sender domain `micros0ft-support[.]com`. |
| T3 | Reviewed the suspicious URL `hxxps://microsoft-security-check[.]com/login`. |
| T4 | Identified urgency, time pressure, brand impersonation, and credential-verification language. |
| T5 | Extracted IOCs for sender, domain, URL, subject, and requested action. |
| T6 | Assigned verdict, severity, confidence, and recommended actions. |

## HUMINT to SOC Translation

My previous work in competitive intelligence and HUMINT-style research involved reviewing incomplete information, checking source credibility, identifying inconsistencies, and writing clear findings for decision-makers.

This phishing triage lab used a similar process in a cybersecurity context.

Instead of evaluating a human source or interview response, I evaluated an email. Instead of checking whether a source was credible, I checked whether the sender domain, display name, and link matched the claimed identity. Instead of looking for inconsistencies in a market or company story, I looked for inconsistencies between the email’s branding, sender, URL, and requested action.

The process was familiar: review the source, validate the details, identify red flags, assess the likely intent, and document the conclusion clearly.

## What I Learned

This lab helped me understand that phishing triage is not just about spotting a “bad-looking” email. A SOC analyst needs to explain why the message is suspicious and what should happen next.

The strongest evidence in this sample was not one single indicator. It was the combination of a lookalike sender domain, a suspicious Microsoft-themed URL, urgency, account-access pressure, and a credential-verification request.

I also learned the value of defanging URLs in public writeups. Even in a mock project, it is a good habit to avoid creating clickable suspicious links.

## Next Steps

For a future version of this lab, I want to expand the analysis by:

- Reviewing full email headers.
- Comparing SPF, DKIM, and DMARC results.
- Using safe domain reputation tools.
- Checking URL redirects in a controlled sandbox.
- Building a phishing triage checklist.
- Writing a user-facing response explaining why the email is unsafe.
- Creating a mock ticket-style SOC case note.

## Author

- GitHub: [Malakh Fuller](https://github.com/MalakhFuller)
- LinkedIn: [Malakh Fuller](https://www.linkedin.com/in/malakhfuller/)
