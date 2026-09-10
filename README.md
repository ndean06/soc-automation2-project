# SOC Automation 2.0

An end-to-end SOC alert-triage automation lab integrating Splunk, n8n, OpenAI, Slack, VirusTotal, DFIR-IRIS, and Claude Desktop through the Model Context Protocol (MCP).

## Project Summary

SOC analysts often spend valuable time collecting alert details, enriching indicators, documenting findings, and transferring information between separate tools. This project demonstrates how those repetitive steps can be automated while keeping the analyst responsible for validating the evidence and making the final escalation decision.

The workflow begins with Windows 10 and Sysmon telemetry in Splunk. When Splunk detects suspicious activity, it sends an alert to an n8n webhook. The webhook passes the alert to an OpenAI agent, which uses VirusTotal and AbuseIPDB as enrichment tools during its analysis. The completed triage output is then sent to DFIR-IRIS for ticket creation and to Slack for analyst notification.

A separate Splunk MCP integration allows Claude Desktop to interact with the Splunk lab through approved tools. Claude can assist with analyst-driven searches and investigation pivots, but its results are validated against the original Splunk evidence.

## Objective

Build an end-to-end proof-of-concept pipeline that demonstrates how a modern Security Operations Center can:

1. Reduce repetitive alert-enrichment and documentation work.
2. Produce consistent and structured triage reports.
3. Improve situational awareness through MITRE ATT&CK mapping.
4. Create investigation cases and analyst notifications automatically.
5. Use AI as decision support while preserving human review.

## Architecture

![SOC Automation 2.0 Architecture](screenshots/01-core-automation/SOC-Auto-2-Proj.png)

See [Architecture.md](Architecture.md) for detailed component roles, communication paths, ports, and security considerations.

## Project Documentation

| Document | Description |
|---|---|
| [Architecture.md](Architecture.md) | Component roles, communication paths, and trust boundaries |
| [Walkthrough.md](Walkthrough.md) | Complete implementation and evidence walkthrough |
| [Threat-Intelligence-Enrichment.md](Threat-Intelligence-Enrichment.md) | VirusTotal and AbuseIPDB integration details |
| [DFIR-IRIS-Integration.md](DFIR-IRIS-Integration.md) | Automated case-creation details |
| [Splunk-MCP-Claude.md](Splunk-MCP-Claude.md) | Claude-to-Splunk MCP integration |
| [Testing-Validation.md](Testing-Validation.md) | Expected results, actual results, and validation status |
| [Troubleshooting-Log.md](Troubleshooting-Log.md) | Problems, root causes, corrections, and lessons |
| [`workflow-exports/`](workflow-exports/) | Sanitized n8n workflow export |
| [`config-examples/`](config-examples/) | Placeholder-only configuration examples |

## Core Workflow

| Stage | Component | Function |
|---:|---|---|
| 1 | Windows 10 and Sysmon | Generate endpoint process, network, file, and registry telemetry |
| 2 | Splunk Enterprise | Ingest telemetry and detect suspicious behavior |
| 3 | Splunk alert action | Send the alert payload to the n8n webhook |
| 4 | n8n | Pass the alert to the OpenAI agent and manage the workflow |
| 5 | VirusTotal and AbuseIPDB | Return hash and IP reputation context when called by the agent |
| 6 | OpenAI agent | Combine the alert evidence and enrichment into a structured triage report |
| 7 | DFIR-IRIS | Create an investigation ticket containing the completed analysis |
| 8 | Slack | Deliver the completed analysis to the analyst |
| 9 | Human analyst | Validate the findings and make the final decision |

## Demonstration Scenario

Security telemetry for this project was generated through a controlled Atomic Red Team simulation on the Windows 10 VM. The lab was prepared by installing the Invoke-AtomicRedTeam execution framework and the Atomics test definitions. Mimikatz test assets were staged to support the authorized credential-access simulation.

### Simulation Details

- **Atomic Red Team technique:** `[Technique ID — Technique Name]`
- **Atomic test number:** `[Test Number]`
- **Detection source:** `[Sysmon Event ID or Splunk detection name]`

The project validates the complete path from endpoint telemetry through analyst notification:

