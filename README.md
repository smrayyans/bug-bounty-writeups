# bug-bounty-writeups

Sanitized write-ups of bugs found through bug bounty programs. Target details, program names and identifying info removed.

Each file is self-contained: summary, how the bug was found, proof of concept, reproduction steps, impact, root cause and fix. Values are replaced; request shapes and observed behavior are kept.

## Writeups

- [Delegation session creation](delegation-session-creation-missing-role-check.md): a staff-only support session endpoint accepts anonymous callers and mints sessions for any email, though the claim step still refuses the login.
- [DNS rebinding to a localhost API](dns-rebinding-local-api-startup-file.md): a desktop app trusts loopback, does not check the Host header, and hands out a session, so a web page can write a shell startup file.
- [Cross-tenant token deletion](graphql-token-deletion-across-tenants.md): a GraphQL mutation resolves a token by identifier alone, so one tenant can permanently delete another tenant's API token.
- [Health endpoint service map](health-endpoint-service-map.md): an unauthenticated health endpoint lists internal auth services and their status, and the names map to live routes.
- [Duplicate keys in offer verification](offer-verification-duplicate-keys.md): a dictionary comprehension drops a duplicate store entry, so verification can pass on a bundle that still carries an unverified entry.
- [PKCE verifier in localStorage](pkce-verifier-persistent-storage.md): the verifier is written to persistent storage as well as session storage, so it outlives the flow.
- [Playbook role assignment](playbook-create-role-assignment.md): creation stores a client-supplied member list, letting a plain team member grant the playbook admin role.
- [Recovery phrase overwrite](recovery-phrase-overwrite-with-session.md): a session token alone replaces the account recovery phrase, with no master password in the request.
- [Reset token leaked to a tracking beacon](reset-token-leaked-to-tracking-beacon.md): the reset page sends the full URL, token included, to an object-storage beacon, where it is kept in the access logs.
- [IDOR in a withdrawal form](state-changing-idor-withdrawal-form.md): a customer can file a withdrawal against another customer's order by changing the order number.
- [Heap buffers leaked on a throwing read](throwing-read-leaks-heap-buffers.md): raw allocations before a read that throws are never freed, so a malformed file leaks memory on every cycle.
- [Unauthenticated CI deploy listeners](unauthenticated-ci-deploy-listeners.md): public deploy webhooks accept anonymous events and disclose their deployment targets.
- [Webhook SSRF to the metadata service](webhook-url-internal-address-ssrf.md): webhook creation accepts loopback and link-local URLs, and the delivery client follows redirects to the metadata endpoint.

## Notes

No program, company, product, domain, token or personal detail is included. Each writeup stands on the observed requests and responses, with values replaced by placeholders.
