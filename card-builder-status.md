# isiBheqe Card Builder — Status Doc

*Paste this at the start of a new chat, along with the current `index.html`, to resume work without re-diagnosing from scratch. Update it yourself as things change — this is a living doc, not a transcript.*

**Last updated:** 24 Aug 2026

---

## What this is

A single self-contained HTML tool (no build step, no server, works offline from disk) for building isiBheqe syllabeme cards with Latin transliteration underneath each glyph. Built for the songbook project's scribes.

- **Live at:** barc-prog.github.io/isibheqe-card-builder/
- **Deployed via:** GitHub Desktop → push to `main` → GitHub Pages, root deploy, file is `index.html`
- **Upload note:** GitHub web upload only *replaces* a file if the name matches exactly (`index.html`) — check the repo file list after uploading from mobile, since some mobile browsers rename downloads and silently create a second file instead of overwriting.
- **Caching note:** GitHub Pages serves with a **10-minute cache header**. If you upload a fix and reload the same tab within 10 min, you may see the old version with no indication it's stale. Open in an incognito/private tab to bypass this and confirm.

## Engine origin

Built on top of three original files from the `isibheqe.org.za/demo` codebase, inlined directly into the single HTML file:
- `isibheqe-glyphs.js` — glyph geometry (`GlyphFactory`, `Glyph`, vowel/consonant drawing)
- `isibheqe-rendering_tracking-fix.js` — `Context` (phrase sequencing, `serialize`/`deserialize`), `makeTarget` (canvas), `makeSvgTarget`
- `key-mappings.js` — physical keyboard → glyph letter mapping

All app code (the card builder itself) sits in a single `<script>` block after these three are inlined.

## Core data model

- **Code string**: the engine's native serialisation — e.g. `[["i",["n"]]," ",["a",["°"]]]`. A phrase is a JSON array; each element is either `" "` / `"\n"` (breaks) or `[vowel, [marks...]]` or a bare vowel string.
- **Record**: the tool's own JSON wrapper — `{entry_id, round, language, latin, gloss, scribe, code, syllables:[{code, latin}, {break:"word"}, ...]}`. `round` is the underlying field name for what the UI now labels "Theme" — **kept as `round` in the JSON deliberately**, for backward compatibility with already-shared entries. Don't rename the field without a migration plan.
- **No "m" key.** Nasals use the `°` modifier (`["a",["°"]]` = "ma"), not a literal `m` consonant. Unknown marks are silently dropped by the engine, not flagged.
- **Latin is typed by hand, never auto-derived**, because the same glyph maps to different Latin spellings across languages (the sha/xa problem — isiZulu vs Xitsonga).

---

## Known bugs

### Confirmed, diagnosed, NOT YET FIXED

