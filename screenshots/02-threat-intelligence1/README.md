# Phase 2 — n8n Orchestration

## Goal

Receive the Splunk alert, validate required fields, enrich supported indicators, request structured AI analysis, and route the result to analyst-facing systems.

## Recommended Workflow Order

1. Webhook receives the Splunk payload.
2. Validation rejects missing or malformed required fields.
3. Normalization creates one consistent alert object.
4. Indicator extraction identifies supported hashes, IPs, URLs, or domains.
5. VirusTotal enrichment runs only when an indicator exists.
6. OpenAI returns output matching the required schema.
7. A validation node checks the structured response.
8. Slack receives a concise notification.
9. DFIR-IRIS receives a case when the configured threshold is met.
10. An error branch records failures without hiding the original alert.

## GitHub Evidence

- Sanitized n8n workflow export
- Full workflow screenshot
- Successful execution screenshot
- Failed-input/error-branch test
- Short explanation of each node

Never export active credentials with the workflow.
