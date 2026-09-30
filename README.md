# Home SOC Lab

A self built home security lab for practicing SOC analyst skills. Generating attacks, capturing them in logs, and detecting them through a SIEM. Made to show hands on work for applications to SOC Analyst and Security Analyst roles.

## Goal

Most of SOC analyst day to day work consists of detecting something that happened on a network, discovering it within logs, understanding the situation, and making decisions on what to do about it. This repo documents me building that pipeline from scratch and working through this process meself.

## Environment

| Role | Tool | Notes |
|---|---|---|
| Hypervisor | VirtualBox | Runs on host machine, isolated from host and home network |
| Attacker | Kali Linux | Source of all offensive activity |
| Target | Windows 10 | Instrumented with Sysmon + Windows Security auditing |
| SIEM | Splunk Enterprise (free tier) | Installed on host machine, receives forwarded logs |
| Log shipping | Splunk Universal Forwarder | Installed on Windows target |

**Network design:** each VM has two adapters — one on NAT (for internet access and log forwarding to the host) and one on an isolated internal only network (`labnet`, 10.10.10.0/24) which is used exclusively for attacker-to-target traffic. All attack traffic in this lab happened inside VirutalBox's internal network and never occurs over the home network or internet.

## Reports

Each report documents one complete analysis cycle from attack to detection. This includes what was run, what it looked like from the attacker's side, what it looked like in the logs, and how an analyst would triage it.

| # | Title | Technique | Status |
|---|---|---|---|
| 01 | [TCP SYN Port Scan](./detection-report-01-port-scan.md) | Nmap `-sS` scan | Complete |

## Roadmap

- [ ] Add a second target: Metasploitable2 (Linux), to compare against Windows detection signatures
- [ ] Additional Nmap scan types (full connect `-sT`, service/version detection `-sV`) and compare log signatures against the SYN scan baseline
- [ ] Basic exploitation attempts (e.g. a known Metasploitable2 vulnerability) with corresponding Sysmon process creation detection
- [ ] Brute force login attempt (RDP or SMB) against the Windows target
- [ ] Build Splunk alters/correlation searches for the detections documented above, so they would run automatically on reocurring attacks
- [ ] Credential dumping / persistence technique and its detection signatures
- [ ] Basic lateral movement scenario between two lab hosts

## Why this exists

I'm a new CS graduate applying to SOC analyst roles. Demonstrated coursework only goes so far, so this lab is to show hands on work in the area, generating traffic that seems malicious, seeing how it shows up in real logs, and practicing thinking it through the way an analyst would. Each report in this repo is a snapshot of that progress.
