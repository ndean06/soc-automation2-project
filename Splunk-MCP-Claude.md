# Splunk MCP and Claude Desktop Integration

## Purpose

This document explains how Claude Desktop was connected to the Splunk lab through a local Model Context Protocol (MCP) server. The integration allows an analyst to request Splunk searches in natural language, review the returned evidence, and identify useful investigation pivots.

This is an analyst-assistance capability. Claude does not replace Splunk as the source of truth, and its conclusions require validation against the original events.

## Role in the Project

The Claude–Splunk MCP integration is separate from the automated alert-triage workflow.

| Workflow | Purpose | Trigger |
|---|---|---|
| Splunk → n8n → OpenAI → DFIR-IRIS and Slack | Automates alert enrichment, triage, documentation, and notification | A scheduled Splunk alert returns matching results |
| Claude Desktop → Splunk MCP → Splunk | Supports analyst-driven searches and follow-up investigation | An analyst submits a natural-language request in Claude Desktop |

The MCP path used in this lab was:

```text
Analyst prompt in Claude Desktop
        ↓
Local Splunk MCP server launched through uv
        ↓
Splunk REST API over HTTPS port 8089
        ↓
Read-only search results returned to Claude
        ↓
Claude organizes the evidence for analyst validation
```

MCP provides a standard way for an AI application to access approved external tools. In this project, the local MCP server exposed Splunk search functionality to Claude Desktop.

## Components

| Component | Function |
|---|---|
| Claude Desktop | Accepts the analyst's natural-language investigation request |
| Local Splunk MCP server | Translates the tool request into a Splunk search and returns the results |
| uv | Manages the Python environment and launches the MCP server with its required dependencies |
| Splunk Enterprise | Searches the indexed Windows Event Log evidence |
| Human analyst | Verifies the query scope, source events, and conclusions |

## Prerequisites

- Claude Desktop installed on the Windows 10 endpoint.
- `uv` installed and available locally.
- A local copy of the Splunk MCP server.
- Network access from the endpoint to the Splunk management interface on TCP port `8089`.
- A dedicated Splunk account with only the permissions required to search the lab index.
- The correct time range, time zone, index, and host values for each investigation.

## Claude Desktop Configuration

The `mcpServers` object was added at the top level of the existing Claude Desktop configuration. Existing objects such as `preferences` were preserved; the configuration file was not replaced with a second JSON object.

The following example is sanitized. Replace each placeholder locally and do not commit the live configuration file.

```json
{
  "mcpServers": {
    "splunk": {
      "command": "C:\\PATH\\TO\\uv.exe",
      "args": [
        "--directory",
        "C:\\PATH\\TO\\splunk-mcp",
        "run",
        "python",
        "splunk_mcp.py",
        "stdio"
      ],
      "env": {
        "SPLUNK_HOST": "SPLUNK_HOST",
        "SPLUNK_PORT": "8089",
        "SPLUNK_USERNAME": "SPLUNK_READ_ONLY_USERNAME",
        "SPLUNK_PASSWORD": "SPLUNK_READ_ONLY_PASSWORD",
        "SPLUNK_SCHEME": "https",
        "VERIFY_SSL": "false"
      }
    }
  }
}
```

Windows paths use escaped backslashes in JSON. An absolute path to `uv.exe` may be necessary because desktop applications do not always inherit the same `PATH` environment as an interactive terminal. The `--directory` argument tells `uv` where to find the MCP project before it runs the Python server.

`VERIFY_SSL=false` was used only because the isolated lab used a self-signed certificate. A production deployment should use a trusted certificate and enable certificate verification.

## Connection Validation

After saving the configuration, Claude Desktop was fully exited and restarted so it could launch the MCP server. The connection was considered successful when the Splunk tools appeared in Claude Desktop and a limited test search returned expected lab data.

<!-- Screenshot: screenshots/04-splunk-mcp-claude/01-claude-mcp-connected.png -->

Useful local validation commands are listed below.

| Tool | Command Purpose | Syntax/Example |
|---|---|---|
| PowerShell | Confirm uv is installed | `uv --version` |
| Command Prompt | Locate the uv executable | `where.exe uv` |
| PowerShell | Test access to the Splunk management port | `Test-NetConnection SPLUNK_HOST -Port 8089` |
| PowerShell | Validate the Claude configuration as JSON | `Get-Content ".\claude_desktop_config.json" -Raw \| ConvertFrom-Json \| Out-Null; Write-Host "JSON is valid"` |

## Investigation 1: Failed Logons

The first MCP investigation reviewed the controlled failed-logon activity that supported the project's Splunk detection. Incorrect credentials were entered manually to generate Windows Security Event ID `4625` records.

