# Memory Overview Page · Spec

What the memory overview page shows when a recorded trace is opened, where every
number comes from, and the smells it surfaces.

> memory overview page · v3 · overview screen · web app

**Job of the memory overview page → Triage, not diagnosis.** Orient someone who
just opened a trace: how much memory, growing or steady, which way to dig
(managed vs native vs file-backed), and where to click. smaps owns the totals
(OS truth); the two profilers explain what's inside. Every element traces to a
real measurement or an honest "unknown."

**Time-erasure.** The page is built to be as non-time-based as possible: it
describes a _selected snapshot_ (or a _diff between two snapshots_), not a
timeline. The one time-based element — the composition-over-time chart — exists
only to pick that snapshot/range. This is what makes traces comparable and
diffable.

## Summary

For a single selected process, the Summary tab shows, top to bottom:

1. **Score cards** — headline process stats.
2. **Memory analysis** — where the anon+swap change went (native vs Java vs
   other), as a single stacked bar.
3. **Composition over time** — the only chart. A stacked area of resident memory
   by region. Click a point to select a snapshot; drag to brush a range. The
   selection drives every section below.
4. **Where did all the memory go?** — a multi-level composition breakdown of the
   selected snapshot (or deltas, when a range is selected).
5. **How much Java memory did you use, and where did it go?** — heap dump
   analysis: reachability, top classes, dominators.
6. **What about bitmaps?** — bitmaps broken out on their own.
7. **How much native memory did you use, and where did it go?** — heapprofd
   coverage and the top allocation call-stacks.

A second **Smaps** tab shows every mapping, grouped into the same taxonomy.

Sections whose data source is absent for the selected process are dropped (not
shown as placeholders), except per-block "unknown" remainders, which are always
shown rather than hidden.

## Selection model (the core idea)

The composition chart is the page's selector. Two modes, mirrored by every
section below:

- **Single snapshot** (click a point) → sections show the **absolute** state at
  that snapshot.
- **Range** (drag to brush two points) → sections show the **delta** (selected −
  baseline). The chart marks the baseline (muted) and selected (red) snapshots;
  section titles turn red while diffing, blue for a single snapshot.

The three data sources resolve a selection differently, because their data has
different shapes:

| Source         | Shape                       | Single snapshot                            | Range                              |
| -------------- | --------------------------- | ------------------------------------------ | ---------------------------------- |
| **smaps**      | complete snapshots          | the snapshot's own contents                | stateB − stateA                    |
| **java_hprof** | complete snapshots          | nearest dump to the snapshot               | nearest(sel) − nearest(base)       |
| **heapprofd**  | incremental (per-dump Δ)    | cumulative from record-start → snapshot    | net allocated between base and sel |

smaps/Java dumps are each a full state. heapprofd is incremental: a single dump
is meaningless alone; a "state" is the running sum of deltas from record-start.
That's the can't-see-the-past limitation — the running total starts at zero when
tracing began, not at process start. The range diff is heapprofd's strongest
view because the pre-trace blind spot cancels out.

## 0 · Inputs

| Source           | Description                                                                                                   |
| ---------------- | ------------------------------------------------------------------------------------------------------------- |
| **smaps**        | Absolute per-mapping resident memory, sampled periodically. The truth. Totals + region breakdown.             |
| **java_hprof**   | Absolute managed-object state (ART heap dump), sampled periodically. Per-class counts, sizes, retention.      |
| **heapprofd**    | Net (alloc−free) by callsite since record start, sampled periodically. malloc only.                           |
| **proc polling** | Cheap counters: `mem.rss`, `mem.rss.watermark`. Plus uptime / oom_score_adj carried on the heap-graph stats.  |

Don't assume a fixed cadence — plot each source at its own real timestamps.

## 1 · Layout

- **Title** — "Memory Overview".
- **Capture strip** — process identity + one entry per data source with a
  colored dot and terse facts (🟢 smaps, 🔵 java_hprof, 🟠 heapprofd).
- **Process selector** — shown when multiple processes have data. Default is the
  process with the most data, weighted: heap dumps (3×) > smaps (2×) >
  native profiles (1×).
