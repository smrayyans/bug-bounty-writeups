# GraphQL token-deletion mutation lets one tenant delete another tenant's API token

## Summary

A personal-API-token deletion mutation resolved the token by identifier alone. A user authenticated to one tenant could pass the identifier of a token belonging to another tenant, and the token was deleted. The deletion was permanent, and the credential stopped authenticating immediately afterwards.

## Context

A multi-tenant retail management platform, where each merchant works behind its own hostname with a separate user list and separate settings. Integrations authenticate with personal access tokens created from a settings screen. The access level needed was an ordinary authenticated session on any tenant: no administrator role, and no reach into the victim's tenant or anything the victim does.

## How I found it

### Where I started

I had two tenants of my own, which made the authorization surface cheap to test: create an object in one, ask for it from the other, see what comes back. I spent most of the first pass on reads — sales, promotions, products, variants, groups, customers, integrations and users — and every one of them refused a foreign identifier with a not-found error that named the type and the identifier it could not resolve.

### The tenant-selection attempts

Before accepting that, I tried to make the server pick the wrong tenant for me. I rewrote the store cookie, added store and retailer headers and store and retailer parameters, and sent one tenant's cookies to the other tenant's hostname. None of it moved: the cross-host request was refused at the session layer, and the rest changed nothing.

### Turning to writes

Reads were well covered, so I stopped asking for objects and started deleting them. Token management looked like the least-travelled path. The settings screen that used to host it now shows a migration banner instead of a link, but the route underneath still served a working form, so I generated a token there and used it.

### The dead end that shaped the evidence

My first delete attempt against a random identifier came back `true`. That looked like a hit, but the mutation returned the same `true` for an identifier that could not exist, so the response proved nothing. I could not use the boolean as evidence. From then on I read the victim tenant's token list before and after every call, and re-used the credential itself to see whether it still worked. That turned a meaningless `true` into a real before-and-after.

### What it meant

The sibling mutation on the same identifier answered a foreign token with a not-found error, so the platform clearly knew how to scope this object. The delete path just did not.

## Proof of concept

Tenant A owns the token and tenant B performs the deletion. Each request goes to the tenant host named in its header and is otherwise a normal authenticated call from that tenant. Operation and field names below are shortened placeholders (`createToken`, `deleteToken`, `tokens` stand in for the server's own names); the arguments, status codes and returned values are as observed, with values replaced.

Tenant A creates a token:

```http
POST /api/graphql HTTP/1.1
Host: store-a.example
Content-Type: application/json
```

Each GraphQL operation below is the body of a POST to `/api/graphql` with the headers shown above; only the operation and its arguments change.

```graphql
mutation CreateToken($n: String!) {
  createToken(request: {name: $n}) { tokenId tokenSecret }
}
```

```http
HTTP/1.1 200 OK

data.createToken.tokenId     = <TOKEN_ID>
data.createToken.tokenSecret = <TOKEN_SECRET>
```

The arguments are `n = demo-token`. Tenant A reads the list back:

```graphql
query { tokens { id name } }
```

```http
HTTP/1.1 200 OK

data.tokens[0].id   = <TOKEN_ID>
data.tokens[0].name = demo-token
```

The token authenticates against tenant A when used as a bearer credential:

```http
GET /api/2.0/retailer HTTP/1.1
Host: store-a.example
```

```http
HTTP/1.1 200 OK

data: id = <TENANT_ID>, status = trial
```

Tenant B does not own this token; its list is empty:

```graphql
query { tokens { id name } }
```

```http
HTTP/1.1 200 OK

data.tokens = []
```

Control: B deletes an identifier that cannot exist, and A's token survives.

```graphql
mutation DeleteToken($id: String!) { deleteToken(id: $id) }
```

```http
HTTP/1.1 200 OK

data.deleteToken = true
```

The argument is `id = <UNKNOWN_ID>`. Re-reading tenant A's list after that control shows the token still present. Then B sends A's real identifier, with `id = <TOKEN_ID>`:

```graphql
mutation DeleteToken($id: String!) { deleteToken(id: $id) }
```

```http
HTTP/1.1 200 OK

data.deleteToken = true
```

Tenant A's list is now empty, and the credential no longer authenticates:

```graphql
query { tokens { id name } }
```

```http
HTTP/1.1 200 OK

data.tokens = []
```

```http
GET /api/2.0/retailer HTTP/1.1
Host: store-a.example
```

```http
HTTP/1.1 401

error = <invalid-token message>
```

The sibling mutation refuses the foreign identifier and leaves the token intact:

```graphql
mutation EditToken($id: String!) {
  editToken(request: {id: $id, name: "renamed-by-attacker"}) { success }
}
```

```http
HTTP/1.1 200 OK

errors[0].message = <not-found for that token id>
errors[0].path    = [editToken]
data              = null
```

The argument is `id = <OTHER_ID>`.

## Steps to reproduce

Preconditions: two tenants that do not share a session, one access token on tenant A, and the identifier of that token.

1. Sign in to tenant A and generate an access token in the settings area.
2. Read the token list on tenant A and note the identifier.
3. Confirm the credential works: call a tenant A endpoint with it as a bearer token. Expect `200`.
4. Sign in to tenant B. Read the token list there and confirm it is empty.
5. From tenant B, delete an identifier that cannot exist, then re-read tenant A's list. The mutation returns `true` and the token is still there — the boolean is not a success signal.
6. From tenant B, delete tenant A's token identifier.
   - **Expected:** a not-found error, the same way the edit mutation answers a foreign identifier.
   - **Actual:** the mutation returns `true`, and tenant A's list comes back empty.
7. Re-use tenant A's bearer credential against the same endpoint as step 3. Expect `401` reporting an invalid access token.

Repeating steps 1 to 7 with a second token produced the same result: present in tenant A's list, deleted from tenant B, list empty afterwards.

## Impact

A token belonging to one tenant was erased from another tenant's session, and the credential stopped working — a `200` with the retailer record beforehand, a `401` after. The owning tenant gets no notification, so an integration that authenticates with that token simply begins failing until somebody notices and issues a replacement. Nothing available to the owner reverses it.

Nothing of the victim's was readable, and the token secret never came back in any response. What was demonstrated is destruction of a credential, plus the outage that follows for whatever depended on it. Repeating it needs more token identifiers, and the listing query accepts no arguments and shows nothing outside the caller's own rows, so no cross-tenant source of identifiers was found.

## Root cause and fix

Nothing in the response indicates that the identifier was resolved against the calling tenant: an identifier that cannot exist, one owned by the caller and one owned by somebody else all produce the same `true`, and the row disappears only in the last case. The observed behavior is consistent with the delete handler looking the record up by primary key and removing it, while the sibling edit handler on the same identifier performs a scoped lookup and reports not-found.

Bind the lookup to the caller's tenant and user in the predicate, answer not-found when nothing matches, and report success only after a row belonging to that caller has really been removed. Audit the other destructive mutations keyed by identifier for the same gap, instead of trusting them because the edit path is correct.

## What I learned

Every read I tried was scoped correctly, and that is why I kept going: a platform that gets lookups right is one where the handler that skips the lookup stands out. The other lesson came from the boolean. When a call returns a value that is always the same, the evidence has to come from the object's state before and after, plus the credential itself — otherwise I am grading my own homework.
