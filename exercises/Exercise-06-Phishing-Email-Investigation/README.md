# Exercise 06 — Phishing Email Investigation

## Overview

In this exercise, I investigated a reported email impersonating Microsoft 365. The goal was to identify phishing indicators, inspect the embedded URL and HTML attachment, review the available threat intelligence, and assess whether there was evidence of user compromise.

## Scenario

A user reported receiving an email asking them to verify their Microsoft 365 mailbox within two hours to avoid access restrictions. The user reported that they had not clicked the link, opened the attachment, or entered their credentials.

## Investigation

### 1. Initial triage and email headers

I checked the file type, calculated its SHA-256 hash, and reviewed the email headers.

The sender address used `micros0ft-support.example`, which imitates the Microsoft name by replacing the letter `o` with the number `0`. The `Reply-To` address pointed to an unrelated Gmail account, and the email authentication results showed `SPF=fail`.

These are suspicious indicators. The SPF result alone does not establish the full authentication status of the message, and the different `Return-Path` is not conclusive on its own.

### 2. URL analysis

The email contained a link whose displayed text was `https://portal.office.com`, while its actual hyperlink target pointed to:

`https://micros0ft-support.example/verify?case=88421`

This mismatch is a deceptive hyperlink intended to make the destination appear legitimate. The destination domain also imitates Microsoft's name.

### 3. Attachment analysis

I extracted and inspected the attached HTML file as text. It contained a mailbox verification page with email and password input fields, which is consistent with a credential-phishing lure.

The sample was a static training artefact. The inspected HTML contained no script, network request, or working form submission. I did not open it in a browser or submit any information.

### 4. Threat intelligence review

I compared the indicators with the CSV file supplied for the exercise. The feed marked the imitation domain as malicious and the sample IP address as suspicious. It also listed `portal.office.com` as a known-good domain and returned no match for the attachment hash.

This was a **simulated training feed**, not a live reputation check. The IP address `203.0.113.45` belongs to the documentation-only TEST-NET-3 range, and the `.example` domains are reserved for examples. They must not be treated as real attacker infrastructure. A hash with no match in this small feed is not proof that a file is safe.

## Key findings

- The sender domain imitates Microsoft and does not match the brand it claims to represent.
- The `Reply-To` address is unrelated to the claimed sender.
- SPF authentication failed in the supplied headers.
- The visible hyperlink text did not match the actual destination.
- The HTML attachment displayed email and password fields, suggesting an attempt to collect credentials.
- The user reported no interaction with the link or attachment and did not submit credentials.

**Assessment:** The message is a high-confidence phishing attempt. Based on the available evidence, there is no confirmed user compromise. This does not rule out activity that was not covered by the supplied evidence.

## Recommended response

1. Quarantine or remove the email and search other mailboxes for the same message.
2. Review email, DNS, proxy, and endpoint logs for activity related to the indicators.
3. Block the malicious destination in the relevant security controls after validating the indicator and following the organisation's change process.
4. Since the user reported no interaction, record that there is no confirmed compromise. If credentials were entered, reset the password, revoke active sessions, and review sign-in activity.

## Evidence

Supporting terminal screenshots are stored in the [`screenshots/`](screenshots/) folder. They show the commands used and the corresponding output.

## Conclusion

The investigation identified multiple indicators of a credential-phishing attempt, especially the sender impersonation and the mismatch between the displayed URL and its actual destination. The attachment supported the same pretext by presenting a mailbox verification form. The user reported no interaction, so no compromise was confirmed from the available evidence.

## Lab note

This exercise used static, simulated training artefacts. No live phishing infrastructure was contacted, no malware was executed, and no credentials were collected. The threat intelligence results were supplied by the lab and were not obtained from external reputation services.
