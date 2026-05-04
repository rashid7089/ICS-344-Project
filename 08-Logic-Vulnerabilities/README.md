# Vulnerability 08: Logic Vulnerabilities

## Summary

This lesson demonstrates a logic vulnerability in the DVSA order-processing workflow. The application allowed billing and order-update operations to be processed in an unsafe sequence, which caused the final order state to become inconsistent with the amount charged.

## Impact

A user could pay for the original item quantity while the final order stored a larger quantity. In the exploit demonstration, the billing request charged the one-item amount, but the final DynamoDB record stored item quantity `10`.

## Affected Components

- `/dvsa/order` API endpoint
- `DVSA-ORDER-BILLING` Lambda function
- `DVSA-ORDER-UPDATE` Lambda function
- `DVSA-ORDERS-DB` DynamoDB table

## Environment

DVSA frontend URL used during testing:

```text
http://dvsa-website-962415228810-us-east-1.s3-website.us-east-1.amazonaws.com/
```

Backend API endpoint used during testing:

```text
https://n1pq1eb7kd.execute-api.us-east-1.amazonaws.com/dvsa/order
```

## Root Cause

The backend did not enforce strict workflow state transitions. Billing did not lock the order immediately when payment started, and the update function allowed order modifications during the billing process. This allowed billing and update requests to overlap and create an inconsistent final order state.

## Fix

The fix was applied in the backend Lambda functions responsible for billing and order updates.

### `order_billing.py`

- Billing now starts only when `orderStatus == 100`
- Billing immediately locks the order by setting `orderStatus = 200`
- DynamoDB `ConditionExpression` checks were added to enforce the expected order state

### `update_order.py`

- Updates are allowed only when `orderStatus == 100`
- Update requests are rejected after billing starts
- DynamoDB `ConditionExpression="orderStatus = :open"` was added to prevent concurrent unsafe updates

## Verification

After remediation:

- Billing still succeeded normally
- The concurrent update request was rejected
- The inconsistent state no longer occurred
- The final order quantity and charged amount remained consistent

## Folder Contents

- `Report/` — Lesson 8 written report
- `Figures/` — screenshots used as exploit, code, and verification evidence
- `Code/` — before/after code change explanations for `order_billing.py` and `update_order.py`

## Security Note

Authorization tokens, personal identifiers, and other sensitive values were redacted from screenshots and documentation.
