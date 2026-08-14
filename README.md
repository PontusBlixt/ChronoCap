# ChronoCap

Chronological PCAP aggregator. Takes two sets of packet captures — typically
from two different network interfaces or capture points — and merges them
into a single, strictly time-ordered output fileset. Similar in spirit to
`mergecap`, but instead of one large output file you get a rotating fileset
sized to your preference, with each file's name carrying the timestamp of
its first packet for easy chronological lookup.

Created by [Pontus Blixt](https://github.com/PontusBlixt). Licensed under
[Apache 2.0](LICENSE.md).

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
ChronoCap.exe -r1 C:\captures\eth0 -r2 C:\captures\eth1 -w C:\merged\chronocap.pcap -size 128 -verify
```

## Arguments

| Flag | Required | Default | Description |
|---|---|---|---|
| `-r1` | Yes | — | Directory containing the first capture set (`.pcapng` files) |
| `-r2` | Yes | — | Directory containing the second capture set (`.pcapng` files) |
| `-w` | No | `.` | Output path prefix — directory + filename prefix. Extension is always normalized to `.pcap` (see [Output file naming](#output-file-naming)) |
| `-size` | No | `64` | Target size per output file, in MB, before rotating to a new file. Must be greater than 0 |
| `-time` | No | `UTC` | `UTC` or `Local` — timezone used for the timestamp embedded in output filenames |
| `-verify` | No | off | Re-read the output after merging and confirm packet count, content, and chronological order all match what was written |
| `-debug-files` | No | off | Print every input file as it's opened (interface, index, path). Very verbose on large capture sets — for diagnosing a stuck/slow run, not routine use |

## Progress reporting and stall detection

For any run — merge or verify — ChronoCap runs a lightweight background
check that gives you two signals, so silence is never ambiguous:

- **Heartbeat:** every 60 seconds, `Still working — N packets processed
  so far, last: <file> frame <n>`. This covers the *entire* run, including
  the `-verify` pass — a common source of confusion, since verify reads
  the output back with no growing file to watch, which can otherwise look
  identical to a genuine hang. A clear `Merge complete... Starting
  verification` message is also printed the moment that phase begins, so
  you always know which part of the run is currently active.
- **Stall warning:** if no packet has been read for 5+ minutes, a warning
  naming the last confirmed file and frame is printed once (not repeated
  every check), and resets automatically if progress resumes.

This only ever detects and reports a stall — it can't recover from one or
skip past it automatically. ChronoCap has no checkpoint/resume
capability, so killing and restarting a run is still a manual decision;
`-debug-files` can help narrow down which input file to investigate
before you do.

The stall threshold is measured **per packet**, not per phase — a
`-verify` pass that legitimately takes hours on a huge capture will not
trigger a false warning as long as it keeps reading packets steadily.
Only a genuine multi-minute gap between individual packets triggers it,
regardless of which phase it happens in.

## Output file naming

```
<prefix>_<yyyy_mm_dd_hh_mm_ss>_<00001>.pcap
```

- Whatever extension you give `-w` (or none at all), ChronoCap normalizes
  it to `.pcap` — the correct name for the classic-pcap format it actually
  writes — and prints a one-line notice when it does so. This applies
  regardless of the precision (micro/nanosecond); `.pcap` covers both,
  distinguished internally by the file's magic number.
- The timestamp portion is taken from the **first packet written to that
  file**, in the timezone selected by `-time` (default UTC).
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
| 1.4 | `-size` now rejects zero/negative values with a clear error instead of silently rotating on every packet; output filename extension is now always normalized to `.pcap` (the accurate name for the format actually written), regardless of what's passed to `-w` |
| 1.5 | Added `-debug-files` for verbose per-file logging; added a background progress reporter — a heartbeat every 60s plus a stall warning after 5 minutes of no progress, covering the *entire* run including `-verify` (fixed a real gap where a slow verify pass gave zero console output and could look like a hang); a clear message now announces the merge→verify transition |