- **Tabs** — Summary · Smaps.

## 2 · Score cards

Headline stats, merged into compact dual-stat cards:

- **Uptime + OOM score** — process uptime and `oom_score_adj` (with a coarse
  label: foreground / perceptible / service / cached). Source: heap-graph stats.
- **Peak RSS anon+swap + RSS spike** — max anon+swap across smaps snapshots, and
  the hi-watermark − max gap from the polled counters.
- **Memory Δ + Trend** — anon+swap change first→last snapshot, absolute and
  normalised to per-hour.

> Not yet implemented: a **GC churn** card (GCs/hour + MB/hour) from
> PostGCMemorySnapshot (statsd atom 924) and churn-mode native sampling — blocked
> on those data sources being present.

## 3 · Memory analysis

A single stacked bar attributing the **anon+swap change** between the baseline
and selected snapshots (or first→last by default) to **Native heap** / **Java
heap** / **Other**, with a headline delta, a per-hour rate, and a legend listing
every region's signed contribution. Source: smaps summary rows
(dalvik / native / other anon+swap).

## 4 · Composition over time (the chart)

- **Type:** stacked area, one series per smaps region (resident + swap): Native,
  Java, File-backed, Graphics, Thread stacks, Other.
- **Interaction:** click a point → select that snapshot; drag → brush a range to
  compare two. A row of snapshot chips mirrors the selection. Markers show the
  selected snapshot (red) and, in range mode, the baseline (muted).
- **Role:** deliberately short — it's the selector, not the focus. Everything
  below reflects the current selection.

## 5 · Where did all the memory go? (composition breakdown)

A multi-level "flamegraph" of the selected snapshot, each row summing to the
snapshot total so blocks line up and widths are comparable. In range mode the
block values become deltas vs the baseline.

- **Level 1:** File-backed · Anonymous · Graphics · Other.
- **Level 2:**
  - File-backed → Native libs (`.so`) · Java code (`.jar`/`.oat`/`.odex`/
    `.vdex`/`.art`) · Resources / APK · Other files.
  - Anonymous → Native · Java · Thread stacks · Other anon.
- **Level 3** (cross-source attribution):
  - Native → **Seen by profiler** (heapprofd unreleased at the selection) ·
    **Allocator overhead** (the remainder).
  - Java → **Reachable** (heap-graph reachable at the nearest dump) ·
    **Unreachable / other**.

Each block carries a **dual-tone shading** (dirty/swap vs clean vs shared) on the
top two levels. Hovering a block shows a plain-language explanation in a footnote
below the bars. An insight callout names the biggest and fastest-growing region.
A link opens the corresponding smaps counter track on the timeline.

> Not yet matching the original taxonomy: File-backed is not split into
> **Shared (zygote)** vs **Private** before the path buckets (only the shading
> conveys it); **ART overhead** is a Java card, not a tree node; and the
> **shared-memory / dmabuf** node is not represented.

## 6 · How much Java memory did you use, and where did it go? (java_hprof)

Requires ≥ 1 heap dump. Resolves to the dump nearest the selected snapshot
(nearest the baseline too, in range mode).

- **Heap reachability** ratio bar — reachable vs unreachable bytes, with per-leg
  and percentage-point deltas in diff mode.
- **Cards** — Heap (reachable + unreachable), Live objects, Registered native,
  ART overhead (from the smaps `.art`/`.oat`/jit paths).
- **Three tables**, each half-width:
  - **Top retainers by class** — Class · Retained · +native · Inst. · Share.
  - **Top dominators** — Class · Retained · R.count · Share.
  - **Top classes by instance count** — Class · Instances · Shallow · Retained ·
    Share of count.
- **Nearest non-library retainer** — each class cell shows a `↳ via <AppClass>`
  second line: the closest non-framework class that dominates its instances, from
  a dominator-tree walk on the last dump (framework + language-runtime packages —
  `java.*`, `android.*`, `kotlin.*`, `dalvik.*`, etc. — are skipped). Computed
  defensively; absent when the heap graph has no dominator tree.
- A link opens the **Heap Dump Explorer** (`#!/heapdump?upid=…&ts=…`).

