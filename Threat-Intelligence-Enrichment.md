# Threat-Intelligence Enrichment

## Purpose

This document explains how the SOC Automation 2.0 workflow used AbuseIPDB and VirusTotal to add threat-intelligence context to a Splunk alert. It covers the indicator flow, sanitized request structure, returned evidence, security controls, limitations, and analyst-validation requirements.

Threat intelligence was treated as supporting context. It did not replace the original Splunk evidence or make the final incident decision.

## Role in the Workflow

The OpenAI agent received selected alert details from the n8n webhook and could call the appropriate enrichment tool when an IP address or file hash was available.

```text
Splunk alert → n8n webhook → OpenAI agent
→ AbuseIPDB or VirusTotal tool → Enrichment results
→ Structured triage report → DFIR-IRIS and Slack
```

For this proof of concept, the original failed-logon alert contained a private lab IP address and no file hash. Controlled indicators were therefore supplied to validate both enrichment integrations.

## Observed and Simulated Indicators

| Indicator | Value | Source | Classification |
|---|---|---|---|
| Original source IP | `192.168.117.1` | Splunk Event ID `4625` results | Directly observed private lab address |
| Enrichment IP | `85.165.104.58` | Controlled test input | Simulated public source used for AbuseIPDB validation |
| Original file hash | None | Splunk failed-logon alert | No hash was observed in the authentication event |
| Enrichment hash | `186b26df63df3b7334043b47659cba4185c948629d857d47452cc1936f0aa5da` | Controlled test input | Test SHA-256 used for VirusTotal validation |

The simulated indicators were kept separate from the observed authentication evidence. They did not determine the final severity of the original failed-logon alert.

## AbuseIPDB Enrichment

### Purpose

AbuseIPDB was used to retrieve reputation and reporting context for a public IPv4 address. The n8n HTTP Request tool submitted the controlled IP to the AbuseIPDB API v2 `check` endpoint.

### Sanitized Request Structure

| Setting | Value |
|---|---|
| Method | `GET` |
| Endpoint | `https://api.abuseipdb.com/api/v2/check` |
| IP parameter | `ipAddress` |
| Report age | `maxAgeInDays=1` |
| Extended results | `verbose` |
| Authentication header | `Key: YOUR_ABUSEIPDB_API_KEY` |
| Response format | `Accept: application/json` |

```json
{
  "url": "https://api.abuseipdb.com/api/v2/check",
  "query": {
    "ipAddress": "CONTROLLED_PUBLIC_IP",
    "maxAgeInDays": "1",
    "verbose": true
  },
  "headers": {
    "Key": "YOUR_ABUSEIPDB_API_KEY",
    "Accept": "application/json"
  }
}
```

The published example uses a placeholder instead of the live API key.

### Validated Results

At the time of testing, AbuseIPDB returned the following context for the controlled public IP:

| Response field | Result |
|---|---|
| Public address | `true` |
| Abuse confidence score | `100` |
| Country | Norway (`NO`) |
| ISP | Telenor Norge AS |
| Usage type | Fixed Line ISP |
| Total reports | `49` |
| Distinct reporting users | `35` |

![AbuseIPDB enrichment result](screenshots/02-threat-intelligence/01-abuseipdb-enrichment.png)

The high score and reporting history showed how reputation data could increase investigation priority if the IP were actually present in an alert. In this project, however, the IP was a simulated enrichment input and was not evidence of an attack against the endpoint.

## VirusTotal Enrichment

### Purpose

VirusTotal was used to retrieve an existing file report for a controlled SHA-256 hash. The workflow queried the API by hash; it did not upload the file.

### Sanitized Request Structure

| Setting | Value |
|---|---|
| Method | `GET` |
| Endpoint | `https://www.virustotal.com/api/v3/files/{hash}` |
| Indicator type | SHA-256 |
| Authentication | n8n VirusTotal credential |
| Response format | `Accept: application/json` |

```json
{
  "method": "GET",
  "url": "https://www.virustotal.com/api/v3/files/CONTROLLED_SHA256",
  "authentication": "predefinedCredentialType",
  "nodeCredentialType": "virusTotalApi",
  "headers": {
    "Accept": "application/json"
  }
}
```

The live API key remained inside the local n8n credential store and was not included in the published workflow export.

### Validated Results

At the time of testing, VirusTotal returned the following file context:

