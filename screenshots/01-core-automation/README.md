# Phase 1 — Detection

## Goal

Generate a repeatable Splunk alert from Windows 10 and Sysmon telemetry.

## Initial Use Case

Detect `vssadmin.exe` attempting to delete Windows shadow copies.

## Deliverables

- SPL detection query
- Alert name, schedule, time range, and trigger condition
- Sanitized example alert payload
- Screenshot of matching telemetry
- Screenshot of the alert configuration with secrets hidden
- False-positive considerations and tuning notes

## Validation

Document the test command or simulation, event time in UTC, expected event, actual Splunk result, and alert trigger status.

## Evidence Checklist

- [ ] Windows/Sysmon events visible in Splunk
- [ ] Detection returns the expected test event
- [ ] Alert triggers once for the test
- [ ] Payload includes required fields
- [ ] Sensitive values removed from screenshots and examples
