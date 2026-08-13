# ChronoCap

Chronological PCAP aggregator. Takes two sets of packet captures — typically
from two different network interfaces or capture points — and merges them
into a single, strictly time-ordered output fileset. Similar in spirit to
`mergecap`, but instead of one large output file you get a rotating fileset
sized to your preference, with each file's name carrying the timestamp of
its first packet for easy chronological lookup.

## Requirements

ChronoCap is distributed as a standalone executable (built with PyInstaller)
— end users don't need Python or scapy installed.

For building from source / development:

- Python 3.9+
- [scapy](https://scapy.net/) — developed and tested against **scapy
  2.7.0**. Newer versions should work; if timestamps ever look wrong after
  a scapy upgrade, see [Known limitations](#known-limitations) below before
  assuming it's a ChronoCap bug.

```
pip install scapy
```

## Basic usage

```
ChronoCap.exe -r1 <path> -r2 <path> -w <output-path-prefix> [options]
```

**Example:**

```
ChronoCap.exe -r1 C:\captures\eth0 -r2 C:\captures\eth1 -w C:\merged\chronocap.pcapng -size 128 -verify
```

## Arguments

| Flag | Required | Default | Description |
|---|---|---|---|
| `-r1` | Yes | — | Directory containing the first capture set (`.pcapng` files) |
| `-r2` | Yes | — | Directory containing the second capture set (`.pcapng` files) |
| `-w` | No | `.` | Output path prefix — directory + filename prefix + extension for output files |
| `-size` | No | `64` | Target size per output file, in MB, before rotating to a new file |
| `-time` | No | `UTC` | `UTC` or `Local` — timezone used for the timestamp embedded in output filenames |
| `-verify` | No | off | Re-read the output after merging and confirm packet count, content, and chronological order all match what was written |

## Output file naming

```
<prefix>_<yyyy_mm_dd_hh_mm_ss>_<00001><extension>
```

- The timestamp is taken from the **first packet written to that file**,
  in the timezone selected by `-time` (default UTC).
- The 5-digit number increments once per rotation and reflects how many
  files ChronoCap actually created — not a fixed counter, so it stays
  accurate even if a rotation happens to land exactly on the last packet.
- UTC is the default specifically because local time can jump backward
  across a DST transition, which would break the chronological sort order
  of the filenames themselves. Use `-time Local` if you specifically want
  wall-clock time matching your own timezone for easier cross-referencing
  during analysis, and are aware of that tradeoff.

## What `-verify` actually checks

Verification re-reads the entire output fileset after merging and performs
three independent checks:

1. **Packet count** — total packets in the output equals total packets read
   from `r1` + `r2`.
2. **Content integrity** — an order-independent checksum (XOR of a blake2b
   hash per packet) computed during the merge is compared against the same
   checksum recomputed from the output files. Catches dropped, duplicated,
   or corrupted packets regardless of where in the stream they'd occur.
3. **Chronological order** — timestamps across the whole output fileset,
   read in file order, must never go backward.

If ordering is broken, the report identifies exactly where: the output
file and frame number (1-based, matching what Wireshark shows if you open
that specific file directly) for both the offending frame and the one
immediately before it, plus both timestamps.

**Cost:** verification adds a full second read pass over the output plus
a hash computation per packet, so it meaningfully increases runtime on
large captures. It's off by default for that reason — turn it on when you
want the assurance, skip it for routine runs once you trust the pipeline.

**Exit codes:** `0` success, `1` a setup error (bad paths, linktype
mismatch), `2` verification ran and found a problem.

For the full explanation of why blake2b + XOR specifically, what it does
and doesn't guarantee, and how violations are located, see
[docs/VALIDATION.md](docs/VALIDATION.md).

## Known limitations

These are understood tradeoffs or edge cases, not fixed bugs — worth
knowing about before you rely on ChronoCap for something high-stakes:

- **Linktype consistency is only checked using the first file** in each of
  `r1` and `r2`. If a folder contains multiple capture files and a later
  one (not the first) has a different linktype than the rest, that's not
  detected.
- **Output filename extension doesn't match the actual format.** Files are
  written as classic nanosecond-precision pcap (not real pcapng), regardless
  of what extension you use in `-w`. Most tools (including Wireshark)
  identify format from the file's magic number rather than its extension,
  so this works in practice, but the naming is technically inaccurate.
- **The `-verify` checksum is XOR-based**, which is order-independent by
  design but is theoretically (not practically) susceptible to a
  specifically-constructed lost-packet-plus-duplicate-packet pair that
  cancels out. Not something you'd hit by accident.
- **scapy's `tsresol` handling has a known upstream edge case** (scapy
  issue #2342): some capture tools encode an interface's timestamp
  resolution in a way that makes scapy silently fall back to microsecond
  resolution instead of the file's actual (e.g. nanosecond) resolution,
  with no error raised. If output timestamps ever look systematically off
  vs. the same packets in Wireshark, check this before assuming a
  ChronoCap bug — verified against a known-good capture in a Python shell
  (checking `metadata.tsresol` directly) is the fastest way to confirm.

For the full timestamp-handling story (nanosecond vs. microsecond, why
`-verify` briefly reported false ordering violations, and how it's now
fixed for good), see [docs/TIMESTAMPS.md](docs/TIMESTAMPS.md).

## Version history

| Version | Changes |
|---|---|
| 1.0 | Initial merge logic (chronological interleave, size-based rotation) |
| 1.1 | Fixed hardcoded `linktype`; guard against empty capture-file sets; exception-safe cleanup on crash |
| 1.2 | Output filenames now embed the timestamp of each file's first packet (lazy file-opening to support this); fixed a trailing-empty-file edge case in the file count |
| 1.3 | Added `-time Local\|UTC`; added `-verify` with count/content/ordering checks and violation location reporting; fixed a nanosecond-vs-microsecond misinterpretation bug in verify's re-read of ChronoCap's own output; consolidated the input-reading and verify-reading code paths into one (`PacketReader`) to prevent that class of bug from recurring |
