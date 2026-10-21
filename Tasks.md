[README](README.md) | [Introduction](Introduction.md) | [Datasets](Datasets.md) | [Dpkt](Dpkt.md) | [Pyarrow](Pyarrow.md) | Tasks ⮕ | [Task 7 Guide](Task-hilbert-map.md) | [flow compare](flow_compare.ipynb) | [flow analysis](flow_analysis.ipynb) | [Slides](slides/)

# Tasks

Complete the tasks below in order. Tasks 1–2 are in the **compare notebook** (`flow_compare.ipynb`); Tasks 3–7 are in the **analysis notebook** (`flow_analysis.ipynb`), which reads the Parquet files the compare notebook produces. Replace each marked section with your code and answer all ten questions in the markdown cells that follow.

Two things worth knowing before you start:

- **How much code you write varies by task.** Tasks 1, 2 and 3 are implementation tasks — look for the `student_code_start` / `student_code_end` markers. In Tasks 4–7 the aggregations and plots are **already written for you**; your input there is to identify three values (`COUNTRY_TOP`, `AS_TOP`, `SCANNER_IP`) from the output and to answer the questions in prose. Don't go hunting for code stubs in Tasks 6 and 7 — there aren't any.
- **"Traffic volume" always means `total_bytes`.** Where a question asks you to rank or aggregate by traffic volume, the provided code sums the `total_bytes` column. Questions that mean packet counts say so explicitly.

## Task 0: Get access to NRP's JupyterHub and run both notebooks in order

Use this link if you need to [get access to NRP's JupyterHub](https://www.caida.org/projects/nids/how-to/access-nrp/)

- step 1. Go to [JupyterHub](https://jupyterhub-west.nrp-nautilus.io)
  - upload and run `flow_compare.ipynb` first — it builds the two flow-record Parquet files
  - then upload and run `flow_analysis.ipynb`, which reads those Parquet files back in
  - complete each task by replacing the marked sections with your code
  - answer all questions
- step 2. Download your completed notebooks, commit, and submit.

## Task 1: Flow Key Extraction (compare notebook)

Given one raw Ethernet frame, parse its IP header and (if present) its TCP/UDP/ICMP header, and return a 5-tuple flow key `(ip_src, port_src, ip_dst, port_dst, proto)` plus the TCP flags observed (or `None` for non-TCP traffic). ICMP has no ports; use its `type`/`code` fields in their place. Return `None` for any frame without an IPv4 layer.

## Task 2: Per-Capture Summary (compare notebook)

Given a completed flow map, compute the total number of distinct flows, the total packet and byte counts across all flows, and a count of flows per protocol. These feed the printed summary and the questions below.

- [ ] Q1: Which transport protocol accounts for the most **flows** in both PCAPs? (The protocol breakdown counts flows, not packets — the two can point different ways.)
- [ ] Q2: Among the top-N flows by bytes in both PCAPs, which TCP destination port was targeted the most? What application is conventionally assigned to that port, and why is it a popular target?
- [ ] Q3: Which PCAP contains more traffic in terms of number of packets, and by how much? Is there a significant increase in traffic contributed by the top-N flows by bytes?

## Task 3: AS Resolution and IP Geolocation (analysis notebook)

Implement `build_prefix_to_asn`, `build_asn_trie`, and `lookup_asn` — reusing the prefix-to-AS mapping and longest-prefix-match trie technique from _nids-bgp-control-plane_ — to resolve each flow's source IP to its origin AS(es). Implement `geolocate_ip` to resolve a source IP to its two-letter country code using the GeoLite2 City database. Together these let `enrich_ip_src` add `ip_src_country` and `ip_src_asn` columns to each capture's flow DataFrame.

## Task 4: Investigate the Surge by Country

The provided cells aggregate traffic volume by `ip_src_country` for both captures. Read their output, answer Q4, and set `COUNTRY_TOP` in the cell that follows.

Note the rankings include a `NaN` row: flows whose source IP the GeoLite2 database does not cover. Check how much traffic sits there before treating a top-5 as the whole picture.

- [ ] Q4: What are the top 5 countries (sorted by traffic volume) contributing to the traffic in PCAP A and PCAP B? What is the biggest change in the ranking between A and B? Call the country with the highest traffic volume in the second capture (PCAP B) **Country<sub>top</sub>**.

## Task 5: Investigate the Surge by AS

The provided cells filter to Country<sub>top</sub> and aggregate traffic volume by `ip_src_asn`. Read their output, answer Q5 and Q6, and set `AS_TOP`. As in Task 4, a `NaN` row collects flows with no covering prefix in the RIB.

- [ ] Q5: What are the top 5 ASes (sorted by traffic volume) within Country<sub>top</sub> in PCAP A and PCAP B? What is the biggest change in the ranking? How many origin ASes are associated with the top traffic contributor in PCAP B — what is this phenomenon called? Call this AS **AS<sub>top</sub>**.
- [ ] Q6: Aggregate the traffic from source IPs within AS<sub>top</sub>. Describe the distribution of traffic volume across the top 5 IPs.

## Task 6: Investigate IP-Level Traffic

Set `SCANNER_IP` to the single top source IP from Task 5 — call it **the scanner**. The provided cells then summarize its flows, destinations, ports and protocols.

- [ ] Q7: How many flows did the scanner create? Describe its traffic pattern — the protocol(s) it used, the destination ports it targeted, and the destinations it probed.

## Task 7: Analyze the Scanning Behavior

See [Task 7 Guidance](Task-hilbert-map.md) for the Hilbert-curve algorithm used here.

The provided cells plot the destination addresses the scanner probed, within the destination /16 it sent the most packets to, as a Hilbert curve map (packet count per destination IP), and then break its packets down by destination port and protocol. Both plots are written for you — your work here is to read them.

- [ ] Q8: Which /24 blocks within that /16 did the scanner probe least? Read the `/24` label printed inside each 16×16 cell — the x and y axes are Hilbert curve positions, **not** octets, so a block cannot be named from its axis coordinates.
- [ ] Q9: Using the packets-by-port-and-protocol plot, what does the scanner's choice of destination ports and protocol reveal about its scanning pattern?
- [ ] Q10: Review the top flows you identified in Task 2. Why can't the scanner be detected simply by screening for the largest flows in network telescope traffic?

[README](README.md) | [Introduction](Introduction.md) | [Datasets](Datasets.md) | [Dpkt](Dpkt.md) | [Pyarrow](Pyarrow.md) | Tasks ⮕ | [Task 7 Guide](Task-hilbert-map.md) | [flow compare](flow_compare.ipynb) | [flow analysis](flow_analysis.ipynb) | [Slides](slides/)
