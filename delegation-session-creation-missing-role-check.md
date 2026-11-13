# Support-session creation lets any guest mint an impersonation session

## Summary

A staff-only support feature let anyone start a support session. A visitor with no account at all could send one POST to the start endpoint and receive a session identifier in its pre-claim state, for any email address they chose. The claim step still refused to complete the login, so no account was accessed.

## Context

A consumer authentication service for a food-delivery platform. The service runs the login, signup and social-auth flows for customer accounts, and also hosts a delegation feature that support staff use to sign in as a customer while assisting them. The access needed was a guest session on the authentication host, obtained by opening the login page. No account, no credentials and no staff role were required.

## How I found it

### Where I started

I was mapping the authentication host: which paths exist, which methods they take, and which of them are reachable without a session. Login flows tend to carry a few staff-facing helpers next to the customer-facing ones, and those are worth checking for a role test.

### The endpoint that did not belong

The support paths stood out because the name describes an action rather than a page. Reading the flow from the response behaviour, the intended sequence is: a staff member authenticates, starts a support session, claims it with their employee credentials, the customer signs in through the support page, and the staff member is then working inside that customer's account.

That first step is the one I tested. I sent the start call from a browser that had never logged in.

### What came back

I expected a refusal, since the caller is an anonymous visitor. The response was a success: an identifier for a new session, the state the client shows before a staff member claims it, and the polling intervals the client uses while it waits. I asked the status endpoint about the identifier and it agreed the session existed. The support login page also rendered when I asked for it with that identifier.

### Where it stopped

The last step is the claim. I posted the identifier to the claim endpoint and got a rejection: the session had not been claimed by a staff member, so the login was refused. That is the boundary that held. Everything before it did not.

### What it meant

Session creation, not impersonation, is what I could demonstrate. The platform was minting support sessions for anonymous callers, and the only thing standing between that and an account was the staff claim.

## Proof of concept

Paths and field names below are generalized; the caller is a guest session, and the values are as observed with identifiers replaced.

```http
POST /auth/support/start?clientId=<CLIENT_ID>&email=user@example.com&scope=*&redirectUrl=https://app.example/&state=probe HTTP/1.1
Host: identity.example
X-CSRF-Token: [REDACTED]
Cookie: [REDACTED]

```

```http
HTTP/1.1 200 OK

id     = <SESSION_ID>
status = pre-claim
poll   = 2500 ms before claim, 1000 ms after
```

The session is live and pollable:

```http
GET /auth/support/status?sessionId=<SESSION_ID>&clientId=<CLIENT_ID> HTTP/1.1
Host: identity.example
X-CSRF-Token: [REDACTED]
Cookie: [REDACTED]
```

```http
HTTP/1.1 200 OK

status = pre-claim
```

The support login page renders for that identifier with no staff involvement:

```http
GET /auth/support/page?sessionId=<SESSION_ID>&clientId=<CLIENT_ID>&redirectUrl=https://app.example/&responseType=code&scope=*&state=probe HTTP/1.1
Host: identity.example
Cookie: [REDACTED]
```

```http
HTTP/1.1 200 OK

<the platform's login page, in support context>
```

The claim step refuses, because nobody has claimed the session:

```http
POST /auth/support/claim HTTP/1.1
Host: identity.example
X-CSRF-Token: [REDACTED]
Cookie: [REDACTED]

session  = <SESSION_ID>
client   = <CLIENT_ID>
scope    = *
return   = https://app.example/
state    = probe
```

```http
HTTP/1.1 401

rejection: the session was never claimed, and the login is refused
```

## Steps to reproduce

Preconditions: none beyond a browser that can load the authentication host.

1. Open the platform's login page on the authentication host and let it complete whatever browser challenge it serves. A guest session is enough.
2. Read the anti-CSRF value from the cookies the page sets.
3. Send the start call to the support-session endpoint with an arbitrary email address, an empty form body, and the anti-CSRF header.
   - **Expected:** a refusal, because the caller holds no staff role.
   - **Actual:** `200` with a new session identifier in its pre-claim state.
4. Ask the status endpoint about the identifier. It reports the same state, so the session exists.
5. Ask for the support login page with the identifier. It renders in support context.
6. Post the identifier to the claim endpoint.
   - **Expected:** an impersonated session, if creation alone were enough.
   - **Actual:** `401`, with the login refused because the session was never claimed by staff.

## Impact

An anonymous visitor could create support sessions bound to any email address, as often as they liked. Each session was live and pollable and carried a rendered login page, which is the state the support tool expects immediately before a staff member claims it.

No account was accessed. The claim step requires an authenticated staff member, and the login was refused without one. What was demonstrated is that the platform creates the object for callers who should not be able to create it, not that the object can be used to reach a customer's data.

## Root cause and fix

The observed behavior is consistent with the start handler treating the caller as staff without checking for a staff claim, while the claim and login handlers do enforce that check: the same identifier is accepted by the status and page endpoints and refused at claim.

Verify a staff role when the session is created, not only when it is claimed. Move the support endpoints off the consumer host, or gate them at the edge so that only the internal support network can reach them, and return a plain refusal to anyone else.

## What I learned

An endpoint that refuses at the last step can still be doing too much at the first. I had a clean refusal in hand and almost stopped there; asking the status endpoint whether the object existed is what turned a rejected login into a finding about creation.
