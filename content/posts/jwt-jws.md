---
date: '2026-04-02T21:30:00+02:00'
draft: false
title: 'The JWT Confusion: Why You Are Probably Talking About JWS'
tags: ["jwt", "jws", "jwe", "jwk", "jwa"]
toc: true
author: theddy
---

When you start to familiarize yourself with JSON Web Tokens (JWT), you often encounter a number of related acronyms - JWS, JWA, JWK, etc, which can feel overwhelming at first. It is not always clear which concept comes "first", how they relate to each other, or what the differences are. In this article, I want to briefly introduce these concepts and clarify the important distinction between JWT and JWS.

## History
The JSON Object Signing and Encryption (JOSE) Working Group was formed in 2011 to standardize signing, encryption, and key representation mechanisms for JSON-based data.

This effort resulted in a family of specifications published in 2015 as RFCs:

1. [JWS (RFC 7515)](https://datatracker.ietf.org/doc/html/rfc7515) - JSON Web Signature
2. [JWE (RFC 7516)](https://datatracker.ietf.org/doc/html/rfc7516) - JSON Web Encryption
3. [JWK (RFC 7517)](https://datatracker.ietf.org/doc/html/rfc7517) - JSON Web Key
4. [JWA (RFC 7518)](https://datatracker.ietf.org/doc/html/rfc7518) - JSON Web Algorithms

This led to the creation of **JWT (JSON Web Token)**, standardized as [RFC 7519](https://datatracker.ietf.org/doc/html/rfc7519) in 2015. BTW, JWT is actually pronounced "jot", not J-W-T.

## The JWT Family
The JOSE (JSON Object Signing and Encryption) working group provide IETF standards designed to securely transfer claims (data) between parties. 

JOSE defines the building blocks:
- JWS - signed messages
- JWE - encrypted messages
- JWK - keys to use
- JWA - allowed algorithms

JWT is built _on top of_ JOSE standards. JWT uses JWS and JWE.

## Glossary

JWS - `JSON Web Signatures`, which provide compact serialization. This looks like a string in three parts separated with dots.

```
base64url( Header ). base64url( Payload ). base64url( Signature )
```

The signature is created by cryptographically signing the base64url-encoded header and payload using a secret signing key (either symmetric or asymmetric). Signatures provides **integrity**/tamper resistance.

JWE - `JSON Web Encryption`, a string in five parts separated with dots.

```
base64url( Header ). base64url( Encrypted Key ). base64url( InitializationVector ). base64url( Ciphertext ). base64url( AuthenticationTag )
```

Those provide **confidentiality**, ensuring only intended recipients can read it.

JWA - `JSON Web Algorithms` - provide information on which algorithms to use for "sealing" the payloads. They define cryptographic algorithms for JWTs.

JWK - `JSON Web Key` - defines a JSON data structure for cryptographic keys.

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
...
  ]
}
```
Those JSON specs describe the key with different fields - "kty" (key type), "use" (public key use), "kid" (key id).

JWKS - `JSON Web Key Set`. Standard way to wrap multiple JWKs into one JSON object. It is a JSON object with a keys member.

JWT - `JSON Web Token` - standartized compact representation of information using claims. 

## What is JWT?

A JSON Web Token looks like this:
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWV9.
MzXXc7P0HPPzPLlXY21kGLBKTdYzq3bdhz76hn8fBYk
```

This is a compact, printable representation of a series of claims, along with a signature to verify its authenticity.

Meaning:
```json
// Header:
{
  "alg": "HS256",
  "typ": "JWT"
}
// Payload:
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
The JWT is a set of claims (data). Claims are definitions about a certain party or object.

Some claims are part of the spec and other are user defined. However, it is worth mentioning that none are required for JWT itself. The spec defines the following:
- "iss" (issuer) - Principal who issued the JWT. Its interpretation is application specific (there is no central authority managing issuers).
- "sub" (subject) - Subject of the JWT (the user). It uniquely identifies the party that this JWT carries information about
- "aud" (audience) - recipients for which the JWT is intended. Array of strings that uniquely identify the intended recipients of this JWT. In other words, when this claim is present, the party reading the data in this JWT must find itself in the aud claim or disregard the data contained in the JWT.
- "exp" (expiration time) - Time after which the JWT expires. A number representing a specific date and time in the format "seconds since epoch".
- "nbf" (not before) - The token is valid after this timestamp.
- "iat" (issued at) - Time when the token was issued. 
- "jti" (JTW ID) - Unique ID for this JWT. 

## JWS
The last part of the JWT is its signature, which is computed based on the JWT's header, payload, and a secret signing key, using the algorithm specified in the header. When a JWT is signed, it is represented as a JWS. The signature is part of the JWS structure. The signature is the third element. This element appears after the last dot (.) in the compact serialization form. It provides the data integrity - if the data in header, payload or signature is changed, then the signature itself won't match, therefore enabling the detection of manipulation.

## JWT vs JWS
Are they the same? 

JWT itself does not require signing or encryption - it is just the format. JWT is a set of claims (the payload). A JWS is a way to protect that payload using a digital signature. For JWT to be used, it should be transmitted securely. A JWT is encoded as a JWS when it is signed.

## Conclusion
JWT and JWS are closely related. A JWT defines what is being transmitted - a set of claims. Whereas a JWS defines how those claims are protected - through a digital signature.

```
JWT (format) + JWS (signature) = signed JWT. 
```

The term "JWT" is often used interchangeably with JWS in practice, and this is generally accepted. Most of the time, when people say "JWT", they are actually referring to a signed token (JWS). However, this is technically imprecise, since JWT is only the data format, while JWS defines how that data is signed. That is why I want to emphasize that understanding this distinction is important.


## Refs
- https://www.jwt.io/
- https://datatracker.ietf.org/wg/jose/history/
- https://datatracker.ietf.org/doc/html/rfc7515
- https://auth0.com/blog/demystifying-jose-jwt-family/
- "The JWT Handbook" by Sebastián E. Peyrott
