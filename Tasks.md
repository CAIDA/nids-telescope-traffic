[README](README.md) | [Introduction](Introduction.md) | [Datasets](Datasets.md) | [Dpkt](Dpkt.md) | [Pyarrow](Pyarrow.md) | Tasks ⮕ | [Task 7](Task-hilbert-map.md) | [flow compare](flow_compare.ipynb) | [flow analysis](flow_analysis.ipynb)

# Tasks

Complete the tasks below in order. Tasks 1–2 are in the **compare notebook** (`flow_compare.ipynb`); Tasks 3–7 are in the **analysis notebook** (`flow_analysis.ipynb`), which reads the Parquet files the compare notebook produces. Replace each marked section with your code and answer all ten questions in the markdown cells that follow.

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

- [ ] Q1: Which transport protocol dominates the network telescope traffic in both PCAPs?
- [ ] Q2: Among the top-N flows by bytes in both PCAPs, which TCP destination port was targeted the most? What application is conventionally assigned to that port, and why is it a popular target?
- [ ] Q3: Which PCAP contains more traffic in terms of number of packets, and by how much? Is there a significant increase in traffic contributed by the top-N flows by bytes?

## Task 3: AS Resolution and IP Geolocation (analysis notebook)

Implement `build_prefix_to_asn`, `build_asn_trie`, and `lookup_asn` — reusing the prefix-to-AS mapping and longest-prefix-match trie technique from _nids-bgp-control-plane_ — to resolve each flow's source IP to its origin AS(es). Implement `geolocate_ip` to resolve a source IP to its two-letter country code using the GeoLite2 City database. Together these let `enrich_ip_src` add `ip_src_country` and `ip_src_asn` columns to each capture's flow DataFrame.

## Task 4: Investigate the Surge by Country

Aggregate traffic volume by `ip_src_country` for both captures.

- [ ] Q4: What are the top 5 countries (sorted by traffic volume) contributing to the traffic in PCAP A and PCAP B? What is the biggest change in the ranking between A and B? Call the country with the highest traffic volume in the second capture (PCAP B) **Country<sub>top</sub>**.

## Task 5: Investigate the Surge by AS

Filter to Country<sub>top</sub> and aggregate traffic volume by `ip_src_asn`.

- [ ] Q5: What are the top 5 ASes (sorted by traffic volume) within Country<sub>top</sub> in PCAP A and PCAP B? What is the biggest change in the ranking? How many origin ASes are associated with the top traffic contributor in PCAP B — what is this phenomenon called? Call this AS **AS<sub>top</sub>**.
- [ ] Q6: Aggregate the traffic from source IPs within AS<sub>top</sub>. Describe the distribution of traffic volume across the top 5 IPs.

## Task 6: Investigate IP-Level Traffic

Filter to the single top source IP from Task 5 — call it **the scanner**.

- [ ] Q7: How many flows did the scanner create? Describe its traffic pattern — the protocol(s) it used, the destination ports it targeted, and the destinations it probed.

## Task 7: Analyze the Scanning Behavior

See [Task 7 Guidance](Task-hilbert-map.md) for the Hilbert-curve algorithm used here.

Visualize the destination addresses the scanner probed, within its most-targeted /16, as a Hilbert curve map (packet count per destination IP).

- [ ] Q8: Which /24 blocks within that /16 was the scanner least likely to probe?
- [ ] Q9: Plot the number of packets the scanner sent, broken down by destination port and protocol. What does this reveal about its scanning pattern?
- [ ] Q10: Review the top flows you identified in Task 2. Why can't the scanner be detected simply by screening for the largest flows in network telescope traffic?

[README](README.md) | [Introduction](Introduction.md) | [Datasets](Datasets.md) | [Dpkt](Dpkt.md) | [Pyarrow](Pyarrow.md) | Tasks ⮕ | [Task 7](Task-hilbert-map.md) | [flow compare](flow_compare.ipynb) | [flow analysis](flow_analysis.ipynb)
