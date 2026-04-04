---
date: '2026-04-02T21:30:00+02:00'
draft: false
title: 'The JWT Confusion: Why You Are Probably Talking About JWS'
tags: ["jwt", "jws", "jwe", "jwk", "jwa"]
toc: true
author: theddy
---

## History
The JSON Object Signing and Encryption (JOSE) Working Group was formed in 2011 to standardize signing, encryption, and key representation mechanisms for JSON-based data.

This effort resulted in a family of specifications published around 2015 as RFCs:

1. [JWS (RFC 7515)](https://datatracker.ietf.org/doc/html/rfc7515) - JSON Web Signature
2. [JWE (RFC 7516)](https://datatracker.ietf.org/doc/html/rfc7516) - JSON Web Encryption
3. [JWK (RFC 7517)](https://datatracker.ietf.org/doc/html/rfc7517) - JSON Web Key
4. [JWA (RFC 7518)](https://datatracker.ietf.org/doc/html/rfc7518) - JSON Web Algorithms

This led to the creation of **JWT (JSON Web Token)**, standardized as [RFC 7519](https://datatracker.ietf.org/doc/html/rfc7519) in 2015. BTW, JWT is actually pronounced "jot", not J-W-T.

## The JWT Family
The JOSE - JSON Object Signing and Encryption, provide IETF standards designed to securely transfer claims between parties. JOSE defines the building blocks:
	- JWS - signed messages
	- JWE - encrypted messages
	- JWK - keys to use
	- JWA - allowed algorithms
JWT is built _on top of_ JOSE standards. It uses JWS and JWE. 

## Glossary

JWS - `JSON Web Signatures`, which provide compact serialization. This looks like a string in three parts separated with dots.

```
base64url( Header ). base64url( Payload ). base64url( Signature )
```

The signature is created by cryptographically signing the base64url-encoded header and payload using a key (either symmetric or asymmetric). Signatures provides **integrity**/tamper resistance.

JWE - `JSON Web Encryption`, a string in five parts separated with dots.

```
base64url( Header ). base64url( Encrypted Key ). base64url( InitializationVector ). base64url( Ciphertext ). base64url( AuthenticationTag )
```

Those provide **confidentiality**, ensuring only intended recipients can read it.

JWA - `JSON Web Algorithms` - provide information on which algorithms to use for "sealing" the payloads.
JWK - `JSON Web Key` - represent keys as JSON objects. 

```json
{
  "keys": [
    {
      "kty": "RSA",
      "use": "sig",
      "n": "sySYvu...",
      "e": "AQAB",
      "kid": "MDc...",
    },
    {
      "kty": "RSA",
      "use": "sig",
```
Those JSON specs describe the key with different fields - kty (key type), use (public key use), kid (key id).

JWKS - `JSON Web Key Set`. Standard way to wrap multiple JWKs into one JSON object. It is a JSON object with a keys member.

JWT - `JSON Web Token` - standartized compact representation of information using claims. 

## What is JWT

A JSON Web Token looks like this:
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWV9.
MzXXc7P0HPPzPLlXY21kGLBKTdYzq3bdhz76hn8fBYk
```

This is a compact, printable representation of a series of claims, along with a signature to verify its authenticity.

Meaning:
```json
Header:
{
  "alg": "HS256",
  "typ": "JWT"
}
Payload:
{
  "sub": "1234567890",
  "name": "John Doe",
  "admin": true
}
```

JWT must always use compact serialization. JWT is a type of token which is a structure token. Again, it is a format for representation of information in the form of claims. 

Some of JWT's applications include:
- Authentication
- Authorization
- Federated identity
- Client-side sessions ("stateless" sessions)
- Client-side secrets

## JWT Claims
JWT ("jot") is a set of claims. Claims are definitions about a certain party or object.

Some are part of the spec and other are user defined. Spec:
- iss (issuer) - Principal who issued the JWT. Its interpretation is application specific (there is no central authority managing issuers).
- sub (subject) - Subject of the JWT (the user). It uniquely identifies the party that this JWT carries information about
- aud (audience) - recipients for which the JWT is intended. Array of strings that uniquely identify the intended recipients of this JWT. In other words, when this claim is present, the party reading the data in this JWT must find itself in the aud claim or disregard the data contained in the JWT.
- exp (expiration time) - Time after which the JWT expires. A number representing a specific date and time in the format "seconds since epoch".
- nbf (not before) - The token is valid after this timestamp.
- iat (issued at) - Time when the token was issued. 
- jti (JTW ID) - Unique ID for this JWT. 

## JWS
When a JWT is signed, it is represented as a JWS. The signature is part of the JWS structure. The signature is the third element. This element appears after the last dot (.) in the compact serialization form. It provides the data integrity.

## JWT vs JWS
Are they the same? 

JWT is a set of claims (the payload). A JWS is a way to protect that payload using a digital signature. For JWT to be used, it must be transmitted securely. A JWT is encoded as a JWS when it is signed.

## Conclusion
JWT and JWS are closely related. A JWT defines what is being transmitted - a set of claims. Whereas a JWS defines how those claims are protected - through a digital signature.

JWT term is often used interchangeable as JWS and this is considered correct. Most of the time, when people say "JWT", they actually mean a signed JWS - and knowing that difference matters.

## Refs
https://www.jwt.io/
https://datatracker.ietf.org/wg/jose/history/
https://datatracker.ietf.org/doc/html/rfc7515
https://auth0.com/blog/demystifying-jose-jwt-family/
"The JWT Handbook" by Sebastián E. Peyrott
