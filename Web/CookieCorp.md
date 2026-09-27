# SunshineCTF 2026 — CookieCorp

**Category:** Web
**Challenge:** CookieCorp
**Target:** `https://tomorrow.web.2026.sunshinectf.games`
**Objective:** Obtain the **Chief's Golden Seal** and retrieve the flag.

---

## 1. Challenge Overview

CookieCorp is a custom-cookie fabrication application.

The challenge description tells us that a baker can:

1. Design a custom cookie recipe.
2. Submit the recipe to the automated **Quality Inspector**.
3. Receive an official seal for an approved recipe.
4. Potentially obtain the much more privileged **Chief's Golden Seal**.

The application explicitly states:

> Only the Chief can award that seal.

That sentence gives us the core authorization requirement:

```text
role = chief
```

The interesting part is that the Quality Inspector is not simply a server-side function. It is a browser-based inspection bot that processes our recipe inside its own browser context.

That distinction becomes the foundation of the exploit.

---

# 2. Reconnaissance

The initial application exposes the following functionality:

```text
/
├── Issue Badge
├── Clock In
├── Bulletin
└── Fabrication / Recipe Review
```

The landing page explains that submitted recipes are loaded directly into the fabrication mixer by the automated Quality Inspector.

This should immediately raise a security question:

```text
Attacker-controlled recipe
            ↓
Automated browser
            ↓
Browser-side processing
            ↓
Security-sensitive action
```

Whenever attacker-controlled data is rendered or executed inside a privileged browser context, we should investigate:

* XSS
* DOM injection
* cookie manipulation
* browser state manipulation
* navigation abuse
* origin trust
* session handling

The most important discovery is that recipe ingredients are converted into browser cookies.

---

# 3. Understanding the Inspector

The Quality Inspector visits:

```text
/review/<recipe-id>
```

During the review process, each ingredient is processed approximately as:

```javascript
document.cookie = name + '=' + value + '; path=/';
```

This gives the attacker a **cookie-writing primitive**.

Conceptually:

```text
Recipe ingredient
      ↓
cookie name/value
      ↓
document.cookie
      ↓
Inspector's browser cookie jar
```

This is much more interesting than the visible recipe functionality suggests.

We are not merely submitting data to a backend parser.

We are controlling persistent browser state inside the Inspector's context.

---

# 4. Inspecting the Cookies

The Inspector's browser contains security-sensitive cookies.

The two important ones are:

```text
session
role
```

The relevant security properties are:

```text
session → HttpOnly + Priority=High
role    → HttpOnly + default/Medium priority
```

The `role` cookie is particularly interesting because it controls the privileged operation.

The target state is:

```text
role=chief
```

However, a direct attempt such as:

```javascript
document.cookie = "role=chief; path=/";
```

does not replace the existing `HttpOnly` cookie.

This behavior is expected.

---

# 5. Why Direct Cookie Overwrite Fails

The `HttpOnly` attribute is specifically designed to prevent browser scripting APIs such as `document.cookie` from exposing that cookie to JavaScript. RFC 6265 describes `HttpOnly` as restricting the cookie to HTTP requests and excluding it from non-HTTP script-access APIs.

Therefore:

```text
Existing:
role=<protected value>; HttpOnly

Attempt:
document.cookie = "role=chief"

Result:
existing HttpOnly role remains
```

So the obvious attack:

```text
overwrite role
```

is blocked.

At this point, the key question becomes:

> Can we remove the existing cookie without being able to modify it directly?

This leads to browser cookie eviction.

---

# 6. Cookie Jar Overflow

Browsers cannot store an unlimited number of cookies.

RFC 6265 explicitly discusses the possibility that an attacker can force a user agent to delete cookies by storing large numbers of them until storage limits are reached.

Chromium's cookie implementation currently defines a per-domain limit of:

```text
180 cookies
```

and a purge target below that limit. Chromium's source also maintains quotas for Low, Medium, and High cookie priorities.

This is the primitive used by the challenge.

We already have a way to create arbitrary cookies:

```javascript
document.cookie = ...
```

Therefore the attack becomes:

```text
Create many attacker-controlled cookies
                ↓
Reach cookie storage limit
                ↓
Trigger eviction
                ↓
Force the browser to remove the target role cookie
```

---

# 7. Cookie Priority Is the Critical Detail

Simply overflowing the cookie jar is not enough.

We need the browser to remove:

```text
role
```

while preserving:

```text
session
```

The application deliberately creates:

```text
session → High priority
role    → Medium/default priority
```

