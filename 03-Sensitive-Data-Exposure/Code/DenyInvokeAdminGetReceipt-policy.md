# Lesson 3 — Sensitive Information Disclosure Config Change

## Resource Changed

AWS IAM role: `serverlessrepo-OWASP-DVSA-OrderManagerFunctionRole-xan2tL8HWALI`  
Inline policy added: `DenyInvokeAdminGetReceipt`

---

## Before Fix

Before remediation, the `DVSA-ORDER-MANAGER` execution role had the AWS managed policy `AWSLambdaRole`.

That policy allowed the Order Manager Lambda function to invoke Lambda functions on all resources:

```json
{
  "Effect": "Allow",
  "Action": [
    "lambda:InvokeFunction"
  ],
  "Resource": [
    "*"
  ]
}
```

---

## Problem

The permission was too broad because it allowed code running inside `DVSA-ORDER-MANAGER` to invoke privileged Lambda functions.

During the exploit, injected backend code invoked:

```text
DVSA-ADMIN-GET-RECEIPT
```

That privileged function generated a signed S3 receipt archive URL, which disclosed sensitive receipt data.

---

## After Fix

An inline explicit Deny policy was added to the `DVSA-ORDER-MANAGER` execution role.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyInvokeAdminGetReceipt",
      "Effect": "Deny",
      "Action": "lambda:InvokeFunction",
      "Resource": "arn:aws:lambda:us-east-1:[ACCOUNT-ID]:function:DVSA-ADMIN-GET-RECEIPT"
    }
  ]
}
```

---

## What Changed

- Added an inline IAM policy named `DenyInvokeAdminGetReceipt`.
- The policy explicitly denies `lambda:InvokeFunction` on `DVSA-ADMIN-GET-RECEIPT`.
- The AWS managed `AWSLambdaRole` policy was not edited directly because it is AWS managed.
- The explicit Deny overrides the existing broad Allow.
- `DVSA-ORDER-MANAGER` can no longer invoke the privileged receipt-generation Lambda function.

---

## Why This Fix Works

Before the fix, injected code running inside `DVSA-ORDER-MANAGER` could call `DVSA-ADMIN-GET-RECEIPT` because the role allowed `lambda:InvokeFunction` on all resources.

After the fix, IAM explicitly denies invocation of `DVSA-ADMIN-GET-RECEIPT`. In AWS IAM, an explicit Deny overrides any Allow. This blocks the sensitive receipt-disclosure path while keeping the normal DVSA order workflow available.

---

## Post-Fix Result

After applying the inline Deny policy, the same malicious payload no longer produced a successful webhook response containing:

```json
{
  "status": "ok",
  "download_url": "..."
}
```

Instead, the request failed or did not generate a new signed receipt download URL.
