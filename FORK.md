# pdf-inspector fork — SaarBarak/pdf-inspector

## What this fork is for

Fixes Hebrew/RTL text-extraction bugs upstream doesn't have a fix for yet:
visual-order text reversal, and a broken-`ToUnicode`-CMap CID mapping bug
(Hebrew `נ` decoding as `ð`). Consumed by the `anydoc` fork's
`[patch.crates-io]` redirect (see its `FORK.md`) — **this is the repo that
actually carries the behavioral fix**; anydoc's own fork is just wiring.

## Frozen base

`upstream-base` tag → `firecrawl/pdf-inspector` @ `65b7fa1` (2026-09-01,
"fix(loader): bound object-stream decompression at load (lopdf 0.44)
(#478)").

**Upstream has moved 24 commits since** (as of 2026-09-16), including a
release bump to 1.20.0. That is expected and fine under this fork's model —
see "Updating" below — not something to treat as urgent or reactively chase.

## What's on top of the base, in order

1. `1bee8d1` — fix(cid): resolve CIDs the ToUnicode CMap never names.
   Hebrew `נ` (U+05E0) was decoding as `ð`; recovers CIDs the embedded
   font's own ToUnicode CMap doesn't name.
2. `f4ed188` — fix(rtl): keep a number separator out of the forward run
   unless digits flank it. A BiDi-ordering edge case in mixed Hebrew/numeral
   text.
3. `f5ff9f7` — fix(rtl): let Hebrew orthography overrule a wrong
   visual/logical verdict. Corrects cases where the reading-order heuristic
   picks the wrong direction for a Hebrew-heavy line.

## Branch

`rtl-fix` is the one long-lived branch — it accumulates every patch this
fork carries. (Renamed from `fix/rtl-upstream-1.17`, which itself
superseded an earlier `fix/rtl-logical-order` — those version-bearing names
are exactly what this rename is meant to stop happening. Don't create a new
differently-named branch for the next patch; branch off `rtl-fix`, merge
back into it, re-tag.)

## Updating to a newer upstream base

Event-driven, not scheduled — do this only when we actually need something
new from upstream (this is exactly why we're touching it now: the OCR
module landed in one of the 24 commits since our freeze point).

1. Move `upstream-base` to the new upstream commit.
2. Rebase `rtl-fix` onto it.
3. Re-run `SysAgentsHarness`'s Hebrew/RTL suite
   (`tests/test_document_parser.py`) against the rebased build — that's the
   real regression gate; this repo's own test suite doesn't cover Hebrew.
4. Tag the result `vX.Y.Z-rtl-fix.N`.

## Adding a new patch

Branch off `rtl-fix`, do the work, merge back in, re-tag. Same process as
above, just without the rebase-onto-a-new-base step if the current
`upstream-base` still has everything you need.
