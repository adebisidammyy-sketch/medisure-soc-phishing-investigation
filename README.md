### SOC Phishing Investigation

Investigation of a real phishing email sent to a staff member at MediSure Health Network. a healthcare provider serving 1.5M+ patients, tracing it from a spoofed Microsoft "unusual sign-in" alert through header analysis, threat intelligence corroboration, and Splunk log correlation, to a confirmed true-positive credential-harvesting attempt with three compromised accounts.


Prepared by: Adebisi Damilola




Overview

ClientMediSure Health NetworkIncident typeSpear-phishing / credential harvesting (impersonating Microsoft account security)VerdictTrue positiveAccounts affected3 (Alice, Bob, Carol)Successful unauthorized logins0Root causeNo DMARC record published for the spoofed sending domain

This repo contains the full SOC investigation report and the staff-facing phishing awareness training deck produced from it.

The Email

A phishing email impersonating "Microsoft account team," reporting a fake unusual sign-in from Russia and urging the recipient to "secure your account" via an embedded link.

From: Microsoft account team <no-reply@access-accsecurity.com>
To: employee@medisurehealthnetwork.com
Subject: Microsoft account unusual sign-in activity

Methodology


Header analysis: examined the raw email source for sender authenticity, relay path, and header inconsistencies (MXToolbox)
Indicator extraction: catalogued IPs, domains, and reply-to addresses as IOCs
Threat intelligence corroboration: checked every IOC independently against VirusTotal, AlienVault OTX, and AbuseIPDB rather than relying on a single source
Log correlation: searched the extracted IOCs against MediSure's proxy, firewall, and authentication logs in Splunk to confirm actual staff interaction
Triage & classification: combined findings into a single verdict and attack classification
Response planning: defined containment, recovery, and lessons-learned actions based on the specific accounts and infrastructure identified


Tools Used

ToolPurposeMXToolboxEmail header analysis; SPF, DKIM, DMARC validationVirusTotalDomain, IP, and URL reputation scanning across security vendorsAlienVault OTXPassive DNS, WHOIS, and indicator threat intelligenceAbuseIPDBHistorical abuse reports for IP addressesSplunk (Free)SIEM log ingestion and correlation across proxy, firewall, and auth logs

Key Findings

Sender authentication failed outright. access-accsecurity.com had no DMARC record published, and the message failed both SPF and DKIM, nothing about the sending path was authenticated. The Reply-To address routed to a personal Gmail account, not Microsoft.

Indicators of Compromise:

IndicatorValueSending IP89.144.44.41Sending domainaccess-accsecurity.comReply-To (attacker)solutionteamrecognizd03@gmail.comMalicious linksign.inTracking pixel domainthebandalisty.comClaimed sign-in IP (in email body, unverified)103.225.77.255


thebandalisty.com: a hidden tracking pixel embedded in the email's HTML, used to silently confirm the message was opened. Flagged malicious by 11/91 vendors on VirusTotal; OTX shows it registered through NameCheap and tagged as a DGA domain, disposable phishing infrastructure, weaponized briefly then dropped.
sign.in: the actual destination behind "secure your account," independently flagged malicious/spam by VirusTotal.
103.225.77.255: the "suspicious sign-in" IP claimed in the email body never appears in MediSure's real traffic logs; it's bait text. AbuseIPDB shows a long report history of other victims describing this exact same template, confirming a reused phishing kit rather than a MediSure-specific attack.


Log correlation (Splunk, index=medisure_logs, 324 events):

UserClicked link (proxy)Contacted attacker IP (firewall)Failed login, Russia (auth)alice@medisurehealthnetwork.comYesYes–bob@medisurehealthnetwork.comYes–Yescarol@medisurehealthnetwork.com–Yes (x2)–

Two of three staff accounts clicked the malicious link within the same day it arrived. One follow-on login attempt from a foreign IP failed, no successful unauthorized access.

Note: firewall log entries in this dataset carry src_ip but not a user field, so Carol's exposure only surfaces by cross-referencing src_ip against proxy/auth logs manually a data quality gap worth flagging on its own.

Verdict

True positive - spear-phishing / credential harvesting disguised as a Microsoft security alert. Sender authentication failed outright, both the link and tracking-pixel domains carry independent malicious flags, and three separate accounts show real interaction with the campaign in the logs.

Response & Mitigation

Full incident response playbook (Section 6 of the report) follows the NIST SP 800-61 lifecycle: Planning → Detection → Analysis → Containment & Eradication → Recovery → Lessons Learned, with Governance as a continuous oversight layer.

Immediate containment actions:

Isolated endpoints for Alice and Carol (outbound connections to attacker IP)
Disabled and force-reset credentials for Alice and Bob (interacted with malicious links)
Quarantined the phishing sample across all mailboxes
Blocked the sending domain, sending IP, and malicious link at the email gateway and firewall
Blocklisted the tracking-pixel domain and claimed sign-in IP

Root cause fix: Enforce DMARC (reject or quarantine policy, not just monitoring) across MediSure's mail environment, this single change would very likely have prevented the email from being delivered at all.

Deliverables in This Repo

Report/MediSure_SOC_Investigation_Report.docx - full investigation report: header analysis, threat intelligence, log correlation, triage verdict, and the complete incident response playbook
Training-deck/MediSure_Phishing_Awareness.pptx - staff-facing awareness training built from this investigation's real findings, covering the 5 signs of phishing, the 30-second hover test, and the report → escalate workflow
Evidence/ - supporting screenshots (MXToolbox, VirusTotal, OTX, AbuseIPDB, Splunk)
Queries/splunk-spl-queries.txt - reference SPL queries used for proxy, firewall, auth, and combined log correlation


References

MXToolbox - Email Header Analyzer and DMARC/SPF/DKIM Lookup Tool
VirusTotal - Domain, IP, and URL Reputation Scanning
AlienVault OTX - Threat Intelligence Platform
AbuseIPDB - IP Address Abuse Reporting Database
Splunk Free - SIEM and Log Analysis Platform
NIST SP 800-61 - Computer Security Incident Handling Guide

MediSure Health Network is a fictional client scenario used for SOC training and practicals.
