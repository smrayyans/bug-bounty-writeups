# Any internet caller can post deploy events to production CI listeners

## Summary

A web estate exposed more than twenty CI event listeners as deploy webhooks, reachable from the internet with no authentication of any kind. There was no shared secret and no signature check, and nothing sat in front of them. One was the production tier. Each listener took a JSON body and returned `202` with its own identifier and a fresh event ID, and each error reply named the internal deployment target it serves.

## Context

A large web estate with several environments, from development through production, and a CI/CD trigger component in front of each. The access needed was none: an unauthenticated HTTPS request from anywhere.

## How I found it

### Where I started

The estate publishes a broad set of hosts, and the naming pattern made the CI endpoints easy to pick out. CI and deploy infrastructure is usually the most valuable target on a perimeter and the one most often guarded by a shared secret rather than a login, so it was worth checking whether that secret was present.

### The first request

A plain `GET /` on a production trigger returned `400` with a body that named the listener and its internal deployment target. That is a lot to hand an anonymous caller, and it answered three questions at once: the component was live, it was the production one, and it was talking to me without a challenge.

### Confirming it was not a decoy

`GET /live` returned `200` with a short `ok` body, the liveness probe of the trigger component. That ruled out a static page imitating the response.

### Sending an event

Then I posted a body that could not match any real event type. The listener answered `202 Accepted` with an event listener UID and a new event ID. I repeated it with a deliberately invalid GitHub signature header, and then with an invalid GitLab token header. Both were accepted. The checks that normally sit in front of a deploy webhook were simply absent.

### The rest of the fleet

From there it was enumeration. The same shape existed across every environment from development through production, and each listener accepted the same anonymous POST. Methods other than POST also reached the parser and came back with the same sink body, so there was no method filter either.

### What I did not claim

A `202` says the listener took the request and picked a trigger. It does not show that a pipeline ran, because a pipeline that fails to be created also returns `202`. The bodies I sent contained no field that a deploy binding reads, so binding resolution would fail. The finding is the absent authentication boundary, and that is where I stopped.

## Proof of concept

The production listener describes itself to an anonymous caller:

```http
GET / HTTP/1.1
Host: ci-prod.example
```

```http
HTTP/2 400

listener  = deploy
target    = prod
listenerUID = (empty)
message   = <malformed body>
```

It is live, not a decoy:

```http
GET /live HTTP/1.1
Host: ci-prod.example
```

```http
HTTP/2 200 OK

ok
```

A production POST with no credential is accepted:

```http
POST / HTTP/1.1
Host: ci-prod.example
Content-Type: application/json

{"check": "probe-one", "event": "unknown-type"}
```

```http
HTTP/2 202

listener  = deploy
target    = prod
listenerUID = <UID>
eventID   = <EVENT_ID>
```

A fake GitHub signature is accepted:

```http
POST / HTTP/1.1
Host: ci-dev1.example
Content-Type: application/json
X-GitHub-Event: ping
X-Hub-Signature-256: sha256=deadbeefgarbage

{"zen": "probe", "hook_id": 0}
```

```http
HTTP/2 202

listener  = deploy
target    = dev1
listenerUID = <UID>
eventID   = <EVENT_ID>
```

An invalid GitLab token is also accepted:

```http
POST / HTTP/1.1
Host: ci-dev1.example
Content-Type: application/json
X-Gitlab-Event: Push Hook
X-Gitlab-Token: garbage-no-secret

{"check": "x"}
```

```http
HTTP/2 202

listener  = deploy
target    = dev1
listenerUID = <UID>
eventID   = <EVENT_ID>
```

The acceptance tier answered identically and named its own deployment target. `PUT`, `DELETE`, `PATCH` and `OPTIONS` on the same path all reached the parser and returned the same sink body.

## Steps to reproduce

1. Find a host running the CI trigger component and request `/` with no credentials.
   - **Expected:** an authentication challenge, or a refusal at the edge.
   - **Actual:** `400` with a body naming the listener and its internal deployment target.
2. Request `/live` on the same host. Expect a denial. Actual: `200` and `ok`, the component's liveness probe.
3. `POST /` with a body that matches no real event type.
   - **Expected:** `401` or `403`.
   - **Actual:** `202` with an event listener UID and a new event ID.
4. Repeat the POST with an invalid `X-Hub-Signature-256`, then with an invalid `X-Gitlab-Token`. Both are accepted, so no webhook secret is verified.
5. Repeat against the production host and one other environment. The production host accepts the same anonymous event.
6. Repeat with `PUT`, `DELETE`, `PATCH` and `OPTIONS`. All four get as far as the parser and come back with the same body.

Do not read the `202` as proof that a pipeline started. Send only bodies that cannot match a deploy binding, as above, so nothing is created.

## Impact

An anonymous caller could submit events to more than twenty deploy listeners, the production tier among them, and every acceptance returned a listener UID, a new event ID and the internal deployment target. The control meant to stand in front of the deploy path, a shared secret or a signature interceptor, was missing on all of them, leaving only the listener's own parsing.

What is demonstrated is acceptance plus disclosure of internal deployment targets. Whether a pipeline actually runs depends on the trigger bindings behind the listener, which are not visible from outside; the bodies used here contained no field a binding resolves, so no pipeline was started and no cluster state was touched.

## Root cause and fix

The observed behavior is consistent with the listeners being published straight to the internet with nothing in front of them: signature headers are ignored, no secret is required, and the component's identification and deployment-target fields are returned to unauthenticated callers.

Put an authentication interceptor on every listener, verify a per-listener webhook signature, and reject anything without a valid one before the event body is parsed. Restrict the listeners to the source ranges of the systems that are supposed to call them, and stop echoing internal deployment targets and listener identifiers in error responses.

## What I learned

Deploy infrastructure announces itself. One anonymous `400` that named its own deployment target was enough to know where to look, and the rest of the work was checking whether the same shape repeated on the other environments, which it did. The discipline that mattered was at the end: an accepted event is not an executed pipeline, and the report had to stop where the evidence stopped.
