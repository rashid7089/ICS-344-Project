# Lesson 8 — `update_order.py` Code Change

## Resource Changed

AWS Lambda function: `DVSA-ORDER-UPDATE`  
File: `update_order.py`

---

## Before Fix

Before remediation, the update function only blocked changes when `orderStatus > 110`.

```python
if response["Item"]["orderStatus"] > 110:
    res = { "status": "err", "msg": "order already paid" }
    return res
```

---

## Problem

This allowed the order to remain editable during part of the billing workflow. A concurrent update request could modify the item list while billing was being processed, creating an inconsistent final state.

For example, the user could be charged for one item while DynamoDB stored quantity `10`.

---

## After Fix — Allow Updates Only for Open Orders

The update function was changed so updates are allowed only when the order is still open.

```python
if response["Item"]["orderStatus"] != 100:
    res = { "status": "err", "msg": "Order cannot be modified after billing has started" }
    return res
```

---

## After Fix — Enforce State Check in DynamoDB

A DynamoDB condition expression was added to the update operation.

```python
response = table.update_item(
    Key={"orderId": orderId, "userId": userId},
    UpdateExpression=update_expr,
    ConditionExpression="orderStatus = :open",
    ExpressionAttributeValues={
        ':itemList': itemList,
        ':open': 100
    }
)
```

---

## What Changed

- Updates are now allowed only when `orderStatus == 100`.
- Update requests are rejected after billing starts.
- DynamoDB enforces the open-order requirement using `ConditionExpression="orderStatus = :open"`.
- This prevents concurrent update requests from changing the item list during billing.

---

## Why This Fix Works

Before the fix, an update request could still modify the order during the billing process.

After the fix, once billing starts and the order is moved out of the open state, update requests are rejected. The DynamoDB condition expression also protects against timing issues by making the database reject updates if the order status changes during concurrent execution.
