# Vulnerability 02: Broken Authentication

## Summary

This lesson demonstrates a Broken Authentication vulnerability in DVSA. The backend decoded JWT payload claims and trusted identity fields such as `username` and `sub` without verifying the JWT signature.

## Impact

An attacker could modify JWT identity claims and impersonate another user. This allowed unauthorized access to protected order data.

## Affected Components

- `/dvsa/order` API endpoint
- `DVSA-ORDER-MANAGER` Lambda function
- Cognito JWT authentication flow
- API Gateway / Lambda backend

## Root Cause

JWT payloads are Base64URL-encoded, not encrypted. The backend decoded the JWT payload and trusted the claims directly without verifying the token signature. Because of this, a modified token could be accepted as valid.

## Fix

The `DVSA-ORDER-MANAGER` Lambda function was updated to verify Cognito JWTs using Cognito public keys before trusting identity claims.

The fix included:

- Fetching Cognito JWKS public keys
- Verifying the JWT signature
- Validating issuer and expiration
- Rejecting invalid or tampered tokens
- Trusting `username`, `cognito:username`, or `sub` only after verification

## Verification

After remediation:

- A valid token still worked normally
- A forged or modified token was rejected with `invalid token`
- The backend no longer accepted tampered JWT payload claims

## Folder Contents

- `Report/` — Lesson 2 written report
- `Figures/` — screenshots used as exploit, code, and verification evidence
- `Code/` — before/after code change explanation for `order-manager.js`

## Security Note

Tokens, Authorization headers, JWT values, user identifiers, and other sensitive data were redacted from screenshots and documentation.
