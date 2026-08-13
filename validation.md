# Validation (`-verify`) in ChronoCap

This document explains what `-verify` actually guarantees, how it's
implemented, and — just as importantly — what it does *not* guarantee.

## The problem it solves

ChronoCap's whole job is to take packets from `r1` and `r2` and rewrite
them in a new, interleaved chronological order across a fresh set of
output files. That makes "did the merge work correctly" harder to check
than a simple before/after diff: you can't compare byte offsets, because
the packets are deliberately not in their original positions anymore.

`-verify` checks three independent things instead, each catching a
different failure mode:

1. **Count** — did every packet make it into the output?
2. **Content** — is every packet's data intact (not dropped, duplicated,
   or corrupted)?
3. **Order** — does the output actually read chronologically, front to
   back?

None of these three implies the others. A merge could get the count
right but corrupt a packet's bytes; it could preserve every packet
perfectly but write them in the wrong order. `-verify` runs all three
because you want to know *which* invariant broke, not just that
something did.

## Check 1 & 2: count and content, via a streaming checksum

The straightforward way to check "is this the same set of packets,
regardless of order" would be to store every packet's hash in a list (or
set) from the source, then compare it against a list from the output.
That works, but costs memory proportional to packet count — for a
capture with millions of packets, that's a lot of hashes held in RAM
simultaneously.

Instead, ChronoCap computes a **single running XOR-aggregated hash**:

```python
def hash_packet(packet_bytes):
    digest = hashlib.blake2b(packet_bytes, digest_size=16).digest()
    return int.from_bytes(digest, "big")

checksum ^= hash_packet(packet_bytes)   # once per packet, source and output
```

### Why blake2b

blake2b is a cryptographic hash function in Python's standard library
(`hashlib`) — no external dependency. Two packets with even a single
differing bit produce completely different hashes (no gradual
"similarity"), so a hash match is, for practical purposes, proof of
identical content. 16 bytes (128 bits) of digest is far more than enough
collision resistance for this use case — we're not defending against a
malicious adversary, just catching accidental data loss or corruption.

### Why XOR specifically

XOR is commutative and associative:

```
a ^ b ^ c  ==  c ^ a ^ b  ==  b ^ c ^ a
```

So the *order* packets are XOR'd in doesn't affect the final result.
ChronoCap XORs every source packet's hash together while merging (in
whatever order they're written), and separately XORs every output
packet's hash together while re-reading (in output order, which is
different from source order by design). If the two running totals match,
the same multiset of packets exists on both sides — with **O(1) memory**,
just one running integer, regardless of how many millions of packets are
involved.

### Why XOR *and not* addition

Addition (`+=`) would also be order-independent, but XOR has a specific
extra property relevant here: identical values cancel out completely
(`x ^ x = 0`). If a packet were somehow duplicated in the output, its
hash would appear twice in the output's checksum and cancel itself out —
which sounds like a weakness, but is exactly why the checksum check is
never used alone. It's always combined with the count check: a
duplicated packet changes the output's packet count even if the checksum
happens to look unaffected. The two checks cover each other's blind
spots.

### The honest limitation

It's theoretically possible to construct an adversarial scenario where
one packet is dropped and a different packet is duplicated such that
their hashes cancel out in the XOR *and* the net count stays the same
(one lost, one duplicated). Both checks would miss this simultaneously.
This isn't something that happens by accident in real network
troubleshooting — it would require deliberately engineering that exact
combination — but it's worth naming plainly rather than overselling what
the checksum proves. The tradeoff (constant memory vs. this narrow
theoretical gap) is the right one for this tool's actual use case.

## Check 3: chronological ordering, with exact location reporting

The ordering check reads the entire output fileset in file order and
confirms timestamps never decrease:

```python
if previous_time is not None and not (previous_time <= timestamp):
    # violation
```

When a violation is found, it's not enough to say "somewhere, something
broke." Each recorded violation captures:

- **`packet_index`** — the global 1-based count across the entire
  output, useful for cross-referencing against other diagnostics.
- **`file`** and **`frame_in_file`** — which specific output file, and
  the frame number *within that file* (1-based). This deliberately
  matches what Wireshark shows if you open that exact file directly —
  not a global frame number, since ChronoCap's output is split across
  multiple files.
- Both the offending frame's location/timestamp **and** the immediately
  preceding frame's — because an ordering break is a relationship
  between two adjacent frames, and you need both to understand what
  happened.

To avoid flooding the report on a badly-broken capture, violations are
capped at `max_violations` (default 20) entries — but the *total* count
found is always reported separately, so you know whether you're looking
at the complete picture or a sample.

## Implementation note: one read path, not two

`verify_output()` re-reads the output fileset using the exact same
`PacketReader` class that reads the `r1`/`r2` input files during the
merge — not a separate, hand-rolled read loop. This matters because of
what we learned the hard way (see [TIMESTAMPS.md](TIMESTAMPS.md)): the
classic-pcap nanosecond-vs-microsecond handling is easy to get subtly
wrong, and having two independent implementations of "read a pcap file
and interpret its timestamp" meant they could silently drift out of
sync — which is exactly what caused the false-positive ordering
violations we debugged. Consolidating to one shared read path doesn't
just remove duplicate code; it removes an entire class of future bug.

## Cost and when to use it

Verification adds:

- A full second read pass over the entire output fileset.
- One blake2b hash computation per packet (in both the merge pass and
  the verify pass).

This is meaningfully more expensive on large captures. It's off by
default for that reason. Recommended usage: turn it on when validating a
new environment, a new capture source, or after any change to ChronoCap
itself; skip it for routine day-to-day runs once you trust the pipeline
is behaving correctly in your environment.

## Exit codes

| Code | Meaning |
|---|---|
| `0` | Success (and, if `-verify` was used, all checks passed) |
| `1` | A setup error — bad paths, linktype mismatch, no capture files found |
| `2` | `-verify` ran and found a problem in at least one of the three checks |

The distinct exit code for verification failures (`2`, separate from `1`)
is intentional — it lets you script around ChronoCap and distinguish
"the tool couldn't even run" from "the tool ran but flagged a data
problem."

## Edge case: zero packets

If the merge produces zero output packets (e.g. genuinely empty input
captures), `verify_output()` short-circuits with `count_ok = (expected
== 0)`, `checksum_ok = (expected == 0)`, and `ordering_ok = True` rather
than attempting to open a file that was never created.
