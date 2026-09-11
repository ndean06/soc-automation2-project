# SOC Automation 2.0 Troubleshooting Log

## Purpose

This document records significant configuration and integration issues encountered while building the SOC Automation 2.0 lab. Each issue includes the symptom, investigation, root cause, correction, and validation.

## Issue Summary

| # | Issue | Root Cause | Resolution | Status |
|---:|---|---|---|---|
| 1 | Windows events not appearing in Splunk | Universal Forwarder targeted the Windows endpoint instead of Splunk | Corrected `outputs.conf` and restarted the forwarder | Resolved |
| 2 | Unable to restart SplunkForwarder | PowerShell was not elevated | Opened PowerShell as Administrator | Resolved |
| 3 | Docker Compose configuration error | Incorrect underscore before an environment variable | Replaced the underscore with a YAML list hyphen | Resolved |
| 4 | n8n image download appeared stalled | Docker was still downloading image layers | Allowed the image pull to finish without cancelling | Resolved |

## 1. Windows Events Not Appearing in Splunk

### Symptom

The Windows endpoint could communicate with the Splunk VM, but searches in Splunk returned no Windows Event Log data.

### Investigation

| Tool | Command Purpose | Syntax/Example |
|---|---|---|
| Command Prompt | Test basic network connectivity | `ping 192.168.117.164` |
| PowerShell | Verify Splunk receiving port | `Test-NetConnection 192.168.117.164 -Port 9997` |
| PowerShell | Confirm the Universal Forwarder was running | `Get-Service -Name SplunkForwarder` |
| PowerShell | Review the forwarding destination | `Get-Content "C:\Program Files\SplunkUniversalForwarder\etc\system\local\outputs.conf"` |

Network connectivity, port `9997`, and the forwarder service were working. However, `outputs.conf` contained the wrong destination:

```ini
server = 192.168.117.163:9997

### Root Cause

The address `192.168.117.163` belonged to the Windows endpoint. The Universal Forwarder was incorrectly sending events back to the endpoint instead of the Splunk server at `192.168.117.164`.

### Resolution

The destination entries in `outputs.conf` were corrected:

```ini
[tcpout]
defaultGroup = default-autolb-group

[tcpout:default-autolb-group]
server = 192.168.117.164:9997

[tcpout-server://192.168.117.164:9997]