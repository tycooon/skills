## Mockup conformance

A user interface built from an approved mockup drifts from it one small decision at a time: a control swapped for the project's nearest component, a selected state that no longer reads, a label lost, a state the mockup implied but didn't draw. Each looks harmless in a diff, and none shows up in a test. A rendered side-by-side is the only check that catches them. This applies whenever a change implements a mockup, an artifact, a canvas or an image, whoever builds it.

### Draw the mockup for the build

- **Draw the project's components as they look.** A mockup cell that shows a bare link where the project renders a tag pill with copy and explorer buttons will not survive the build: the component brings its decoration along and the row grows. Draw the real look, or mark the cell as needing a compact variant.
- **Write the behaviour down.** A static canvas shows no click, key or scroll. Put a short note on the canvas for each interaction the design depends on: what a row click does, which elements stay pinned, what Enter does in a form or dialog, what a long value does when it doesn't fit.

### Render both, the same way

- **The mockup.** Serve its source with a static server (`python3 -m http.server`) and open it in a browser. A design tool's runtime scripts that fail to load don't matter: the markup and styles render. An image mockup is used as it is.
- **The build.** The dev server, against stub data shaped like the data the mockup shows: the same counts, long values and edge rows. Real data that differs makes every row look like a deviation.
- **Same viewport, same state.** One pair per view at the mockup's own width and at phone width (390 px): the same tab selected, the same row expanded, the same dirty field. Save them as `<view>-<width>-mockup.png` and `<view>-<width>-built.png` in an ignored folder.
- **One command.** Script the pairs and the measurements below (a Playwright script in the same ignored folder is enough), so the reviewer and the post-deploy check rerun them instead of skipping them.
- **States the mockup implies.** A static mockup draws one moment. Loading, refetching, empty, error and a long list are still part of the interface: render them in the built page and judge them against the mockup's look.

### The conformance table

The implementer writes it in the PR or MR description: one row per element the mockup shows, covering regions, controls, labels and wording, and states.

| Element | Result |
|---|---|
| Selected tab | `Match` |
| Filter placeholders | `Deviates: the shared filter component has no placeholder slot` |
| Per-tab counts | `Dropped: no count endpoint; out of scope` |

`Deviates` and `Dropped` each carry a reason: data that doesn't exist, a project component that must win, a brief that overrode the mockup. A difference with no row is an unexplained deviation. A project component that wins over the mockup gets its own row even when it looks close: its pill, buttons or glyphs are exactly what adds height.

The table also carries measured rows, mockup value next to built value, taken from the rendered pages rather than judged by eye:

- the height and text line count of a typical row, card or list item;
- the font family and size of the figures the page is about (times, amounts, the headline value) and of the body text;
- the background colour of the main surfaces;
- the gap between repeated items.

A row height off by more than about 20 %, an extra line of text, or a figure in another font family is `Deviates` with a reason, or a defect.

### Interactions

Screenshots show none of these, so each one gets a browser check in the change's tests, not a look:

- **Hit area equals highlight.** Every clickable element accepts a click anywhere on its hover or focus highlight. Click near each corner of the highlighted box and assert the click reached the control.
- **Keys.** Enter in a form submits it; an ordinary confirmation dialog focuses its primary action, so Enter confirms and Escape cancels, while a destructive one keeps focus on Cancel so a stray Enter dismisses it. The check presses Enter in both kinds of dialog.
- **Jumps and pinned elements.** An in-page jump lands at once, and its target isn't hidden under a pinned header or bar.
- **Phone width.** Nothing from the desktop layout, such as a pinned table header, shows over the phone layout.
- **Mounted hidden.** Behaviour that measures the page still works for an element mounted while its tab or panel was hidden.
- **What the mockup notes say.** Each behaviour note on the mockup (row click, pinned search) is exercised.

### Review

The reviewer renders the pairs and takes the measurements again rather than reading the implementer's screenshots, which show what the implementer chose to show. Walk the table against the pairs, then the pairs against the mockup for anything the table left out. Look at every pair, phone width included. Record the evidence in one top-level PR or MR comment of its own, posted before the verdict: the pairs rendered (view and width) and the measured values next to the table's. The verdict keeps the one-line format review-pr sets, and a review with no such comment at its head is not a conformance review. Nobody writes "matches the mockup" without such a pair behind it.

- A missing table on a change built from a mockup is a finding.
- A visible difference with no row, or a reason that doesn't hold, is a finding: P2 when a user would notice it (layout, hierarchy, density, wording, a selected or loading state that doesn't read, an interaction that fails), P3 when it is cosmetic (spacing, a shade).
- Accepted `Deviates` and `Dropped` rows are not findings.

Judge structure, hierarchy, density, wording, controls and states, not pixels: the project's tokens and components legitimately shift spacing and type by a little, but not a row's line count, a figure's font family or a surface's colour.

### After deploy

Repeat the pairs and the measurements against production for the views that shipped, at both widths, as part of the post-deploy check, and compare each production page with its mockup side by side, not only that it works. Real data shows what the stub didn't: long names, missing values, the real count of rows. A deviation found there goes back to the change's issue.
