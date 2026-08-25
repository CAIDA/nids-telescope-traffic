README ⮕ | [Introduction](Introduction.md) | [Datasets](Datasets.md) | [Dpkt](Dpkt.md) | [Pyarrow](Pyarrow.md) | [Tasks](Tasks.md) | [Task 7](Task-hilbert-map.md) | [flow compare](flow_compare.ipynb) | [flow analysis](flow_analysis.ipynb)

### Network Infrastructure Data Science (NIDS) Module

---

# Investigating a Traffic Surge in the UCSD Network Telescope

## Learning Objectives

The goal of this assignment is to build flow records from raw network telescope packet captures in a memory-efficient, streaming way; enrich those flows with IP geolocation and BGP-derived origin-AS information (reusing the prefix-to-AS mapping and longest-prefix-match technique from **nids-bgp-control-plane**); and use the enriched flow data to investigate a real traffic surge — narrowing the investigation from country, to AS, to a single source IP, to that IP's per-port scanning behavior, and visualizing its scan footprint with a Hilbert curve map.

<img src="images/hilbert-curve.png" width="150px">

## Slides

- [ETP week 06 Telescope](slides/ETP-Week-06-etp-Telescope-Plenary.pptx)
- [ETP week 06 Expanse Assignment](slides/ETP-Week-06-expanse_assignment.pdf)

## Overview

Start by reading **Introduction** to get the background needed to understand the assignment. **Datasets** explains each dataset and how to access it.

- step 1 [read the introduction](Introduction.md)
- step 2 [read dataset overviews](Datasets.md)
- step 3 [review the tasks](Tasks.md)
- step 4 log into NRP's JupyterHub (instance with **at least 16 GB memory, 24 GB recommended** — see [Datasets](Datasets.md)), upload and complete both notebooks **in order**
  - Detailed instructions to access NRP: [How to access NRP](https://www.caida.org/projects/nids/how-to/access-nrp/)
  - complete `flow_compare.ipynb` first — it builds the two flow-record Parquet files the analysis notebook reads
  - then complete `flow_analysis.ipynb`, which reads those Parquet files back in and performs the investigation
  - complete each task by replacing the marked sections with your code
  - answer all ten questions in the markdown cells that follow
- step 5 download your working notebooks and replace ⬅ deliverables
- step 6 commit and push to github

### Directory Structure

```
nids-etp-traffic
├- Introduction.md                          # Introduction and background
├- Datasets.md                              # Dataset overview and access instructions
├- Dpkt.md                                  # Guide - parsing telescope packets with dpkt (Task 1)
├- Pyarrow.md                               # Guide - persisting flow records to Parquet with pyarrow
├- Tasks.md                                 # Task checklist and instructions
├- Task-hilbert-map.md                      # Guidance for Task 7's Hilbert curve visualization
├- requirements.txt                         # Python dependencies
├- flow_compare.ipynb                    ⬅  # compare notebook - build flow records. Complete / Commit / Push
└- flow_analysis.ipynb                   ⬅  # analysis notebook - investigate the surge. Complete / Commit / Push
```

### Glossary

- **Network Telescope (Darknet)**: A block of globally routed but unused IP address space instrumented to capture all incoming traffic. Since nothing legitimate is ever addressed to it, essentially everything it observes is unsolicited — scanning, backscatter from spoofed attack traffic, or misconfiguration.
- **Flow**: A summary record aggregating every packet sharing the same 5-tuple (source IP, source port, destination IP, destination port, protocol) seen in a capture into a single row — packet count, byte count, first/last-seen timestamps, and observed TCP flags — without retaining each individual packet.
- **PCAP**: A packet-capture file format; the raw input used to build flow records in this assignment.
- **dpkt**: A fast, low-level Python library for pulling packets out of a PCAP. Used in the compare notebook to decode each frame's `Ethernet → IP → TCP/UDP/ICMP` layers; chosen over higher-level parsers (scapy, pyshark) because it stays fast and memory-flat across millions of packets. See [Dpkt](Dpkt.md).
- **Scanner**: A host observed systematically probing many destination addresses and/or ports, characteristic of reconnaissance activity.
- **MOAS (Multi-Origin AS)**: A prefix announced as originating from more than one AS. Introduced in _nids-bgp-control-plane_; you may encounter it here while resolving a traffic source's origin AS.
- **Origin AS**: The AS that originates a BGP prefix announcement — the last ASN in the AS path.
- **Longest-prefix match**: Attributing an IP address to the most-specific (highest `/N`) prefix that covers it — used here to resolve a source IP's origin AS, and in _nids-bgp-control-plane_ to count addresses per AS.
- **GeoLite2 / MaxMind**: A free IP geolocation database (this assignment uses the City edition) mapping an IP address to a country, among other fields.
- **RouteViews / OSDF / bgpkit**: The BGP data collection, distributed-fetch, and MRT-parsing infrastructure introduced in _nids-bgp-control-plane_ and reused here to resolve origin ASes.
- **Parquet**: A columnar file format used here to persist flow records between the two notebooks, so the compare notebook's raw per-packet state can be freed from memory before the analysis notebook's work begins. Written with **pyarrow** (see [Pyarrow](Pyarrow.md)).
- **pyarrow**: The Python binding for Apache Arrow, used in the compare notebook to write flow records straight to Parquet without first building a pandas `DataFrame`, so only one copy of the data is resident at a time. See [Pyarrow](Pyarrow.md).
- **Hilbert curve**: A space-filling curve that maps a 1-D index to 2-D coordinates while preserving locality — nearby indices land on nearby points, at every scale. Used here to lay an IPv4 /16 block out as a 256×256 image so that scanned and unscanned regions of the address space form visually compact clusters rather than scattered noise.

README ⮕ | [Introduction](Introduction.md) | [Datasets](Datasets.md) | [Dpkt](Dpkt.md) | [Pyarrow](Pyarrow.md) | [Tasks](Tasks.md) | [Task 7](Task-hilbert-map.md) | [flow compare](flow_compare.ipynb) | [flow analysis](flow_analysis.ipynb)
