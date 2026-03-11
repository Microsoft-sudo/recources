---
name: csrf
description: Advanced CSRF testing with exploit validation, token analysis, SameSite bypass, JSON CSRF, CORS abuse, and safe exploit confirmation
---

# CSRF

Cross-site request forgery abuses ambient authority (cookies, HTTP auth) across origins.

CSRF only exists when **a cross-origin request can trigger a state-changing action using the victim's authenticated session without requiring a secret the attacker cannot obtain**.

The agent must **confirm exploitability before reporting**.

If exploitation cannot be confirmed, mark the issue as **Not Exploitable**.

---

# Attack Surface

## Session Types

Prioritize endpoints that rely on **cookie-based authentication**.

CSRF is most likely when:

- Authentication uses session cookies
- Cookies are automatically sent by browsers
- No explicit user interaction is required

Examples:

- Traditional web apps
- Cookie-based REST APIs
- GraphQL APIs using cookies
- File upload endpoints
- Legacy admin panels

Low probability targets:

- APIs using Authorization: Bearer tokens
- APIs requiring signed requests
- APIs requiring client certificates

If no cookies are sent cross-origin → mark **Not CSRF Prone**.

---

# Authentication Flows

High-priority flows:

- Login
- Logout
- Password change
- Email change
- MFA enable/disable
- Device trust registration
- Session invalidation
- Account linking

These endpoints must **always require CSRF protection**.

---

# OAuth/OIDC

Evaluate CSRF in:

- /authorize
- /logout
- /connect
- /disconnect
- redirect_uri flows

Check for:

- missing state parameter
- weak state validation
- state reuse

Exploitability requires **actual account binding or session impact**.

---

# High-Value Targets

Prioritize endpoints performing **state-changing operations**:

- Email change
- Password change
- Phone number change
- Payment details
- Plan upgrades
- Account deletion
- API key generation
- SSH key addition
- OAuth client linking
- Role/permission changes
- Admin actions
- File upload/delete
- Access control modifications

Only report if **the action succeeds without a valid CSRF defense**.

---

# Reconnaissance

## Session and Cookies

Inspect cookies:

- HttpOnly
- Secure
- SameSite

Interpretation:

SameSite=Strict  
→ Cookies not sent cross-site → CSRF unlikely

SameSite=Lax  
→ Cookies sent on **top-level GET navigation**

SameSite=None  
→ Cookies sent on **all cross-site requests**

Verify whether session cookies are sent during:

- cross-origin GET navigation
- cross-origin POST forms
- iframe loads
- fetch/XHR requests

If cookies are not sent → **Not exploitable**.

---

## Token and Header Checks

Locate anti-CSRF tokens.

Common locations:

- hidden form fields
- meta tags
- JavaScript variables
- custom headers

Example token names:

```
csrf_token
authenticity_token
xsrf-token
x-csrf-token
```

Test:

1. Remove token
2. Send empty token
3. Reuse old token
4. Reuse token across sessions
5. Change request method
6. Modify request path

Exploit only if server **accepts the request without validating the token**.

---

## Method and Content-Types

Check if state changes occur via:

- GET
- HEAD
- OPTIONS

These methods should **never modify server state**.


Test content-types that **avoid CORS preflight**:

```
application/x-www-form-urlencoded
multipart/form-data
text/plain
```


Many backends **auto-parse these into JSON**.

---

## CORS Profile

Check response headers:

```
Access-Control-Allow-Origin
Access-Control-Allow-Credentials
```

Dangerous configuration:

```
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
```


Or dynamic origin reflection.

CORS misconfiguration can **convert CSRF into data exfiltration**.

---

# Key Vulnerabilities

## Navigation CSRF

Generate HTML exploit.:

```
<html>
<body>

<form action="TARGET" method="POST">
<input type="hidden" name="param" value="value">
</form>

<script>
document.forms[0].submit();
</script>

</body>
</html>
```

Confirm:

- Cookies are sent
- Request succeeds
- State changes

---

## Simple Content-Type CSRF

Use:


application/x-www-form-urlencoded
multipart/form-data
text/plain


These do **not trigger CORS preflight**.

If accepted without token → exploitable.

---

## JSON CSRF

Test if server parses JSON from:

- text/plain
- form encoded
- multipart


Example:

```
{"email":"attacker@email.com"}
```


