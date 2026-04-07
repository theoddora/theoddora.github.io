---
date: '2026-04-08T11:16:25+03:00'
draft: true
title: 'Beyond Bearer: Sender-Constrained Tokens with DPoP and mTLS'
tags: ["oauth"]

---

## Access tokens

The OAuth 2.0 RFC 6749 defined the authorization framework and explained how we can obtain access tokens. Additionally, a separate RFC 6750 defined Bearer tokens and how to use them - transmit in HTTP requests. Bearer tokens are a type of access token. Anyone who holds a Bearer token can use it — no cryptographic proof of ownership required. 

Bearer Tokens have over a decade of proven track record, but they have a fundamental weakness: if stolen, the attacker can use them. This is the motivation behind sender-constrained tokens. A client must prove possession of a cryptographic key bound to the token.

In this article, we will unpack the sender-constrained tokens and compare them with the bearer tokens:
- Bearer tokens
- Sender-constrained tokens:
  - DPoP (Demonstrating Proof of Possession) —  - 2023
  - mTLS (Mutual TLS) Certificate-Bound Access Token — RFC 2020

## How DPoP (Demonstrating Proof of Possession) works

The client generates a public/private key pair. 
Private key never leaves the client

```mermaid
sequenceDiagram
    participant Client
    participant AuthorizationServer as Authorization Server
    participant ResourceServer as Resource Server

    Client-)Client: 1. Generate public/private key.
    Client<<-->>AuthorizationServer: 2. Authorization Code Flow (completes).
    Client-)Client: 3. Create DPoP Proof (JWT containing public key).
    Client-->>AuthorizationServer: 4. POST /token with DPoP: {public_key, signature}
    AuthorizationServer-)AuthorizationServer: 5. Verify signature. Embed public key hash in the token ("cnf" claim).
    AuthorizationServer->>Client: 6. Access token (containing public key hash).
    Client-)Client: 7. Create a fresh DPoP Proof signed with private key each time. 
    Client->>ResourceServer: 8. GET /resource Authorization: DPoP token, DPoP: {Fresh signature}
    ResourceServer-)ResourceServer: Check DPoP Proof signature using public key hash in token
    ResourceServer-->>Client: 10. 200 OK
```