Chromium's cookie eviction logic explicitly considers cookie priority when selecting cookies for deletion. Its current source shows separate purge rounds for Low, Medium, and High priority cookies and protects quotas at each priority level.

Therefore the desired state is:

```text
Before overflow:

session = valid
role    = protected

        ↓

Cookie jar overflow

        ↓

role     ❌ evicted
session  ✅ retained
```

This is the central trick.

---

# 8. Constructing the Payload

The challenge writeup confirms that approximately **200 dummy ingredients** are sufficient in its Chromium environment.

The payload concept is:

```text
junk0=AAAA
junk1=AAAA
junk2=AAAA
...
junk199=AAAA
role=chief
```

Each entry becomes a cookie operation in the Inspector's browser.

Conceptually:

```javascript
document.cookie = "junk0=AAAA; path=/";
document.cookie = "junk1=AAAA; path=/";
document.cookie = "junk2=AAAA; path=/";

/* ... */

document.cookie = "junk199=AAAA; path=/";

document.cookie = "role=chief; path=/";
```

The important detail is the **ordering**.

The `role=chief` operation must occur after the jar has been flooded.

---

# 9. Full Exploitation Logic

The sequence is:

```text
Existing cookies
│
├── session=VALID        [High + HttpOnly]
└── role=...             [Medium + HttpOnly]
│
▼
Attacker controls recipe ingredients
│
▼
Create ~200 junk cookies
│
▼
Cookie jar reaches Chromium per-domain limit
│
▼
Eviction occurs
│
▼
Medium-priority cookies become eviction candidates
│
▼
role cookie is removed
│
▼
session survives
│
▼
document.cookie = "role=chief"
│
▼
role=chief is now stored
│
▼
Inspector requests protected endpoint
│
▼
/api/seal
│
▼
Chief's Golden Seal
│
▼
FLAG
```

This is a multi-stage browser-state attack rather than a conventional injection vulnerability.

---

# 10. Proof-of-Concept

A minimal representation of the primitive is:

```javascript
for (let i = 0; i < 200; i++) {
    document.cookie = `junk${i}=AAAA; path=/`;
}

document.cookie = "role=chief; path=/";
```

In the actual challenge, we do not need to inject `<script>`.

The application's recipe-processing mechanism itself performs the cookie writes.

That means the real attack surface is:

```text
Ingredient name/value
        ↓
document.cookie
```

rather than:

```text
Ingredient
        ↓
<script>...</script>
```

This distinction explains why a conventional XSS payload is unnecessary.

---

# 11. Why XSS Is Not the Solution

One natural hypothesis is:

```html
<script>
document.cookie = "role=chief";
</script>
```

However, this is not the intended path.

The challenge sanitizes/escapes recipe data sufficiently to prevent the obvious HTML/JavaScript injection route. The published solution specifically rules out XSS through ingredient names/values and instead uses cookie-jar overflow.

Therefore the vulnerability is not:

```text
Stored XSS
```

or:

```text
Reflected XSS
```

The interesting primitive is already available legitimately:

```text
attacker input
   ↓
cookie creation
```

---

# 12. Triggering the Golden Seal

After the `role` cookie is evicted and recreated:

```http
Cookie: session=<valid-session>; role=chief
```

the Inspector's subsequent request to:

```text
/api/seal
```

is interpreted as coming from the Chief.

This crosses the application's authorization boundary:

```text
ordinary baker
       ↓
browser manipulation
       ↓
role=chief
       ↓
privileged endpoint
```

The server then awards:

```text
Chief's Golden Seal
```

and returns the challenge flag.

---

# 13. Vulnerability Classification

This challenge does not appear to rely on a specific CVE.

There is no evidence that the intended solution requires exploitation of a known third-party software vulnerability such as:

```text
CVE-XXXX-XXXX
```

Instead, the weakness is an **application-level security design flaw combined with browser cookie behavior**.

The most relevant classifications are:

### 13.1 Client-Controlled Authorization State

The application trusts:

```text
role=chief
```

from the client cookie to determine authorization.

This is dangerous because the browser is not a trusted authorization boundary.

The correct model is:

```text
session
   ↓
server-side identity
   ↓
server-side role
   ↓
authorization decision
```

rather than:

```text
cookie role
   ↓
authorization
```

---

### 13.2 Cookie Jar Overflow / Cookie Eviction

The browser has bounded cookie storage.

The attacker can create many cookies and force eviction.

RFC 6265 explicitly notes that storing large numbers of cookies can force a user agent to evict existing cookies.

Chromium documents the implementation limit and priority-aware eviction behavior in its CookieMonster implementation.

---

### 13.3 Security Boundary Bypass Through Browser State

