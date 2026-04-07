---
date: '2026-04-07T18:16:25+03:00'
draft: true
title: 'Understanding OIDC Part 2: Deep dive'
tags: ["oauth", "oidc"]

---


How do we ususally learn technologies? We check the documentation, understand it, implement something and explain your thought process. Repeat the process to make your understanding deeper. Starting with it we see the following diagram:

(TODO: add diagram)

The end-goal of this sequience of articles is to understand OIDC, why do we use it, what problems does it solve, why not use OAuth 2.0. 

But first, in order to understand OIDC, we need to have an understanding of its fountaion (underpinnings in the image above). We have covered the JW(*) part in a previous article (TODO: Add link). In this one we will focus on the OAuth 2.0.


## History
### Identity use-cases (pre-2010)
- Simple login (forms and cookies)
- Single sign-on across sites (SAML)

### Identity use-cases (pre-2014)
- Simple login (OAuth 2.0) *Authentication*
- Single sign-on across sites (OAuth 2.0) *Authentication*
- Mobile app login (OAuth 2.0) *Authentication*
- **Delegated authorization**  (OAuth 2.0) *==Authorization==*
