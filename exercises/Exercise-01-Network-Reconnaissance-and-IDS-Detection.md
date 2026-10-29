# Exercise 01 — Network Reconnaissance & IDS Detection

**Author:** Milan Nikolic  
**Project:** Cybersecurity Homelab  
**Exercise:** 01 — Network Reconnaissance & IDS Detection  
**Date:** October 29, 2026  
**Status:** Successfully Completed

## 1. Objective

The objective of this exercise was to perform controlled network reconnaissance against a deliberately vulnerable target in an isolated cybersecurity lab and verify that suspicious network activity could be detected by Suricata and forwarded to Wazuh for centralized analysis.

This exercise focused on understanding the relationship between network scanning, intrusion detection, security event collection, and incident investigation.

## 2. Lab Environment

The exercise used the following components:

| Component | Role |
|---|---|
| Kali Linux | Attacking and reconnaissance machine |
| OPNsense | Firewall and network traffic routing |
| Suricata | Network intrusion detection system (IDS) |
| Metasploitable | Deliberately vulnerable target machine |
| Wazuh | Security information and event management (SIEM) platform |

The attacker and target were connected through the lab's segmented network architecture, with OPNsense routing traffic between the relevant network segments.

Private IP addresses have been replaced with descriptive placeholders to avoid exposing internal network details.

## 3. Methodology

### 3.1. Network Connectivity Verification

Before testing, the routing configuration was checked to ensure that traffic from the attacking machine reached the target through OPNsense.

Connectivity to the target was verified before performing reconnaissance.

### 3.2. Port Scanning

Nmap was used to scan TCP ports 1–1000 on the target machine.

The scan identified the following open ports:

| Port | Commonly Associated Service |
|---:|---|
| 21 | FTP |
| 22 | SSH |
| 23 | Telnet |
| 25 | SMTP |
| 53 | DNS |
| 80 | HTTP |
| 111 | RPCBind / Portmapper |
| 139 | NetBIOS Session Service |
| 445 | SMB |
| 512 | rexec |
| 513 | rlogin |
| 514 | rsh |

These results provide an initial overview of the target's exposed services. The service names are based on conventional port assignments; additional enumeration would be required to verify the actual services, versions, configurations, and associated vulnerabilities.

### 3.3. Controlled SSH Scanning Activity

Controlled TCP SYN probes were sent from the attacking machine to the target's SSH service on TCP port 22.

The purpose was to generate test traffic and determine whether the configured intrusion detection system recognized the activity.

No successful compromise was established as part of this exercise.

### 3.4. Suricata Alert Analysis

Suricata generated the following alert:

- **Signature:** ET SCAN Potential SSH Scan
- **Signature ID (SID):** `2001219`
- **Source IP:** `ATTACKER_IP`
- **Destination IP:** `TARGET_IP`
- **Destination port:** `22`
- **Protocol:** TCP
- **Alert timestamp:** October 29, 2026, at 17:19:11

The alert indicated network activity matching the detection rule for potential SSH scanning.

This is a detection of suspicious behavior, not proof that an attacker successfully accessed or compromised the target.

### 3.5. Wazuh Event Verification

The corresponding security event was located in Wazuh Threat Hunting.

The event was processed under custom Wazuh rule ID `100100`, confirming that the Suricata alert reached the SIEM and was available for centralized investigation.

This validated the integration between the network intrusion detection system and the security monitoring platform.

## 4. Findings

The exercise produced the following results:

- Nmap identified 12 open TCP ports on the target.
- Controlled SSH SYN probes generated a Suricata alert.
- Suricata identified the activity using SID `2001219`.
- The source and destination addresses in the alert matched the test environment.
- Wazuh received the corresponding event under rule ID `100100`.
- No evidence of successful authentication, unauthorized access, or system compromise was established by this exercise.

## 5. Security Analysis

The detection indicates that network activity matched a known pattern associated with potential SSH scanning. Such behavior can occur during legitimate security testing, automated reconnaissance, or malicious activity.

The alert should therefore be treated as an indicator requiring investigation rather than definitive evidence of a security breach.

A further investigation would examine:

1. The source of the activity and whether it was authorized.
2. The timing, frequency, and volume of connection attempts.
3. SSH authentication logs and relevant system events on the target.
4. Additional Suricata alerts and related Wazuh events.
5. Whether any successful authentication or unauthorized changes occurred.

Correlating network alerts with endpoint and authentication logs would help determine whether the activity remained reconnaissance or progressed to an attempted or successful intrusion.

## 6. Conclusion

The exercise was successfully completed.

The attacking machine performed network reconnaissance against the designated target, and the monitoring infrastructure detected the controlled SSH scanning activity. Suricata generated the expected alert, and Wazuh received the corresponding event under the configured custom rule.

This demonstrates that the lab's network detection and centralized monitoring pipeline is functioning as intended for the tested scenario.

The exercise also reinforces an important incident response principle: **a security alert is evidence of detected activity, not automatic proof of compromise.** Additional evidence is necessary to determine the impact and whether a security incident occurred.

## 7. Skills Practiced

- Network reconnaissance with Nmap
- TCP port identification
- Basic TCP SYN scanning concepts
- Network traffic monitoring
- Suricata IDS alert analysis
- Wazuh SIEM event investigation
- Source and destination analysis
- Security event correlation
- Initial incident assessment and evidence-based reporting

## 8. Limitations and Next Steps

This exercise focused on reconnaissance and alert validation. It did not include exploitation, credential testing, or verification of successful unauthorized access.

Potential follow-up exercises include:

- Enumerating selected services and validating their configurations.
- Investigating SSH authentication events.
- Correlating Suricata alerts with endpoint telemetry.
- Creating and testing custom Wazuh detection rules.
- Writing an incident report based on correlated network and host evidence.

All future testing should remain within the authorized, isolated lab environment.
