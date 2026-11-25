# Exercise 07 — Linux SSH Brute-Force Detection

## Overview

In this exercise, I built and tested a small SSH monitoring scenario in my Proxmox homelab. The goal was to detect repeated SSH authentication failures from two different perspectives:

- **Wazuh** for host-based authentication monitoring.
- **Suricata** for network-based SSH scan detection.

All tests were performed against systems in my own lab using a non-existent test account and dummy passwords.

## Lab Environment

| Component | Role |
|---|---|
| Kali Linux | Source system for controlled SSH tests |
| Ubuntu Server | SSH target and Wazuh agent |
| Wazuh Manager | Central collection and analysis of security events |
| OPNsense | Routing, firewall rules and network address translation |
| Suricata | Network traffic inspection using ET Open rules |

The Wazuh Manager and agent were both running version **4.14.8**. Internal IP addresses, hostnames and other lab-specific identifiers are intentionally omitted.

## Network Preparation

The lab systems were placed on separate network segments. Before testing, I configured the required routing and firewall access so that:

- The Ubuntu agent could reach the Wazuh Manager on TCP ports **1514** and **1515**.
- Kali could reach the Ubuntu SSH service on TCP port **22**.
- The existing isolation between network segments remained in place, with only the required traffic permitted.

The initial Wazuh connection tests timed out. Packet capture showed the TCP connection request reaching OPNsense and a SYN-ACK returning with an unexpected layer-2 destination. I adjusted the `reply-to` setting on the specific WAN pass rule. After applying the change, connectivity tests to both Wazuh ports succeeded.

I also added temporary static routes on Ubuntu and Kali. These routes were created with `ip route` and are **not persistent**; they must be added again after the respective VM is restarted.

## Wazuh Agent and Log Collection

I installed **Wazuh Agent 4.14.8-1** on Ubuntu and configured it to register with the Wazuh Manager. After enabling and starting the service, the agent appeared as **Active** in the Wazuh Dashboard.

The agent already had a `journald` collection entry in its `ossec.conf` configuration. I confirmed that SSH authentication events were available through the system journal, so I kept the existing journald configuration instead of adding a second input for `auth.log`. This avoided collecting the same authentication events through two separate inputs.

I first tested the logging pipeline locally with a connection attempt using a non-existent username. Ubuntu recorded the invalid-user and failed-password messages, and Wazuh generated **Rule 5710**.

## SSH Brute-Force Simulation

From Kali, I used **THC Hydra** with a small wordlist containing eight dummy passwords and a non-existent test username. The test was limited to the Ubuntu SSH service in my lab and used one parallel task.

Ubuntu recorded repeated authentication failures. The Wazuh Dashboard showed the corresponding SSH events, including:

| Wazuh rule | Detection | Result |
|---|---|---|
| **5710** | Attempt to log in using a non-existent user | Triggered |
| **5712** | Brute-force activity involving a non-existent user | Triggered |
| **2502** | User missed the password more than once | Also observed |

Rule **5712** was generated after the configured correlation threshold of eight matching authentication failures was reached. The alert had level **10**.

### NAT observation

The Ubuntu SSH logs and Wazuh events identified the OPNsense WAN address as the source of the forwarded SSH attempts, rather than the original Kali address. This was caused by source NAT on the path between the ATTACK_NET segment and the Ubuntu network.

This is an important distinction when investigating events: the host-based log showed the translated source, while Suricata monitoring on ATTACK_NET was able to see the original Kali source before NAT.

## Suricata SSH Scan Detection

Initially, Suricata was monitoring only the LAN interface. The Kali-to-Ubuntu test traffic did not cross that interface, so no matching alert was generated there.

I updated the Suricata configuration to:

- Monitor both **LAN** and **ATTACK_NET**.
- Keep the existing HOME_NET definition for the lab LAN and add the Ubuntu target as a single protected host.
- Keep Kali outside HOME_NET so that it was treated as the external source for this test.

I then made a separate network test using six individual TCP connection attempts to the Ubuntu SSH port with `nc`. This was a separate test from the Hydra authentication simulation.

Suricata generated the following alert:

| Field | Result |
|---|---|
| Alert | **ET SCAN Potential SSH Scan** |
| SID | **2001219** |
| Protocol | TCP |
| Source | Kali test system |
| Destination | Ubuntu SSH service |
| Destination port | **22** |
| Interface | **ATTACK_NET** |

This confirmed that Suricata was inspecting the correct interface and detecting the SSH connection pattern. The alert was a network-level scan detection; it did not identify password attempts or reveal SSH credentials because SSH traffic is encrypted.

## Findings

- Wazuh successfully collected SSH authentication events from Ubuntu through `journald`.
- Rule **5710** identified attempts using a non-existent username.
- Rule **5712** correlated repeated failures and generated a level 10 alert.
- Suricata rule **2001219** detected the separate sequence of SSH connection attempts on ATTACK_NET.
- Monitoring the correct interface and defining HOME_NET correctly were necessary for the Suricata test to work.
- Source NAT changed the source address visible in Ubuntu's SSH logs. Capturing traffic on ATTACK_NET preserved visibility of the original Kali source.
- Wazuh and Suricata provided complementary results: Wazuh identified authentication failures on the endpoint, while Suricata identified a network connection pattern.

## Conclusion

The exercise successfully demonstrated SSH brute-force monitoring with Wazuh and SSH scan detection with Suricata. It also showed how routing, firewall rules, NAT and IDS interface selection affect what each security tool can observe.

The two detections came from separate tests: Wazuh Rule 5712 was triggered during the Hydra authentication simulation, while Suricata SID 2001219 was triggered during the separate six-connection `nc` test.

## Screenshots

Screenshots and supporting evidence for this exercise are stored in the `screenshots/` directory.
