# Exercise 05 — Active Directory Authentication Attack Investigation

**Focus:** Detecting repeated Kerberos authentication failures, investigating successful authentication, and distinguishing suspicious activity from normal user error.

## 1. Overview

The objective of this exercise was to investigate repeated failed authentication attempts against an Active Directory test account.

Wazuh generated alerts for failed authentication activity. Since repeated password failures can indicate password guessing, I reviewed the related Windows Security events, identified the source workstation, and checked whether successful authentication followed the failures.

The goal was not only to generate an alert, but to determine whether the activity indicated a real account compromise or benign user behaviour.

All tests were performed in an isolated homelab using a dedicated, non-privileged test account.

## 2. Lab Environment

| Component | Role |
|---|---|
| Windows Server / Domain Controller | Active Directory authentication and Security event collection |
| Windows workstation | Source of the controlled authentication attempts |
| Wazuh Manager and Dashboard | SIEM collection, alerting, correlation, and investigation |

Sensitive hostnames, domain details, account identifiers, IP addresses, SIDs, and other environment-specific values have been omitted from this document.

## 3. Scenario and Test Method

The scenario represented a user account receiving several incorrect password attempts, followed by a successful authentication.

Before testing, I checked the domain Account Lockout Policy. The lockout threshold was set to **Never** in this isolated lab, allowing the controlled attempts without locking the test account.

On the Domain Controller, I enabled the relevant auditing settings:

- Logon — Success and Failure
- Kerberos Authentication Service — Failure
- Credential Validation — Success and Failure

The Wazuh agent was already configured to collect the Windows **Security** event channel. Its existing event filter did not exclude Event ID `4771`.

From the workstation, I used the following command to request an SMB connection with the test account:

```cmd
net use \\<DOMAIN_CONTROLLER>\IPC$ /user:<DOMAIN>\<TEST_ACCOUNT> *
```

The password was entered at the prompt. I intentionally entered an incorrect password several times, then repeated the command with the correct password. The successful attempt returned:

```text
The command completed successfully.
```

## 4. Windows Security Event Analysis

The main event in this investigation was **Event ID 4771 — Kerberos pre-authentication failed**.

The failed events contained the following evidence:

| Field | Observed value / meaning |
|---|---|
| Event ID | `4771` |
| Account | Dedicated test account |
| Client | Lab workstation |
| Service | Kerberos ticket-granting service (`krbtgt`) |
| Failure code | `0x18` |
| Pre-authentication type | `2` |

The `0x18` failure code indicated invalid Kerberos pre-authentication information. In this controlled test, it was consistent with the intentionally incorrect password entries.

Other relevant Windows Security events reviewed during the investigation:

| Event ID | Meaning | How it was used |
|---|---|---|
| `4624` | An account successfully logged on | Reviewed for context. The sampled event associated with a machine/service account was not treated as proof that the test account had logged on. |
| `4625` | An account failed to log on | Reviewed as a general failed-logon event. One event found on the workstation was a local interactive logon (`Logon Type 2`, loopback source), so it was not treated as the primary evidence for the Domain Controller's Kerberos failures. |
| `4768` | A Kerberos authentication ticket (TGT) was requested | A successful result (`0x0`) was observed for the test account after the failed attempts. |
| `4769` | A Kerberos service ticket was requested | Successful ticket requests (`0x0`) were observed for the test account after the failed attempts. |
| `4771` | Kerberos pre-authentication failed | Primary evidence for the repeated failed Kerberos attempts. |
| `4776` | NTLM credential validation | Considered during the investigation, but the observed failed attempts were Kerberos events (`4771`). |

The successful `4768` and `4769` events, together with the successful `net use` result, showed that the test account subsequently authenticated successfully. A `4624` event associated with a machine/service context was not used by itself as proof that the test account had logged on.

## 5. Wazuh Detection

Wazuh collected the Security events from the Domain Controller and generated the following alerts:

| Rule ID | Level | Description / purpose |
|---|---:|---|
| `60104` | 5 | Built-in **Windows audit failure event** rule. It identified the failed Windows audit event. |
| `100301` | 5 | Custom rule for a Kerberos pre-authentication failure (`4771`) with status `0x18`. |
| `100302` | 10 | Custom correlation rule for five matching failures from the same account and source within 300 seconds. |

The custom correlation alert included the preceding events in `previous_output`. The five failed attempts used for validation occurred within approximately 88 seconds, so the `100302` alert triggered as expected.

### Custom rule 100301 — Kerberos pre-authentication failure

This rule detects an individual Event ID `4771` with status `0x18`. It uses the built-in rule `60104` as its parent and adds conditions specific to the Kerberos failure being investigated.

### Custom rule 100302 — Repeated Kerberos failures

