# Vulnerability 06: Denial of Service

## Summary

This lesson demonstrates a Denial of Service vulnerability in DVSA. The backend accepted and processed repeated requests without rate limiting or throttling, which could allow an attacker to overload the serverless application.

## Impact

An attacker could send many requests quickly, causing excessive Lambda invocations and backend resource usage. This could slow down the service, increase AWS costs, or make the application unavailable for legitimate users.

## Affected Components

- `/dvsa/order` API endpoint
- `DVSA-ORDER-MANAGER` Lambda function
- API Gateway request handling
- Backend Lambda invocation flow

## Root Cause

The system did not enforce request rate limits or throttling. Each incoming request was processed without checking request frequency or volume, and backend logic could trigger additional Lambda operations for every request.

## Fix

Rate limiting and request control were added to prevent unlimited request processing.

The fix included:

- Adding request control logic in the Lambda handler
- Returning HTTP `429 Too Many Requests` for excessive requests
- Applying or recommending API Gateway throttling
- Reducing unnecessary backend processing during high request volume
- Using monitoring/logging to detect abnormal traffic patterns

## Verification

After remediation:

- Normal requests within the allowed limit still worked
- Excessive repeated requests were rejected
- The system returned HTTP `429 Too Many Requests`
- Backend resources were protected from unlimited request processing

## Folder Contents

- `Report/` — Lesson 6 written report
- `Figures/` — screenshots used as exploit, code, and verification evidence
- `Code/` — before/after code and configuration change explanation

## Security Note

Authorization tokens, API keys, account identifiers, and other sensitive values were redacted from screenshots and documentation.
