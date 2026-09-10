# SOC Automation 2.0 — Implementation Walkthrough

## Overview

This walkthrough documents how the SOC Automation 2.0 lab was built and connected. It follows the alert from Windows endpoint telemetry through Splunk detection, n8n orchestration, AI-assisted analysis, threat-intelligence enrichment, case creation, and analyst notification.

The optional Claude Desktop and Splunk MCP integration is documented separately because it provides analyst-driven access to Splunk rather than participating in the automated alert workflow.

See [Architecture.md](Architecture.md) for the complete system diagram, component roles, and communication paths.

## 1. Lab Environment

The project was built in an isolated virtual lab using the following systems:

| System | Primary Role |
|---|---|
| Windows 10 VM | Endpoint used to generate controlled security telemetry |
| Sysmon | Detailed process, network, file, and registry logging |
| Splunk Universal Forwarder | Sends Windows and Sysmon events to Splunk |
| Splunk Enterprise VM | Log ingestion, search, detection, and alert generation |
| n8n VM | Workflow orchestration and integration management |
| OpenAI API | Structured alert analysis and triage-report generation |
| VirusTotal and AbuseIPDB | Hash, URL, domain, and IP-reputation enrichment |
| DFIR-IRIS | Investigation case management |
| Slack | Real-time analyst notification |
| Claude Desktop and Splunk MCP | Optional analyst-assisted Splunk search interface |

The VMs communicated over a private lab network. Credentials, API keys, tokens, and complete webhook URLs are excluded from this repository.

<!-- Add the final architecture diagram or link here if desired. -->

## 2. Windows Telemetry and Splunk Ingestion

Sysmon was installed on the Windows 10 endpoint to capture detailed endpoint activity. The Splunk Universal Forwarder collected the selected Windows Event Logs and transmitted them to the Splunk server.

The forwarder was configured to use the Splunk receiving port:

```text
Windows 10 + Sysmon → Splunk Universal Forwarder → Splunk TCP 9997
```

The following items were validated independently:

- Network connectivity between the Windows and Splunk VMs
- Splunk receiving port `9997`
- Splunk Universal Forwarder service status
- Forwarding destination in `outputs.conf`
- Active TCP connection from the endpoint to Splunk
- Arrival of Windows and Sysmon events in Splunk

The final Splunk search confirmed that events from the Windows endpoint were being indexed successfully.

<!-- Screenshot: screenshots/01-core-automation/01-splunk-telemetry-ingestion.png -->

> A forwarding-destination issue was identified and corrected during setup. The root cause and resolution are documented in [Troubleshooting-Log.md](Troubleshooting-Log.md).

## 3. Controlled Security Simulation

Security telemetry was generated through an authorized Atomic Red Team simulation on the Windows 10 VM. Invoke-AtomicRedTeam and the Atomics test definitions were installed in the isolated lab, and the required test assets were staged.

### Simulation Details

| Field | Value |
|---|---|
| MITRE ATT&CK technique | `[Add verified technique ID and name]` |
| Atomic test number | `[Add verified test number]` |
| Test name | `[Add exact Atomic Red Team test name]` |
| Execution time | `[Add date and time in UTC]` |
| Detection source | `[Add Sysmon Event ID and/or Splunk detection name]` |

The simulation produced repeatable endpoint activity that could be detected in Splunk and submitted to the automation workflow.

<!-- Screenshot: screenshots/01-core-automation/02-atomic-red-team-simulation.png -->

## 4. Splunk Detection and Alert

Splunk searched the incoming endpoint telemetry for the controlled activity. The detection logic was reviewed to confirm that the returned event matched the expected host, user, process, and timestamp.

### Detection Details

| Field | Value |
|---|---|
| Detection name | `[Add exact Splunk alert name]` |
| Index | `[Add index]` |
| Sourcetype | `[Add sourcetype]` |
| Trigger condition | Number of results greater than `0` |
| Alert action | Send webhook request to n8n |

Add the final sanitized SPL query here after verifying it against the saved Splunk detection:

```spl
[Add final detection query]
```

The saved search was configured as an alert. When matching results were returned, Splunk sent the selected alert fields to the n8n webhook.

<!-- Screenshot: screenshots/01-core-automation/03-splunk-detection-results.png -->
<!-- Screenshot: screenshots/01-core-automation/04-splunk-alert-configuration.png -->

## 5. n8n Workflow Orchestration

n8n was deployed in Docker and used as the central workflow orchestrator. The workflow received the Splunk alert, prepared the data for analysis, invoked the AI agent and enrichment tools, and delivered the final output to DFIR-IRIS and Slack.

### Core Workflow Nodes

| Node | Function |
|---|---|
| Webhook | Receives the Splunk alert payload |
| OpenAI agent | Analyzes the alert and produces the structured triage report |
| VirusTotal enrichment | Retrieves supported hash, file, URL, or domain context |
| AbuseIPDB enrichment | Retrieves IP-reputation and abuse-reporting context |
| DFIR-IRIS HTTP request | Creates or updates the investigation case |
| Slack message | Sends the analyst notification |

The workflow preserved the original alert evidence and passed only the required fields to external services.

