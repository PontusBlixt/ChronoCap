# Timestamps in ChronoCap

This document covers how ChronoCap represents, reads, and writes packet
timestamps — including the details that bit us once already. If output
timestamps ever look wrong, start here before assuming it's a new bug.

## The `Timestamp` class

Internally, every packet's capture time is represented as a `Timestamp`:
a raw integer tick count plus a resolution (ticks per second). This lets
ChronoCap compare timestamps from sources with *different* resolutions
(e.g. a microsecond-resolution capture merged against a nanosecond-
resolution one) without ever converting to a lossy float:

```python
class Timestamp:
    def __init__(self, ticks, resolution):
        self.ticks = ticks
        self.resolution = resolution
```

Comparisons (`__lt__`, `__le__`) cross-multiply rather than divide, so two
timestamps at different resolutions compare exactly:

```python
self.ticks * other.resolution  <  other.ticks * self.resolution
```

This is the same trick as comparing fractions `a/b` and `c/d` via
`a*d` vs `c*b` — it avoids any rounding error from converting one
resolution into the other.

## Reading pcapng input (r1 / r2)

pcapng-format files carry timestamp resolution as **per-packet metadata**
(`tshigh`, `tslow`, `tsresol`), not as a single value for the whole file.
This is because a single pcapng file can, in principle, contain multiple
interfaces with different capture resolutions.

Confirmed against scapy's source: `tsresol` in the metadata scapy hands
back is **already normalized** into ticks-per-second (e.g. `1_000_000` or
`1_000_000_000`) — not the raw pcapng-encoded resolution byte (where the
spec packs base-2-vs-base-10 and an exponent into one byte). No extra
conversion is needed on our side; `get_timestamp()` uses `tsresol`
directly as the `Timestamp`'s resolution.

**Known upstream edge case (scapy issue #2342):** some capture tools
encode an interface's `if_tsresol` option in a way that scapy doesn't
parse as expected, causing it to silently fall back to the default
`1_000_000` (microsecond) resolution instead of the file's actual
resolution — with no error raised. If you ever see ChronoCap's output
timestamps look systematically low-resolution or slightly off compared
to the same packets in Wireshark, check this first:

```python
from scapy.all import RawPcapReader
reader = RawPcapReader("suspect_file.pcapng")
packet_data, metadata = next(reader)
print(metadata.tsresol)   # expect 1000000000 for a nanosecond capture
```

If it prints `1000000` when you know the capture is nanosecond-precision,
this is the scapy-level issue, not a ChronoCap bug.

## Reading/writing classic pcap output

ChronoCap's output files are written as **classic pcap** (not real
pcapng) with nanosecond precision (`RawPcapWriter(..., nano=True)`).
Classic pcap format signals its precision through the file's **magic
number** in the 24-byte global header, not per-packet:

| Magic number | Meaning |
|---|---|
| `0xa1b2c3d4` | Microsecond precision |
| `0xa1b23c4d` | Nanosecond precision |

### The bug we found and fixed

Unlike pcapng, **scapy does not normalize this for classic pcap.** The
raw `usec` field in `PacketMetadata` is handed back exactly as stored in
the file — meaning it's either a true microsecond count or a true
nanosecond count depending on the magic number, and it's the caller's
job to check `reader.nano` to know which.

ChronoCap's `-verify` feature reads back its own output (always
nanosecond-format), but `get_timestamp()`'s classic-format branch
originally assumed microseconds unconditionally. The result: a
nanosecond value (up to 999,999,999) got misinterpreted as a microsecond
value, throwing the reconstructed timestamp off by hundreds of "phantom"
seconds — which surfaced as spurious chronological-ordering violations
during verification, with nonsensical displayed timestamps.

**The fix:** `get_timestamp(metadata, is_nano=False)` now takes an
explicit `is_nano` flag, threaded through from `reader.nano` at both call
sites (`PacketReader.get_next_packet` and, since the read-path
consolidation, `verify_output` via the same `PacketReader`):

```python
resolution = 1_000_000_000 if is_nano else 1_000_000
ticks = metadata.sec * resolution + metadata.usec
```

**Note on why this only ever affected `-verify`:** ChronoCap's own r1/r2
inputs are always pcapng, which takes the already-normalized `tshigh`
code path — never the classic-format branch. Only `-verify`, reading back
ChronoCap's own nanosecond-precision classic-pcap output, exercises that
branch for real. If you ever extend ChronoCap to accept classic `.pcap`
input files directly, re-test this path carefully — the guard against
regressing is the shared `PacketReader` class, not a standalone check.

### Extension vs. actual format

Output files get whatever extension you specify via `-w` (typically
`.pcapng`), but the bytes written are classic pcap, not real pcapng.
Most tools (Wireshark included) identify format from the magic number
rather than the extension, so this works in practice — it's just
technically inaccurate naming, not a functional problem.

## Filename timestamps: `-time UTC` vs `-time Local`

`format_timestamp_for_filename()` renders the *first packet's* timestamp
into `yyyy_mm_dd_hh_mm_ss` for use in the output filename.

- **UTC (default):** avoids a real failure mode — if the machine running
  ChronoCap crosses a Daylight Saving Time transition, naive local time
  can jump *backward* by an hour, which would break the chronological
  sort order of the filenames themselves (a file created after the DST
  fallback could get an earlier-looking timestamp than one created
  before it).
- **Local:** matches your own wall-clock time, which can make
  cross-referencing against locally-sourced data or logs easier during
  analysis. Trade-off: filename ordering across a DST boundary is not
  guaranteed to stay strictly chronological in this mode.

This is a separate function from `format_timestamp_display()`, which
renders sub-second (microsecond) precision for diagnostic output like
`-verify`'s violation reports — filenames intentionally only need
whole-second granularity, since finer precision there would just make
filenames longer without adding practical value for browsing a
directory listing.

## Quick troubleshooting checklist

If a timestamp anywhere in ChronoCap's output looks wrong:

1. Is it a filename timestamp or a `-verify` diagnostic timestamp? They
   use different formatting functions but the same underlying
   `Timestamp` — if the underlying value is wrong, both will be wrong.
2. Is the source a pcapng file? Check `metadata.tsresol` directly (see
   the scapy issue #2342 check above).
3. Is it specifically `-verify` output that looks wrong, and the file
   in question is nanosecond-precision? Confirm `reader.nano` is being
   read correctly for your scapy version — this is exactly the class of
   bug that bit us once.
