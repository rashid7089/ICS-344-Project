# Lesson 2 — Broken Authentication Code Change

## Resource Changed

AWS Lambda function: `DVSA-ORDER-MANAGER`  
File: `order-manager.js`

---

## Before Fix

The backend decoded the JWT payload and trusted the `username` claim without verifying the JWT signature.

```js
var auth_data = jose.util.base64url.decode(token_sections[1]);
var token = JSON.parse(auth_data);
var user = token.username;
```

# Problem

JWT payloads are Base64URL-encoded, not encrypted. Because the backend did not verify the token signature, an attacker could modify identity fields such as username or sub and impersonate another user.

The security mistake was trusting client-provided JWT claims after decoding them, instead of cryptographically verifying the JWT first.

## Helper Code Added

The fix added helper logic near the top of order-manager.js to fetch Cognito JWKS, cache the key store, and verify JWT signatures before trusting claims.

```js
const https = require('https');

let _jwksCache = { keystore: null, fetchedAt: 0 };

async function getCognitoKeystore() {
  const region = process.env.AWS_REGION;
  const userPoolId = process.env.userpoolid;
  const jwksUrl = `https://cognito-idp.${region}.amazonaws.com/${userPoolId}/.well-known/jwks.json`;

  const jwks = await fetchJson(jwksUrl);
  const keystore = await jose.JWK.asKeyStore(jwks);

  _jwksCache = { keystore, fetchedAt: Date.now() };
  return keystore;
}

async function verifyCognitoJwt(jwt) {
  const keystore = await getCognitoKeystore();
  const result = await jose.JWS.createVerify(keystore).verify(jwt);
  const claims = JSON.parse(result.payload.toString("utf8"));

  return claims;
}
```
The full implementation also validates issuer, expiration, and token use before returning the claims.

## After Fix

The backend now extracts the Bearer token from the Authorization header, verifies it using Cognito JWT verification, and only then trusts identity claims.

```js
var auth_header = (headers.Authorization || headers.authorization || "");
var jwt = auth_header.replace(/^Bearer\s+/i, "").trim();

if (!jwt) {
  return callback(null, resp(401, { status: "err", msg: "missing authorization" }));
}

verifyCognitoJwt(jwt).then((claims) => {
  var user = claims.username || claims["cognito:username"] || claims.sub;

  if (!user) {
    return callback(null, resp(401, { status: "err", msg: "missing subject" }));
  }

  var isAdmin = false;
  ```

## Invalid Token Handling

If JWT verification fails, the backend rejects the request instead of trusting the modified token.

```js
}).catch((e) => {
  console.log("JWT verify failed:", e);
  return callback(null, resp(401, { status: "err", msg: "invalid token" }));
});
```

## What Changed
- Added JWT verification using Cognito public keys.
- Added JWKS retrieval and key-store caching.
- Removed direct trust in decoded JWT payload data.
- Verified token signature before reading username, cognito:username, or sub.
- Added validation for issuer, expiration, and token use.
- Added error handling to reject invalid or tampered tokens.

## Why This Fix Works

The backend now verifies the JWT cryptographically before trusting identity claims. If an attacker modifies the JWT payload, the original signature no longer matches the modified content. The verification step fails, and the backend returns invalid token.

This prevents forged tokens from being used to impersonate other users while still allowing valid Cognito tokens to work normally.
