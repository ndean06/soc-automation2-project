# SOC Automation 2.0

An end-to-end SOC alert-triage automation lab integrating Splunk, n8n, OpenAI, Slack, VirusTotal, AbuseIPDB, DFIR-IRIS, and Claude Desktop through the Model Context Protocol (MCP).

## Project Summary

SOC analysts often spend valuable time collecting alert details, enriching indicators, documenting findings, and transferring information between separate tools. This project demonstrates how those repetitive steps can be automated while keeping the analyst responsible for validating the evidence and making the final escalation decision.

The workflow begins with Windows 10 Security Event Logs in Splunk. When Splunk detects suspicious activity, it sends an alert to an n8n webhook. The webhook passes the alert to an OpenAI agent, which uses VirusTotal and AbuseIPDB as enrichment tools during its analysis. The completed triage output is then sent to DFIR-IRIS for alert creation and to Slack for analyst notification.

A separate Splunk MCP integration allows Claude Desktop to interact with the Splunk SIEM through approved tools. Claude can assist with analyst-driven searches and investigation pivots, but its results are validated against the original Splunk evidence.

### What the AI Does

The OpenAI agent receives selected fields from the Splunk alert rather than the complete raw event. It uses those details to:

- Summarize the detected activity.
- Identify the affected user and endpoint.
- Request supported VirusTotal and AbuseIPDB enrichment.
- Suggest relevant MITRE ATT&CK mappings for analyst validation.
- Assign an initial severity and explain the supporting rationale.
- Recommend investigation and response actions.
- Generate a structured report for DFIR-IRIS and Slack.

The AI does not modify Splunk data, automatically contain endpoints, block IP addresses, or make the final incident decision.

A separate Claude Desktop integration uses a local Splunk MCP server to support analyst-driven, read-only searches. Claude helps organize returned events and identify investigation pivots, but Splunk remains the source of truth.

## Objective

Build an end-to-end proof-of-concept pipeline that demonstrates how a modern Security Operations Center can:

1. Reduce repetitive alert-enrichment and documentation work.
2. Produce consistent and structured triage reports.
3. Improve situational awareness through MITRE ATT&CK mapping.
4. Create investigation alerts and analyst notifications automatically.
5. Use AI as decision support while preserving human review.

## Architecture

![SOC Automation 2.0 Architecture](screenshots/01-core-automation/SOC-Automation-Project-Diagram.png)

See [Architecture.md](Architecture.md) for detailed component roles, communication paths, ports, and security considerations.

## Project Documentation

| Document | Description |
|---|---|
| [Architecture.md](Architecture.md) | Component roles, communication paths, and trust boundaries |
| [Walkthrough.md](Walkthrough.md) | Complete implementation and evidence walkthrough |
| [Threat-Intelligence-Enrichment.md](Threat-Intelligence-Enrichment.md) | VirusTotal and AbuseIPDB integration details |
| [DFIR-IRIS-Integration.md](DFIR-IRIS-Integration.md) | Automated alert-creation and case-escalation details |
| [Splunk-MCP-Claude.md](Splunk-MCP-Claude.md) | Claude-to-Splunk MCP integration |
| [Testing-Validation.md](Testing-Validation.md) | Expected results, actual results, and validation status |
| [Troubleshooting-Log.md](Troubleshooting-Log.md) | Problems, root causes, corrections, and lessons |
| [`workflow-exports/`](workflow-exports/) | Sanitized n8n workflow export |
| [`config-examples/`](config-examples/) | Placeholder-only configuration examples |

## Core Workflow

| Stage | Component | Function |
|---:|---|---|
| 1 | Windows 10 Security Event Logs | Generate authentication, PowerShell, and Defender telemetry |
| 2 | Splunk Enterprise | Ingest telemetry and detect suspicious behavior |
| 3 | Splunk alert action | Send the alert payload to the n8n webhook |
| 4 | n8n | Pass the alert to the OpenAI agent and manage the workflow |
| 5 | VirusTotal and AbuseIPDB | Return hash and IP reputation context when called by the agent |
| 6 | OpenAI agent | Combine the alert evidence and enrichment into a structured triage report |
| 7 | DFIR-IRIS | Create an investigation alert containing the completed analysis |
| 8 | Slack | Deliver the completed analysis to the analyst |
| 9 | Human analyst | Validate the findings and make the final decision |

