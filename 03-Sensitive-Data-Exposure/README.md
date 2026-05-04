# Vulnerability 03: Sensitive Data Exposure

## Summary

This lesson demonstrates a Sensitive Information Disclosure vulnerability in DVSA. A normal authenticated user could trigger backend code execution through the `/dvsa/order` action field and invoke a privileged admin Lambda function.

## Impact

The exploit generated a signed S3 receipt archive URL through `DVSA-ADMIN-GET-RECEIPT`. This allowed unauthorized access to receipt archive data outside the intended admin workflow.

## Affected Components

- `/dvsa/order` API endpoint
- `DVSA-ORDER-MANAGER` Lambda function
- `DVSA-ADMIN-GET-RECEIPT` Lambda function
- Order Manager IAM execution role
- S3 receipts bucket

## Root Cause

The vulnerable path combined unsafe backend execution with overly broad IAM permissions. The `DVSA-ORDER-MANAGER` execution role was allowed to invoke Lambda functions broadly using `lambda:InvokeFunction` on `Resource: "*"`. This allowed injected code running inside the user-facing order function to invoke the privileged receipt-generation function.

## Fix

An inline IAM Deny policy named `DenyInvokeAdminGetReceipt` was added to the `DVSA-ORDER-MANAGER` execution role.

The fix explicitly denies:

- `lambda:InvokeFunction`
- on `DVSA-ADMIN-GET-RECEIPT`

Because explicit Deny overrides Allow in IAM, the Order Manager role can no longer invoke the admin receipt Lambda.

## Verification

After remediation:

- The same malicious payload no longer generated a successful webhook `download_url`
- Postman returned an error instead of sensitive receipt data
- The normal DVSA browsing and order workflow remained available

## Folder Contents

- `Report/` — Lesson 3 written report
- `Figures/` — screenshots used as exploit, IAM configuration, and verification evidence
- `Code/` — IAM policy/configuration change used for remediation

## Security Note

Authorization tokens, signed S3 URLs, AWS credential parameters, webhook URLs, and other sensitive values were redacted from screenshots and documentation.
