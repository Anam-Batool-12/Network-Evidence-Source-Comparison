# Findings: Network Evidence Source Comparison

## Summary
This lab simulated a reconnaissance-phase intrusion (Nmap scan + HTTP request) against a DVWA target and captured the same attack simultaneously through three independent evidence sources: raw packet capture (PCAP), network security monitoring (Zeek), and signature-based intrusion detection (Suricata). The goal was to determine what each source uniquely contributes to an investigation, rather than assuming any single tool is sufficient on its own.

## Finding 1: No single source is sufficient alone
Each tool answered a different investigative question:
- **PCAP answered:** "What bytes actually crossed the wire?"
- **Zeek answered:** "What sessions and application-layer transactions occurred?"
- **Suricata answered:** "Was any of this malicious, and what kind of activity is it?"

An investigation relying on only one source would have gaps. PCAP alone would require manually reconstructing that a scan happened. Suricata alone would confirm suspicious activity but not show the exact HTTP requests involved. Zeek alone would show clean session data but offer no judgment on intent.

## Finding 2: Evidence sources can fail silently — verification matters
Zeek's live capture on `docker0` initially produced no `http.log` at all, because Docker's virtual NIC triggers checksum-offloading behavior that Zeek's default settings treat as invalid/corrupted packets, silently dropping them. This was only caught by noticing `http.log` and `dns.log` were missing after the run and checking Zeek's own warning output. Re-analyzing the same PCAP offline with `-C` (ignore checksums) recovered full HTTP visibility.

**Implication for real investigations:** an analyst who assumes "no log entries" means "no activity" could reach a false negative conclusion. Cross-checking Zeek's output against the raw PCAP surfaced the gap.

## Finding 3: Suricata is fastest to triage, but least detailed
Suricata alerts were the quickest way to answer "did anything bad happen here?" — signatures like `POSSBL PORT SCAN (NMAP -sS)` and `ET SCAN Possible Nmap User-Agent Observed` immediately named the activity. However, the alerts alone do not show full request/response content — for that, an analyst still needs to pivot to Zeek's `http.log` or the raw PCAP.

## Finding 4: Attribution signals differ by source
The two distinct actions in this test — the automated Nmap NSE scan and the manual `curl` request — were both visible, but identified differently depending on the source:
- In Zeek's `http.log`, they were distinguishable by `user_agent` field (`Nmap Scripting Engine` vs `curl/8.20.0`)
- In Suricata, only the Nmap traffic triggered the `ET SCAN Possible Nmap User-Agent Observed` signature — the manual curl request did not match any signature, since it resembled normal traffic
- This shows Suricata's detection depends on known signatures matching known tool fingerprints; a modified or less common user-agent could evade this specific alert

## Finding 5: Absence of evidence is itself evidence
No `dns.log` was produced by Zeek because the attack targeted an IP address directly, with no domain resolution involved. This is a valid, expected finding rather than a tooling failure — worth explicitly documenting so it isn't mistaken for missing data.

## Conclusion
Effective network forensics and threat hunting depend on correlating multiple evidence sources rather than trusting any single tool. Suricata provides fast triage and classification; Zeek provides structured, queryable session and protocol detail; PCAP provides the immutable ground truth needed to verify or extend either. This three-source correlation model — alert → structured log → raw packet — is the evidence methodology carried forward into the [Intelligent-NIDS-ThreatHunting-Platform](https://github.com/Anam-Batool-12/Intelligent-NIDS-ThreatHunting-Platform-Hybrid-NIDS-ML-Suricata-Zeek), where it underpins how alerts get investigated and confirmed.

## Limitations of This Lab
- Single attack type tested (reconnaissance); results may differ for other attack categories (e.g. exploitation, data exfiltration, lateral movement)
- Docker bridge networking introduced a tooling quirk (checksum offloading) not present on physical networks — worth noting for anyone reproducing this on bare-metal or VM-based labs
- Suricata's custom/local rules (`CUSTOM NULL Scan Detected`, etc.) were pre-existing on this system; a fresh Suricata install with only default Emerging Threats rules may not reproduce identical alerts
