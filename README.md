# Network Evidence Source Comparison

**Comparative forensic analysis of PCAP, Zeek, and Suricata as network evidence sources during a simulated intrusion** — evaluating unique detection capabilities, evidence granularity, and correlation methodology across all three sources.

![Python](https://img.shields.io/badge/tools-tcpdump%20%7C%20Zeek%20%7C%20Suricata-informational)
![Platform](https://img.shields.io/badge/platform-Kali%20Linux-557C94)
![Status](https://img.shields.io/badge/status-complete-brightgreen)

## Overview
Network defenders rarely rely on a single data source during an investigation. This project answers a practical question: **when the same intrusion happens, what does each evidence source actually give you — and what does it miss?**

A single attack was captured simultaneously through three independent tools, then compared side by side.

## Attack Scenario
- **Target:** DVWA (Damn Vulnerable Web Application) in a Docker container
- **Attacker:** Kali Linux host
- **Actions:** `nmap -sV -A` reconnaissance scan (NSE scripts) followed by a manual `curl` HTTP request
- **Capture:** tcpdump, Zeek, and Suricata all monitoring the same interface concurrently

## Evidence Sources
| Source | What it captured |
|---|---|
| **PCAP** (`pcaps/`) | Raw packet bytes via tcpdump — the ground truth |
| **Zeek** (`zeek-analysis/`) | Structured `conn.log`, `http.log`, `files.log` |
| **Suricata** (`suricata-analysis/`) | Signature-based alerts (`eve.json`, `fast.log`) — port scan, XMAS/NULL scan, and Nmap user-agent detections |

## Key Result
No single source was sufficient alone. Suricata triaged fastest ("this is a scan"), Zeek gave the structured HTTP detail ("here's exactly what was requested"), and PCAP provided the raw proof needed to verify either. Full breakdown in [`comparison.md`](comparison.md), with detailed analysis and lessons learned in [`findings.md`](findings.md).

## Notable Technical Issue Solved
Zeek's live capture on Docker's virtual interface initially dropped HTTP traffic due to checksum-offloading behavior on the `docker0` bridge — a common gotcha in containerized/virtual lab environments. Fixed by re-running Zeek offline against the saved PCAP with the `-C` (ignore checksums) flag. Documented in `findings.md`.

## Repository Structure
```
network-evidence-comparison/
├── pcaps/                      # raw tcpdump capture
├── zeek-analysis/logs*/        # conn.log, http.log, files.log
├── suricata-analysis/alerts/   # eve.json, fast.log, stats.log
├── screenshots/                # tool output evidence
├── comparison.md                # evidence comparison table
├── findings.md                  # detailed analysis and conclusions
└── README.md
```

## Part of a Larger Body of Work
This evidence-correlation methodology (alert → structured log → raw packet) directly feeds into the [Intelligent-NIDS-ThreatHunting-Platform](https://github.com/Anam-Batool-12/Intelligent-NIDS-ThreatHunting-Platform-Hybrid-NIDS-ML-Suricata-Zeek), where it underpins how detected alerts get investigated and confirmed.