## Demonstration Scenario

This project used a controlled, multi-stage simulation on the Windows 10 endpoint. The activity was intentionally generated to test both automated alert triage and analyst-driven investigation.

### Phase 1 — Failed-Logon Simulation

Five incorrect Windows credentials were manually entered within approximately 16 seconds. This generated Windows Security Event ID `4625` events for user `ndean` on `DESKTOP-ESM4I8F`.

The Splunk detection identified the rapid failed-logon pattern and triggered the automated workflow:

```text
Windows Event ID 4625 → Splunk detection → n8n webhook
→ OpenAI triage and threat-intelligence enrichment
→ DFIR-IRIS alert and Slack notification → Analyst validation
```

This phase simulated password-guessing behavior associated with MITRE ATT&CK `T1110.001 — Password Guessing`. Because the failed logons were manually generated from a familiar internal source and no successful unauthorized login was observed, the activity was not treated as a confirmed hostile attack.

### Phase 2 — PowerShell and Atomic Red Team Activity

Additional activity was then generated using the Invoke-AtomicRedTeam framework to simulate PowerShell-based execution associated with MITRE ATT&CK `T1059.001 — PowerShell`.

Mimikatz test assets were also downloaded as part of the authorized exercise. Microsoft Defender detected the download and successfully quarantined it, generating Event IDs `1116` and `1117`.

Claude Desktop used the Splunk MCP server to perform read-only searches of this additional telemetry. The results confirmed:

- PowerShell module logging through Event ID `4103`.
- PowerShell script-block logging through Event ID `4104`.
- Invoke-AtomicRedTeam framework and module activity.
- Mimikatz detection through Defender Event ID `1116`.
- Successful quarantine through Defender Event ID `1117`.

The observed PowerShell activity supported the mapping to `T1059.001`. However, the returned evidence did not independently confirm the exact Atomic test command.

The available evidence also did not confirm that Mimikatz executed, accessed LSASS memory, dumped credentials, or compromised the endpoint. Therefore, the Mimikatz download was documented as a detected and quarantined test asset rather than successful credential-dumping activity.

## Key Features

### Splunk Detection

- Ingested Windows Security, PowerShell, and Defender events through the Splunk Universal Forwarder.
- Created detection logic for the controlled Event ID `4625` failed-logon activity.
- Configured Splunk to trigger the automation workflow when the search returned at least three failures within five minutes.

### n8n Orchestration

- Received Splunk alerts through a webhook.
- Normalized alert fields for consistent downstream processing.
- Connected the OpenAI agent to VirusTotal and AbuseIPDB enrichment tools.
- Routed the completed analysis to DFIR-IRIS and Slack.
- Preserved the original alert evidence for analyst review.

### AI-Assisted Triage

- Used OpenAI to generate a structured triage report.
- Allowed the OpenAI agent to request VirusTotal and AbuseIPDB enrichment when supported indicators were present.
- Separated observed facts from analytical assessment.
- Included severity, MITRE ATT&CK mapping, evidence limitations, and recommended next steps.
- Kept the final escalation decision under human control.

## Bonus Integrations

### Threat-Intelligence Enrichment

The original failed-logon alert contained a private lab source IP and no observed file hash. To validate the enrichment integrations, a controlled public IP was submitted to AbuseIPDB and a test SHA-256 hash was submitted to VirusTotal.

These indicators were labeled as simulated enrichment inputs. They provided supporting context for integration testing but were not treated as evidence observed in the original authentication alert and did not determine its final severity.

See [Threat-Intelligence-Enrichment.md](Threat-Intelligence-Enrichment.md).

### DFIR-IRIS Alert Management

