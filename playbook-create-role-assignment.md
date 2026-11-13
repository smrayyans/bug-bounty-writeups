# Playbook creation accepts a client-supplied member list and grants an admin role

## Summary

The playbook creation endpoint saved the `members` array from the request body without asking whether the caller was permitted to set it. It only asked whether the caller could create a playbook at all. A plain team member, holding neither administrator rights nor permission to manage roles, could create a playbook and make any user its administrator.

## Context

A team collaboration platform with a playbooks feature. Each playbook keeps a membership list and its own roles, separate from team roles. The access needed was an ordinary team member account; the account granted the role held no administrator standing on the playbook and no permission to manage roles elsewhere.

## How I found it

### Where I started

I was looking at object-level permissions inside playbooks, because the feature layers its own role model on top of the team one. Anything that accepts a membership list from a client is worth reading: that list is where a product decides who may add whom, and therefore where the decision is easiest to leave out.

### First impression

Creating a playbook sends a body with a title, a team, a visibility flag and, optionally, a member array. Each entry carries a user and a set of roles. In the interface, the creator ends up as the playbook's only administrator. Sending the array myself was the obvious next step.

### The attempt

I logged in as a plain member and created a playbook whose member array named a second user with the administrator role. The response was `201 Created`. Reading the playbook back showed the array stored exactly as sent, with the submitted roles written to the record. The second user administered a playbook I had just created, and I was not on it at all.

### The comparison that made it a bug

An ordinary member being allowed to create a playbook is normal. What was not normal is that creation skipped the check the update path performs: on update, altering the member list requires the membership-management permission, and any role above a plain member requires role-management. The create handler never ran that check.

Two other paths in the same family behaved properly: import discards whatever list you send and leaves the importer as the sole administrator, while duplicate makes the copier its administrator. Plain creation was the one path that kept what the client sent.

### Bounding it

I checked what the bypass does not do. No data crosses team lines, and no existing playbook can be taken over, since creation refuses a body that carries an identifier up front and team membership is still required. What it gives is unauthorized role assignment on a playbook of the caller's own making.

## Proof of concept

The caller is a plain member with an ordinary login session. The account being made administrator is `<USER_ID>`, and role strings are replaced with `<admin>` and `<member>`.

```http
POST /plugins/playbooks/api/v0/playbooks HTTP/1.1
Host: workspace.example
Authorization: Bearer [REDACTED]
Content-Type: application/json

{"title": "poc",
 "team_id": "<TEAM_ID>",
 "public": true,
 "members": [ { "user_id": "<USER_ID>", "roles": ["<admin>", "<member>"] } ]}
```

```http
HTTP/1.1 201 Created

{ "id": "<PLAYBOOK_ID>" }
```

Reading it back shows the array was stored as sent, not sanitized:

```http
GET /plugins/playbooks/api/v0/playbooks/<PLAYBOOK_ID> HTTP/1.1
Host: workspace.example
Authorization: Bearer [REDACTED]
```

```http
HTTP/1.1 200 OK

{ "id": "<PLAYBOOK_ID>",
  "title": "poc",
  "public": true,
  "members": [ { "user_id": "<USER_ID>",
                 "roles": ["<admin>", "<member>"] } ] }
```

## Steps to reproduce

Preconditions: two accounts on the same team, one of them an ordinary member, and a session for the member account.

1. Sign in as the member and keep the session token from the login response.
2. Create a playbook, sending a member array that names the second account with the administrator role.
   - **Expected:** a refusal, because assigning that role requires the role-management permission the caller does not hold.
   - **Actual:** `201 Created`.
3. Read the playbook back. The member array holds the second account with the roles that were sent, with the caller absent from it.
4. As a control, send the same array through the import endpoint and read the imported playbook back. The supplied list is dropped and the importer is the only administrator.

## Impact

A plain member could carry out a role assignment the product reserves for the role-management permission, and could put arbitrary team users on a new playbook without the membership permission. The granted role controls membership, roles, properties and the playbook's visibility, so a caller could create a playbook whose controlling role belongs to an account they picked.

The scope is the playbook that was created. No cross-team data was read, no existing playbook was taken over, and team membership is still required to create one. The update path could not be exercised on the instance available to me; its check is visible in the plugin source.

## Root cause and fix

The response behaves as if the create handler takes the membership list from the request body and writes it without comparing it to what the caller may set. The update handler on the same object does make that comparison, requiring membership-management for any change to the list and role-management for any role above a plain member, and create does not.

Run the same check on creation: compare what was sent with the server's default of the creator as sole administrator, require membership-management for any change and role-management for anything above a plain member. Refuse the request rather than silently stripping entries, so the response cannot be used to probe which parts were accepted.

## What I learned

The permission model was not missing; it was implemented on one path and not on its sibling. When a feature has create, update, import and duplicate routes that all take the same object, the useful question is not whether a check exists, but whether every route that takes this field runs it. The one that quietly keeps what the client sent is the one to open.
