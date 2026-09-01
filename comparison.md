# Evidence Source Comparison: PCAP vs Zeek vs Suricata

## Research Question
What unique forensic evidence does each network data source provide during an intrusion?

## Attack Scenario
- **Target:** DVWA (Damn Vulnerable Web Application) running in a Docker container (172.17.0.2)
- **Attacker:** Kali Linux host (172.17.0.1)
- **Actions:** `nmap -sV -A 172.17.0.2` (recon scan with NSE scripts) followed by a manual `curl http://172.17.0.2/login.php` request
- **Capture window:** All three tools ran concurrently on the `docker0` interface during the attack

## Evidence Comparison Table

| Evidence Type | PCAP | Zeek | Suricata |
|---|---|---|---|
| Source/Destination IP | Yes — every packet | Yes — `conn.log` | Yes — every alert |
| Full packet payload | Yes — complete bytes | No — summarized only | No — summarized only |
| DNS activity | N/A this scenario | N/A this scenario | N/A this scenario |
| HTTP metadata (method, URI, status, user-agent) | Yes, but requires manual reassembly | Yes — structured in `http.log` | Partial — user-agent visible in ET SCAN alert payload only |
| Detection/alerting | No — raw data only | No — Zeek does not alert by default | Yes — 6+ distinct signatures fired |
| Protocol breakdown (TCP/UDP/ICMP/ARP counts) | Yes — via `tshark -z io,phs` | Partial — per-connection only | No |
| Session/connection summarization | No — requires tooling (tshark) | Yes — native `conn.log` per session | No — alert-centric, not session-centric |
| File extraction capability | Yes — with `tshark --export-objects` | Yes — native `files.log` | No |
| Attack classification / naming | No | No | Yes — e.g. "POSSBL PORT SCAN (NMAP -sS)", "CUSTOM XMAS Scan Detected" |
| Analysis effort required | High — everything manual | Medium — pre-parsed logs | Low — pre-classified alerts |

## Key Findings Per Source

### PCAP (tcpdump)
- Captured 2,419 total frames: 2,393 TCP, 6 ICMP, plus small UDP/IGMP/ARP background noise
- `tshark -z conv,tcp` showed 15+ distinct TCP sessions from 172.17.0.1 to 172.17.0.2:80, matching the Nmap scan's parallel connection behavior
- Ground truth for all other sources — Zeek and Suricata both derive their view from the same underlying packets
- No native interpretation: identifying the attack required external tooling (tshark filters)

### Zeek
- Live capture on the interface initially missed HTTP parsing due to invalid checksums from Docker's virtual NIC (`docker0`) — checksum offloading caused Zeek to silently discard packets
- Re-running Zeek offline against the saved PCAP with `-C` (ignore checksums) fixed this and produced full `http.log` and `files.log`
- `http.log` cleanly recorded 30+ HTTP transactions, showing the Nmap NSE user-agent (`Mozilla/5.0 (compatible; Nmap Scripting Engine...)`) on most requests, and the later manual `curl/8.20.0` request on `/login.php` returning `200`
- No DNS activity recorded — expected, since the attack targeted an IP directly
- Zeek describes *what happened* in a session but does not judge whether it was malicious

### Suricata
- Fired distinct signatures across three detection layers:
  - Network-layer: `POSSBL PORT SCAN (NMAP -sS)`, `CUSTOM Possible Port Scan (many ports, single source)`
  - Scan-technique-layer: `CUSTOM NULL Scan Detected`, `CUSTOM XMAS Scan Detected`
  - Application-layer: `ET SCAN Possible Nmap User-Agent Observed` (directly fingerprinted the Nmap NSE user-agent string), `SURICATA HTTP Response excessive header repetition`
- Suricata is the only source that assigned severity/priority and a named classification to the traffic
- Did not need to be paired with any other tool to conclude "this is reconnaissance" — the alert text alone states it

## Correlation Methodology
1. Suricata's alerts flag *that* something suspicious happened and roughly *why* (signature name)
2. Zeek's `http.log`/`conn.log` provide the *structured detail* of what was sent and received in each flagged session (via matching IP:port:timestamp)
3. PCAP provides the *raw proof* — the actual bytes on the wire — for any claim that needs verification or deeper forensic review (e.g. confirming a payload, extracting a file)

This three-layer correlation (alert → structured log → raw packet) is the methodology this comparison feeds into the [Intelligent-NIDS-ThreatHunting-Platform](https://github.com/Anam-Batool-12/Intelligent-NIDS-ThreatHunting-Platform-Hybrid-NIDS-ML-Suricata-Zeek).
