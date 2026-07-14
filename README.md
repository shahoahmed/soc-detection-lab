# SOC Detection & Incident Response Lab

A hands-on Security Operations Center lab where I built a detection-and-response
environment, executed a real credential-theft attack chain against a Windows
endpoint, and detected, triaged, and investigated it end to end using EDR
telemetry.

Built by working through Eric Capuano's
[*So You Want to Be a SOC Analyst?*](https://blog.ecapuano.com/p/so-you-want-to-be-a-soc-analyst-intro)
series and documenting the full workflow. Credit for the lab design and attack
scenario goes to Eric Capuano; the environment build, execution, detection
review, and investigation captured here are my own hands-on work.

---

## What this lab demonstrates

- Standing up a multi-VM attacker/victim environment
- Configuring endpoint telemetry (Sysmon) and an EDR/SIEM sensor
- Executing a realistic Command-and-Control (C2) attack chain
- Detecting the attack through EDR telemetry and detection rules
- Triaging alerts and investigating root cause across process, network, and
  file-system data
- Mapping observed activity to MITRE ATT&CK

---

## Environment & tools

| Component | Details |
|-----------|---------|
| Attacker VM | Ubuntu 22.04 (VMware Workstation), hosting the C2 server |
| Victim VM | Windows 11 Enterprise (evaluation) |
| Network | Lab network on the `192.168.2.0/24` subnet |
| EDR / SIEM | LimaCharlie (endpoint sensor + cloud console) |
| Endpoint telemetry | Sysmon + Windows Event logs |
| C2 framework | Sliver |
| Attack tooling | procdump (LSASS memory dump) |
| Enrichment | VirusTotal (file-hash reputation) |

---

## Lab architecture

```
   ┌────────────────────┐         ┌────────────────────┐
   │   Ubuntu Attacker  │         │   Windows Victim   │
   │   (Sliver C2)      │────────▶│   (Sysmon +        │
   │   192.168.2.129    │  attack │    LimaCharlie EDR)│
   └────────────────────┘         │   192.168.2.130    │
                                  └─────────┬──────────┘
                                            │ telemetry
                                            ▼
                                  ┌────────────────────┐
                                  │  LimaCharlie Cloud │
                                  │  Detections /      │
                                  │  Timeline / Output │
                                  └────────────────────┘
```

---

## Walkthrough

### 1. Building the environment
Deployed the attacker and victim VMs and configured static networking on the lab
subnet.

![Network setup](screenshots/01-network-setup.jpg)

### 2. Deploying telemetry and the EDR sensor
Installed Sysmon for detailed process, network, and image-load telemetry, then
enrolled the LimaCharlie sensor and verified events were reaching the console.

![Sysmon telemetry](screenshots/02-sysmon-telemetry.jpg)

### 3. Reducing endpoint defenses (attacker foothold)
Disabled Microsoft Defender via Group Policy so the attack could execute in a
controlled lab. This keeps the focus on **detecting** the activity through EDR
rather than relying on prevention, and mirrors what an attacker attempts after
gaining a foothold.

![Defender disabled](screenshots/03-defender-disabled.jpg)

*MITRE ATT&CK: T1562.001 – Impair Defenses: Disable or Modify Tools*

### 4. Executing the attack chain
From the Sliver C2 server, ran an implant on the victim (`WEIRD_BASEBALL.exe`),
established a session, and used `procdump` to dump LSASS process memory, the
classic credential-access technique.

![LSASS attack](screenshots/04-lsass-attack.jpg)

*MITRE ATT&CK: T1055 – Process Injection · T1003.001 – OS Credential Dumping:
LSASS Memory · T1071 – Application Layer Protocol (C2)*

### 5. Detecting the activity
Reviewed LimaCharlie detections firing on the LSASS dumping activity and
configured a webhook output to forward detections downstream.

![Detection fired](screenshots/05-detection-fired.jpg)

Confirmed the underlying sensitive-process-access telemetry in the sensor
timeline.

![Process access events](screenshots/06-process-access-events.jpg)

*MITRE ATT&CK: T1003.001 – OS Credential Dumping: LSASS Memory*

### 6. Triage and investigation
Pivoted through the timeline, process tree, and network data to reconstruct the
attack, then enriched the payload hash against VirusTotal.

![Triage timeline](screenshots/07-triage-timeline.jpg)

![VirusTotal enrichment](screenshots/08-virustotal-enrichment.jpg)

---

## Detections observed

| Detection | Technique | Data source |
|-----------|-----------|-------------|
| LSASS process access | T1003.001 | Sysmon / EDR sensor |
| LSASS memory dumping | T1003.001 | EDR detection rule + webhook output |
| Suspicious process execution (`WEIRD_BASEBALL.exe`) | T1204 / T1071 | Sysmon process-create |
| Outbound C2 connection | T1071 | Network telemetry |

---

## What I learned

- **End-to-end visibility is everything.** Detection only works when telemetry,
  the sensor, and the console are wired together correctly. A gap anywhere is a
  blind spot.
- **Detection over prevention.** Disabling Defender and still catching the attack
  through EDR made the value of detection engineering concrete.
- **Thinking like the attacker sharpens the defender.** Running the LSASS dump
  myself made the resulting detection and its data source obvious.
- **Triage is a pivot game.** Moving between process, network, and file-system
  views to reconstruct an attack is the core loop of SOC work.
- **Tuning matters.** Separating real signal from noise is what keeps a SOC from
  drowning in alerts.

---

## Credit

Lab design, attack scenario, and methodology from Eric Capuano's
*So You Want to Be a SOC Analyst?* series. This repository documents my own
hands-on completion, execution, and investigation of that lab.