| Response field | Result |
|---|---|
| Meaningful name | `ManageEngine-OpManager.msi` |
| File type | MSI |
| Malicious detections | `24` |
| Suspicious detections | `0` |
| Undetected engines | `35` |
| Reputation | `-63` |
| Relevant tags | `signed`, `revoked-cert`, `msi` |

![VirusTotal enrichment result](screenshots/02-threat-intelligence/02-virustotal-enrichment.png)

The results demonstrated that the VirusTotal tool could retrieve file-reputation data and return it to the AI agent. The test hash was not observed in the failed-logon alert and did not influence the alert's observed severity.

## How the AI Used the Results

The OpenAI agent combined the selected alert fields and enrichment responses into a structured triage report. The final output separated direct observations from the controlled enrichment scenario.

| Report area | Data used |
|---|---|
| Alert summary | Splunk alert name, endpoint, user, timestamp, and failed-attempt count |
| Observed severity | Five failed logons with no confirmed successful unauthorized login |
| IP context | AbuseIPDB results for the controlled public IP |
| Hash context | VirusTotal results for the controlled SHA-256 hash |
| ATT&CK mapping | `T1110.001 — Password Guessing` for the simulated failed-logon pattern |
| Recommended actions | Authentication review, successful-logon correlation, endpoint review, and continued monitoring |

The original alert was assessed as Medium severity. The public-IP reputation was presented as a simulated priority consideration, and the test hash was identified as unrelated to the observed authentication event.

## Security and Data Handling

- The project used authorized lab data rather than production or customer information.
- API keys, credential identifiers, authentication headers, and private webhook paths were removed from the published files.
- Live credentials remained in excluded local n8n configuration and credential storage.
- Only the indicators required for each lookup were submitted to the external services.
- No file was uploaded to VirusTotal; the workflow queried an existing report by hash.
- The public IP and SHA-256 hash were clearly labeled as controlled test inputs.
- Screenshots were reviewed for exposed secrets before publication.

Public threat-intelligence services should not be used to submit confidential indicators or files without confirming the organization's data-handling requirements and the provider's terms.

## Validation

The following checks confirmed that the enrichment path worked as intended:

| Validation check | Result |
|---|---|
| AbuseIPDB request completed | Passed |
| AbuseIPDB response returned reputation fields | Passed |
| VirusTotal file-report request completed | Passed |
| VirusTotal response returned analysis statistics | Passed |
| Results were available to the OpenAI agent | Passed |
| Simulated indicators were labeled in the final report | Passed |
| Original Splunk evidence remained identifiable | Passed |
| Published examples contained no live API keys | Passed |

## Limitations

- The public IP and SHA-256 hash were controlled inputs rather than indicators observed in the original alert.
- The proof-of-concept prompt supplied the test indicators directly instead of extracting them dynamically from every alert type.
- Private, reserved, and loopback IP addresses were not automatically filtered before enrichment.
- External API availability, quotas, rate limits, and data freshness can affect the returned context.
- Reputation results can contain false positives or outdated reporting and require analyst validation.
- AI-generated interpretations may be incomplete or incorrect and must be compared with the source telemetry.

## Future Improvements

- Add n8n logic to classify IP addresses and skip private, reserved, and loopback ranges.
- Extract public IPs and hashes dynamically from supported Splunk alert fields.
- Call VirusTotal only when a supported hash is directly observed in the alert evidence.
- Return enrichment results through a defined JSON schema for reliable downstream mapping.
- Add API timeout handling, retries, rate-limit handling, and failure notifications.
- Cache recent reputation results to reduce duplicate API requests.
- Record enrichment timestamps and source references for analyst verification.

## Sanitized Workflow Configuration

The complete sanitized n8n workflow export is available here:

[`workflow-exports/soc-automation-2-sanitized.json`](workflow-exports/soc-automation-2-sanitized.json)

The export contains the enrichment node configuration, n8n expressions, AI-tool connections, and request structure. Credential identifiers, API keys, private webhook paths, and sensitive configuration values were removed before publication.

## References

- [AbuseIPDB API v2 documentation](https://docs.abuseipdb.com/)
- [VirusTotal API v3 file-report endpoint](https://docs.virustotal.com/reference/file-info)
- [Project walkthrough](Walkthrough.md)
- [Testing and validation](Testing-Validation.md)

