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
4. build: let the Python bindings build without the native OCR path.
   `python` was `["pyo3", "ocr"]`, so every Python consumer compiled ONNX
   runtime, a bundled PDFium and a TLS stack whether or not it OCRs
   anything — 262 crates against 99. Now `python = ["pyo3"]`, with
   `python-ocr` for the full path. Unlike items 1-3 this fixes nothing
   about extraction; it exists so the `anydoc` fork can pin its *Python*
   dependency here (see below) at a sane install cost.

   **Upstreamable, and worth offering.** It follows upstream's own stated
   intent — the `ocr` feature is commented there as opt-in precisely "so
   default library, renderer-only, and browser consumers do not inherit
   inference or HTTP/TLS", which the Python bindings were quietly
   contradicting.

## Why anydoc must pin this fork's *Python* package too

`anydoc` consumes this repo twice: its Rust core links the crate (redirected
by `[patch.crates-io]`), and its Python layer imports the Python package.
Only the first was ever redirected — the second resolves `pdf-inspector`
from PyPI, i.e. upstream, i.e. **without items 1-3**. So anydoc's Azure OCR
path rebuilds documents through unpatched code and Hebrew comes back
character-reversed, in the one fork that exists to stop exactly that.

Measured on a real Hebrew document, same file, same page:

| Path | Result |
|---|---|
| Rust core (this fork, via `[patch.crates-io]`) | `עיריית תל אביב` |
| Python `pdf-inspector` 1.20.0 (PyPI upstream) | `ביבא לת תייריע` |

Verified that pinning anydoc's Python dependency here fixes it through both
APIs that code calls, `extract_pages_markdown_bytes` and
`extract_text_with_positions_bytes`. Item 4 is what makes that pin
affordable.

## Branch

**`develop`** is the one long-lived branch — it accumulates every patch
this fork carries, and releases are tagged directly off it (no separate
release branch; `main` never receives these commits). Renamed from
`rtl-fix`, which itself was renamed from `fix/rtl-upstream-1.17`, which
superseded an earlier `fix/rtl-logical-order` (**not merged into this
line at all** — a rebase makes new commits even for an identical fix, so
that branch is a dead end, kept around only for history/its own tag).
Those version-bearing names are exactly what this rename is meant to stop
happening — `develop` describes the branch's role, not whichever patch
came first.

**Working on it, including in parallel with Saar:** never commit directly
to `develop`. Cut a short-lived branch off it per patch, merge back via PR
when it's done, then re-tag. `develop`'s tip is then always either fully
done or not yet touched — never half-finished — which is what makes
concurrent work by more than one person safe on a single branch.

**`main` stays a plain, untouched mirror of upstream**, not a merge target.

**When to introduce a second branch (a `main-fork` line):** only if either
(a) a release needs to be cut while another patch is genuinely mid-flight
and can't be merged or set aside, as a recurring situation, or (b) two
deployments need to diverge onto separately-maintained patch sets. Neither
applies today.

## Updating to a newer upstream base

Event-driven, not scheduled — do this only when we actually need something
new from upstream (this is exactly why we're touching it now: the OCR
module landed in one of the 24 commits since our freeze point).

1. Move `upstream-base` to the new upstream commit.
2. Rebase `develop` onto it.
3. Re-run `SysAgentsHarness`'s Hebrew/RTL suite
   (`tests/test_document_parser.py`) against the rebased build — that's the
   real regression gate; this repo's own test suite doesn't cover Hebrew.
4. Tag the result `vX.Y.Z-fork.N`, where `X.Y.Z` is the upstream base this
   fork sits on and `N` counts this fork's releases against it.

   **Why not `-rtl-fix.N` any more.** That name described the *first* patch,
   not what a tag off `develop` actually is now — the branch was renamed
   away from `rtl-fix` for exactly this reason (see Branch, above), and the
   tags never followed. The RTL fix is permanently part of `develop`; every
   tag carries it, so naming it in the tag says nothing. Patch 4 made the
   mismatch obvious: it is a build change with no RTL content at all, and
   `v1.17.0-rtl-fix.3` would have been actively misleading.

   Existing `-rtl-fix.N` tags stay as they are — they are immutable history
   and the `anydoc` fork pins one of them. Only new tags use `-fork.N`, and
   `fork-wheels.yml` triggers on `v*-fork.*` to match.

## Adding a new patch

Branch off `develop`, do the work, merge back in, re-tag. Same process as
above, just without the rebase-onto-a-new-base step if the current
`upstream-base` still has everything you need.
