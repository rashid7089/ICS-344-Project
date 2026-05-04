# Lesson 8 — `order_billing.py` Code Change

## Resource Changed

AWS Lambda function: `DVSA-ORDER-BILLING`  
File: `order_billing.py`

---

## Before Fix

Before remediation, billing was allowed when the order status was less than `120`.

```python
status = int(json.dumps(response["Item"]['orderStatus'], cls=DecimalEncoder))
if status < 120:
```

---

## Problem

This created a workflow gap. Billing could start while the order was still editable, which allowed a concurrent update request to change the item quantity while payment was being processed.

As a result, the final DynamoDB record could show a higher item quantity while the total amount still reflected the original smaller quantity.

---

## After Fix — Start Billing Only for Open Orders

The billing function was changed so billing starts only when the order is still open.

```python
status = int(json.dumps(response["Item"]['orderStatus'], cls=DecimalEncoder))
if status == 100:
```

---

## After Fix — Lock Order Immediately

As soon as billing starts, the order is locked by changing `orderStatus` from `100` to `200`.

```python
try:
    table.update_item(
        Key={
            "orderId": orderId,
            "userId": userId
        },
        UpdateExpression="SET orderStatus = :processing",
        ConditionExpression="orderStatus = :open",
        ExpressionAttributeValues={
            ":processing": 200,
            ":open": 100
        }
    )
except Exception as e:
    print("Order lock failed:", str(e))
    res = {"status": "err", "msg": "order already locked"}
    return res
```

---

## After Fix — Protect Final Payment Update

The final payment update was also protected with a DynamoDB condition expression.

```python
expression_attributes = {
    ':orderstatus': res['status'],
    ':paymentTS': ts,
    ':total': Decimal(cartTotal).quantize(TWOPLACES),
    ':token': res['confirmation_token'],
    ':processing': 200
}
```

```python
response = table.update_item(
    Key=key,
    UpdateExpression=update_expression,
    ConditionExpression="orderStatus = :processing",
    ExpressionAttributeValues=expression_attributes
)
```

---

## What Changed

- Billing now starts only when `orderStatus == 100`.
- Billing immediately locks the order by setting `orderStatus = 200`.
- DynamoDB enforces the lock using `ConditionExpression="orderStatus = :open"`.
- The final payment update only succeeds if the order is still in processing state.
- This reduces the chance of concurrent update requests creating an inconsistent order.

---

## Why This Fix Works

Before the fix, billing and update operations could overlap. The billing function calculated the total while the update function could still change the item list.

After the fix, the order is locked as soon as billing starts. Once the order moves to `orderStatus = 200`, update requests should no longer be allowed to modify the item list. The DynamoDB condition expression also enforces this rule at the database layer.