The application assumes:

```text
HttpOnly role cookie
        =
protected role
```

But `HttpOnly` only prevents script access to that cookie; it does not make the application's authorization design inherently safe.

The challenge demonstrates a deeper lesson:

```text
Protecting a credential from direct JavaScript access
≠
Making the credential a trustworthy authorization primitive
```

---

# 14. Is This a `HttpOnly` Bypass?

Strictly speaking, this exploit should not be described as "breaking HttpOnly."

We are not reading the existing cookie.

We are not extracting the existing cookie's secret.

We are causing the browser to discard it.

Then we create a new cookie with the same name:

```text
old:
role=<protected value>

        ↓ eviction

new:
role=chief
```

So the primitive is better described as:

```text
Cookie Eviction
+
Cookie Name Re-Creation
+
Client-side Role Trust
```

rather than:

```text
HttpOnly bypass
```

That distinction is technically important.

---

# 15. Why Cookie Priority Matters

Without the priority difference:

```text
session
role
```

could both become eviction candidates.

That could destroy the Inspector's authenticated session before we reach the privileged endpoint.

The challenge avoids this by making:

```text
session = High
role    = Medium
```

so the desired effect is:

```text
role     → remove
session  → preserve
```

Chromium's source explicitly contains separate priority quotas and purge rounds for Low, Medium, and High cookies.

This is why the challenge's implementation is not merely "add a lot of cookies."

The exploit is:

```text
cookie flooding
+
eviction policy
+
priority manipulation
+
role recreation
```

---

# 16. Browser Dependence

Cookie limits and eviction behavior are implementation details and should not be assumed to be identical across browsers.

The cookie specification defines minimum capabilities and allows user agents to evict cookies; it does not prescribe a universal "180 cookies everywhere" rule.

For Chromium specifically, the current implementation documents:

```text
Per-domain maximum: 180
Purge amount:       30
```

and maintains priority quotas.

Therefore the following statement is precise:

```text
200 cookies worked against the challenge's Chromium inspector
```

It should not be generalized into:

```text
200 cookies will always work against every browser.
```

For a real assessment, the exact user-agent and cookie-store behavior must be verified.

---

# 17. Complete Exploit Chain

The final exploit can be summarized as:

```text
                      RECON
                        │
                        ▼
               Identify recipe review
                        │
                        ▼
              Quality Inspector bot
                        │
                        ▼
             Analyze browser context
                        │
                        ▼
               Discover cookie sink
                        │
                        ▼
            document.cookie primitive
                        │
                        ▼
              Identify role cookie
                        │
                        ▼
             HttpOnly blocks direct
                 modification
                        │
                        ▼
             Analyze cookie limits
                        │
                        ▼
             Flood cookie jar
                        │
                        ▼
              Trigger eviction
                        │
                ┌───────┴───────┐
                ▼               ▼
         role = evicted     session = kept
                │
                ▼
          role=chief created
                │
                ▼
             /api/seal
                │
                ▼
        Chief's Golden Seal
                │
                ▼
              FLAG
```

---

# 18. Alternative Attacks Considered

### Direct role overwrite

```javascript
document.cookie = "role=chief";
```

**Result:** fails because the existing cookie is `HttpOnly`.

---

### XSS

Attempting to inject JavaScript through the ingredient data is not required and is blocked by the challenge's encoding behavior.

**Result:** not the intended primitive.

---

### Session theft

This is unnecessary.

The goal is not to steal:

```text
session
```

but to preserve the valid session while manipulating:

```text
role
```

---

### Authentication bypass

Also unnecessary.

We already have a valid Inspector session.

The vulnerability is entirely in the authorization state carried by the `role` cookie.

---

# 19. Root Cause

The root cause is architectural.

The application effectively performs:

```text
Browser cookie
     ↓
role
     ↓
authorization decision
```

while simultaneously allowing attacker-controlled recipe data to manipulate the browser's cookie jar.

That creates a dangerous combination:

```text
Attacker-controlled browser state
            +
Security-sensitive cookie
            +
Authorization based on cookie value
```

The application's trust model is therefore fundamentally too permissive.

---

# 20. Secure Design

A secure implementation should never derive a privileged role directly from an attacker-modifiable cookie.

Instead:

```text
Cookie:
session=<opaque random identifier>
```

Then:

```text
Server:
session
   ↓
lookup session
   ↓
identify user
   ↓
lookup role server-side
   ↓
authorize request
```

For example:

```python
session = request.cookies.get("session")

user = session_store.get(session)

if not user:
    return unauthorized()

if user.role != "chief":
    return forbidden()

return award_golden_seal()
```

