[README](README.md) | Introduction ⮕ | [Datasets](Datasets.md) | [Dpkt](Dpkt.md) | [Pyarrow](Pyarrow.md) | [Tasks](Tasks.md) | [Task 7](Task-hilbert-map.md) | [flow compare](flow_compare.ipynb) | [flow analysis](flow_analysis.ipynb)

# Introduction and Background

### Reading

- [CAIDA network telescope datasets and research](https://catalog.caida.org/search?query=network+telescope) — background on the darknet infrastructure that collected the traffic used in this module
- [Internet background radiation (Wikipedia)](https://en.wikipedia.org/wiki/Internet_background_noise) — what a network telescope actually observes
- [Hilbert curve (Wikipedia)](https://en.wikipedia.org/wiki/Hilbert_curve) — the space-filling curve used to visualize scan footprints in Task 7

### Prerequisite NIDS Modules

- [Understanding the BGP Control Plane](https://github.com/CAIDA/nids-bgp-control-plane) — this module reuses its prefix-to-AS mapping and longest-prefix-match technique to resolve origin ASes for observed traffic

## Introduction

A **network telescope** (or **darknet**) is a block of globally routed IP address space that hosts no real services. Because nothing legitimate should ever be addressed to it, essentially everything a telescope receives is unsolicited: scanning, backscatter from spoofed denial-of-service traffic, and misconfigured software. The **UCSD Network Telescope**, operated by CAIDA, is one of the largest such instruments on the Internet, and continuously records this "background radiation."

This module gives you two short packet captures (`PCAP_A`, `PCAP_B`) from a /16 of the telescope, taken about 7 minutes apart on 2026-06-26 (08:53 and 09:00 UTC), with destination addresses anonymized into `10.0.0.0/8` and source addresses left intact. Between the two captures, the telescope observed a **traffic surge** — a sudden jump in volume. Your job, across two notebooks, is to figure out what caused it.

### From Packets to Flows

Working packet-by-packet does not scale — a single hour of telescope traffic already contains millions of packets. The compare notebook streams each PCAP once, without ever materializing the whole file in memory, and aggregates packets sharing the same **5-tuple** (source IP, source port, destination IP, destination port, protocol) into a **flow** record: a packet count, a byte count, first/last-seen timestamps, and the set of TCP flags observed. The resulting flow records — orders of magnitude fewer rows than raw packets — are persisted to **Parquet** files, so the raw per-packet state can be dropped from memory before the analysis notebook's work begins. Parsing each frame into its protocol layers is done with the **dpkt** library ([Dpkt](Dpkt.md)); the flow records are handed to the analysis notebook as Parquet files written with **pyarrow** ([Pyarrow](Pyarrow.md)).

### From Flows to Root Cause

A flow's 5-tuple tells you *what* happened, but not *who* is behind it or *where* they're from. The analysis notebook enriches every flow's source IP with:

- a **country**, via the **MaxMind GeoLite2** geolocation database, and
- an **origin AS**, via the same **prefix-to-AS mapping and longest-prefix-match** technique you built in _nids-bgp-control-plane_, applied here to a live BGP RIB snapshot rather than the router table you counted addresses in previously.

With those two extra columns, you can aggregate traffic at whatever granularity the investigation needs — by country, then by AS within a country, then down to a single IP — following the volume until it stops concentrating. Along the way you may encounter a prefix associated with more than one origin AS, a **MOAS** — the same phenomenon introduced in _nids-bgp-control-plane_, seen here from the traffic side rather than the routing-table side.

### Visualizing a Scan

Once you've narrowed the surge down to a single source IP (call it **the scanner**), the natural next question is *how* it scans: which protocols, which ports, and which destinations. Because a /16 has 65,536 addresses, a simple list of "destination IP → packet count" is hard to read as a whole. Task 7 instead lays the destination /16 out as a **Hilbert curve** — a space-filling curve that maps a 1-D sequence onto a 2-D grid while preserving locality, so that addresses which are numerically close stay visually close. Regions of the address space the scanner covered light up as a spatial pattern in the resulting image, making it easy to see, for example, whether it swept complete /24 blocks, skipped others, or clustered around specific subnets.

### Why Screening for "Big Flows" Isn't Enough

It's tempting to assume that a scanner producing a traffic surge would show up as one of the biggest flows by volume. Task 7 asks you to revisit the top flows you found in Task 2 and explain why that assumption can fail — think about what a flow actually counts, and how a scan that touches many destinations spreads its packets across them.

#### Optional Reading

- [CAIDA scanning and Internet background radiation research](https://catalog.caida.org/search?query=scanning)
- [Border Gateway Protocol (Wikipedia)](https://en.wikipedia.org/wiki/Border_Gateway_Protocol)

[README](README.md) | Introduction ⮕ | [Datasets](Datasets.md) | [Dpkt](Dpkt.md) | [Pyarrow](Pyarrow.md) | [Tasks](Tasks.md) | [Task 7](Task-hilbert-map.md) | [flow compare](flow_compare.ipynb) | [flow analysis](flow_analysis.ipynb)