The workflow creates a structured investigation alert containing the alert summary, severity, host, user, MITRE ATT&CK mapping, enrichment context, evidence, and recommended investigation steps. An analyst can review the alert and escalate it into a case when additional investigation is required.

See [DFIR-IRIS-Integration.md](DFIR-IRIS-Integration.md).

### Splunk MCP and Claude Desktop

Claude Desktop connects to the Splunk lab through a local MCP server. This allows Claude to run approved Splunk tools, review returned events, and assist with identifying investigation pivots using natural-language requests. Claude's findings are then compared with the original Splunk results before they are accepted.

See [Splunk-MCP-Claude.md](Splunk-MCP-Claude.md).

## Technologies Used

| Category | Technologies |
|---|---|
| Endpoint telemetry | Windows 10, Windows Event Logs |
| SIEM | Splunk Enterprise, Splunk Universal Forwarder |
| Automation | n8n, webhooks, JSON, APIs |
| AI analysis | OpenAI API |
| Threat intelligence | VirusTotal, AbuseIPDB |
| Alert and case management | DFIR-IRIS |
| Notification | Slack |
| Analyst assistance | Claude Desktop, Model Context Protocol, Splunk MCP |
| Infrastructure | Ubuntu, Docker, Windows PowerShell |

## Project Results

- Built and validated an end-to-end alert-triage workflow in an isolated lab.
- Connected endpoint detection, orchestration, enrichment, AI analysis, alert management, and analyst notification.
- Produced consistent triage output from a repeatable alert payload.
- Added failure testing and troubleshooting documentation for key integrations.
- Demonstrated how AI can assist investigation without replacing evidence validation or analyst judgment.

No numerical time-saving claim is made because a formal before-and-after performance study was not completed.

## Evidence

| Project Area | Evidence Folder |
|---|---|
| Core Splunk-to-Slack workflow | [`screenshots/01-core-automation/`](screenshots/01-core-automation/) |
| VirusTotal and AbuseIPDB enrichment | [`screenshots/02-threat-intelligence/`](screenshots/02-threat-intelligence/) |
| DFIR-IRIS alert creation | [`screenshots/03-dfir-iris/`](screenshots/03-dfir-iris/) |
| Splunk MCP and Claude | [`screenshots/04-splunk-mcp-claude/`](screenshots/04-splunk-mcp-claude/) |


## Security and Sensitive-Data Handling

- The project used authorized lab data and did not contain production or customer information.
- API keys, passwords, credential identifiers, authentication headers, and private webhook URLs were removed from the published files.
- Live credentials remained in excluded local n8n and Claude Desktop configuration files.
- Only necessary alert fields were sent externally, and simulated enrichment indicators were clearly labeled.
- Screenshots were reviewed for sensitive information before publication.
- Disabled certificate verification was limited to the isolated lab; production use would require trusted TLS certificates.

## Lessons Learned

- Network connectivity, service status, destination configuration, and data ingestion must be validated separately.
- Automation is most reliable when each integration is tested independently before the full workflow is enabled.
- Structured input and output reduce inconsistent AI responses and make downstream processing easier.
- Error branches and duplicate-event handling are as important as the successful workflow path.
- AI provides the most value when it organizes evidence and recommends pivots without being treated as the source of truth.

## Future Improvements

- Automatically distinguish public IP addresses from private or reserved addresses before enrichment.
- Submit hashes to VirusTotal only when they are present in the original alert.
- Require structured JSON output from the AI agent.
- Dynamically map AI severity to the correct DFIR-IRIS severity ID.
- Add authenticated or signed webhook requests.
- Add retries, timeout handling, failure notifications, and duplicate-alert prevention.
- Escalate DFIR-IRIS alerts into cases only after analyst review.
- Use trusted TLS certificates and least-privilege service accounts.
- Measure manual versus automated triage time and report consistency.

## Disclaimer

This project was created for defensive cybersecurity education and authorized lab testing. It does not contain production credentials, confidential organizational data, or instructions intended for unauthorized activity.
