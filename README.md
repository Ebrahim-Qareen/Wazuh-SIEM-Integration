# Wazuh SIEM Integration Lab

![Wazuh](https://img.shields.io/badge/Wazuh-4.x-3B82F6?style=flat-square)
![Suricata](https://img.shields.io/badge/Suricata-IDS-EF4444?style=flat-square)
![OS](https://img.shields.io/badge/agents-Windows%20%C2%B7%20Ubuntu-1F2937?style=flat-square)
![ATT&CK](https://img.shields.io/badge/MITRE-ATT%26CK%20mapped-6B7280?style=flat-square)

A multi-OS Wazuh SIEM lab built inside the **Cybersecurity Corp** home environment: Windows and Linux agents, file integrity monitoring, VirusTotal enrichment, Suricata network detection, and DVWA as an attack target — with custom rules written and validated for each attack that was run against it.

The goal is not "install Wazuh". The goal is: **run an attack → see it in the SIEM → write the rule that catches it reliably → document what a Tier 1 analyst should do with the alert.**

## Topology

```
                    ┌──────────────────────────┐
                    │  Sophos XG  (perimeter)  │
                    └────────────┬─────────────┘
                                 │  LAN 10.10.10.0/24
       ┌─────────────────┬───────┴────────┬──────────────────┐
       │                 │                │                  │
┌──────┴──────┐  ┌───────┴───────┐  ┌─────┴──────┐   ┌───────┴───────┐
│ Wazuh Mgr   │  │ WS2025 DC     │  │ Win 11/10  │   │ Ubuntu + DVWA │
│ 10.10.10.2  │  │ cybersecurity │  │ endpoints  │   │ + Suricata    │
│ + Dashboard │  │ .local        │  │ (agents)   │   │ (agent)       │
└─────────────┘  └───────────────┘  └────────────┘   └───────────────┘
                                 │
                       ┌─────────┴─────────┐
                       │ Kali Purple       │  attacker
                       │ 192.168.50.10     │
                       └───────────────────┘
```

| Host | Role | Wazuh component |
|---|---|---|
| Wazuh manager | Manager + indexer + dashboard (single node) | Manager 4.x |
| Windows Server 2025 | Domain controller `cybersecurity.local` | Agent + Sysmon + AD audit policy |
| Windows 11 / 10 / 7 | Domain-joined workstations | Agent + Sysmon |
| Ubuntu | DVWA web target, Suricata sensor | Agent (syslog + `eve.json`) |
| Kali Purple | Attacker | — |

## What is integrated

| Integration | Where | What it gives the analyst |
|---|---|---|
| **Sysmon → Wazuh** | Windows agents | Process creation, network connections, registry and image loads with parent/child context |
| **File Integrity Monitoring** | `/etc`, `/var/www/html`, `C:\Windows\System32\drivers\etc`, web root | Real-time change alerts with who-data (user + process) |
| **VirusTotal** | Manager `integrations` block on FIM events | Hash lookup on every new/modified file in monitored paths |
| **Suricata → Wazuh** | Ubuntu sensor, `eve.json` ingested by the agent | Signature-based network alerts correlated with host events |
| **Active response** | Manager | Automatic `firewall-drop` on repeated SSH/web brute force |
| **DVWA** | Ubuntu | Web attack surface for SQLi / XSS / command injection / brute-force scenarios |

## Attack → detection matrix

| Attack (from Kali) | ATT&CK | Evidence source | Custom rule | Level |
|---|---|---|---|---|
| Nmap SYN / service scan | T1046 | Suricata ET SCAN | `100100` | 7 |
| SSH brute force (Hydra) | T1110.001 | `sshd` auth log | `100110` (freq 8/120s) | 10 |
| DVWA SQL injection | T1190 | Apache access log | `100120` (PCRE2) | 12 |
| DVWA reflected / stored XSS | T1189 | Apache access log | `100121` (PCRE2) | 12 |
| DVWA command injection | T1059.004 | Apache access log + `auditd` | `100122` | 13 |
| Web shell dropped in web root | T1505.003 | FIM + VirusTotal | `100130` | 14 |
| EICAR / known-bad file on endpoint | T1204.002 | FIM + VirusTotal | `100131` | 13 |
| Mimikatz / LSASS access | T1003.001 | Sysmon EID 10 | `100140` | 14 |
| New local admin created | T1136.001 | Security EID 4720/4732 | `100150` | 12 |
| Scheduled task persistence | T1053.005 | Security EID 4698 / Sysmon EID 1 | `100151` | 11 |

Rule levels follow the same scheme used in production: **0–6 log only · 7–11 medium triage · 12–14 high, enrich and review · 15 critical, auto-block**.

## Repository layout

```
.
├── README.md
├── docs/
│   ├── build-guide.md            # manager, agents, Sysmon, FIM + VirusTotal, Suricata, active response
│   └── attack-walkthroughs.md    # each attack: command, what fired, analyst actions
├── rules/
│   └── local_rules.xml           # custom rules, IDs 100100–100199, ATT&CK-tagged
└── decoders/
    └── local_decoder.xml
```

## Working method

1. **Decoder first.** Confirm the field names in `Tools → Ruleset Test` before writing any rule — `data.srcip` vs `srcip` mistakes are the number-one cause of rules that never fire.
2. **PCRE2 over keywords.** Anchored, case-insensitive patterns with word boundaries; XML entities encoded (`&lt; &gt; &quot; &apos;`).
3. **One rule, one behaviour.** Base/classification rules stay at level 3 and are never silenced — they prove the feed is alive. Escalation rules hang off the base rule, never off a sibling.
4. **Validate, then deploy.** Every rule is reproduced in `Tools → Ruleset Test` with a real log line before it is loaded on the manager.
5. **Document the analyst side.** Every alert has a triage note: what it means, false-positive sources, first three checks, escalation criteria.

## Quick start

```bash
# manager (Ubuntu 22.04)
curl -sO https://packages.wazuh.com/4.x/wazuh-install.sh && sudo bash ./wazuh-install.sh -a

# drop in custom rules and decoders, then reload
sudo cp rules/local_rules.xml /var/ossec/etc/rules/
sudo cp decoders/local_decoder.xml /var/ossec/etc/decoders/
sudo systemctl restart wazuh-manager
```

Full step-by-step build: [`docs/build-guide.md`](docs/build-guide.md). Attack-by-attack results: [`docs/attack-walkthroughs.md`](docs/attack-walkthroughs.md).

## Author

Ebrahim Mohamed — SOC Analyst / Detection Engineer / Cybersecurity Instructor — [LinkedIn](https://linkedin.com/in/EbrahimMohamed)
