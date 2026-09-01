# Network Evidence Source Comparison

Comparative forensic analysis of PCAP, Zeek, and Suricata as network evidence sources during a simulated intrusion — evaluating unique detection capabilities, evidence granularity, and correlation methodology across all three sources.

## Attack Scenario
Nmap reconnaissance scan (`-sV -A`) + HTTP GET request against a DVWA (Damn Vulnerable Web Application) Docker target, simulating a recon-phase intrusion.

## Evidence Sources
- **PCAP** (`pcaps/`) — raw packet capture via tcpdump
- **Zeek** (`zeek-analysis/`) — connection, HTTP, and file logs
- **Suricata** (`suricata-analysis/`) — IDS alerts and signatures

## Structure
See `comparison.md` for the evidence comparison table and `findings.md` for detailed analysis.

## Part of
This methodology feeds into the [Intelligent-NIDS-ThreatHunting-Platform](https://github.com/Anam-Batool-12/Intelligent-NIDS-ThreatHunting-Platform-Hybrid-NIDS-ML-Suricata-Zeek).
