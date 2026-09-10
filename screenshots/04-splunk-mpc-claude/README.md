# Bonus 1 — VirusTotal Enrichment

## Goal

Enrich supported indicators extracted from the alert and return concise reputation context to the triage workflow.

## Evidence to Capture

- Indicator-extraction logic
- API request with the key hidden
- Parsed response fields
- No-indicator branch
- Rate-limit or API-failure branch
- Before-and-after triage comparison

## Safety Rule

Do not upload confidential files or unrestricted telemetry. Query only the indicators required for the authorized lab investigation.
