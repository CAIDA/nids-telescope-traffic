[README](README.md) | [Introduction](Introduction.md) | [Datasets](Datasets.md) | [Dpkt](Dpkt.md) | [Pyarrow](Pyarrow.md) | [Tasks](Tasks.md) | Task 7 Guide ⮕ | [flow compare](flow_compare.ipynb) | [flow analysis](flow_analysis.ipynb) | [Slides](slides/)

# Task 7 Guidance: Visualizing a Scan with a Hilbert Curve

This guide shows how the scanner's destination addresses, from Task 7, are laid out as a 2-D image that preserves address-space locality.

### Why Not Just List Destinations?

The destination /16 contains 65,536 possible addresses. (The notebook picks the /16 the scanner sent the most packets to. Because the telescope's destination space is anonymized into `10.0.0.0/8`, in practice there is only one such /16 here — the selection matters when the same code is pointed at traffic spanning more address space.) A table of "destination IP → packet count" is complete but hard to read as a whole — you can't easily tell whether the scanner covered the block systematically, skipped ranges, or clustered around a few subnets. Laying all 65,536 addresses out on a 2-D grid solves this, *provided* the layout keeps numerically nearby addresses visually nearby — otherwise the image is just noise.

### Why a Hilbert Curve (and Not Row-Major Order)

The obvious layout — address `256*o2 + o3` maps to raster position `(o2, o3)` — already keeps the third and fourth octets close together in one direction, but jumps a full row for every increment of the third octet, splitting a contiguous range like `x.x.5.250`–`x.x.6.10` across two visually distant rows. A **Hilbert curve** is a space-filling curve with a stronger guarantee: any two indices that are close together in the 1-D sequence map to points that are close together in 2-D, **at every scale** — a contiguous range of addresses always forms a compact, connected region in the image, not just within a single row. This is why Hilbert curves are a standard choice for visualizing IP address space (and other hierarchical numeric spaces).

<img src="images/hilbert-curve.png" width="150px">


### The Algorithm

A Hilbert curve of **order** `k` covers a `2^k × 2^k` grid. For an order-8 curve (256×256 — exactly enough for one /16's third and fourth octets), converting a 1-D distance `d` (0 to 65,535) to an `(x, y)` coordinate works by peeling off 2 bits of `d` at a time, from the most significant end, and using them to rotate/reflect a running `(x, y)` position:

```python
def _rot(n, x, y, rx, ry):
    if ry == 0:
        if rx == 1:
            x = n - 1 - x
            y = n - 1 - y
        x, y = y, x
    return x, y

def hilbert_d2xy(order, d):
    n = 1 << order
    x = y = 0
    t = d
    s = 1
    while s < n:
        rx = 1 & (t // 2)
        ry = 1 & (t ^ rx)
        x, y = _rot(s, x, y, rx, ry)
        x += s * rx
        y += s * ry
        t //= 4
        s *= 2
    return x, y
```

### Mapping Addresses to the Grid

For a destination IP `o0.o1.o2.o3` within the target `o0.o1.0.0/16`, the distance along the curve is `d = o2 * 256 + o3` (the address's position within the /16, treating the third and fourth octets as one 16-bit number). Feed `d` to `hilbert_d2xy(8, d)` to get its `(x, y)` position in the 256×256 image, then accumulate that destination's packet count into `grid[y][x]`.

### Worked Example

```
Destinations probed (within 10.42.0.0/16):
  10.42.0.1   -> 40 packets   (o2=0, o3=1   -> d=1)
  10.42.0.2   -> 40 packets   (o2=0, o3=2   -> d=2)
  10.42.5.200 -> 12 packets   (o2=5, o3=200 -> d=1480)

hilbert_d2xy(8, 1)    -> (1, 0)     # next to d=0's origin corner at (0, 0)
hilbert_d2xy(8, 2)    -> (1, 1)     # adjacent to (1, 0) - d=1 and d=2 are close on the curve
hilbert_d2xy(8, 1480) -> (13, 53)   # far from the above - a different region of the /16

grid[0][1]   += 40
grid[1][1]   += 40
grid[53][13] += 12
```

The two addresses that are numerically adjacent (`.0.1` and `.0.2`) land on adjacent pixels; the address 1,479 addresses away lands somewhere else in the image entirely — exactly the locality property that makes gaps and clusters in the scanner's coverage visible at a glance.

### Reading the Image: `(x, y)` Is Not `(o2, o3)`

The curve *reorders* the address space, so the axes of the finished image are **Hilbert curve positions, not octets**. In the example above `o2=0, o3=1` lands at `(x=1, y=0)` — not `(0, 1)`. You cannot name a block by reading its coordinates off the axes.

What the curve does guarantee is that each **/24** — 256 consecutive values of `d` — occupies exactly one **aligned 16×16 cell** of the image. That is what the 16-pixel gridlines mark, and the notebook prints each cell's third octet inside it. To answer Q8, read those printed labels.

[README](README.md) | [Introduction](Introduction.md) | [Datasets](Datasets.md) | [Dpkt](Dpkt.md) | [Pyarrow](Pyarrow.md) | [Tasks](Tasks.md) | Task 7 Guide ⮕ | [flow compare](flow_compare.ipynb) | [flow analysis](flow_analysis.ipynb) | [Slides](slides/)