> Not yet implemented: separate **Shortest-path** and **Dominator-tree**
> flamegraph links (currently a single explorer link).

## 7 · What about bitmaps? (java_hprof)

Reachable `android.graphics.Bitmap` instances, grouped by dimensions and storage
backing, from the dump nearest the selection.

- **Cards** — Total bitmaps · Bitmap memory (broken down heap vs ashmem, with an
  estimate of pixel bytes for non-heap backings) · Largest group · share of Java
  retained.
- **Tables** — Largest bitmaps and Most frequent bitmaps, both grouped by
  dimensions, with a "Share of bitmaps" bar.
- An insight names the dominant dimensions and, where available, the nearest
  non-library class retaining the bitmaps.

## 8 · How much native memory did you use, and where did it go? (heapprofd)

Requires ≥ 1 native profile. Opens with a warning that the profiler only sees
allocations made **after** tracing started — it can't explain the past.

- **Profiler coverage** ratio bar — heapprofd unreleased vs the smaps native
  allocator footprint at the selection (the rest predates the trace).
- **Cards** — RSS anon+swap · Seen by profiler · Allocator overhead + unseen ·
  Thread stacks (with thread count).
- **Top allocation call-stacks** — loaded **on demand** for the selected window
  (single snapshot → record-start → snapshot; range → baseline → selected), so
  the table tracks the selection and shows Δ unreleased while diffing. Columns:
  call-stack snippet · (Δ) unreleased · allocs · share of profiled. Snippets are
  rendered **vertically** (leaf → root) and contain **app frames only** — frames
  whose mapping lives under `/data/app` or `/data/data`; system/runtime library
  frames are dropped. Empty/loading states are shown explicitly (e.g. selecting
  before the profiler started yields "no app allocations captured here").

## 9 · Smaps tab

"Every mapping, grouped" — the raw `/proc/<pid>/smaps`, folded into the same
taxonomy as the composition breakdown.

- **Cards** — Total RSS · RSS anon+swap · Total PSS · Private dirty.
- **Tree** — two-level group/subgroup taxonomy with collapsible nodes and
  per-group totals; long groups are capped with a "+N more" row.
- **Flat toggle** — undo the grouping and list individual mappings.
- **Regex filter** — filter mappings by path (falls back to substring on an
  invalid regex).
- **All-columns toggle** — adds PSS, private clean, shared dirty/clean and swap
  to the default RSS / private-dirty / swap columns.

## 10 · What the page explicitly does not claim

- It is **not** total RSS in the headline anon+swap figures — file-backed pages
  are reclaimable and tracked separately in the composition breakdown.
- The region breakdown is best-effort attribution by mapping name; the
  unexplained remainder is shown, not hidden.
- The native profiler is **malloc-scoped**, post-record-start only.
- The "nearest non-library retainer", the file-backed buckets, and the level-3
  cross-source splits are heuristics; they show their evidence rather than
  asserting.
- No object values exist in the dump (no string contents, no bitmap pixels) — so
  no duplicate/boxing analysis.

## 11 · Known gaps & issues

- **Spec gaps:** GC-churn card; two named flamegraph links; File-backed
  shared/private split + ART/dmabuf composition nodes. GC-churn and dmabuf are
  data-source-blocked.
- **Retainer query is unvalidated at scale** — the dominator-tree walk is scoped
  to the last dump per process and depth-capped, and wrapped defensively, but
  hasn't been profiled on large heaps.
- **Native call-stack top-N** is recomputed per selected window (correct
  membership), at the cost of a per-selection query; `QuerySlot` keeps it from
  re-running the whole page load and retains stale rows while reloading.
- smaps snapshots can be attributed to a different upid than the parent process
  (some appear under `<unknown>`/`init`) on older builds; needs a recent
  platform build for clean smaps.

---

**One-line summary:** smaps gives the honest totals and region breakdown; the
Java dump and native profiler explain what's inside their regions; a single
chart selects a snapshot or a range, and every section re-frames itself as an
absolute view or a diff. Total is truth, breakdown is best-effort, unknown is
shown.
