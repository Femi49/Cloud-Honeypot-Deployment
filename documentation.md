# Cloud HOneypot Deployment — Detailed Documentation

This document expands on the project [README](./README.md) with a full process-by-process account of the deployment, configuration, monitoring, analysis, and validation work performed, backed by the evidence in [`/images`](./images).

---

## 1. Objective

Deploy a multi-honeypot sensor (T-Pot 24.04.1) on a cloud host to:

1. Passively attract and log unsolicited internet scanning/attack traffic.
2. Keep real administrative access fully separated from the attacker-facing surface.
3. Visualize and analyze the resulting telemetry (source IPs, countries, ports, credentials, alert categories).
4. Validate the pipeline end-to-end with a controlled brute-force test.
5. Perform a short attribution exercise on a notable attacking IP.

## 2. Environment

| Item | Value |
|---|---|
| Cloud provider | AWS (EC2) |
| OS | Ubuntu |
| Hostname | `ip-172-xx-xx-xx` |
| Platform | T-Pot 24.04.1 (Telekom Security) |
| Container runtime | Docker / Docker Compose |
| SIEM/visualization | Elastic Stack (Elasticsearch + Kibana) |
| IDS | Suricata |
| Attack testing tool | THC-Hydra (from a Kali Linux client) |

---

## Process 1 — EC2 Provisioning & Network Design

An EC2 instance was provisioned and its AWS Security Group was configured to split traffic into two categories:

- **Bait ports**, left open to `0.0.0.0/0` so the honeypots receive organic attack traffic.
- **Real admin access**, moved off the standard port and locked to a single trusted source IP.

### Security group rules (evidence: `aws-security-group-rules.png`)

| Rule | Type | Protocol | Port | Source | Purpose |
|---|---|---|---|---|---|
| sgr-0b105408c2e22b2a9 | SSH | TCP | 22 | `0.0.0.0/0` | **Bait** — routed to the Cowrie honeypot |
| sgr-0de96a51a6efe5c20 | Custom TCP | TCP | 64295 | `10x.xx.xx.xxx/32` | **Real** SSH admin access, single trusted IP only |
| — | Custom TCP | TCP | 64297 | `10x.xx.xx.xxx/32` | Secondary management port, same trusted IP |
| — | Custom TCP | TCP | 23 | `0.0.0.0/0` | Bait — Telnet |
| — | HTTP | TCP | 80 | `0.0.0.0/0` | Bait — routed to Tanner |
| — | RDP | TCP | 3389 | `0.0.0.0/0` | Bait |
| — | HTTPS | TCP | 443 | `0.0.0.0/0` | Bait — routed to H0neytr4p |
| — | SMB | TCP | 445 | `0.0.0.0/0` | Bait — routed to Dionaea |

This design means anything hitting port 22 on the public IP is talking to Cowrie, not a real shell — the actual management channel lives on `64295/64297` and only accepts connections from one allow-listed IP.

---

## Process 2 — T-Pot Installation

The official T-Pot installer was run on the instance. Key install-time output (evidence: `tpot-installation-terminal.png`):

- Docker images pulled for each honeypot/support component, including:
  `tanner`, `h0neytr4p`, `elasticsearch`, `medpot`, `fatt`, `tpotinit`, plus 6 additional images.
- Installer flagged: *"Please review for possible honeypot port conflicts... While SSH is taken care of, other services such as SMTP, HTTP, etc. might prevent T-Pot from starting."*
- Post-install `netstat` output confirmed the **real** SSH daemon was rebound to `64295/tcp` (both IPv4 and IPv6 listeners), separate from the honeypot-facing port 22.
- Final instruction: *"Done. Please reboot and re-connect via SSH on tcp/64295."*

## Process 3 — Post-Install OS Maintenance

Before/around the reboot, standard OS housekeeping was reviewed (evidence: `ec2-terminal-updates.png`):

