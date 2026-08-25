[README](README.md) | [Introduction](Introduction.md) | Datasets ⮕ | [Dpkt](Dpkt.md) | [Pyarrow](Pyarrow.md) | [Tasks](Tasks.md) | [Task 7](Task-hilbert-map.md) | [flow compare](flow_compare.ipynb) | [flow analysis](flow_analysis.ipynb) | [Slides](slides/)

# Datasets

## System Requirements

The compare notebook streams a raw telescope traffic sample per capture and builds an in-memory flow map before writing it to Parquet. Run both notebooks on an instance with **at least 16 GB of memory (24 GB recommended)**.

The captures are only minutes long, but they are not small: each reduces to **millions of distinct flow records**, and the flow map holding them is what dominates memory — not the packet stream, which is processed one frame at a time and never fully resident. That is why the notebooks process one capture at a time and free each flow map (`del` + `gc.collect()`) before starting the next.

## UCSD Network Telescope PCAP Samples

The traffic data is a short sample — small only relative to the full telescope feed — collected by one /16 of the CAIDA **UCSD Network Telescope** — a block of routed but unused address space that receives only unsolicited traffic (scanning, backscatter, misconfiguration; see [Introduction](Introduction.md)).

| Variable | Timestamp (UTC) | Notes |
|---|---|---|
| `PCAP_A` | 2026-06-26 08:53 | first capture |
| `PCAP_B` | 2026-06-26 09:00 | second capture — the traffic surge appears here |

- **Destination IPs are anonymized** into `10.0.0.0/8`.
- **Source IPs are left as originally observed** — this is what makes the country/AS/IP attribution in the analysis notebook possible.
- Both files are **gzip-compressed pcaps** served over http(s) from an NRP Ceph S3-compatible endpoint. The compare notebook's streaming reader (`iter_packets`) opens them without ever downloading or decompressing the whole file up front — it sniffs the gzip magic bytes rather than trusting the `.gz` suffix, since the URL's query string can follow it. Each frame is then parsed with the **dpkt** library — see [Dpkt](Dpkt.md) for the packet object model (`Ethernet → IP → TCP/UDP/ICMP`) and the parsing best practices this module follows.

## Flow Record Parquet Files

The compare notebook writes one Parquet file per capture, which the analysis notebook reads back in. See [Pyarrow](Pyarrow.md) for how these files are written — directly from the flow map, without a pandas round-trip — and read back:

| Column | Description |
|---|---|
| `ip_src`, `port_src` | Source IP and port |
| `ip_dst`, `port_dst` | Destination IP and port (anonymized into `10.0.0.0/8`) |
| `proto` | `TCP`, `UDP`, `ICMP`, or `OTHER` |
| `total_packets`, `total_bytes` | Aggregate counts for this 5-tuple. `total_bytes` sums the bytes *captured* per frame, so under a capture snap length it understates what was on the wire |
| `first_seen`, `last_seen` | Timestamps of the first/last packet seen in this flow |
| `tcp_flags_seen` | Comma-separated TCP flags observed (empty for non-TCP flows) |

The analysis notebook adds two more columns during enrichment (Task 3):

| Column | Description |
|---|---|
| `ip_src_country` | Two-letter country code from GeoLite2, or `None` if not found |
| `ip_src_asn` | Comma-separated origin ASN(s) for `ip_src` from longest-prefix match, or `None` if uncovered |

## MaxMind GeoLite2 City Database

The analysis notebook geolocates each flow's source IP using MaxMind's **GeoLite2 City** database. For this module the database is already hosted (gzip-compressed) on the same NRP Ceph S3 endpoint as the PCAPs — you do not need your own MaxMind account or license key. `build_geoip_reader` streams and decompresses it the same way `iter_packets` streams a PCAP: it does not assume a `.gz`-suffixed URL is actually served that way.

The hosted snapshot is named by its build date and edition:

```
2026-06-24 . GeoLite2-City . mmdb . gz
└── date ──┘  └── edition ─┘   │      └ gzip transport wrapper
                               └ MaxMind DB binary format
```

### File format: the MaxMind DB (`.mmdb`)

`.mmdb` is MaxMind's own **binary, self-describing, read-only** database format, purpose-built for "which record covers this IP?" lookups. One file is three back-to-back sections:

| Section | What it holds |
|---|---|
| **Search tree** | A binary trie over IP-address bits — the index that turns an IP into a data offset. |
| **Data section** | The actual records (country, city, location, …) in MaxMind's typed encoding. |
| **Metadata** | Describes the database itself; found by scanning from EOF backward for the marker `\xab\xcd\xef MaxMind.com`. |

**Search tree.** A lookup walks a binary trie one address bit at a time, most-significant first: bit `0` follows a node's left record, bit `1` its right record. Each record either points to the next node, signals "not in the database," or points into the data section — so a lookup is at most 32 steps for IPv4 (128 for IPv6). IPv4 and IPv6 share **one tree**; in a v6 database an IPv4 address is reached through its IPv4-mapped position. This "trie over address bits" is the same longest-prefix-match idea as the `pytricia` ASN trie on the BGP side — MaxMind just ships it prebuilt on disk.

**Data section.** Records use MaxMind's typed encoding (maps, arrays, UTF-8 strings, doubles, unsigned ints, booleans). Repeated strings — country names, time zones — are stored once and referenced by **pointer**, which keeps the file compact. A GeoLite2 **City** record is a nested map; the fields the `geoip2` reader surfaces:

