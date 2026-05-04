# Vulnerability 07: Over-Privileged Functions

## Summary

This lesson demonstrates an Over-Privileged Functions vulnerability in DVSA. A Lambda execution role had broader IAM permissions than required, including full access policies for Lambda and DynamoDB.

## Impact

If the Lambda function was misused or compromised, the excessive permissions could allow access to actions outside the function’s intended purpose. This could include reading, modifying, or deleting DynamoDB data, or interacting with Lambda functions unnecessarily.

## Affected Components

- AWS Lambda function execution role
- IAM role permissions
- `AWSLambda_FullAccess` managed policy
- `AmazonDynamoDBFullAccess` managed policy
- DynamoDB backend resources

## Root Cause

The Lambda function was assigned broad managed IAM policies instead of a limited least-privilege policy. Permissions such as `dynamodb:*` on `Resource: "*"` gave the function more access than it needed.

## Fix

The fix applied the principle of least privilege to the Lambda execution role.

The fix included:

- Removing overly broad managed policies
- Removing `AWSLambda_FullAccess`
- Removing `AmazonDynamoDBFullAccess`
- Creating a restricted custom policy
- Allowing only required DynamoDB actions such as `GetItem` and `Query`
- Scoping access to a specific DynamoDB table

## Verification

After remediation:

- The role no longer had broad FullAccess policies
- High-risk actions such as `DeleteTable`, `DeleteItem`, and unrestricted DynamoDB access were denied
- Required read actions such as `GetItem` and `Query` remained allowed
- The Lambda function retained only the permissions needed for its intended behavior

## Folder Contents

- `Report/` — Lesson 7 written report
- `Figures/` — screenshots used as IAM policy and verification evidence
- `Code/` — least-privilege IAM policy/configuration change explanation

## Security Note

AWS account identifiers, ARNs, role names, and other sensitive environment-specific values may be redacted in screenshots and documentation.
