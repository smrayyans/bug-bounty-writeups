# A throwing read after raw allocations leaks heap buffers on every malformed file

## Summary

Two read paths in a proving library allocate heap buffers with `new[]` and then call a helper that throws on a short read. The `delete[]` calls sit after the throwing call, so every read that fails leaves the buffers allocated. A crafted data file that fails at the right offset leaks roughly 800 bytes each time the worker touches it.

## Context

A proving component of a storage-based proof-of-work system. A long-running worker process reads large data files on a fixed cadence and proves against them. The finding is in the C++ source of the prover: no request, no account, and no network position are involved. The trigger is a file the worker is pointed at.

## How I found it

### Where I started

I was reading the prover's read path for exception safety, not looking for a logic bug. Large-file parsers are a good place to look because they mix raw buffers with reads that can fail halfway, and the failure path is easy to write once and never test.

### What the code did

The line-point reader allocates three buffers at the top of the function, then reads into each one with a helper that throws `std::runtime_error` when the stream reports `fail()`. The three `delete[]` calls are at the bottom. Nothing between the allocations and the frees is exception-safe, so a throw at any of the reads after the first allocation leaks whatever was allocated so far.

The class constructor has the same shape: one buffer allocated before a loop, and a read inside the loop that can throw.

### Why a file can trigger it

The read helper throws on `fail()`, which covers end of file, a short read and a disk error. A file with a valid header and a truncated body reaches the read that fails, every time the worker proves against it. No corruption of memory, no crash on the failing call; the buffers are simply never freed.

### Measuring it

I added up the buffer sizes for a default-size file: about 15 bytes, 107 bytes and 700 bytes, so roughly 800 bytes per failing call. With the worker's roughly nine-second cadence that is about 330 KB an hour, or 7.7 MB a day.

### What it meant

On a normal machine this is not an exploit; it is a slow leak that becomes an outage on the hardware where it is least welcome.

## Proof of concept

The two allocations-before-throw patterns, with identifiers generalized.

```cpp
uint64_t readLineEntry(std::ifstream& file, uint8_t table, uint64_t position) {
    auto* line_point = new uint8_t[line_point_size + 7];        // allocation 1
    readExact(file, line_point, line_point_size);               // can throw

    auto* stubs = new uint8_t[stubs_size_bits / 8 + 7];         // allocation 2
    readExact(file, stubs, stubs_size_bits / 8);                // can throw

    auto* deltas = new uint8_t[max_deltas_size_bits / 8];       // allocation 3
    readExact(file, &encoded_deltas_size, sizeof(encoded_deltas_size)); // can throw here,
                                                                // after all three allocations

    // ... decode ...

    delete[] line_point;   // not reached if any read above threw
    delete[] stubs;        // not reached
    delete[] deltas;       // not reached
    return final_line_point;
}
```

```cpp
Prover::Prover(std::ifstream& file, uint32_t table_count) {
    auto* c2 = new uint8_t[c2_size];                            // allocation before the loop
    for (uint32_t i = 0; i < c2_entries - 1; i++) {
        readExact(file, c2, c2_size);                           // can throw inside the loop
        // ...
    }
    delete[] c2;          // not reached if a read threw
}
```

The helper that throws:

```cpp
static void readExact(std::ifstream& file, uint8_t* target, uint64_t size) {
    file.read(reinterpret_cast<char*>(target), size);
    if (file.fail()) {
        throw std::runtime_error("short read");
    }
}
```

A file that reproduces it:

1. Build a valid header so the constructor and the table lookups succeed.
2. Truncate the body at a point inside the line-point record, after the first fields have been read and before the delta data.
3. Point the worker at the file and let it run one proof cycle. Each cycle leaks the buffers.

Measured budget for a default-size file: about 15 + 107 + 700 bytes, roughly 800 bytes per failing call, one call every nine seconds or so.

| Rate | Value |
|---|---|
| Per failing call | ~800 bytes |
| Per hour | ~330 KB |
| Per day | ~7.7 MB |
| Time to exhaust 1 GB | ~130 days |
| Time to exhaust 16 GB | ~5-6 years |

The rate multiplies per file: ten crafted files leak ten times as fast.

## Steps to reproduce

Preconditions: a build of the prover and a worker that reads files from a directory.

1. Construct a data file with a valid header and a body truncated inside the line-point record.
2. Add it to the worker's file set.
3. Run the worker and watch its resident memory across proof cycles.
   - **Expected:** memory flat, because a failed read should not retain anything.
   - **Actual:** resident memory rises by roughly 800 bytes per cycle that touches the file, and does not fall back.
4. Repeat with several crafted files to confirm the rate scales with the number of files.

## Impact

A truncated or crafted data file held by a worker makes it leak roughly 800 bytes per proof cycle for as long as it stays in the set. On a large machine that takes years to matter; on a constrained or long-running worker it reaches an out-of-memory crash in months, and each additional crafted file multiplies the rate. When the worker dies, proving stops until it is restarted, and the operator sees a memory exhaustion rather than a bad file.

No memory is corrupted and nothing is disclosed. The cost is availability and the work lost while the process is down.

## Root cause and fix

The observed behavior is consistent with manual buffer lifetime: the buffers are freed only on the normal return path, and the read helper signals failure by throwing, so any failed read skips every free after it.

Use a `std::vector<uint8_t>` or a `std::unique_ptr<uint8_t[]>`, either of which frees the buffer during unwinding, and pass `.data()` or `.get()` to the read helper. The same change applies to the constructor's buffer. There is no case in these functions where a raw `new[]` buys anything that the wrappers do not.

## What I learned

The interesting part of a read path is not the read, it is everything that runs when the read fails. Once I started checking the failure path on purpose, the allocation order made the bug obvious: three allocations, three reads, three frees at the end, and no way out of the middle.