- Kernel status: running `6.17.0-1007-aws`, with `6.17.0-1009-aws` pending — flagged as a pending kernel upgrade requiring a reboot to fully apply.
- Services restarted automatically post-update: `irqbalance`, `multipathd`, `packagekit`, `polkit`, `rsyslog`, `ssh`, `udisks2`.
- Service restarts deferred (require manual/scheduled action): `ModemManager`, `dbus` restart hook, `networkd-dispatcher`, `systemd-logind`, `unattended-upgrades`.
- No containers or VM guests required restarting at this point.

## Process 4 — Service Verification

`systemctl status tpot.service` was used to confirm the platform came up correctly (evidence: `tpot-service-active-status.png`):

- **Active: active (running)**, service enabled at boot.
- Managed via `docker compose -f /home/ubuntu/tpotce/docker-compose.yml up`.
- Startup log confirmed each honeypot module initializing in sequence, including `Ipphoney`, `Log4Pot`, `Mailoney`, `Medpot`, `Miniprint`, `Redishoneypot`, `Sentrypeer`, `Tanner`, `Wordpot` (with a staggered start delay between modules).

## Process 5 — Accessing the T-Pot Web Console

The T-Pot landing page (evidence: `tpot-web-dashboard.png`) confirmed version **T-Pot 24.04.1** and exposed quick links to:

- **Tools:** Attack Map, CyberChef, Elasticvue, Kibana, Spiderfoot
- **Reference:** SecurityMeter, T-Pot ReadMe, T-Pot @ GitHub

---

## 3. Honeypot Inventory

The following honeypot daemons were active as part of the T-Pot stack, based on install logs and the Kibana honeypot breakdown:

| Honeypot | Emulates / Targets |
|---|---|
| **Cowrie** | SSH & Telnet — the highest-volume target in this deployment |
| **Dionaea** | SMB and assorted network/IoT/malware-delivery services |
| **Tanner** | Web application attack surface (paired front end for HTTP probing) |
| **H0neytr4p** | HTTPS-facing honeypot |
| **Ciscoasa** | Cisco ASA VPN login emulation |
| **Ipphoney** | IPP network printers |
| **Log4Pot** | Detects Log4Shell-style (CVE-2021-44228) exploitation attempts |
| **Mailoney** | SMTP |
| **Medpot** | HL7 healthcare protocol |
| **Miniprint** | Network printer emulation |
| **Redishoneypot** | Redis |
| **Sentrypeer** | SIP/VoIP fraud and scanning |
| **Wordpot** | WordPress-specific attack surface |

---

## Process 6 — Monitoring Stack Setup

Two visualization layers were used side by side:

1. **T-Pot Attack Map** — a live, self-hosted geographic feed with a "Live Feed" table (time, source IP, IP reputation, country, honeypot, protocol, port, hostname), plus Top IPs / Top Countries / Dashboard tabs.
2. **Kibana (Elastic Stack)** — a `>T-Pot` dashboard with panels for attack totals, histograms by port/honeypot/country, IP reputation, OS fingerprinting, Suricata alerts, and credential tag clouds.

---

## Process 7 — Data Collection Timeline

Snapshots were captured at several points to show how quickly an unadvertised host attracts attention:

| Stage | Evidence | Observation |
|---|---|---|
| **T0 — immediately after deployment** | `attack-map-fresh-deployment.png` | 0 attacks in the last 1 min / 1 hr / 24 hr — clean baseline |
| **~T+1 hour** | `attack-map-live-feed.png` | 13 attacks in the trailing 1h/24h window; sources in Russia and the US hitting **Tanner** (HTTP) and **Cowrie** (Telnet) |
| **~T+18 hours** | `attack-map-live-feed-after-18h.png` | 1,042 attacks logged in the trailing 24h; live feed shows HTTPS hits on **H0neytr4p**, HTTP on **Tanner**, and SSH on **Cowrie** from the US and Netherlands |
| **~T+24 hours (Kibana)** | `kibana-honeypot-attacks-24h.png` | ~8,000 total attacks: **Cowrie ~6k**, **Dionaea 883**, **Tanner 199**, **H0neytr4p 92**, **Ciscoasa 1** |

