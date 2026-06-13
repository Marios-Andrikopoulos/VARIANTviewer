# Dev Notes - V² Variant Viewer

A running log of technical decisions, bugs found, and things learned while building this tool.

---

## Entry 1 - Architecture Decision: Why Canvas?

The first question was how to render the table. The first and most familiar answer for me was a straightforward `<table>` element with multiple '<tr>', but that fell apart pretty fast when faced with real genomics data - a WES run produces ~50,000 variants, a WGS run can hit 4–5 million. Rendering that many DOM nodes locks the browser for seconds and makes the tool effectively useless.

The two solutions I found most frequently were:

1. **Virtual DOM scrolling** - render only visible rows as real HTML elements, swap them as the user scrolls (React Window, TanStack Virtual, or a hand-rolled version)
2. **Canvas rendering** - paint rows as pixels directly, no DOM nodes at all

I couldn't quite get the first one to work (see Entry 3), so I went with Canvas which I was a bit more familiar with. The upsides were quite compelling: zero DOM overhead, complete control over layout and styling, very fast for read-only data. For this however, we give up accessibility, text selection, and browser-native click semantics - all of which have to be re-implemented manually. The details page that appears upon clicking a variant partially adresses this.

---

## Entry 2 - Column-of-Arrays Data Layout

The variant data is stored as parallel arrays (`V.chrom[]`, `V.pos[]`, `V.gene[]`, ...) rather than an array of objects (`[{chrom, pos, gene}, ...]`).

This is a struct-of-arrays (SoA) pattern. When you filter or sort, you're usually reading one or two fields across all N variants. With SoA, those reads are ideally contiguous in memory, so the CPU prefetcher handles it well. With array-of-structs, each variant object is a separate heap allocation and field reads jump around.

---

## Entry 3 - The INP Problem

After adding virtual scroll, performance was still poor on load - measured a 23,496ms Interaction to Next Paint value. The Chrome profiler showed the presentation delay was the culprit, not input delay or processing time.

Likelly root cause: setting `scrollSpacer.style.height = (N * ROW_H) + "px"` synchronously inside the data-load handler. Setting a height on a DOM element that's in the document forces a synchronous layout recalculation. With N = 50,000 and the spacer inside a flex container, the browser had to resolve the entire layout tree before it could paint anything - hence 23 seconds.

---

## Entry 4 - Index-Based Filtering

Filtering by chromosome/classification/type uses pre-built inverted indexes (`IDX.chrom`, `IDX.class`, `IDX.type`) - Maps from value to list of variant IDs. When multiple filters are active, the result is the intersection of those lists.

This avoids scanning all N variants on every filter change. For a single active filter, lookup is O(1). For multiple filters, intersection is O(min set size) with a Set for fast membership testing.

The search query is applied after index filtering as a linear scan, which to my mind is acceptable because by that point the candidate set is usually smaller.

---

## Entry 5 - `clearAll()` Mutates Arrays In-Place
 
When resetting the viewer, `clearAll()` does this:
 
```js
Object.keys(V).forEach(key => V[key] = []);
```
 
rather than:
 
```js
V = {}; // or V = { chrom: [], pos: [], ... }
```
 
`V` is referenced in many places throughout the code and if you reassign `V` to a new object, any code that captured a reference to the old `V` now holds a stale reference pointing at the old data. By mutating in-place (replacing each array with a new empty array), `V` itself stays the same object at the same memory address. All existing references to `V` remain valid and automatically see the cleared state.
 
I struggled with this in other JavaScript projects, so I thought it prudent to include this as a note here. The symptom of getting it wrong is that data appears to persist after a clear, or old data shows up briefly after reload.

---

## Entry 6 - Chromosome Sort with NaN

The chromosome sort initially used `parseInt(a) - parseInt(b)`. Here I forgot about the concept of biological sex it seems. `parseInt('X')` returns `NaN`. Any comparison involving `NaN` returns `false`, so the sort comparator becomes non-deterministic for sex chromosomes - the browser's sort algorithm produces unpredictable ordering.

Fix:

```js
const n = x => x === 'X' ? 23 : x === 'Y' ? 24 : parseInt(x);
return n(a) - n(b);
```

Maps X and Y to sentinel values above the autosomes so they sort to the end correctly. Same pattern used in both `renderDashboard` and `populateDropdowns`.

---

## Entry 7 - HiDPI / Retina Canvas

On a 2× display, a canvas drawn at CSS pixel dimensions looks blurry - each CSS pixel is 2 physical pixels, and the canvas is being upscaled. Fix is to set `canvas.width = cssWidth * dpr` and `canvas.height = cssHeight * dpr`, then call `ctx.scale(dpr, dpr)` before drawing. All drawing code then uses CSS pixel coordinates as normal.

This is handled in `setupCanvasScale()` for dashboard charts and inline for the table canvas in `resize()`. If you ever add a new canvas element, always go through `setupCanvasScale` - don't set width/height directly.

---


## What's Next

- **Web Worker VCF parser** - move `parseVCFData` off the main thread; this will hopefully keep the UI responsive during large file loads
- **ClinVar API integration** - on variant selection, fetch live classification data instead of relying on INFO field keywords; hopefully do it in a way that doesn't make ClinVar hate me.
- **Export** - filtered variant list to CSV/TSV/VCF
- **IGV.js embed** - deep link from the detail panel into a genomic browser view