This rule detects five matches of rule `100301` within 300 seconds, requiring the same account and source address. It raises a Level 10 alert and maps the activity to **MITRE ATT&CK T1110.001 — Password Guessing**.

The rule detects a repeated-failure pattern. It does not determine by itself whether the cause is an attacker or a legitimate user entering an incorrect password, and it does not detect a subsequent successful logon.

### Custom rules used in the exercise

The following is the tested rule logic. The group and rule IDs are retained; environment-specific values are represented through event fields rather than hard-coded identifiers.

```xml
<group name="windows,authentication,kerberos,">

  <rule id="100301" level="5">
    <if_sid>60104</if_sid>
    <field name="win.system.eventID">^4771$</field>
    <field name="win.eventdata.status">^0x18$</field>

    <description>Kerberos pre-authentication failed for $(win.eventdata.targetUserName) from $(win.eventdata.ipAddress).</description>
    <group>kerberos,authentication_failed,</group>
  </rule>

  <rule id="100302" level="10" frequency="5" timeframe="300">
    <if_matched_sid>100301</if_matched_sid>
    <same_field>win.eventdata.targetUserName</same_field>
    <same_field>win.eventdata.ipAddress</same_field>

    <description>Possible Kerberos password guessing: 5 failed attempts against the same account from the same source within 5 minutes.</description>

    <mitre>
      <id>T1110.001</id>
    </mitre>

    <group>kerberos,authentication_failed,correlation,</group>
  </rule>

</group>
```

## 6. Investigation Findings

The investigation established the following:

- Multiple failed Kerberos authentication attempts were recorded for the same test account.
- The failed attempts originated from the same lab workstation.
- The events contained status code `0x18`, consistent with incorrect password attempts in this test.
- Wazuh detected individual failures with rule `100301` and generated the Level 10 correlation alert `100302` after five matching events within the configured time window.
- Successful Kerberos ticket events (`4768` and `4769`) followed the failed attempts.
- The SMB connection test subsequently completed successfully with the correct password.
- No additional suspicious accounts or evidence of unauthorized account access were identified during this exercise.

The Wazuh alert correctly identified a repeated authentication-failure pattern. However, the known test sequence and the surrounding evidence were consistent with a legitimate user entering an incorrect password several times and then authenticating successfully. No account compromise was established.

## 7. MITRE ATT&CK Mapping

| Technique | Name | Relevance |
|---|---|---|
| `T1110.001` | Password Guessing | Used by custom rule `100302` to classify repeated failed password attempts. |

The mapping describes the behaviour the rule is designed to detect. It is not, by itself, proof that an attack took place.

## 8. Final Assessment

**Assessment: Benign activity in the context of this lab exercise; no confirmed account compromise.**

The exercise demonstrated that repeated failed authentication attempts can look suspicious and should generate an alert, but the alert must be investigated before classifying the activity as malicious.

Wazuh successfully collected and detected the Kerberos failures. The custom rules provided visibility into both individual failures and repeated attempts from the same source. Reviewing the surrounding authentication events made it possible to distinguish the detected pattern from a confirmed attack.

## 9. Possible Improvements

- **Account Lockout Policy:** The lab used a `Never` lockout threshold to allow controlled testing. In a real domain, configure an appropriate lockout threshold and duration through domain policy, based on the organisation's risk and operational requirements.
- **Authentication correlation:** Extend the investigation to correlate repeated failures with later successful authentication for the same account and source. This should be a separate detection or investigation step; rule `100302` currently detects failures only.
- **Alert enrichment:** Include account criticality, workstation context, and other related authentication activity to help analysts assess the alert.
- **Response automation:** Consider notifications or a controlled response workflow. Do not automatically disable or lock an account solely because this alert fired without considering false positives and business impact.
- **Archive management:** JSON event archiving was enabled during troubleshooting to inspect raw events. Disable archive collection after the exercise if it is no longer needed, since archiving all events can increase storage use.

## 10. Skills Practiced

- Active Directory authentication monitoring
- Windows Security Event Log analysis
- Kerberos event investigation (`4768`, `4769`, `4771`)
- Wazuh SIEM alert triage
- Custom Wazuh rule creation and validation
- Event correlation by account and source
- MITRE ATT&CK mapping
- Distinguishing a suspicious pattern from confirmed malicious activity
- Basic incident assessment and reporting

## 11. Evidence to Include in the Repository

Add sanitized screenshots alongside this document if you want to show the investigation workflow:

- Windows Security Event ID `4771` with the Kerberos failure code.
- Wazuh built-in alert `60104`.
- Wazuh custom single-event alert `100301`.
- Wazuh correlation alert `100302`, including its previous-event context.
- Successful Kerberos events `4768` and `4769` for the test account.

Before publishing, remove or mask internal IP addresses, real hostnames, domain names, account names, SIDs, GUIDs, logon IDs, ports, and any other infrastructure-specific identifiers.