## Process 8 — Traffic Analysis

Detailed breakdowns from Kibana (evidence: `kibana-attacks-by-port-country-os.png`, `kibana-suricata-alerts-credential-tagclouds.png`):

- **By destination port:** heaviest volumes on `445` (SMB), `22` (SSH), `80` (HTTP), `3306` (MySQL), and `23` (Telnet).
- **By honeypot:** Cowrie dominant by a wide margin, Dionaea a distant second, with Tanner/H0neytr4p/Ciscoasa making up the remainder.
- **By country:** top sources included Nigeria, France, China, Chile, and Canada in the main panel, with the United States, Vietnam, Bulgaria, Brazil, and the United Kingdom also appearing prominently in the extended country breakdown.
- **Attacker IP reputation:** the large majority of sources were flagged as "known attacker," with a smaller share tagged "bot, crawler."
- **Passive OS fingerprinting (p0f):** a mix of Windows NT kernel variants, Windows 7/8, and multiple Linux kernel families (2.2.x–3.x, 3.11+, 2.4.x–2.6.x, 3.1–3.10), indicating attacks from both compromised Windows hosts and Linux-based scanning infrastructure.
- **Suricata alert categories:** dominated by "Generic Protocol Command Decode" and "Potentially Bad Traffic," with two sharp spikes in "Attempted Administrator Privilege Gain" alerts during the collection window, plus steady background "Misc Attack" and "Attempted Information Leak" alerts.
- **Username tag cloud:** common credential-stuffing usernames (`root`, `admin`, `ubuntu`, `sa`) appeared alongside raw SIP/RTSP protocol strings (`OPTIONS sip:nm SIP/2.0`, `Call-ID`, `Max-Forwards`) — a sign that VoIP scanning traffic against Sentrypeer was also being captured and surfaced in the same field.
- **Password tag cloud:** overwhelmingly weak/default credentials (`123456`, `12345678`, `password`, `iloveyou`, `princess`, `qwerty`, `rockyou`, `monkey`), plus a large "(blank)" bucket representing no-password or anonymous login attempts.

## Process 9 — Attacker Attribution Case Study

One of the most active sources, **`5.101.64.6`** (Russia — repeatedly hit the Tanner/HTTP honeypot), was investigated with an IP reputation/WHOIS lookup tool (evidence: `attacker-ip-reputation-lookup.png`):

| Field | Value |
|---|---|
| IP | 5.101.64.6 (alive, ICMP ~46.4 ms) |
| Network owner | PINDC-AS, RU |
| ASN | 34665 |
| Network block | 5.101.64.0/24 |
| Org name | Petersburg Internet Network Ltd. |
| Location | Saint Petersburg, Russian Federation |
| Abuse contact | abuse@pindc.ru |
| Block assigned | 2012-06-26 |

**Interpretation:** the address belongs to a commercial Russian hosting/ISP block rather than a residential ISP, consistent with rented VPS or bot infrastructure commonly used for broad internet-wide scanning rather than a single compromised home machine.

## Process 10 — Controlled Validation (SSH Brute-Force Test)

To confirm the Cowrie honeypot correctly captures credential attacks end-to-end, a controlled brute-force test was run from my Kali Linux client (evidence: `hydra-ssh-bruteforce-test.png`):

```bash
hydra -t 4 -V -L /usr/share/wordlists/rockyou.txt -P /usr/share/wordlists/rockyou.txt ssh://<target-ip>
```

**Parameters:**
- `-t 4` — 4 parallel tasks
- `-V` — verbose, prints every login attempt
- `-L` / `-P` — username and password lists, both set to `rockyou.txt`

**Scale:** Hydra queued the full username × password cross product (14,344,399 × 14,344,399 ≈ 2.06 × 10¹⁴ possible combinations) — the intent here was to generate a sustained, realistic stream of credential attempts against the honeypot, not to actually exhaust the list.