<!-- Screenshot: screenshots/01-core-automation/05-n8n-core-workflow.png -->

A sanitized workflow export should be stored in [`workflow-exports/`](workflow-exports/). Credential values, tokens, and webhook secrets must be removed before publishing.

## 6. OpenAI Triage Agent

The OpenAI agent received the normalized alert context from n8n. The prompt required a structured response so the output could be used consistently by DFIR-IRIS and Slack.

### Required Triage Output

- Executive alert summary
- Observed evidence and affected assets
- Severity and confidence assessment
- MITRE ATT&CK mapping
- Threat-intelligence enrichment results
- Evidence gaps and limitations
- Recommended investigation steps
- Suggested containment or escalation actions

The prompt instructed the model to distinguish observed evidence from analytical conclusions and to avoid inventing missing facts.

<!-- Screenshot: screenshots/01-core-automation/06-openai-triage-output.png -->

## 7. Threat-Intelligence Enrichment

VirusTotal and AbuseIPDB were connected as enrichment tools available to the OpenAI agent through the n8n workflow.

### VirusTotal

VirusTotal was used when the alert contained a supported hash, URL, domain, or other indicator. The response added reputation and detection context to the triage report.

<!-- Screenshot: screenshots/02-threat-intelligence/01-virustotal-enrichment.png -->

### AbuseIPDB

AbuseIPDB was used when the alert contained an IP address. The response added confidence, reporting, and reputation context to the analysis.

<!-- Screenshot: screenshots/02-threat-intelligence/02-abuseipdb-enrichment.png -->

Enrichment did not replace the original Splunk evidence. It was treated as supporting context and remained subject to analyst validation.

See [Threat-Intelligence-Enrichment.md](Threat-Intelligence-Enrichment.md) for the field mappings and sanitized request examples.

## 8. DFIR-IRIS Case Creation

After the AI triage report was returned, n8n sent the selected fields to DFIR-IRIS. The resulting case provided a central location for investigation tracking.

The case included:

- Alert and detection name
- Severity and confidence
- Affected host and user
- Evidence summary
- MITRE ATT&CK mapping
- VirusTotal and AbuseIPDB context
- Recommended investigation steps
- Original event references

<!-- Screenshot: screenshots/03-dfir-iris/01-dfir-iris-case-created.png -->

See [DFIR-IRIS-Integration.md](DFIR-IRIS-Integration.md) for the API mapping and case-field details.

## 9. Slack Analyst Notification

n8n also sent a concise summary to Slack so the analyst could review the alert in real time. The Slack message contained enough information for initial prioritization while directing the analyst to the full DFIR-IRIS case and Splunk evidence.

The notification included:

- Alert name and severity
- Affected endpoint
- Short evidence summary
- Enrichment highlights
- Recommended next action
- DFIR-IRIS case reference, when available

<!-- Screenshot: screenshots/01-core-automation/07-slack-analyst-notification.png -->

## 10. Claude Desktop and Splunk MCP

Claude Desktop was configured as an optional analyst-assistance interface. It communicated with a local Splunk MCP server, which acted as the controlled query bridge to the Splunk REST API over HTTPS port `8089`.

```text
Claude Desktop ↔ Splunk MCP server ↔ Splunk REST API
```

This path allowed an analyst to request approved Splunk searches using natural language and receive summarized results. It did not trigger the n8n workflow and did not replace direct validation in Splunk.

Configuration examples must use placeholders instead of real Splunk credentials.

<!-- Screenshot: screenshots/04-splunk-mcp-claude/01-claude-mcp-configuration-sanitized.png -->
<!-- Screenshot: screenshots/04-splunk-mcp-claude/02-claude-splunk-search-results.png -->

See [Splunk-MCP-Claude.md](Splunk-MCP-Claude.md) for configuration and validation details.

## 11. End-to-End Validation

The completed workflow was validated from telemetry generation through analyst notification:

1. Controlled activity generated Windows and Sysmon events.
2. The Universal Forwarder delivered the events to Splunk.
3. The Splunk detection returned the expected matching event.
4. Splunk triggered the n8n webhook.
5. n8n submitted the alert to the OpenAI agent.
6. VirusTotal and AbuseIPDB returned applicable enrichment.
7. OpenAI produced the required structured triage report.
8. n8n created the DFIR-IRIS case.
9. n8n delivered the Slack notification.
10. The analyst compared the automation output with the original Splunk evidence.

Detailed expected results, actual results, and evidence references are maintained in [Testing-Validation.md](Testing-Validation.md).

## Outcome

The project demonstrates how SIEM detection, workflow orchestration, threat intelligence, AI-assisted analysis, case management, and analyst notification can be combined into one repeatable SOC triage process.

The automation reduces repetitive evidence-handling work while preserving human responsibility for validation, escalation, and response decisions.

## Security and Publishing Notes

- All activity was performed in an isolated and authorized lab.
- Screenshots must be reviewed for credentials, tokens, and complete webhook URLs.
- Workflow and configuration exports must contain placeholders only.
- Private keys, `.env` files, credential databases, and live API tokens must not be committed.
- Hostnames and private IP addresses may be sanitized when they are not necessary to understand the evidence.
