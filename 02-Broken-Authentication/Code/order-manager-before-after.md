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

# Before Fix

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
