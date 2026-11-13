# Duplicate keys in offer verification let a signed offer carry an unverified entry

## Summary

An offer-verification function collapsed the offer's store entries into dictionaries keyed by store identifier. When an offer contained two entries for the same store, the second silently replaced the first, and the comparison ran against the surviving entry only. The rest of the bundle still held both, so a verification that passed described less than the transaction contained.

## Context

A wallet implementation for a decentralised data store. Offers carry maker and taker entries, each naming a store and the value it proves, and the wallet checks those entries against reference data before presenting an offer to the user. The finding is in that verification function; no account or network position is involved.

## How I found it

### Where I started

Offer verification is the part of a wallet that decides whether the thing the user is about to sign matches the thing they were shown, so I read it looking for places where the comparison could cover less than the bundle.

### The shape of the check

The function builds two dictionaries from the offer: one mapping a store identifier to its new root, one mapping a store identifier to the values it proves. Both are dictionary comprehensions over the same list of entries. It then compares those dictionaries against the reference data it was given.

That structure assumes one entry per store. A list of entries does not guarantee that.

### The question I had

If the same store appears twice, what does the comprehension do? Python keeps the last value for a repeated key. So the dictionary holds one entry, and the comparison checks that one.

### Confirming it

I built a summary with two entries for the same store: one with a root I called the malicious root and a dependency I called the secret, one with a benign root and a public dependency. Running the same comprehensions over it produced a single-entry dictionary holding the benign pair. The malicious entry had not been rejected; it had been dropped from the view the verifier compares.

The source also carries a note in the same function asking whether duplicate entries should be checked for consistency, which told me the case was known and unhandled.

### What it meant

The verifier can return success for a bundle that contains an entry it never looked at.

## Proof of concept

The aggregation, reduced to the two comprehensions. Names are generalized; the structure is the point.

```python
# one dictionary entry per store, last write wins
maker_from_offer = {
    entry.store_id: entry.claimed_root
    for entry in summary.offered
}

taker_from_offer = {
    dependency.store_id: [hash32(value) for value in dependency.values]
    for entry in summary.offered
    for dependency in entry.dependencies
}
```

With one entry per store, the comparison behaves as intended:

```python
maker_from_offer = {store_a: good_root}
maker_from_reference = {store_a: good_root}
# equal, no error
```

With two entries for the same store, the first is overwritten before the comparison:

```python
summary.offered = [
    entry(store_a, claimed_root=malicious_root, dependencies=[secret_value]),
    entry(store_a, claimed_root=good_root,     dependencies=[public_value]),
]

maker_from_offer  # {store_a: good_root}       <- first entry discarded
taker_from_offer  # {store_a: [public_value]}  <- first entry discarded

maker_from_offer == maker_from_reference  # True
taker_from_offer == taker_from_reference  # True
```

The function returns without raising, while `summary.offered` still holds both entries. Nothing in the verifier removes the discarded one from the bundle it was handed.

## Steps to reproduce

Preconditions: the wallet library checked out locally, and reference data for a store the caller controls.

1. Build a summary whose offered list holds two entries for the same store: one carrying a different root and different dependency values, one matching the reference data.
2. Call the verification function with that summary and reference data matching the second entry.
   - **Expected:** an error, because the offer describes two different states for one store.
   - **Actual:** verification succeeds; the comparison only saw the second entry.
3. Inspect the offered list after the call. Both entries are still there.
4. Repeat with the malicious entry second and the benign entry first. The same collapse happens in the other direction, which shows the outcome depends on ordering rather than on consistency.

## Impact

An offer that names the same store twice passes verification as long as one of its entries matches the reference data. The verification result therefore describes a subset of the bundle, not the bundle. A user relying on that result to decide whether to sign is deciding about a transaction that still contains the entry the verifier discarded.

The demonstration is at the function level: the dictionaries collapse and the comparison succeeds. A full signing run through the wallet interface was not performed, so what the wallet does with the surviving bundle after verification is outside what this writeup shows.

## Root cause and fix

The observed behavior is consistent with the verifier normalising a list into a dictionary keyed by store identifier, which cannot represent the duplicate case, and then comparing the normalised form rather than the original entries. A note in the function already asks whether duplicates should be checked for consistency.

Reject an offer that names the same store more than once, or verify every entry against its own reference rather than comparing collapsed maps. The check should compare counts as well as values, so a duplicate cannot reduce the set being verified.

## What I learned

Verification code has its own version of the confused-deputy problem: the thing it checks and the thing that gets signed have to be the same object. Here the check built a smaller copy of the data and compared that. Reading a check for what it might not be looking at is more useful than reading it for whether it is correct on the inputs it expects.
