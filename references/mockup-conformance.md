## Mockup conformance

A user interface built from an approved mockup drifts from it one small decision at a time: a control swapped for the project's nearest component, a selected state that no longer reads, a label lost, a state the mockup implied but didn't draw. Each looks harmless in a diff, and none shows up in a test. A rendered side-by-side is the only check that catches them. This applies whenever a change implements a mockup, an artifact, a canvas or an image, whoever builds it.

### Render both, the same way

- **The mockup.** Serve its source with a static server (`python3 -m http.server`) and open it in a browser. A design tool's runtime scripts that fail to load don't matter: the markup and styles render. An image mockup is used as it is.
- **The build.** The dev server, against stub data shaped like the data the mockup shows: the same counts, long values and edge rows. Real data that differs makes every row look like a deviation.
- **Same viewport, same state.** One pair per view at the mockup's own width and at phone width (390 px): the same tab selected, the same row expanded, the same dirty field. Save them as `<view>-<width>-mockup.png` and `<view>-<width>-built.png` in an ignored folder.
- **States the mockup implies.** A static mockup draws one moment. Loading, refetching, empty, error and a long list are still part of the interface: render them in the built page and judge them against the mockup's look.

### The conformance table

The implementer writes it in the PR or MR description: one row per element the mockup shows, covering regions, controls, labels and wording, and states.

| Element | Result |
|---|---|
| Selected tab | `Match` |
| Filter placeholders | `Deviates: the shared filter component has no placeholder slot` |
| Per-tab counts | `Dropped: no count endpoint; out of scope` |

`Deviates` and `Dropped` each carry a reason: data that doesn't exist, a project component that must win, a brief that overrode the mockup. A difference with no row is an unexplained deviation.

### Review

The reviewer renders the pairs again rather than reading the implementer's screenshots, which show what the implementer chose to show. Walk the table against the pairs, then the pairs against the mockup for anything the table left out.

- A missing table on a change built from a mockup is a finding.
- A visible difference with no row, or a reason that doesn't hold, is a finding: P2 when a user would notice it (layout, hierarchy, wording, a selected or loading state that doesn't read), P3 when it is cosmetic (spacing, a shade).
- Accepted `Deviates` and `Dropped` rows are not findings.

Judge structure, hierarchy, wording, controls and states, not pixels: the project's tokens and components legitimately shift spacing and type.

### After deploy

Repeat the pairs against production for the views that shipped, at both widths, as part of the post-deploy check. Real data shows what the stub didn't: long names, missing values, the real count of rows. A deviation found there goes back to the change's issue.
