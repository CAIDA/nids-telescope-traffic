[README](README.md) | [Introduction](Introduction.md) | [Datasets](Datasets.md) | Dpkt ⮕ | [Pyarrow](Pyarrow.md) | [Tasks](Tasks.md) | [Task 7 Guide](Task-hilbert-map.md) | [flow compare](flow_compare.ipynb) | [flow analysis](flow_analysis.ipynb) | [Slides](slides/)

# dpkt Guide: Parsing Telescope Packets

This guide builds up the [`dpkt`](https://github.com/kbandla/dpkt) knowledge you
need for **Task 1** (`flow_key`) in `flow_compare.ipynb`, one piece at a time. Each
section is a small, copy-pasteable snippet followed by the *why* and the
gotchas — the same "building blocks" the provided `iter_packets` reader and the
`flow_key` you write are assembled from. Read [Datasets](Datasets.md) first for
the capture format (gzip-compressed pcap, Ethernet frames, destination IPs
anonymized into `10.0.0.0/8`), then come back here for how to actually pull a
5-tuple out of each frame.

Every idea below is also a *best practice* — parsing telescope traffic at scale
means millions of packets, so the choices that keep memory flat and skip
the packets you don't care about matter as much as the API itself.

## 1. Why dpkt (and not scapy)?

Packet parsing in Python has several libraries, and they are **not**
interchangeable at this scale. The right choice is the whole first best
practice: **match the parser to the size of the data.**

| Library | What it's good at | Why not here |
|---|---|---|
| **dpkt** | Fast, lazy, tiny per-packet overhead; streams straight off a file/socket | — (this is what we use) |
| **scapy** | Beautiful interactive API, auto-decodes every layer by linktype, huge protocol coverage | 10–50× slower and far more memory per packet — it eagerly builds a full object graph for every layer. Fine for a few thousand packets in a REPL; the wrong tool for millions. |
| **pyshark** | Wraps Wireshark/`tshark`, the richest dissection available | Shells out to `tshark` and parses its JSON/XML — orders of magnitude too slow for bulk offline work. |
| **raw `struct.unpack`** | Absolute fastest, zero dependencies | You'd reimplement Ethernet/IP/TCP header parsing by hand — error-prone, and dpkt already does it in C-speed Python. |
| **nfstream** | Builds flow records for you, out of the box | It would do Task 1 *and* Task 2 automatically — which defeats the point of learning to turn packets into flows yourself. |

dpkt wins here because it is **fast and lazy**: it parses only the headers it is
asked for and hands the rest back as raw bytes, so nothing is decoded until you
reach for it. That laziness is what lets the compare notebook stream a capture of
any size with flat memory — the two here hold over 22 million packets between
them, and the same code would handle a multi-gigabyte capture unchanged. It is the
same memory-discipline theme that runs through
the whole module (stream the pcap, aggregate into flows, free the raw state
before the analysis notebook).

## 2. Opening a capture — `dpkt.pcap.Reader`

`dpkt.pcap.Reader` wraps a binary stream and iterates it as
`(timestamp, raw_frame_bytes)` pairs. It parses only the pcap global header and
each per-record header as it goes — the frame payload is handed back
**unparsed**:

```python
import dpkt

with open("capture.pcap", "rb") as f:
    reader = dpkt.pcap.Reader(f)
    for ts, buf in reader:
        # ts  is a float unix timestamp
        # buf is the raw Ethernet frame, still bytes — nothing decoded yet
        ...
```

- **Stream, never buffer the whole file.** Iterating the `Reader` pulls one
  record at a time. Do **not** call `reader.readpkts()` (which returns a list of
  *every* packet) on a telescope capture — that materializes the entire file in
  memory, exactly what streaming avoids.
- **`Reader` assumes an Ethernet link layer.** Unlike scapy, it does **not**
  auto-select a decoder from the pcap's linktype — you decode the frame yourself
  (see §3). These captures are Ethernet, so that's what `flow_key` assumes.
- **You don't open the file yourself here.** The provided `iter_packets` helper
  already wraps `dpkt.pcap.Reader` around a stream that is sniffed for the gzip
  magic bytes (`\x1f\x8b`) and transparently decompressed — see
  [Datasets](Datasets.md). Your job in `flow_key` starts from the `buf` it
  yields.

## 3. The layered object model

A parsed packet is a chain of layers, each exposing the next one up through its
`.data` attribute:

```
Ethernet ─.data→ IP ─.data→ TCP / UDP / ICMP ─.data→ payload bytes
```

You decode the raw frame by constructing the outermost layer, then walk `.data`
inward:

```python
eth = dpkt.ethernet.Ethernet(buf)   # decode the Ethernet frame
ip  = eth.data                      # the next layer up — but WHAT is it?
```

- **Type-check every layer before you use it.** `eth.data` might be `dpkt.ip.IP`,
  but it might just as easily be ARP, IPv6, or something else — a telescope sees
  all of it. Never assume; gate with `isinstance`:

  ```python
  if not isinstance(ip, dpkt.ip.IP):
      return None            # not IPv4 → out of scope for this task
  ```

  This is why `flow_key` returns `None` for non-IPv4 frames instead of crashing on
  the next attribute access. IPv6 (`dpkt.ip6.IP6`) is intentionally out of scope
  and falls through this same gate.
- **`.data` is lazy.** Constructing `dpkt.ethernet.Ethernet(buf)` does not decode
  TCP until you actually touch `ip.data` — accessing only the layers you need is
  what keeps parsing cheap.

## 4. Reading addresses

`ip.src` and `ip.dst` are **not strings** — they are the raw 4-byte,
network-order address as it appeared on the wire. Convert them to dotted-decimal
with `socket.inet_ntoa`:

```python
import socket

src_ip = socket.inet_ntoa(ip.src)   # b'\n\x00\x00\x01' → '10.0.0.1'
dst_ip = socket.inet_ntoa(ip.dst)
```

- Printing `ip.src` directly gives you bytes, not an address — a common first
  surprise.
- `inet_ntoa` is IPv4-only (4 bytes). IPv6 would need `socket.inet_ntop`, but
  we've already excluded IPv6 at the §3 gate.

## 5. The transport layer

Below IP, `ip.data` is the transport segment. Which class it is tells you the
protocol, and different protocols expose their "ports" differently:

```python
if isinstance(ip.data, dpkt.tcp.TCP):
    proto = "TCP"
    sport, dport = ip.data.sport, ip.data.dport
elif isinstance(ip.data, dpkt.udp.UDP):
    proto = "UDP"
    sport, dport = ip.data.sport, ip.data.dport
elif isinstance(ip.data, dpkt.icmp.ICMP):
    proto = "ICMP"
    sport, dport = ip.data.type, ip.data.code   # ICMP has no ports
else:
    proto = "OTHER"
    sport, dport = 0, 0
```

- **Handle every branch, including `OTHER`.** Telescope traffic is not all TCP —
  assuming `ip.data` is TCP is the fastest way to a crash on a UDP or ICMP packet.
- **ICMP has no ports.** Its `type`/`code` fields identify the message (echo
  request, unreachable, …); we slot them into the port positions so ICMP still
  produces a well-formed 5-tuple key.

## 6. TCP flags are a bitfield

For TCP, `ip.data.flags` is a single integer whose bits are the SYN/ACK/FIN/…
flags. dpkt gives you the bit masks as `dpkt.tcp.TH_*` constants; test a bit with
a bitwise `&`:

```python
if flags & dpkt.tcp.TH_SYN:   # SYN bit set?
    ...
```

The notebook **provides** the decoder that turns that integer into a short string
like `"S"`, `"SA"`, or `"FA"` — you just call it:

```python
_TCP_FLAG_BITS = (
    (dpkt.tcp.TH_SYN,  "S"),
    (dpkt.tcp.TH_ACK,  "A"),
    (dpkt.tcp.TH_FIN,  "F"),
    (dpkt.tcp.TH_RST,  "R"),
    (dpkt.tcp.TH_PUSH, "P"),
    (dpkt.tcp.TH_URG,  "U"),
)

def _tcp_flags_str(flags):
    return "".join(letter for bit, letter in _TCP_FLAG_BITS if flags & bit) or "."
```

- One integer can carry several flags at once (a SYN-ACK has both `TH_SYN` and
  `TH_ACK` set), so you test each bit independently rather than comparing for
  equality.
- Only TCP has these flags; leave `tcp_flags` as `None` for UDP/ICMP/OTHER.

## 7. Robustness: malformed packets

A real capture is not clean. Truncated frames, snap-length cutoffs, and outright
garbage all occur, and when dpkt runs out of bytes mid-header it raises
`dpkt.dpkt.NeedData` (a subclass of `dpkt.dpkt.UnpackError`). If one bad frame
propagates, it aborts the entire multi-million-packet run.

**Best practice: guard the parse and skip bad packets** rather than trusting
every frame:

```python
def flow_key(buf):
    try:
        eth = dpkt.ethernet.Ethernet(buf)
        ...
    except (dpkt.dpkt.NeedData, dpkt.dpkt.UnpackError):
        return None            # unparseable frame — skip it, don't crash the run
```

Returning `None` funnels malformed frames into the same "skip this packet" path
that `build_flow_map` already uses for non-IPv4 frames, so a handful of corrupt
packets cost you those packets and nothing more. (Optionally keep a counter of
how many you dropped, so a *suspicious* number of unparseable frames doesn't pass
silently.) The `flow_key` you complete in Task 1 uses exactly this guard.

## 8. Best practices, all together

- **Match the parser to the scale** — dpkt for millions of packets, not scapy
  (§1).
- **Stream, don't buffer** — iterate the `Reader`; never `readpkts()` a large
  capture (§2).
- **Type-check every layer** with `isinstance` before touching it; return `None`
  for non-IPv4 (§3).
- **Convert addresses** with `socket.inet_ntoa` — `ip.src`/`ip.dst` are raw
  bytes (§4).
- **Handle every transport branch**, including UDP, ICMP (type/code), and OTHER
  (§5).
- **Treat TCP flags as a bitfield** — test each bit with `&` (§6).
- **Guard against malformed packets** — wrap the parse in `try/except` and skip
  the bad ones (§7).
- **Let dpkt stay lazy** — only walk into the layers you actually need, so a
  packet you reject early is never fully decoded.

[README](README.md) | [Introduction](Introduction.md) | [Datasets](Datasets.md) | Dpkt ⮕ | [Pyarrow](Pyarrow.md) | [Tasks](Tasks.md) | [Task 7 Guide](Task-hilbert-map.md) | [flow compare](flow_compare.ipynb) | [flow analysis](flow_analysis.ipynb) | [Slides](slides/)