### Analyst Request

```text
Using the Splunk MCP server, run a read-only search for Windows Event ID 4625
in index mydfir-project on DESKTOP-ESM4I8F between September 11, 2026
11:00 and 11:10 UTC. Return timestamp, computer, user, source IP, failure
reason, logon type, authentication package, and failed-attempt count.
Do not modify Splunk data or saved searches.
```

### Search Scope

| Field | Value |
|---|---|
| Index | `mydfir-project` |
| Host | `DESKTOP-ESM4I8F` |
| Event code | `4625` |
| Time range | September 11, 2026, 11:00–11:10 UTC |

### Results

Claude returned five failed logons within approximately 16 seconds.

| Timestamp (UTC) | Computer | User | Source IP | Result |
|---|---|---|---|---|
| 2026-09-11 11:03:55.044 | `DESKTOP-ESM4I8F` | `ndean` | `192.168.117.1` | Bad password or unknown username |
| 2026-09-11 11:03:57.734 | `DESKTOP-ESM4I8F` | `ndean` | `192.168.117.1` | Bad password or unknown username |
| 2026-09-11 11:04:04.189 | `DESKTOP-ESM4I8F` | `ndean` | `192.168.117.1` | Bad password or unknown username |
| 2026-09-11 11:04:07.195 | `DESKTOP-ESM4I8F` | `ndean` | `192.168.117.1` | Bad password or unknown username |
| 2026-09-11 11:04:11.747 | `DESKTOP-ESM4I8F` | `ndean` | `192.168.117.1` | Bad password or unknown username |

The events also shared the following fields:

- Logon type: `3` (network)
- Authentication package: `NTLM`
- Status and substatus: `0xC000006D` and `0xC000006A`
- Workstation name: `NKD-HP-PC`
- MITRE ATT&CK demonstration mapping: `T1110.001 — Password Guessing`

The five-event burst matched the project's detection condition, but the evidence did not establish a hostile attack. The source address was a familiar private lab address, and the activity was generated by manually entering incorrect credentials. This distinction demonstrates why a detection match still requires analyst context.

<!-- Screenshot: screenshots/04-splunk-mcp-claude/02-claude-failed-logon-query.png -->

## Investigation 2: PowerShell, Atomic Red Team, and Mimikatz Evidence

A separate analyst-led MCP search reviewed controlled activity from September 13, 2026. This activity did not trigger the original failed-logon automation. It was used to demonstrate how Claude could search Splunk for related endpoint evidence and organize the results.

### Analyst Request

```text
Using the Splunk MCP server, run a read-only search in index mydfir-project
for activity on DESKTOP-ESM4I8F during September 13, 2026. Look for
PowerShell, Invoke-AtomicTest, T1059.001, and Mimikatz. Return timestamp,
computer, user, event code, process or script path, command or script summary,
and matching evidence. Do not modify Splunk data or saved searches.
```

### Evidence Summary

| Time (UTC) | Event | Evidence | Interpretation |
|---|---:|---|---|
| 18:54:37 | `1116` | Microsoft Defender identified `HackTool:Win32/Mimikatz` in a partially downloaded Mimikatz archive | Mimikatz-related content was detected during download |
| 18:54:42 | `1117` | Microsoft Defender reported a successful quarantine action | The detected content was quarantined |
| 19:05:58 | `4103` | PowerShell module logging recorded archive-related commands | PowerShell was used during the setup activity |
| Around 19:06:30 | `4104` | Script Block Logging recorded Invoke-AtomicRedTeam modules and configuration files | The Atomic Red Team framework was loaded |
| Around 19:19:30 | `4104` | Numerous sequential script-block chunks were recorded | Additional PowerShell content was logged and required deeper review |

Examples of observed Atomic Red Team framework files included:

- `config.ps1`
- `Attire-ExecutionLogger.psm1`
- `Invoke-AtomicRedTeam.psm1`
- `Install-AtomicsFolder.ps1`

### Interpretation Boundaries

- The literal value `T1059.001` did not appear in the returned events. The PowerShell technique mapping was an analyst inference based on the observed PowerShell telemetry.
- Framework-loading evidence was present, but the exact command `Invoke-AtomicTest T1059.001` was not independently confirmed in the returned data.
- Defender detection and quarantine proved that Mimikatz-related content was downloaded and blocked. The evidence did not prove Mimikatz execution, LSASS access, credential dumping, or endpoint compromise.
- Event ID `4104` records did not contain an extracted user field in this dataset. Attribution to `ndean` was inferred from nearby Event ID `4103` records and the surrounding session timeline.
- The multi-part script blocks were not classified as malicious without first reconstructing and reviewing their complete content.