**Colour view miscounts syllables relative to strip/editor.**
Root cause confirmed: in `splitCodeUnits`, the space-detection line
```js
var t = u.text.replace(/\s+/g,"");
if(t === '" "'){ ... }
```
strips whitespace before comparing against `'" "'` — but that strips out the very space it's checking for, so the check can never match. Every space gets miscounted as a real glyph, shifting the colour view's numbering by one for every space passed. This is why clicking the 7th colour-view snippet opens the 6th syllable in the editor, and why a 10-syllable phrase with 2 spaces shows "12" in the colour view while the strip/editor correctly show "10 of 10" (the strip and editor were never wrong — they count from `state.tokens` directly; only the colour view's separate hand-rolled parser has this bug).
**Fix is small and localised** to that one comparison in `splitCodeUnits` — not yet applied or verified.

### Logged, not yet actioned

- **Overall spacing feels too loose.** Suspected side effect of the nasal-clipping frame fix (the render frame was enlarged from ~88×130 to 200×230 to stop everyday consonants like `f`/`s`/nasals from clipping — see Fixed section below). Not yet traced to a specific CSS rule; needs investigation, not just a guess-and-tweak.
- **No way to insert a space between syllables from inside the structured syllable editor.** Prev/Next/Delete exist; adding a space currently requires dropping to the raw code string. Worth designing as a first-class editor action, not a raw-text workaround, since spaces are already first-class tokens in the data model.

---

## Fixed this project, worth knowing about (in case of regression)

- **Nasal/consonant clipping** — old glyph render frame (88×130 px at unit 5) clipped 320 of 983 possible vowel+consonant+modifier combinations, including plain `f` and `s` with no marks — not nasal-specific, a systemic frame-sizing bug. Fixed by sizing the frame from an exhaustive sweep (verified 0/983 overflow). This required keeping **three places in sync**: `renderSyllable` (strip/editor canvas), `cardOps` (card layout, `AR` constant), `buildCardSvg`/`glyphSvgParts` (SVG export) — all now reference shared `GLYPH_FRAME_W_U`/`GLYPH_FRAME_H_U`/`GLYPH_OFFSET_X_U`/`GLYPH_OFFSET_Y_U` constants. **If glyphs ever look wrong on the exported card but fine in the strip (or vice versa), check these three are still using the same constants** — they drifted out of sync once already during this build.
- **Glyph-slot overlap** — the frame enlargement above made drawn glyphs ~21% wider than their card layout slot, which would have overlapped adjacent glyphs on export. Fixed with a clamp in `cardOps` (`if(dw > gw){ dw = gw; dh = dw/AR; }`) that inscribes the glyph inside its slot on both axes. Confirmed zero overlaps on a real 24-syllable stress phrase.
- **Latin carry-forward on code edits** — originally compared old/new tokens by array index, so inserting or deleting even one syllable while hand-editing shifted every position after it and silently wiped their Latin. Replaced with a proper LCS (longest common subsequence) alignment in `loadFromText`, which matches syllables by content and relative order rather than raw position — correctly handles duplicate syllables too (tested: three identical `["a",["k"]]` tokens with an insert between them each kept their own distinct Latin).
- **Stale syllable editor on file load** — `loadFromText` never called `closeSyllableEditor()`, so if the editor was open when a different file loaded, it kept a closure reference to the *old* token index — typing into it could silently corrupt the *new* phrase at the wrong position. Fixed: `loadFromText` now closes the editor first.
- **Portrait mobile layout** — was previously a forced "rotate your phone" gate; replaced with a genuine single-column portrait layout and a collapsible keyboard drawer (collapsed by default only on narrow screens, checked via `matchMedia`).
- **Portrait/landscape media query collision** — the new portrait rule (`max-width:900px`, caps keyboard at `46vh` for stacking) and an older, separate "compact landscape" rule (`max-height:560px`, two-column layout, built in an earlier session) both match on a real phone in landscape (e.g. 844×390 satisfies both thresholds simultaneously). The portrait height-cap was bleeding into the two-column layout, squashing the keyboard. Fixed by making the two modes explicitly mutually exclusive in both CSS and JS (`isPortraitDrawerMode()` checks width *and* height before engaging the drawer). **If touching either media query again, re-check this interaction** — it's an easy regression to reintroduce.
- **Descender clipping** (g/y/p tails cut off on card text) — caused by sub-pixel baseline coordinates with no explicit `textBaseline`; fixed by rounding coordinates and setting `textBaseline` explicitly.
- **Double-space bug** — was actually a leftover duplicate code block from an earlier edit registering every click listener twice, not an engine quirk as first assumed. Removed the duplicate.

---

## Design decisions worth NOT re-litigating

- **Square (1080×1080) is the default card shape**, better for WhatsApp Status (less letterboxing). **Landscape (1440×1080) is opt-in**, better for long phrases (measured: same real phrase needs 88% scale in square vs 100%/no scaling in landscape). Only width changes between shapes — height stays 1080 in both, since all vertical layout tuning (text wrap budget, footer clearance) was built against that.
- **Storage/export philosophy**: the *record* (code string + per-syllable Latin) is canonical; the *image* (PNG/SVG) is always derived from it, never the other way round. Don't build a workflow that treats the image as a source of truth.
- **No Google Drive / cloud folder for hand-off.** Considered and rejected — the tool has no server, so it can't *fetch* a file from Drive automatically, only a human downloading-then-re-uploading, which isn't actually a one-action improvement. Instead: **Share card** (native share sheet, feature-detected, image only) and **Copy record** (minified single-line JSON, meant for pasting straight into a WhatsApp message, not for reading).
- **Copyright/attribution**: any future Wikipedia-sourced reference material (e.g. the consonant chart) should be a static, hand-copied snippet with attribution — never a live iframe/fetch, which would reintroduce a network dependency this tool deliberately doesn't have.
- **No headless browser available in this build environment** (Chrome download blocked by network policy). All fixes are verified via Node.js DOM/canvas stubs exercising the *actual* shipped functions, not just standalone logic tests. This has caught real bugs (e.g. functions accidentally deleted by earlier find-replace edits) but cannot catch genuine rendering/visual bugs — those need a real device check after upload.

---

## Testing approach (for whoever picks this up)

There's an informal regression suite as standalone `.js` files (not committed to the repo, live only in the working session) that stub `document`/`canvas`/`navigator` well enough to `vm.runInContext` the actual app script and call its real functions. Re-run after any change:
- Edge cases (empty phrase, bare vowels, syllabic nasals, long phrases, unknown marks)
- Both card shapes against a real stress phrase
- Syllable editor operations (vowel swap, mark toggle, delete)
- LCS carry-forward (deletion, insertion, duplicate syllables)
- Keyboard collapse default-state across viewport sizes

If continuing in a fresh chat, these test files won't exist yet — regenerate the relevant one before trusting a fix, don't skip verification just because it's a "small" change. Several bugs in this build were introduced by supposedly-small edits.

---

## How to hand off to yet another new chat

1. Update this doc yourself (or ask Claude to update it) before switching.
2. Download the current `index.html` from the chat's outputs.
3. Start the new chat **inside this Project** (keeps Songbook/Rest-of-Project docs and Project memory attached).
4. Upload both this doc and `index.html`, paste a one-line pointer to what you want done next.