Or:


email=attacker@email.com


If backend reconstructs JSON → CSRF possible.

---

## Login CSRF

Steps:

1. Force victim logout
2. Submit login form with attacker credentials
3. Victim becomes logged into attacker account

Impact:

- data exposure
- session confusion
- phishing setups

---

## OAuth/OIDC CSRF

Test flows without state validation.

Attack:

- attacker initiates OAuth flow
- victim completes authorization
- victim account binds to attacker service

---

## File and Action Endpoints

Commonly vulnerable:

- file uploads
- delete endpoints
- admin actions

Multipart requests often bypass token checks.

---

## GraphQL CSRF

Check if GraphQL accepts:

- GET requests
- persisted queries

Example:

```
GET /graphql?query=mutation{deleteAccount}
```


If cookies are sent → mutation may execute.

---

## WebSocket CSRF

WebSocket handshake includes cookies.

Test cross-origin connections:

```
new WebSocket("wss://target.com/socket")
```


Server must enforce **Origin validation**.

---

# Bypass Techniques

## SameSite Bypass

SameSite=Lax allows cookies during:

- top-level GET navigation

Exploit if state changes via GET.

Example:

```
https://target.com/deleteAccount?id=123
```

---

## Origin / Referer Bypass

Test:


```
Origin: null
Referer: null
```


Or navigation from:


about:blank
data:
sandbox iframe


Some frameworks incorrectly accept null origins.

---

## Method Override

Many frameworks support:


```
_method=DELETE
X-HTTP-Method-Override: DELETE
```

Allows destructive actions via POST.

---

## Token Weaknesses

Detect:

- missing token validation
- token not session-bound
- token reuse
- predictable tokens
- tokens in URL

Double submit cookie flaws:


csrf_token cookie = request token


If attacker can set cookie → bypass possible.

---

## Content-Type Switching

Switch between:


form-urlencoded
multipart
text/plain


Different parsers may trigger different validation logic.

---

## Header Manipulation

Try removing headers:


```
Origin
Referer
```


Test if server still processes request.

---

# Special Contexts

## Mobile / Hybrid Apps

Check:

- WebViews
- deep links
- cookie reuse

Hybrid apps often mix **cookies and API requests**.

---

## Integrations

Check:

- internal tools
- webhook endpoints
- staff dashboards

These often expose **GET actions without CSRF protection**.

---

# Chaining Attacks

Combine CSRF with:

- CSRF + IDOR → modify other user resources
- CSRF + Clickjacking → trick user interaction
- CSRF + OAuth → bind victim to malicious account

---

# Testing Methodology

1. Inventory endpoints
2. Identify state-changing requests
3. Check authentication model
4. Check CSRF tokens
5. Test token removal
6. Test cross-origin delivery
7. Test SameSite behavior
8. Test CORS misconfiguration
9. Test parser differences

---

# Exploit Confirmation (Required)

Before reporting, confirm:

1. Cross-origin request successfully triggers the action
2. Victim cookies are automatically included
3. No valid CSRF token required
4. Origin/Referer validation missing or bypassed
5. State change confirmed

Validation evidence:

- before/after state
- response difference
- visible UI change

---

# False Positives

Mark **Not Exploitable** when:

- CSRF token required and validated
- Origin/Referer properly enforced
- Cookies not sent cross-origin
- SameSite=Strict prevents requests
- Endpoint is read-only
- Only idempotent actions affected

---

# Safe Exploitation

Allowed:

- change test email
- toggle settings
- create harmless objects

Never perform:

- destructive actions
- account deletion
- financial transfers

---

# Impact

Confirmed CSRF can lead to:

- account takeover via login CSRF
- email/password modification
- financial transactions
- admin actions
- API key creation
- role escalation

Impact severity depends on affected functionality.

---

# Pro Tips

1. Always attempt **preflightless attacks**
2. Test **GET state changes**
3. Evaluate **login CSRF**
4. Inspect **OAuth flows**
5. Test **content-type parser differences**
6. Check **method overrides**
7. Combine with **clickjacking**

---

# Summary

CSRF exists only when:

1. Authentication uses automatically sent credentials
2. A state-changing action is reachable cross-origin
3. No secret is required that the attacker cannot supply
4. The action succeeds without user interaction

Always confirm exploitability before reporting.