The browser should not be trusted to tell the server:

```text
"I am Chief."
```

The server should already know whether the authenticated identity is Chief.

---

# 21. Mitigations

### Do not store authorization roles as trusted client state

Prefer server-side session state.

---

### Treat cookies as untrusted input

Even:

```text
HttpOnly
Secure
SameSite
```

do not magically turn arbitrary cookie values into trusted authorization claims.

---

### Use authenticated/signed authorization tokens correctly

If a client-side token must carry claims, the server must cryptographically authenticate and validate those claims, including:

```text
issuer
audience
expiration
signature
role
```

and must consider revocation and privilege transitions.

---

### Remove the attacker-controlled cookie creation primitive

The recipe reviewer should not blindly execute:

```javascript
document.cookie = ...
```

from arbitrary attacker-controlled ingredient names.

Use a constrained data representation instead.

---

### Separate inspection from privileged authority

The Quality Inspector should never automatically possess:

```text
Chief privileges
```

merely because it is a privileged browser bot.

The review environment should use the minimum privileges necessary.

---

# 22. Professional Pentest Methodology

The reusable methodology from this challenge is broader than the specific exploit.

### Phase 1 — Recon

Identify:

```text
Routes
Authentication
Bots
Reviewers
Background jobs
Browser automation
```

The key discovery here was:

```text
/review/<id>
```

---

### Phase 2 — Data Flow Analysis

Track attacker-controlled input:

```text
recipe
 ↓
ingredient
 ↓
browser processing
 ↓
cookie creation
```

This is more useful than blindly fuzzing endpoint names.

---

### Phase 3 — Security-State Enumeration

Inspect:

```text
Cookies
Local Storage
Session Storage
Authorization headers
DOM state
```

Find which values affect privilege.

Here:

```text
role
```

was the critical state.

---

### Phase 4 — Boundary Testing

Test:

```text
Can role be overwritten?
Can role be deleted?
Can role be duplicated?
Can cookie Path/Domain be abused?
Can storage limits affect it?
```

The direct overwrite fails, so move to browser-state manipulation.

---

### Phase 5 — Browser Behavior Analysis

Study:

```text
Cookie quotas
Cookie eviction
Priority
Secure
HttpOnly
Domain
Path
SameSite
Partitioning
```

Then determine whether the browser's state machine can be manipulated.

---

### Phase 6 — Exploit Construction

Build the smallest deterministic chain:

```text
Flood
→ Evict
→ Recreate
→ Trigger
```

Avoid unnecessary primitives.

---

### Phase 7 — Impact Verification

Do not stop at:

```text
role=chief
```

Verify the actual security impact:

```text
/api/seal
        ↓
Golden Seal
        ↓
Flag
```

This proves privilege escalation rather than merely cookie manipulation.

---

# 23. Final Exploit

Conceptually:

```javascript
for (let i = 0; i < 200; i++) {
    document.cookie = `junk${i}=1; path=/`;
}

document.cookie = "role=chief; path=/";
```

The challenge's recipe mechanism supplies the equivalent browser-side operations.

The important exploit requirements are:

```text
~200 attacker-controlled cookies
+
role=chief as the final cookie
```

The exact count is browser/environment dependent, so the 200-cookie value should be treated as the challenge-specific working payload rather than a universal constant.

---

# 24. Post-Exploitation

Once the browser reaches:

```text
session=<valid>
role=chief
```

the protected endpoint recognizes the Inspector as the Chief.

The application awards:

```text
Chief's Golden Seal
```

and exposes the flag.

The final flag is:

```text
sun{c00kie_jar_0verfl0w_ev1cts_the_chief}
```

---

# 25. Final Takeaway

The key lesson from CookieCorp is not simply:

```text
"Create 200 cookies."
```

The real lesson is:

```text
Do not confuse a browser security attribute
with a secure authorization architecture.
```

`HttpOnly` prevents script access to a cookie, but it does not make a client-side role trustworthy.

The challenge combines four concepts:

```text
1. Attacker-controlled cookie creation
2. Cookie storage limits / eviction
3. Priority-aware browser state management
4. Client-side authorization trust
```

Together they create:

```text
HttpOnly role
      ↓
cannot overwrite directly
      ↓
cookie jar overflow
      ↓
role eviction
      ↓
recreate role=chief
      ↓
privileged request
      ↓
Chief's Golden Seal
      ↓
sun{c00kie_jar_0verfl0w_ev1cts_the_chief}
```

That makes CookieCorp a particularly good example of a **Web application logic vulnerability built from browser state semantics rather than a conventional injection bug**.