```text
Sysmon event → Splunk detection → n8n webhook → OpenAI agent
→ VirusTotal and AbuseIPDB tool calls → structured triage
→ DFIR-IRIS ticket and Slack notification → analyst validation
```

## Key Features

### Splunk Detection

- Ingested Windows 10 and Sysmon telemetry through the Splunk Universal Forwarder.
- Created detection logic for the controlled Atomic Red Team activity.
- Configured Splunk to trigger the automation workflow when the search returned a matching event.

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
- Included severity, confidence, MITRE ATT&CK mapping, evidence gaps, and recommended next steps.
- Kept the final escalation decision under human control.

## Bonus Integrations

### Threat-Intelligence Enrichment

The OpenAI agent uses VirusTotal for supported hash or indicator reputation and AbuseIPDB for IP-address reputation. The returned context becomes part of the agent's analysis without replacing the original alert evidence.

See [Threat-Intelligence-Enrichment.md](Threat-Intelligence-Enrichment.md).

### DFIR-IRIS Case Management

The workflow creates a structured investigation case containing the alert summary, severity, host, user, MITRE ATT&CK mapping, enrichment, evidence, and recommended investigation steps.

See [DFIR-IRIS-Integration.md](DFIR-IRIS-Integration.md).

### Splunk MCP and Claude Desktop

Claude Desktop connects to the Splunk lab through a local MCP server. This allows Claude to run approved Splunk tools, review returned events, and assist with identifying investigation pivots using natural-language requests. Claude's findings are then compared with the original Splunk results before they are accepted.

See [Splunk-MCP-Claude.md](Splunk-MCP-Claude.md).

## Technologies Used

| Category | Technologies |
|---|---|
| Endpoint telemetry | Windows 10, Sysmon, Windows Event Logs |
| SIEM | Splunk Enterprise, Splunk Universal Forwarder |
| Automation | n8n, webhooks, JSON, APIs |
| AI analysis | OpenAI API |
| Threat intelligence | VirusTotal, AbuseIPDB |
| Case management | DFIR-IRIS |
| Notification | Slack |
| Analyst assistance | Claude Desktop, Model Context Protocol, Splunk MCP |
| Infrastructure | Ubuntu, Docker, Windows PowerShell |

## Project Results

- Built and validated an end-to-end alert-triage workflow in an isolated lab.
- Connected endpoint detection, orchestration, enrichment, AI analysis, case management, and analyst notification.
- Produced consistent triage output from a repeatable alert payload.
- Added failure testing and troubleshooting documentation for key integrations.
- Demonstrated how AI can assist investigation without replacing evidence validation or analyst judgment.

No numerical time-saving claim is made because a formal before-and-after performance study was not completed.

## Evidence

| Project Area | Evidence Folder |
|---|---|
| Core Splunk-to-Slack workflow | [`screenshots/01-core-automation/`](screenshots/01-core-automation/) |
| VirusTotal and AbuseIPDB enrichment | [`screenshots/02-threat-intelligence/`](screenshots/02-threat-intelligence/) |
| DFIR-IRIS case creation | [`screenshots/03-dfir-iris/`](screenshots/03-dfir-iris/) |
| Splunk MCP and Claude | [`screenshots/04-splunk-mcp-claude/`](screenshots/04-splunk-mcp-claude/) |


## Security Considerations

- The project was built in an isolated and authorized lab environment.
- API keys, passwords, tokens, active webhook URLs, and unrestricted configuration files are not committed.
- Public configuration examples contain placeholders only.
- Only the alert fields required for analysis are sent to external services.
- AI-generated findings are treated as analytical leads until confirmed in the source telemetry.
- Automated containment actions require testing and human approval before use.

## Lessons Learned

- Network connectivity, service status, destination configuration, and data ingestion must be validated separately.
- Automation is most reliable when each integration is tested independently before the full workflow is enabled.
- Structured input and output reduce inconsistent AI responses and make downstream processing easier.
- Error branches and duplicate-event handling are as important as the successful workflow path.
- AI provides the most value when it organizes evidence and recommends pivots without being treated as the source of truth.

## Disclaimer

This project was created for defensive cybersecurity education and authorized lab testing. It does not contain production credentials, confidential organizational data, or instructions intended for unauthorized activity.