These boundaries prevent the AI summary from overstating what the source telemetry proves.

<!-- Screenshot: screenshots/04-splunk-mcp-claude/03-claude-atomic-red-team-query.png -->

## Analyst Validation

Claude's response was treated as an organized summary of the MCP search results, not as independent evidence. Validation included:

1. Confirming the index, host, event codes, and UTC time range.
2. Comparing timestamps and field values with the original Splunk events.
3. Separating directly observed facts from inferred technique mappings and user attribution.
4. Confirming that the searches did not modify Splunk data, saved searches, or configuration.
5. Avoiding compromise claims that were not supported by endpoint evidence.

## Security Considerations

- The live Claude Desktop configuration was excluded from the repository because it contained the Splunk username and password.
- The published configuration uses placeholders and does not reveal credentials or unrestricted local paths.
- Splunk access should use a dedicated least-privilege service account restricted to the required indexes and search capabilities.
- Natural-language instructions such as “read-only” express analyst intent but do not enforce authorization. Splunk roles and the MCP tool design must provide the actual control.
- Only approved MCP servers and tools should be enabled.
- Search results may contain usernames, hostnames, internal IP addresses, commands, and other sensitive log data. Screenshots and examples should be reviewed before publication.
- HTTPS was used for the Splunk REST connection. Disabled certificate verification was limited to the self-signed lab environment.
- MCP server code and dependencies should be reviewed and version-pinned before use with sensitive systems.

## Troubleshooting

| Symptom | Likely Cause | Correction |
|---|---|---|
| Claude does not show the Splunk tools | Claude did not reload the configuration | Fully exit Claude Desktop and reopen it |
| Claude configuration will not load | Invalid JSON or a second top-level JSON object was added | Merge `mcpServers` into the existing object and validate the JSON |
| MCP server fails to start | Claude cannot locate uv | Use the absolute path to `uv.exe` |
| Server file cannot be found | The MCP directory is relative or incorrect | Use an absolute Windows path with escaped backslashes |
| Splunk connection fails | Management port, host, service, or firewall issue | Test TCP port `8089` and confirm the Splunk service is available |
| Authentication fails | Incorrect credentials or insufficient Splunk permissions | Verify the dedicated account and its assigned role |
| TLS validation fails | The lab uses a self-signed certificate | Use the lab-only exception temporarily; install a trusted certificate for production |
| Search returns no results | Incorrect index, host, time range, time zone, or field name | Recheck the search scope directly in Splunk |
| Claude overstates a finding | Observations and inferences were not clearly separated | Compare the summary with the raw events and revise the conclusion |

Additional integration problems and their resolutions are recorded in [Troubleshooting-Log.md](Troubleshooting-Log.md).

## Limitations

- Claude Desktop and the local MCP server must be running for interactive searches.
- Search quality depends on accurate time ranges, time zones, field extractions, and available telemetry.
- Some event types did not expose a normalized user field.
- Claude can omit details, misinterpret context, or infer more than the logs support.
- A read-only instruction in a prompt is not a substitute for technical access controls.
- The test demonstrated investigation assistance, not autonomous incident response.

## Future Improvements

- Use a dedicated Splunk service account limited to approved indexes and search endpoints.
- Replace the self-signed certificate, set `VERIFY_SSL` to `true`, and validate the certificate chain.
- Restrict the MCP server to an explicit allowlist of read-only search tools.
- Add limits for search time ranges, returned event counts, and permitted indexes.
- Record MCP tool calls and SPL queries for audit review.
- Return a structured schema containing observations, inferences, confidence, and direct event references.
- Improve field normalization for Windows Security, Defender, and PowerShell events.
- Add a connection-health test and documented dependency versions.
- Store credentials through a more secure secrets-management method where supported.

## Related Documentation

- [Architecture.md](Architecture.md)
- [Walkthrough.md](Walkthrough.md)
- [Testing-Validation.md](Testing-Validation.md)
- [Troubleshooting-Log.md](Troubleshooting-Log.md)

## References

- [Model Context Protocol overview](https://modelcontextprotocol.io/docs/2026-07-28/getting-started/intro)
- [Claude Desktop local MCP server guidance](https://support.claude.com/en/articles/10949351-getting-started-with-local-mcp-servers-on-claude-desktop)
- [Splunk REST API reference](https://help.splunk.com/en/splunk-enterprise/rest-api-reference)

## Disclaimer

This integration was created in an isolated and authorized cybersecurity lab. The example searches and results are provided for defensive education. AI-generated findings must be verified against the original telemetry before any investigation or response decision is made.
