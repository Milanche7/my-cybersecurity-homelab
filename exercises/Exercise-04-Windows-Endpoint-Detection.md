# Exercise 04 — Windows Endpoint Detection & Threat Hunting

## Objective

Monitor Windows process creation with Sysmon, forward the events to Wazuh, and detect PowerShell launched with an encoded command.

## Lab Setup

- Windows client with Sysmon and the Wazuh agent.
- Wazuh Manager for collecting and analyzing endpoint events.
- OPNsense with Suricata for network-based detection.

The Wazuh agent was configured to collect the Sysmon Operational event channel:

```xml
<localfile>
    <location>Microsoft-Windows-Sysmon/Operational</location>
    <log_format>eventchannel</log_format>
</localfile>
```

## Test

First, I ran basic PowerShell commands:

```powershell
whoami; hostname; Get-Date
```

Then I ran the same commands through PowerShell's `-EncodedCommand` parameter. The decoded Base64 payload was:

```powershell
whoami; hostname; Get-Date
```

These commands were benign and used only to generate process-creation telemetry for the test.

## Results

- Sysmon generated **Event ID 1 — Process Create** for the PowerShell process.
- The event contained the `-EncodedCommand` argument in its command line.
- Sysmon also recorded child processes such as `whoami.exe` and `hostname.exe`, with PowerShell listed as their parent process.
- The Wazuh agent successfully forwarded the Sysmon events to the Wazuh Manager.
- A temporary diagnostic rule (`100202`) confirmed that Sysmon Event ID 1 events were reaching Wazuh.
- Wazuh's built-in rule **92057** detected the encoded PowerShell execution. The alert had **level 12** and was mapped to **MITRE ATT&CK T1059.001 — PowerShell**.
- No new Suricata alerts were generated during this test.

## Detection Notes

The test payload was harmless. The presence of `-EncodedCommand` does not by itself prove malicious activity. It can also be used by legitimate administration, automation, remote-management, and deployment tools.

During an investigation, the command line should be reviewed together with the account, parent process, child processes, and other endpoint or network events. This context helps distinguish expected administrative activity from suspicious execution.

The custom Wazuh rule `100201` was added to the local rules file, but its independent activation was not confirmed during this exercise. The successful detection documented above came from the built-in rule `92057`.

## Wazuh and Suricata

Wazuh detected the local process execution through Sysmon endpoint telemetry. Suricata did not generate an alert because this test focused on local process creation and did not produce network activity that matched a Suricata detection rule.

## Conclusion

This exercise demonstrated how Sysmon process-creation events can be collected by Wazuh and used to detect encoded PowerShell execution. It also highlighted the importance of reviewing process context and not treating every encoded PowerShell command as malicious.
