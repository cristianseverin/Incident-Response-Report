# [The Cyber Range](http://joshmadakor.tech/cyber-range) Project

# Cyber Defense Final — Advanced Honey Pot w/Live Breach

In this project, I investigated suspicious activity against a Windows VM hosting MySQL in The Cyber Range. I correlated database audit logs, Microsoft Defender for Endpoint telemetry, and network flow data to reconstruct the incident and guide response actions.

_**Initial State:**_ An internet-exposed lab VM with broadly accessible services and remote MySQL root access.

_**Project Outcome:**_ Completed an incident investigation, documented findings and evidence gaps, performed containment and hardening, and compared MDE investigation packages. Full database recovery and remediation effectiveness remain unverified.

---

## Technology Utilized

- Microsoft Azure Virtual Machines
- Microsoft Defender for Endpoint (MDE)
- Advanced Hunting and Kusto Query Language (KQL)
- Azure Log Analytics
- Azure Network Security Groups (NSGs) and network flow analytics
- MySQL Server and MySQL Workbench
- Windows Defender and Windows Firewall

---

## Table of Contents

- [Step 1: Environment and Evidence Review](#step-1-environment-and-evidence-review)
- [Step 2: MySQL Authentication Investigation](#step-2-mysql-authentication-investigation)
- [Step 3: Database Activity and Ransom Message](#step-3-database-activity-and-ransom-message)
- [Step 4: Windows Authentication Investigation](#step-4-windows-authentication-investigation)
- [Step 5: Network Traffic Correlation](#step-5-network-traffic-correlation)
- [Step 6: Incident Timeline](#step-6-incident-timeline)
- [Step 7: Containment and Hardening](#step-7-containment-and-hardening)
- [Step 8: Recovery and Connectivity Troubleshooting](#step-8-recovery-and-connectivity-troubleshooting)
- [Step 9: MDE Investigation Package Comparison](#step-9-mde-investigation-package-comparison)
- [Step 10: Final Incident Report](#step-10-final-incident-report)
- [Lessons Learned](#lessons-learned)

---

## Step 1: Environment and Evidence Review

I reviewed telemetry from the Azure Windows VM `corp-wsy-wt`, which hosted the MySQL server. The investigation focused on September 28 through October 1, 2026.

The evidence sources included:

| Source | Investigation Purpose |
|---|---|
| MySQL authentication logs | Review database connection attempts and source addresses |
| MySQL query logs | Identify database reads, ransom-message writes, and destructive statements |
| DeviceLogonEvents | Investigate Windows authentication and account activity |
| DeviceProcessEvents | Review process execution and available command lines |
| DeviceNetworkEvents | Correlate endpoint connections with suspicious activity |
| DeviceFileEvents | Review file activity and potential incident artifacts |
| DeviceRegistryEvents | Investigate configuration changes and possible persistence |
| NTANetAnalytics | Review network flows, traffic volume, and access rules |

I used KQL to filter the evidence by device, time, source address, and activity type. Written timeline entries use UTC; the original screenshots display Pacific time.

---

## Step 2: MySQL Authentication Investigation

I investigated remote MySQL connections and identified root access from the principal suspicious source, `64.89.163.141`.

The database authentication records, combined with accepted TCP/3306 connections, supported direct remote access to the MySQL service. Broad remote root privileges increased the potential impact of that access.

**Finding:** Remote privileged database access was established. How the credentials were obtained was not determined from available logs.

<img width="929" height="444" alt="image" src="https://github.com/user-attachments/assets/e990f286-7b2d-46c8-a8f4-dca173a8a53c" />

---

## Step 3: Database Activity and Ransom Message

After the root connection, the query logs recorded database enumeration, ransom-message creation, and bulk database reads.

The principal activity included:

- Creation of a `RECOVER_YOUR_DATA` database and table.
- Insertion of a ransom message demanding cryptocurrency payment.
- 31 unrestricted `SELECT *` statements during the principal collection window.
- Requests to drop four databases.
- Requests to remove binary logs, change privileges, and shut down the service.

The ransom message was also visible in MySQL Workbench.

**Finding:** The evidence established an extortion attempt and potentially destructive database activity. Query logs record received statements, so they did not independently prove that every command succeeded. The ransom message’s claims of data theft and deletion also required corroboration.

<img width="925" height="445" alt="image" src="https://github.com/user-attachments/assets/76d6c3ce-9fa5-40f9-83f6-0290237c2e9f" />

<img width="906" height="349" alt="image" src="https://github.com/user-attachments/assets/6958a7aa-9f67-466c-912e-6db8ec1c73b6" />

<img width="1232" height="481" alt="image" src="https://github.com/user-attachments/assets/ed21ee28-9bf9-4677-91e5-b130f59a009e" />

---

## Step 4: Windows Authentication Investigation

I reviewed Windows logon events and correlated successful administrator network authentication from public IP addresses with SMB traffic.

The investigation also examined process, file, and registry activity for signs of execution or persistence.

**Finding:** Suspicious Windows network authentication was observed. Attacker-controlled Windows execution, a completed public RDP session, and a direct connection between the Windows and MySQL activity were not determined from available logs.

<img width="914" height="469" alt="image" src="https://github.com/user-attachments/assets/e89272db-6059-4b99-b3c9-9d831ae64786" />

---

## Step 5: Network Traffic Correlation

I correlated endpoint connections and network flow analytics with the database activity.

During the principal collection window, flow records showed **4,909,352 bytes, approximately 4.9 MB, sent from the VM to the principal suspicious source**. Network analytics also recorded suspicious traffic permitted by the broad inbound rule `danger-allow-all_inbound`.

**Finding:** The timing and outbound volume supported likely data transfer. Network flows did not establish the exact contents transferred.

<img width="1078" height="394" alt="image" src="https://github.com/user-attachments/assets/7c4633a7-a67b-43ce-a2f8-f895ef758ed6" />

---

## Step 6: Incident Timeline

I combined database, endpoint, and network evidence to reconstruct the sequence of events.

| Date and Time, UTC | Observed Activity |
|---|---|
| September 28, 05:19:43–05:19:47 | Accepted MySQL connection followed by root access from the principal source |
| September 28, 05:20:18–05:20:19 | Database enumeration and ransom-message creation |
| September 28, 05:20:30–05:22:30 | Bulk database-read requests |
| September 28, 05:22:30–05:22:33 | Four DROP DATABASE statements submitted |
| September 28, 05:22:41–05:22:50 | Ransom message recreated; log removal, privilege changes, and shutdown requested |
| September 28, 06:22–06:25 | Public-source Windows administrator authentication correlated with SMB |
| September 30 and October 1 | Additional ransom-database creation requests observed |

The ransom-message writes preceded the principal bulk-read sequence. Attribution of all observed activity to a single operator remained unresolved.

---

## Step 7: Containment and Hardening

I performed the following response actions in the lab:

- Isolated the VM using Microsoft Defender for Endpoint.
- Hardened the VM’s Azure Network Security Group.
- Blocked public internet access to MySQL.
- Ran a full Windows Defender malware scan.
- Enabled Windows Firewall.
- Deleted the separate `administrator` account.
- Disabled the Guest account.
- Secured the retained local account with strong credentials.
- Addressed the network-accessible MySQL root account.

These actions addressed exposed services and unnecessary privileged access.

**Validation gap:** The available evidence did not establish the malware scan findings, final effective network rules, or the exact retained configuration of the remote MySQL root account.

<img width="1062" height="482" alt="image" src="https://github.com/user-attachments/assets/10e93078-3a50-4a72-8409-ddec778e3322" />


<img width="931" height="445" alt="image" src="https://github.com/user-attachments/assets/13a6fa8f-2449-4ccc-8d3c-07266305a36d" />


<img width="688" height="421" alt="image" src="https://github.com/user-attachments/assets/96a270c6-9405-4e7c-87ab-42abb824be45" />


<img width="737" height="403" alt="image" src="https://github.com/user-attachments/assets/5d1cfa7a-66be-4c35-a84b-af69b4e5eb18" />


<img width="926" height="449" alt="image" src="https://github.com/user-attachments/assets/01a10d47-52fb-43f5-9c43-cad308214d2b" />


<img width="928" height="497" alt="image" src="https://github.com/user-attachments/assets/2a9df8b3-09bf-4fbe-afee-73df978fad0d" />


---

## Step 8: Recovery and Connectivity Troubleshooting

After hardening the NSG, I started the VM and removed it from isolation.

Enabling Windows Firewall interrupted Azure Bastion connectivity. I restored management access using Azure’s remote-access configuration reset.

This step provided practice troubleshooting connectivity during incident recovery and highlighted the importance of preserving an approved management path when changing security controls.

**Recovery status:** Management access was restored. Successful database restoration and full incident recovery were not determined from available logs.


---

## Step 9: MDE Investigation Package Comparison

I compared two MDE investigation packages to examine changes in authentication, processes, services, scheduled tasks, local accounts, and other endpoint artifacts.

| Review Area | Key Finding |
|---|---|
| Authentication | Nine successful privileged NTLM network logons from three public sources |
| Failed logons | 72 failed network logons from 11 public IP addresses |
| Firewall configuration | Profiles changed from enabled to disabled before the observed public authentication; attribution unresolved |
| Processes | Snapshot differences did not establish attacker-controlled execution |
| Scheduled tasks and accounts | No new malicious persistence mechanism or backdoor account established |
| Evidence limitations | Log retention, incomplete command lines, and collection activity limited conclusions |

The earlier package was the earliest available collection, rather than a proven clean baseline.

**Finding:** The comparison helped separate endpoint changes from evidence of malicious activity. It did not independently establish data theft, while the separate MySQL and network evidence supported likely transfer.


---

## Step 10: Final Incident Report

I consolidated the investigation into an incident report covering:

- Executive summary and incident classification.
- Affected assets and potential impact.
- Indicators of compromise.
- A chronological incident timeline.
- Access conditions and unresolved attack-vector questions.
- Containment, hardening, and recovery actions.
- Evidence references and investigation limitations.
- Prioritized recommendations.
- MDE investigation package comparison.

The report distinguishes observed activity, analyst assessments, and outcomes that could not be verified.

[Final Incident Report](https://docs.google.com/document/d/1jmHK_rTeGdQTFzmNylHFGP8WDvX-UHmsDCNDeyGMo0M/edit?usp=sharing)

---

## Lessons Learned

- **Restrict service exposure.** Limit database and Windows administration services to approved networks and management paths.
- **Apply least privilege.** Broad remote root access increases the impact of compromised database credentials.
- **Correlate evidence sources.** Authentication logs, database statements, endpoint events, and network flows each explain a different part of the incident.
- **Validate response outcomes.** A completed configuration change or restored connection does not prove that recovery is complete.
- **Document uncertainty.** Distinguishing confirmed facts from likely activity makes an incident report more useful and defensible.

This project strengthened my experience with KQL, database investigations, endpoint analysis, incident response, and technical reporting.

