# Advanced-flow auto-layout

Heuristics for computing card `x`/`y` coordinates when *generating* an advanced flow programmatically (not hand-placing them). Naive uniform spacing validates fine but looks broken in the Homey app: cramped connectors, cards landing exactly level with their source, tall text cards squeezed against their neighbors. Tuned through iterative visual feedback across several rounds.

## Layout model

Depth (longest path from a trigger) is the column (`x`) axis. Each column's cards are packed vertically (`y`) by the barycenter (average `y`) of their already-placed parents, ties broken by insertion order.

```
COL_W   ~340-460px   column width per depth level
ROW_H   ~120-180px   base row height
```

## Widen spacing around convergence AND divergence points

A card with **2+ incoming** edges — an explicit `any`/`all` merge, or a plain action fed by multiple parallel branches (e.g. one `create_notification` reached from 3 different condition checks) — needs roughly double the column width from its sources, or the lines crowd on entry.

The mirror case matters just as much: a card with **2+ distinct outgoing** targets (nearly every `condition` card, via `outputTrue`/`outputFalse`) needs its children spread apart with extra row height, or the lines leaving its bottom edge have no room to diverge before reaching their targets.

Judge this structurally — in-degree/out-degree — not by card `type`. A `create_notification` fed by 4 parallel checks needs the same treatment as an explicit `any` card.

```python
weight   = 2 if len(parents) >= 2 else 1                 # column weight for edges INTO a convergence point
row_gap  = ROW_H_HUB if is_hub_or_fanout else ROW_H       # ~210-220px vs ~120-180px
```

## Never let a card land exactly level with its parent

Connectors visually exit from the **bottom** of a card. If a child's computed `y` lands exactly on its (single) parent's `y` — which plain barycenter placement does by default for every 1-to-1 chain step — the connector bows into an unwanted arc instead of drawing straight. Add a small deliberate jitter (~45px) whenever the computed position would coincide exactly with any parent's `y`, including the subtler case where a multi-parent barycenter happens to land exactly on one specific parent (e.g. an evenly-spaced trio's average equals the middle one).

## Size text-heavy cards proportionally

`create_notification`/`push_text` cards render their `text` arg on the card face — a long interpolated message (with `[[homey:device:...|cap]]` tokens) renders visibly taller/wider than a plain action card. Estimate displayed length by collapsing each `[[...]]` token to a short placeholder first (so a long device UUID doesn't inflate the estimate), then scale row/column spacing proportionally to the estimated wrapped-line count.

## Give section notes room to breathe

If cards are grouped into logical sections marked by `note` cards, give the note extra vertical gap (~1.6x base row height) before the first real card — otherwise a chain starting right after the note crowds the section header.
