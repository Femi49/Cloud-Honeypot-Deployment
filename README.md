# Cloud-Based Honeypot Deployment and Threat Intelligence Monitoring Using T-Pot on AWS

📌 Project Overview

This project involved deploying T-Pot, an open-source multi-honeypot platform, on an AWS EC2 instance to create a controlled environment for observing malicious activity, collecting attack telemetry, and analyzing security events.

The project demonstrates practical experience with cloud security, honeypot deployment, network security monitoring, log analysis, threat intelligence, and Linux administration.

The T-Pot environment was exposed to the internet in a controlled manner to attract automated scanning, reconnaissance, brute-force attempts, and other malicious activities. The collected telemetry was analyzed through T-Pot's monitoring and visualization capabilities.

![AWS](https://img.shields.io/badge/AWS-EC2-orange?style=for-the-badge&logo=amazonaws)
![Linux](https://img.shields.io/badge/Linux-Ubuntu-orange?style=for-the-badge&logo=linux)
![T-Pot](https://img.shields.io/badge/Honeypot-T--Pot-red?style=for-the-badge)
![Elastic](https://img.shields.io/badge/Monitoring-Elastic-005571?style=for-the-badge&logo=elastic)
![Suricata](https://img.shields.io/badge/IDS-Suricata-blue?style=for-the-badge)
![Cybersecurity](https://img.shields.io/badge/Focus-Cybersecurity-black?style=for-the-badge)

**Goals:**
- Stand up a production-style honeypot sensor in the cloud
- Safely expose common attacker-facing ports while keeping real admin access locked down
- Collect and visualize live attack telemetry (source IPs, countries, credentials, ports, protocols)
- Validate the sensor by running a controlled brute-force test against it and confirming capture
- Investigate a top attacking IP for reputation/attribution

The project provided practical experience in:

- Cloud infrastructure deployment
- Honeypot deployment
- Network security monitoring
- Attack detection
- Security event analysis
- Threat intelligence
- Log analysis
- Linux administration
- Security visualization

## Architecture

| Component | Details |
|---|---|
| Host | AWS EC2 instance (Ubuntu), hostname `ip-172-31-29-29` |
| Platform | T-Pot 24.04.1 |
| Honeypots | Cowrie, Dionaea, Tanner, H0neytr4p, Ciscoasa, Ipphoney, Log4Pot, Mailoney, Medpot, Miniprint, Redishoneypot, Sentrypeer, Wordpot |
| Detection | Suricata (network IDS alerts) |
| Visualization | Kibana / Elastic Stack, T-Pot Attack Map (Live Feed, Top IPs, Top Countries) |
| Management | Admin SSH moved off the honeypot port, restricted by security group to a single source IP |

### Network / security group design

Real SSH management access was moved to a **custom high port (64295)** and locked to a single admin IP via the AWS security group, so the internet-facing `22/tcp` traffic goes to the Cowrie honeypot instead of a real shell. Common attacker-facing ports were opened to `0.0.0.0/0` so the honeypots would receive organic scan/attack traffic:

- `22/tcp` (SSH) → Cowrie honeypot
- `23/tcp` (Telnet)
- `80/tcp`, `443/tcp` (HTTP/HTTPS)
- `445/tcp` (SMB)
- `3389/tcp` (RDP)
- `64295/tcp` — real SSH admin access, restricted to a single trusted IP

## 📁 Repository Structure

```text
tpot-aws-honeypot/
│
├── README.md
│
├── images/
│   ├── aws-security-group-rules.png
│   ├── tpot-installation-terminal.png
│   ├── ec2-terminal-updates.png
│   ├── tpot-service-active-status.png
│   ├── tpot-web-dashboard.png
│   ├── attack-map-fresh-deployment.png
│   ├── attack-map-live-feed.png
│   ├── attack-map-live-feed-after-18h.png
│   ├── kibana-honeypot-attacks-24h.png
│   ├── kibana-attacks-by-port-country-os.png
│   ├── kibana-suricata-alerts-credential-tagclouds.png
│   ├── attacker-ip-reputation-lookup.png
│   └── hydra-ssh-bruteforce-test.png
│
└── documentation/
    ├── deployment.md
    ├── monitoring.md
    └── findings.md-
```
⚠️ Disclaimer

This project was conducted for educational, defensive security research, and cybersecurity portfolio purposes.

The honeypot was intentionally exposed to internet traffic to observe attack activity. No unauthorized access to third-party systems was performed as part of this project.