**Sample attempts observed:** `123456/123456`, `123456/12345`, `123456/123456789`, `123456/password`, `123456/iloveyou`, `123456/princess`, `123456/1234567`, `123456/rockyou`, `123456/12345678`, `123456/abc123`.

**Result:** these attempts subsequently appeared in Cowrie's captured logs and Kibana's credential tag clouds, confirming the capture pipeline (honeypot → log shipper → Elasticsearch → Kibana) worked end-to-end.

> **Note:** Hydra's own banner explicitly states it should not be used against systems without authorization. This test was run only against infrastructure owned and operated for this project.

---

## 4. Security & Operational Notes

- Real administrative access is fully isolated from the attack surface: the honeypot stack answers on the "expected" ports (22, 23, 80, 443, 445, 3389), while actual SSH management lives on `64295`/`64297`, restricted to a single source IP.
- `tpot.service` is enabled at boot and supervised via Docker Compose, so the honeypot stack restarts automatically after a reboot or crash.
- A kernel upgrade was pending at the time of the install-time screenshot and had not yet been applied — flagged here as an outstanding maintenance item (see Recommendations).

## 5. Key Findings

- An unadvertised cloud host starts receiving unsolicited scans within roughly an hour of going live, and accumulates four-figure attack counts within a day.
- SSH and SMB are, by a wide margin, the most targeted services — consistent with widespread internet-wide scanning for weak credentials and legacy protocol exposure.
- Credential attempts are dominated by default/weak passwords and a large volume of blank/anonymous attempts, reinforcing the value of disabling password auth / enforcing key-based access on real infrastructure.
- Elastic/Kibana plus T-Pot's built-in Attack Map together provide enough visibility (source, country, port, protocol, credential) to characterize an attack campaign without any custom tooling.
- A quick WHOIS/IP-reputation lookup on a top attacking IP was enough to confirm it as flagged hosting infrastructure rather than a residential host.

## 6. Recommendations / Next Steps

- Apply the pending kernel upgrade during a scheduled maintenance window and reboot.
- Extend the collection window beyond 24 hours to establish weekly/seasonal attack patterns.
- Add outbound abuse reporting (e.g., to the `abuse@pindc.ru`-style contacts surfaced during attribution) for the most persistent offenders.
- Consider enabling T-Pot's additional threat-intel/reporting integrations for longer-term trend tracking.
- Periodically rotate the admin-access port and allow-listed IP as an added precaution.

---

## 7. Evidence Index

| File (in `/images`) | What it documents |
|---|---|
| `aws-security-group-rules.png` | AWS security group rules — bait ports vs. restricted admin ports |
| `tpot-installation-terminal.png` | T-Pot installer output, Docker image pulls, port-conflict check |
| `ec2-terminal-updates.png` | OS/kernel update status and deferred service restarts |
| `tpot-service-active-status.png` | `systemctl status tpot.service` — active, honeypot modules starting |
| `tpot-web-dashboard.png` | T-Pot 24.04.1 web console and tool links |
| `attack-map-fresh-deployment.png` | Attack Map at T0 — zero attacks, clean baseline |
| `attack-map-live-feed.png` | Attack Map ~1 hour after deployment — first live attacks |
| `attack-map-live-feed-after-18h.png` | Attack Map after ~18 hours — 1,042 attacks in the trailing 24h |
| `kibana-honeypot-attacks-24h.png` | Kibana 24h attack totals by honeypot |
| `kibana-attacks-by-port-country-os.png` | Kibana breakdowns by port, honeypot, country, IP reputation, and OS |
| `kibana-suricata-alerts-credential-tagclouds.png` | Suricata alert histogram and username/password tag clouds |
| `attacker-ip-reputation-lookup.png` | WHOIS/reputation lookup for top attacking IP `5.101.64.6` |
| `hydra-ssh-bruteforce-test.png` | Controlled Hydra SSH brute-force validation test |
