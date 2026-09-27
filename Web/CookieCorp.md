# SunshineCTF 2026 — CookieCorp

## Challenge

**Category:** Web  
**Challenge:** CookieCorp  
**Flag format:** `sun{...}`

## Overview

CookieCorp is a custom-cookie fabrication service. A recipe is submitted to a robotic Quality Inspector for review, and the application uses cookies to track the session and user role.

The important distinction is that the application protects the `role` cookie with `HttpOnly`, while the browser-side ingredient system can still create cookies. The intended path is to manipulate the browser's cookie jar until the lower-priority protected role cookie is evicted, then create a replacement `role=chief` cookie.

## Recon

The application exposes a review flow where the Quality Inspector visits:

`/review/<id>`

The page writes submitted ingredients into browser cookies using JavaScript:

```javascript
document.cookie = name + '=' + value + '; path=/'
```

This creates a useful cookie-injection primitive.

## Cookie Analysis

Two cookies are especially important:

- `session` — marked `HttpOnly` and `Priority=High`
- `role` — marked `HttpOnly` and uses the default/medium priority

The application later relies on the `role` cookie for authorization.

A direct attempt to create:

```
role=chief
```

does not work because JavaScript cannot directly modify or read an existing `HttpOnly` cookie.

So the problem becomes: **how can the protected `role` cookie be removed without directly modifying it?**

## Cookie Jar Overflow

Chromium limits the number of cookies stored for a domain. When the cookie jar becomes full, cookies can be evicted, and cookie priority affects which cookies are retained.

The application gives us a convenient way to create many cookies through the ingredient mechanism.

By submitting a large number of unique ingredients, the browser's cookie jar can be filled.

The important priority relationship is:

```
session  -> High priority
role     -> Medium/default priority
dummy cookies -> lower/equal priority
```

The overflow causes the lower-priority `role` cookie to be evicted while the high-priority `session` cookie survives.

This is not an `HttpOnly` bypass in the strict sense. The original `HttpOnly` cookie is not read or modified by JavaScript. It is evicted by cookie-storage behavior, after which a new cookie with the same name can be created.

## Exploit

The attack flow is:

1. Submit a recipe containing a large number of unique ingredient names.
2. Make the Quality Inspector visit the generated review page.
3. The browser creates hundreds of cookies.
4. The cookie jar reaches its per-domain limit.
5. The lower-priority `role` cookie is evicted.
6. The attacker creates a new cookie:
   ```
   role=chief
   ```
7. The application now receives the attacker-controlled `role=chief` cookie.
8. Requesting the Chief-only functionality grants the Chief's Golden Seal and returns the flag.

A minimal conceptual payload is:

```text
dummy1=x
dummy2=x
dummy3=x
...
dummy200=x
role=chief
```

The exact number needed depends on browser cookie handling, but Chromium's per-domain cookie limit makes the overflow practical.

## Why the Exploit Works

The vulnerability is the combination of:

- client-controlled cookie creation
- authorization based on a cookie value
- different cookie priorities
- browser cookie-jar eviction behavior
- the ability to submit enough unique cookie names to trigger eviction

The important point is that `HttpOnly` protects a cookie from non-HTTP script access; it does not make the cookie impossible for the browser to evict.

## Attack Chain

```text
Ingredient input
      |
      v
document.cookie
      |
      v
Many attacker-controlled cookies
      |
      v
Cookie jar reaches capacity
      |
      v
Lower-priority role cookie evicted
      |
      v
Create role=chief
      |
      v
Chief authorization
      |
      v
Golden Seal
      |
      v
FLAG
```

## Classification

This challenge demonstrates a combination of:

- Cookie jar overflow / cookie eviction
- Client-side trust in authorization cookies
- Broken access control
- Business logic abuse
- Browser cookie priority behavior

There is no need to associate the challenge with a specific CVE.

## Important Detail

This should **not** be described as simply bypassing `HttpOnly`.

The original `HttpOnly` cookie remains protected from JavaScript access. The attack instead abuses cookie-storage behavior to remove the original cookie and then establishes a replacement cookie with the desired value.

## Flag

```
sun{c00kie_jar_0verfl0w_ev1cts_the_chief}
```

## Takeaway

The challenge is a good example of why security properties of individual browser mechanisms cannot be considered in isolation.

`HttpOnly` can prevent JavaScript from directly modifying a sensitive cookie, but an application that treats a client-controlled cookie as an authorization boundary still needs to account for cookie replacement, duplicate-name behavior, eviction, and browser-specific storage rules.
