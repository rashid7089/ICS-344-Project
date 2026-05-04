# Lesson 6 — Denial of Service Code / Config Change

## Resource Changed

AWS Lambda function: `DVSA-ORDER-MANAGER`  
File: `order-manager.js`

API Gateway route: `/dvsa/order`

---

## Before Fix

Before remediation, the backend processed incoming requests without rate limiting or request throttling. Each request was accepted and could trigger additional backend operations.

```js
exports.handler = (event, context, callback) => {
    var req = JSON.parse(event.body);

    const lambda_client = new LambdaClient();
    const command = new InvokeCommand(params);
    lambda_client.send(command);
};
```

---

## Problem

The function accepted repeated requests without checking request volume or frequency. This made the application vulnerable to Denial of Service because an attacker could send many requests quickly and force the backend to keep processing them.

The risk was increased because each request could trigger additional backend work, such as invoking another Lambda function. This can increase resource consumption, slow down the service, increase cost, or make the application unavailable for legitimate users.

---

## After Fix — Lambda Request Limit

A basic request limit was added so excessive requests are rejected instead of being processed indefinitely.

```js
let requestCount = 0;
const MAX_REQUESTS = 10;

exports.handler = (event, context, callback) => {
    requestCount++;

    if (requestCount > MAX_REQUESTS) {
        return callback(null, {
            statusCode: 429,
            body: JSON.stringify({ message: "Too many requests" })
        });
    }

    var req = JSON.parse(event.body);

    const lambda_client = new LambdaClient();
    const command = new InvokeCommand(params);
    lambda_client.send(command);
};
```

---

## After Fix — API Gateway Throttling

Rate limiting should also be enforced at the API Gateway layer so excessive traffic is blocked before it reaches Lambda.

Example configuration concept:

```text
API Gateway throttling:
- Rate limit: restrict requests per second
- Burst limit: restrict short traffic spikes
- Return HTTP 429 Too Many Requests when exceeded
```

---

## What Changed

- Added request control to reduce unlimited request processing.
- Added a maximum request threshold.
- Excessive requests return HTTP `429 Too Many Requests`.
- API Gateway throttling is recommended to block abuse before Lambda execution.
- Reduced unnecessary backend Lambda invocations during high traffic.

---

## Why This Fix Works

Before the fix, every request was accepted and processed, allowing a user to overload backend resources by sending many requests quickly.

After the fix, excessive requests are rejected once the allowed threshold is exceeded. API Gateway throttling adds another layer of protection by limiting traffic before it reaches the Lambda function. This helps preserve availability, reduce resource exhaustion, and protect normal users.

---

## Post-Fix Result

After applying rate limiting, repeated request testing showed that normal requests still worked within the allowed limit, while excessive requests were rejected with:

```text
HTTP 429 Too Many Requests
```

This confirms that the system no longer processes unlimited requests without restriction.
