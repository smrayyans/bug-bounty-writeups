# A session token alone is enough to overwrite the recovery phrase

## Summary

The endpoint that configures recovery accepted a request that carried a session token and nothing derived from the master password. A caller with nothing but a session received `200`, and the stored recovery hash was overwritten with a hash made from a phrase the caller picked. The owner's own phrase stopped verifying.

## Context

A password manager, used through a browser. Vault contents are encrypted client-side under a master password, and recovery works by storing a data key encrypted under a recovery phrase. The only access needed was a valid session token for the account, obtained by signing in normally. The endpoint itself asked for no master password and challenged no second factor.

## How I found it

### Where I started

Recovery is one of the few places in a password manager where a second secret is meant to stand in for the first. That makes it worth reading closely, because the whole premise of the feature is that the server should not be able to move the recovery material on its own.

### What the panel showed

Configuring recovery is a two-step experience. The panel asked for the master password before it would continue, which is the control you would expect to see. Watching the traffic showed that the password went to a separate validation call, and the request that actually configured recovery carried no password field at all: an envelope with an encrypted session, and a payload with the recovery blob.

### Testing the request directly

I replayed that request with my session and a recovery blob I built myself, and it answered `200` with an empty body. Then I put 60 random bytes in place of the blob — a random IV, a random tag and a random ciphertext — and got `200` again. Whatever the server was checking, the content of the blob was not part of it.

The control was the same call with the session taken out. It never reached the handler; the session filter refused it, which made the separation clean: only the session distinguishes the accepted call from the refused one.

### Checking the owner's side

A success response proves nothing was stored, so I looked at the account that owned the recovery phrase. The phrase recorded earlier now verified as `doesnt_exist`, and the account held the phrase material I had sent. That is stored state, not a cosmetic reply.

### The part I could not show

I also checked whether the planted phrase opened the vault. It did not. A session-only client is not given the account's data key, the vault sync returns the vault still encrypted, and the backup retrieval answers that the login token has expired. The overwrite is real; the vault is not opened by it.

## Proof of concept

The envelope carries the session; the payload carries the recovery material.

```http
POST /api/rest/authentication/recovery_setup HTTP/1.1
Host: vault.example
Content-Type: application/json

{
  "session": "<session token>",
  "payload": {
    "recovery_encrypted_data_key": "<60-byte blob>",
    "recovery_auth_hash": "<32-byte hash>"
  }
}
```

```http
HTTP/1.1 200 OK

```

`recovery_encrypted_data_key` is a 12-byte IV followed by AES-256-GCM ciphertext over the account's data key. Anything 60 bytes long is accepted, provided it is not all zeros; the tag is not verified. The SDK source derives the hash from the phrase with HKDF-SHA512, so material generated from any phrase produces an acceptable pair.

With 60 random bytes as the blob, the answer is the same:

```http
POST /api/rest/authentication/recovery_setup HTTP/1.1
Host: vault.example
Content-Type: application/json

{
  "session": "<session token>",
  "payload": {
    "recovery_encrypted_data_key": "<60 random bytes>",
    "recovery_auth_hash": "<32 random bytes>"
  }
}
```

```http
HTTP/1.1 200 OK
```

Without the session, the call does not reach the handler:

```http
POST /api/rest/authentication/recovery_setup HTTP/1.1
Host: vault.example
Content-Type: application/json

{
  "payload": {
    "recovery_encrypted_data_key": "<60-byte blob>",
    "recovery_auth_hash": "<32-byte hash>"
  }
}
```

```http
HTTP/1.1 401 Unauthorized

{ "error": "<key check failed>" }
```

The client retries with a new key; the session layer refuses the second attempt:

```http
HTTP/1.1 403 Forbidden

where    = <session filter>
details  = <no session>
```

## Steps to reproduce

Preconditions: an account with recovery already configured, so a phrase the owner knows is on file, and a session token for that account.

1. Sign in and confirm the recovery phrase verifies.
2. Lift the session token from the envelope the vault itself sends.
3. Submit the recovery-configuration request with that session and blob material derived from a phrase you picked.
   - **Expected:** a refusal, since no master password was presented to this endpoint.
   - **Actual:** `200` with an empty body.
4. Repeat with 60 random bytes as the blob. It also returns `200`.
5. Remove the session and resend. Expect the session filter to stop it; no handler runs.
6. On the owner's account, verify the phrase from step 1. Expect it to still verify. Actual: it reports that the object no longer exists, and the account now holds the caller's material.
7. As the session-only caller, ask for the data key and sync the vault. Expect the planted phrase to open it. Actual: no data key is returned, the sync response is still encrypted, and the backup call reports an expired login token.

## Impact

Someone holding a session token could swap the account's recovery capability for a phrase of their choosing, and the owner's existing phrase stopped working. The owner can only repair it by configuring recovery again, and no screen in the vault indicated a change. Email delivery was not tested; no notification appeared inside the product.

The caller did not obtain the vault; the data key is not returned to a session alone.

## Root cause and fix

The response behaves as if the handler checks only that a session is present and that the recovery blob is not all zeros: material built from a random IV and random ciphertext is accepted exactly like a well-formed blob. The password prompt in the panel belongs to a separate call whose result never reaches this request, so the observed behavior is consistent with the server trusting the client to have performed that check first.

Make the recovery-configuration request carry proof of the master password and check it on the server, so a session alone cannot satisfy it. Re-authentication and an owner notification on any change to recovery material are the obvious companions.

## What I learned

A prompt in the interface is not a control until the request that matters carries its result. The other thing I took from this one is to measure the claim down: the overwrite was easy to prove, while "and therefore the vault opens" was a separate claim the data-key calls did not support, so it stayed out.
