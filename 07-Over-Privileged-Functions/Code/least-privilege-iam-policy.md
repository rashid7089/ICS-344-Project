# Lesson 7 — Over-Privileged Functions Config Change

## Resource Changed

AWS IAM role attached to the Lambda function  
Example role: `serverlessrepo-OWASP-DVSA-AdminGetOrdersRole`

---

## Before Fix

Before remediation, the Lambda execution role had overly broad AWS managed policies attached, such as:

```text
AWSLambda_FullAccess
AmazonDynamoDBFullAccess
```

These policies allowed the function to perform many actions that were not required for its intended purpose.

Example of the risky permission pattern:

```json
{
  "Effect": "Allow",
  "Action": [
    "dynamodb:*"
  ],
  "Resource": [
    "*"
  ]
}
```

---

## Problem

The Lambda function was over-privileged because it had more permissions than it needed. Instead of allowing only the required read operations, the role allowed broad access to AWS Lambda and DynamoDB resources.

This violated the principle of least privilege. If the function was misused or compromised, the excessive permissions could be used to read, modify, or delete data and interact with resources outside the function’s intended scope.

---

## After Fix — Least Privilege Policy

The broad managed policies were removed and replaced with a custom least-privilege policy.

Example restricted policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:Query"
      ],
      "Resource": "arn:aws:dynamodb:[REGION]:[ACCOUNT-ID]:table/[TABLE-NAME]"
    }
  ]
}
```

---

## What Changed

- Removed `AWSLambda_FullAccess`.
- Removed `AmazonDynamoDBFullAccess`.
- Replaced broad permissions with a custom least-privilege policy.
- Limited DynamoDB permissions to only required read actions.
- Scoped access to a specific DynamoDB table instead of all resources.
- Removed unnecessary high-risk actions such as table deletion or unrestricted modification.

---

## Why This Fix Works

Before the fix, the Lambda function could perform unnecessary and risky AWS actions because its IAM role had full-access managed policies.

After the fix, the function only has the permissions required for its intended task. Even if the Lambda function is misused, the potential damage is reduced because the role cannot perform broad DynamoDB or Lambda actions.

This follows the principle of least privilege and reduces the attack surface.

---

## Post-Fix Result

After applying the restricted policy, IAM Policy Simulator showed that high-risk actions such as `DeleteTable`, `DeleteItem`, or unrestricted scans were denied, while required read actions such as `GetItem` and `Query` remained allowed.

This confirms that the excessive permissions were removed while legitimate function behavior was preserved.
