# Account reset token is sent to a third-party tracking beacon and kept in its logs

## Summary

The account host loads a tracking image on every page, and the image URL carries the full page URL in a query parameter. On the reset route the page URL contains the reset token, so simply opening a reset link sends the token to an object-storage host, where it is written into that bucket's access logs.

## Context

The authentication and account site of a financial services platform. Account resets are sent as a link containing a single-use token; the token is accepted both by an API endpoint and by a browser route in the single-page app. The access needed was a reset email for an account, which the account owner controls.

## How I found it

### Where I started

I was looking at the reset flow end to end, from the email to the credential change, because reset links are the one place a URL is expected to carry a secret. That makes every request the page makes while the token is in the address bar worth reading.

### What the page sent

With the network panel open on the reset route, the page loaded an image from an object-storage host. The request had nothing to do with the reset. The image source was built in a small inline script that appends the current page URL to a query parameter.

On the reset route the current page URL is the reset link, token included. So the token left the platform in a plain image request, before the user had typed anything.

### Confirming the value

I read the request out of the network panel and the token was there in the parameter, in full. Then I checked whether that token was enough on its own: I posted it to the reset API with a new credential, and the API accepted it. Signing in with the new credential worked.

### The part I could not test

The exposure depends on who can read the bucket's access logs, and I had no access to them. What I could confirm is that the token is transmitted to the bucket; the log entry follows from the request being served.

### What it meant

A secret that is supposed to be seen once by the account owner is copied to a third-party system as a side effect of rendering the page.

## Proof of concept

Opening the browser route with the token in the path:

```http
GET /reset/<TOKEN> HTTP/1.1
Host: auth.example
```

The tracking image the page loads while that route renders:

```http
GET /bg.png?x-id=<ID>&x-s=https://auth.example/reset/<TOKEN> HTTP/1.1
Host: <object-storage-host>
```

The `x-s` parameter is the page URL, appended by the inline script, so the token travels as part of it.

The same token is accepted by the reset API:

```http
POST /api/account/reset/<TOKEN>/ HTTP/1.1
Host: auth.example

<the new credential, submitted twice as the endpoint expects>
```

```http
HTTP/1.1 200 OK
```

Signing in with the new credential then succeeds, which is how I confirmed the token had been used for a real reset and not just accepted.

## Steps to reproduce

Preconditions: an account that can request a reset, and access to whatever receives the tracking request.

1. Request an account reset and open the link from the email.
2. Take the token out of the link and build the browser route, `/reset/<TOKEN>`.
3. Open that route with the network panel recording, filtered to the object-storage host.
   - **Expected:** the page makes no request carrying the token.
   - **Actual:** an image request goes out with the full page URL, token included, in a query parameter.
4. Post the same token to the reset API with a new credential.
5. Sign in with the new credential. The session opens, confirming the reset.

## Impact

Every visit to a reset link sends that token to an object-storage host outside the reset flow, and the request is recorded in that bucket's access logs. Anyone who can read those logs — through compromised credentials for the bucket, an internal role with analytics access, or a downstream pipeline that ingests the logs — can recover tokens for accounts that have requested a reset and use them against the reset API.

The token alone was enough to set a new credential on the account I tested, which had no second factor on the reset. Whether a token alone is sufficient where a second factor is enforced at the credential change was not tested, and the writeup does not claim it.

The exposure also outlives the reset: the request is logged when the page loads, and the token stays in the log after the credential has been changed.

## Root cause and fix

The observed behavior is consistent with a page-level tracking snippet that forwards the entire current URL, which is fine on marketing pages and wrong on any route whose path or query carries a secret.

Send a fixed value, or the pathname with sensitive segments removed, instead of the full URL. Move the token out of the address bar: deliver it in the fragment, or POST it, so a page URL never contains it. Rotate reset tokens that have already been exposed, and shorten their lifetime so a logged token expires before it is useful.

## What I learned

The network panel is part of the threat model for any page that handles a secret in its URL. I had been treating the reset page as a form, and the request that mattered was not the one the form sent.
