# OAuth PKCE verifier is written to localStorage and outlives the browser session

## Summary

An OAuth SDK extension stored the PKCE `code_verifier` in `sessionStorage` and then wrote the identical record to `localStorage`. Because `localStorage` is persistent, the verifier, the OAuth state value and the redirect metadata survived tab closes and browser restarts. Any script on the same origin could read them, and an abandoned flow left them there indefinitely.

## Context

A client-side authentication SDK used by applications to run OAuth redirect flows in the browser. The code ships as published package source and is bundled into the production login page, so the behaviour is visible in the delivered JavaScript. No account, session or privilege was involved; this is a storage-handling finding in the client library.

## How I found it

### Where I started

A browser OAuth flow has one secret that must stay short-lived: the PKCE verifier. It proves that the client finishing the code-for-token exchange is the one that started the flow, and it only does that job while nobody else can read it. So when I review an OAuth client, the storage calls are the first thing I look for.

### What the code did

The redirect start path wrote the verifier and its metadata twice:

```js
sessionStorage.setItem(PKCE_STORAGE_KEY, JSON.stringify({ codeVerifier, ...meta }));
localStorage.setItem(PKCE_STORAGE_KEY,   JSON.stringify({ codeVerifier, ...meta }));
```

The comment beside the first line explained that `sessionStorage` survives the redirect without entering the iframe, which is correct. The second line duplicates the same record into persistent storage, which removes the property the first line was chosen for.

The read path checked session storage first and fell back to local storage, so the `localStorage` write is not needed for the flow to work. Cleanup runs only after a flow completes.

### Confirming it in a browser

Source is enough to see the defect, but I wanted the persistence to be observable, so I built a self-contained page that repeats the write, the read and a simulated tab close.

### The result

After the simulated restart, `sessionStorage` was empty and `localStorage` still held the verifier, the state value and the redirect metadata. That is the whole finding: the record outlives the window it was designed for.

## Proof of concept

The pattern, reduced to the two storage calls and the cleanup condition.

```js
// start of the redirect flow
if (successResult?.pkceMetadata && codeVerifier) {
  // sessionStorage survives the same-tab redirect.
  sessionStorage.setItem("pkce_record", JSON.stringify({
    codeVerifier,
    ...successResult.pkceMetadata,   // state, redirectUri, appID, provider
  }));

  // unnecessary: the same record, in persistent storage
  localStorage.setItem("pkce_record", JSON.stringify({
    codeVerifier,
    ...successResult.pkceMetadata,
  }));
}
```

The read path prefers session storage and falls back to local storage:

```js
const storedInSession = sessionStorage.getItem("pkce_record");
const storedInLocal   = localStorage.getItem("pkce_record");
sessionStorage.removeItem("pkce_record");
localStorage.removeItem("pkce_record");

const metadata = storedInSession
  ? JSON.parse(storedInSession)
  : storedInLocal
    ? JSON.parse(storedInLocal)
    : undefined;
```

Cleanup is reached only when a flow completes. A caller who abandons it, by closing the tab, losing the network or navigating away, leaves the record in place.

The browser demonstration:

```js
// inputs: KEY is the storage key; state and redirectUri come from the flow
const KEY = "pkce_record";
const state = "<state>";
const redirectUri = "<redirect-uri>";

// 1. write, as the redirect start does: a 128-character base64url verifier
const bytes = crypto.getRandomValues(new Uint8Array(96));
const codeVerifier = btoa(String.fromCharCode(...bytes))
  .replace(/\+/g, "-").replace(/\//g, "_").replace(/=+$/, "");
sessionStorage.setItem(KEY, JSON.stringify({ codeVerifier, state, redirectUri }));
localStorage.setItem(KEY,   JSON.stringify({ codeVerifier, state, redirectUri }));

// 2. read, as any same-origin script can
const stolen = JSON.parse(localStorage.getItem(KEY));

// 3. survive a tab close: sessionStorage is cleared, localStorage is not
sessionStorage.clear();
console.log(localStorage.getItem(KEY));
```

## Steps to reproduce

1. Open the redirect flow in a browser with the developer console available.
2. Start an OAuth sign-in and stop at the provider page. In application storage, confirm the key exists in both session storage and local storage.
3. Read the local-storage entry. It holds the 128-character verifier plus the state value, redirect URI, application ID and provider.
4. Close the tab and reopen the site. Expect the entry to be gone. Actual: it is still there, unchanged.
5. Repeat the flow and abandon it before completion. Confirm the entry remains; nothing in the library removes it on timeout.

## Impact

The verifier and the flow metadata stayed readable to any JavaScript on the origin for as long as the record was present, rather than for the duration of the redirect. An attacker who obtained the authorization code for a flow could pair it with a verifier read from local storage, and the window for that is the lifetime of the stored record rather than the length of the redirect.

On its own this is a storage weakness, not an account takeover. Reading the entry requires script execution on the origin, and completing the exchange requires the authorization code as well. Those are separate conditions, and neither is demonstrated here.

## Root cause and fix

The observed behavior is a redundant dual write: the redirect flow needs one store that survives the redirect, and the code writes the record to two. The persistent copy adds nothing, since the read path already prefers the ephemeral one, and the only cleanup runs on successful completion.

Remove the `localStorage` write for the verifier and keep the record in `sessionStorage` alone. Clear it on every exit path, including abandonment, and give it a short expiry. Keeping the verifier in memory is better still where the flow allows it.

## What I learned

The proof that the persistent write was unnecessary came from the read path, not the write path. Retrieval tried session storage first and only fell back to local storage, so the local copy could be deleted without breaking the redirect. When two lines do the same job, the fallback order tells you which one is load-bearing.