| Field (on the response) | Example | Notes |
|---|---|---|
| `.country.iso_code` | `"US"` | Two-letter ISO code — **the only field this module keeps** (`ip_src_country`). |
| `.country.name` | `"United States"` | Localized; `.country.names` is the full per-language map. |
| `.continent.code` / `.name` | `"NA"` / `"North America"` | |
| `.city.name` | `"Ashburn"` | Often `None` at country-level resolution. |
| `.subdivisions.most_specific.iso_code` | `"VA"` | State/region; may be empty. |
| `.location.latitude` / `.longitude` | `39.0`, `-77.5` | Approximate; `.accuracy_radius` is in km. |
| `.postal.code` | `"20149"` | May be `None`. |

**Metadata.** A trailing map describing the file: `database_type` (`"GeoLite2-City"`), `build_epoch`, `ip_version` (`6`), `record_size`, `node_count`, `languages`, and a text `description`. Because the schema travels inside the file, the reader needs no external type declarations — the same self-describing property Parquet gives us on the flow-record side ([Pyarrow](Pyarrow.md)).

**Editions.** MaxMind ships **Country** (smaller — continent/country only) and **City** (adds city, subdivisions, latitude/longitude, postal). This module uses the City edition for consistency with CAIDA tooling, but `geolocate_ip` reads only `.country.iso_code`.

**Compression is only transport.** The `.mmdb` format is itself uncompressed; the `.gz` suffix is just a gzip wrapper for the download. `build_geoip_reader` peeks the first two bytes for the gzip magic (`\x1f\x8b`) rather than trusting the suffix, then decompresses — the identical sniff `iter_packets` uses for the PCAPs.

### Opening the database — `build_geoip_reader`

The reader is **provided**. It opens the URL (or local path), sniffs and undoes gzip, and hands the bytes to a `geoip2` reader in `MODE_FD` — which reads the whole (tens-of-MB) database into memory in one `stream.read()`, so the underlying stream can close as soon as the function returns:

```python
import geoip2.database
import maxminddb

def build_geoip_reader(source):
    with contextlib.closing(_open_binary_stream(source)) as raw:
        buffered = raw if hasattr(raw, "peek") else io.BufferedReader(raw)
        is_gzip = buffered.peek(2)[:2] == b"\x1f\x8b"          # sniff, don't trust ".gz"
        stream = gzip.GzipFile(fileobj=buffered) if is_gzip else buffered
        return geoip2.database.Reader(stream, mode=maxminddb.MODE_FD)
```

Unlike the PCAPs, which we stream incrementally because the technique has to hold up on captures far larger than these, a GeoLite2 City database is small enough to hold fully in memory — so `MODE_FD` (one read) is the right call here, not streaming.

### Looking up an IP

Open the reader once (as a context manager) and reuse it for every lookup — rebuilding it per-IP would re-download and re-parse the whole database. `reader.city(ip)` returns a response object; IPs the database doesn't cover (private/reserved ranges, unallocated space) raise `AddressNotFoundError`:

```python
import geoip2.errors

with build_geoip_reader(GEOIP_DB) as reader:
    resp = reader.city("8.8.8.8")
    resp.country.iso_code            # 'US'
    resp.city.name                   # 'Mountain View' (or None)
    resp.location.latitude           # 37.4056

    reader.city("10.0.0.1")          # raises geoip2.errors.AddressNotFoundError
```

Completing `geolocate_ip` (Task 3) is a matter of wrapping that one call in a `try` / `except geoip2.errors.AddressNotFoundError` that returns `None` for uncovered IPs — the same "skip what you can't resolve" habit `flow_key` uses for non-IPv4 frames ([Dpkt](Dpkt.md)). `enrich_ip_src` then opens one reader per capture, looks up each **unique** source IP once, and maps the country codes onto the `ip_src_country` column.

## BGP Routing Table (RIB) Snapshot

Origin-AS resolution reuses the **RouteViews** RIB snapshot and **bgpkit** MRT parser from _nids-bgp-control-plane_ — see that module's Datasets page, in the [nids-bgp-control-plane repository](https://github.com/CAIDA/nids-bgp-control-plane), for the full RIB/MRT format description. The snapshot is fetched automatically via **OSDF**; building the `prefix → origin AS` map from a full RIB (tens of millions of routes) can take several minutes, so `build_prefix_to_asn` caches the result to disk (`prefix_to_asn.pkl`) and reuses it on subsequent runs instead of re-fetching.

Which snapshot: the notebook lists the RIB archive for collector `COLLECTOR` (`route-views3`) under the month `RIB_MONTH` (`2026.06`) and takes the **first** RIB of that month. The captures are from 2026-06-26, so the routing view is a few weeks older than the traffic it is used to attribute. Prefix origins change slowly enough that this is fine for the module, but it is an approximation, not a snapshot contemporaneous with the captures — worth remembering if an attribution ever looks surprising.

Note one difference from the prior module's environment: this one has **`pybgpkit-parser`** installed rather than `pybgpkit`, imported as `import pybgpkit_parser as bgpkit` — the `bgpkit.Parser(...)` API itself is unchanged.

[README](README.md) | [Introduction](Introduction.md) | Datasets ⮕ | [Dpkt](Dpkt.md) | [Pyarrow](Pyarrow.md) | [Tasks](Tasks.md) | [Task 7](Task-hilbert-map.md) | [flow compare](flow_compare.ipynb) | [flow analysis](flow_analysis.ipynb) | [Slides](slides/)
