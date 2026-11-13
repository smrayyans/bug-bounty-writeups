# Unauthenticated health endpoint exposes the authentication service's internal service map

## Summary

A health endpoint on the authentication host answered without authentication and, in doing so, listed the internal services behind the host and their status. Each name pointed at a functional area of the login stack, from delegation to phone verification, and the naming convention was enough to derive HTTP paths that turned out to exist.

## Context

The authentication service of a food-delivery platform, the host that runs consumer login, signup and social sign-in. The access needed was none: an unauthenticated GET.

## How I found it

### Where I started

Health and status endpoints are usually the cheapest thing to check on a host that hands out logins, because they are meant for operators and are easy to forget at the edge. I requested the common health path and got JSON instead of a page.

### What it returned

A status of `UP`, and underneath it a list of components with their own states. The component names were not generic. They described the authentication services the host runs: delegation, phone verification, social providers, device remembering, magic links, guest sessions.

### The part that made it worth reporting

A name like that is a hint about a path. The services were named after their function, and the host's routes follow the same vocabulary, so I translated a few names into paths by hand and probed them. A delegation path answered `405`, which means the route exists and wants a different method. A phone-verification path answered `303`, an active redirect. Neither was linked from the login page I had loaded.

I did not need to fuzz or read the JavaScript bundle to find them. That is the difference between this and a version banner: the disclosure tells a reader what to look for.

### Confirming the mapping

I checked each derived path with a single request and recorded the method and status. Where the naming convention was ambiguous, I left the path out of the writeup rather than guessing.

### What it meant

The endpoint published an index of the login stack. It did not publish anything a user had put there.

## Proof of concept

The request:

```http
GET /actuator/health HTTP/1.1
Host: identity.example
Accept: application/json
```

```http
HTTP/1.1 200 OK

status = UP
components (names generalized):
  delegation          UP
  phoneVerification   UP
  socialProvider      UP
  webauthn            UP
  magicLink           UP
  guestAuthentication UP
  ...                 (further entries)
```

Paths derived from two of those names, probed once each:

```http
GET /auth/support/start HTTP/1.1
Host: identity.example
```

```http
HTTP/1.1 405 Method Not Allowed
```

```http
GET /auth/phone/check/eligibility HTTP/1.1
Host: identity.example
```

```http
HTTP/1.1 303 See Other
```

A `405` means the route exists and rejects the method; a `303` means it redirected. Neither path appeared in the login page's links, and both were reachable without a session.

## Steps to reproduce

1. Request `/actuator/health` on the authentication host with no session.
   - **Expected:** `404`, a redirect to login, or a body with only an overall status.
   - **Actual:** `200` with the component list and per-component states.
2. Take a component name and derive the route path from the same vocabulary, for example a delegation component to `/auth/support/...`.
3. Request the derived path with `GET`.
   - **Expected:** a `404` if the route does not exist.
   - **Actual:** `405` for a POST-only route, or `303` for an active redirect.
4. Repeat for the remaining names. Record only the paths that return a method or redirect response, and ignore the ones that do not resolve.

## Impact

The endpoint disclosed the internal service layout of the authentication host and the functional areas each service covers, and it did so without any credential. Combined with the host's route naming, that list shortened the work of finding unlinked authentication endpoints from fuzzing or bundle analysis to a handful of hand-written requests.

Nothing beyond names and health states was returned. The response carried no user records, credentials, tokens or configuration values, and the derived endpoints were only confirmed to exist, not exercised.

## Root cause and fix

The observed behavior is consistent with the health endpoint being exposed at the edge with component details enabled by default, so the detail view that operators use is served to the public.

Serve only the aggregate status publicly, with component details disabled, and restrict the management paths to an internal network or an authenticated operator route. The same rule applies to the rest of the management surface, not just this one path.

## What I learned

The value of a leaked name is not the name. It is that the name lets you skip the part of the work where you would otherwise have to guess. I nearly wrote this off as a banner, and what changed my mind was probing two of the derived paths and seeing that the vocabulary carried through to real routes.
