# IDOR in a withdrawal form lets one customer file a withdrawal against another's order

## Summary

A customer-facing withdrawal form built a withdrawal request from whatever order number the request body contained. Nothing in the response tied that number to the logged-in customer, and a number that could not exist produced the same result. Any authenticated customer could file a withdrawal request against another customer's order.

## Context

An e-commerce web application with customer accounts, order history, and a self-service form for withdrawing from a purchase contract. The only access required is a normal customer account. No special role, no elevated privileges, no knowledge of the victim beyond their order number.

## How I found it

### Where I started

I was working through the order-management area as an ordinary customer, looking at the parts that change something rather than the pages that only display. Most of that area is read-only: an order list, an order detail page, a cancel page. The withdrawal form was the one place where a customer asks for something to actually happen to a contract, so it was the one worth reading closely.

### First impression

The other order pages addressed an order by a long opaque identifier, and each of them only ever showed me my own. The withdrawal form didn't ask for anything like that. It rendered a plain text field for an order number and posted it as `orderNumber`, with a free-text `reason` next to it. A number typed by the user, going straight into a request body.

### Attempts

I submitted the form with my own order number first, just to see what the request looked like. It posted to the form's action path and came back as a redirect to a confirmation page. Nothing surprising yet.

So I went back to the rest of the order area with an identifier belonging to my second account. The detail page wouldn't render and pushed me back to a different page, the cancel page errored out, and that account's order list came back as an empty array. Every read path I touched refused me. That contrast is what sent me back to the form.

### The one action that changes something

If every page that displays an order checks the session against that order, then the action that changes an order should check it too. And this form took the number as free input, which meant I could put any number in it. I sent the same request again with the number replaced by the first account's order number, leaving the request otherwise unchanged from a legitimate submission by the second account.

### Confirming with a fake order number

I got the same redirect to the confirmation page. That on its own could have been a page that always redirects, so I sent a number I made up, one that could not correspond to any order. Same redirect, same confirmation. Whatever the server was doing with the number, it wasn't telling the two apart.

The acknowledgment email settled it. It arrived at the account that submitted the form and quoted the other account's order number.

### What it meant

A withdrawal request now sat against an order I didn't own, created from my session. It was there whether or not anything came of it.

## Proof of concept

Account B submits the withdrawal form with Account A's order number. Everything else in the request is Account B's own.

```http
POST /withdraw/?/submit HTTP/2
Host: target.example
Cookie: session=[REDACTED]
Origin: https://target.example
Content-Type: application/x-www-form-urlencoded

orderNumber=1001&reason=test
```

```http
HTTP/2 302
location: https://target.example/withdraw/confirmation
```

`1001` is Account A's order, and the request is otherwise a legitimate submission by Account B.

A number that cannot exist produces the identical response:

```http
orderNumber=9999&reason=test
```

```http
HTTP/2 302
location: https://target.example/withdraw/confirmation
```

## Steps to reproduce

1. Create two customer accounts, A and B. Place an order on A so it has an order number. B needs no order of its own.
2. Log in as B and open the withdrawal form.
3. Turn on an intercepting proxy.
4. Submit the form once with any value and catch the request in the proxy.
5. Replace `orderNumber` with A's order number. Leave the rest of the request untouched.
6. Forward the request.
   - **Expected:** a rejection, the same way the order detail and cancel pages reject a foreign identifier.
   - **Actual:** `302` to the confirmation page. The withdrawal request is accepted.
7. Follow the redirect. B sees the confirmation page, and B's account receives an acknowledgment email quoting A's order number.

For the existence check, repeat step 5 with a number that cannot exist. The response is the same `302` to the same confirmation page.

## Impact

A customer could create a withdrawal request against another customer's order number, from their own session, without the order owner's knowledge. The request lands in the support queue attributed to the submitting account and quoting the victim's order number.

What was demonstrated is the creation of that request. The application routes withdrawal requests to a person for review before anything is done to the contract, so whether the victim's order actually moved depended on that review, not on the request alone.

## Root cause and fix

Nothing in the response indicates that the posted order number was resolved against the session: a number that cannot exist gets the same success redirect as a real one, and so does an order belonging to another customer. The observed behavior is consistent with the withdrawal record being written from the request body without an ownership lookup, on an action where the order pages already behave as if they perform one.

The fix is to resolve the order number before the record is written, confirm it is owned by the session's customer, and turn down anything else with a generic message, matching what the other order pages do.

## What I learned

The check I was looking for wasn't missing all over the order area. It was missing on exactly one request. Comparing the action that changes something against the pages that only read the same object is what told me where to look.
