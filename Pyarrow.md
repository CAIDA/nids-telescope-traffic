[README](README.md) | [Introduction](Introduction.md) | [Datasets](Datasets.md) | [Dpkt](Dpkt.md) | Pyarrow ⮕ | [Tasks](Tasks.md) | [Task 7](Task-hilbert-map.md) | [flow compare](flow_compare.ipynb) | [flow analysis](flow_analysis.ipynb) | [Slides](slides/)

# pyarrow Guide: Persisting Flow Records to Parquet

This guide covers the [`pyarrow`](https://arrow.apache.org/docs/python/) you need
in order to hand flow records from the compare notebook to the analysis notebook. The compare notebook builds a
`flow_map` in memory, writes it out as a **Parquet** file with pyarrow, and frees
it; the analysis notebook reads that file back with pandas. The write path
(`write_flow_map_parquet`) is **provided** — this page explains *why* it is
written the way it is, so the pattern is yours to reuse.

Like the [dpkt guide](Dpkt.md), the theme is memory discipline: the whole reason
to split the work across two notebooks and a file on disk is so the raw
per-packet state never has to coexist with the analysis.

## 1. Why pyarrow / Parquet here?

**Parquet** is a columnar, compressed on-disk format. It is the handoff between
the two notebooks: the compare notebook reduces millions of packets to a `flow_map`, writes
it to Parquet, and exits; the analysis notebook reads the Parquet back and never has to see
a raw packet. Columnar storage means the analysis notebook can also read back *only* the
columns it needs.

The subtler choice is writing it with pyarrow **directly** rather than through a
pandas `DataFrame`:

| Approach | Cost |
|---|---|
| **pyarrow direct** (`pa.table(...)` → `pq.write_table`) | Builds the Arrow table straight from the column lists you already have. One copy. |
| **`pandas.DataFrame(...).to_parquet(...)`** | Materializes a full pandas DataFrame *in addition to* the still-live `flow_map` — a second complete copy of the data, at peak memory, for no benefit on the write path. |
| **`polars`** | Also efficient, but an extra dependency this module doesn't otherwise need. |

Because `flow_map` is still resident when we write (it isn't freed until
`process_capture` calls `del` + `gc.collect()` afterward), avoiding a second
in-memory copy is exactly the kind of peak-memory saving that keeps these
captures — millions of flow records each — inside a 24 GB instance, and that
lets the same code scale to far longer stretches of telescope traffic.

## 2. Building columnar arrays from `flow_map`

Arrow (and Parquet) are **column-oriented**: you provide one list per column, not
one record per row. So the write function walks `flow_map` once, appending each
field to its own list, then wraps the whole dict of lists in a single
`pa.table(...)` call:

```python
import pyarrow as pa
import pyarrow.parquet as pq

columns = {
    "ip_src": [], "port_src": [], "ip_dst": [], "port_dst": [], "proto": [],
    "total_packets": [], "total_bytes": [], "first_seen": [],
    "last_seen": [], "tcp_flags_seen": [],
}
for key, stats in flow_map.items():
    columns["ip_src"].append(key[0])
    columns["port_src"].append(key[1])
    # ... one append per column ...
    columns["tcp_flags_seen"].append(",".join(sorted(stats.tcp_flags_seen)))

table = pa.table(columns)     # dict-of-lists → Arrow Table, inferring types
```

- **Build columns directly** from the source structure — don't assemble
  per-row dicts or a DataFrame first. `pa.table` takes a dict of equal-length
  lists and infers each column's type.
- **A Python `set` has no Parquet type.** `stats.tcp_flags_seen` is a `set`, so we
  join it into a single comma-separated string (`"SA,S"`) before writing —
  Parquet stores scalars and lists, not sets.

## 3. Writing — `pq.write_table`

One call persists the table to disk:

```python
pq.write_table(table, "capture_a.flow.parquet")
```

That's the whole write. pyarrow handles the columnar encoding and compression;
the resulting file is self-describing (it carries its own schema), so the reader
side needs no separate type declarations.

## 4. Reading back — `pd.read_parquet`

The analysis notebook reads the file straight into a pandas DataFrame:

```python
import pandas as pd

df_a = pd.read_parquet("capture_a.flow.parquet")
```

The pandas DataFrame is the right shape for the analysis notebook's work (group-bys,
enrichment joins), and by the time it exists the compare notebook's `flow_map` is long
gone. The **write → `del` → read** round-trip is deliberate: it guarantees only
one representation of the data is ever resident, so the raw per-packet state is
freed before the analysis begins.

## 5. Best practices, all together

- **Use Parquet for the handoff** — columnar, compressed, self-describing, and
  lets the reader pull only the columns it needs (§1).
- **Write straight from your source columns with pyarrow** — don't round-trip
  through a pandas DataFrame on the *write* path; it doubles peak memory (§1).
- **Feed `pa.table` a dict of column lists**, built in one pass over the source
  (§2).
- **Flatten unsupported types** — join a `set` to a string; Parquet has no set
  type (§2).
- **Free the source after writing** — `del flow_map; gc.collect()`, as
  `process_capture` already does, so the file on disk becomes the *only* copy
  (§4).
- **Read back only what you need** — the next stage rarely needs every column.

[README](README.md) | [Introduction](Introduction.md) | [Datasets](Datasets.md) | [Dpkt](Dpkt.md) | Pyarrow ⮕ | [Tasks](Tasks.md) | [Task 7](Task-hilbert-map.md) | [flow compare](flow_compare.ipynb) | [flow analysis](flow_analysis.ipynb) | [Slides](slides/)
