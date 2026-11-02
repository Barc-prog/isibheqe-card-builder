# isiBheqe Card Builder — Status Doc

*Paste this at the start of a new chat, along with the current `index.html`, to resume work without re-diagnosing from scratch. Update it yourself as things change — this is a living doc, not a transcript.*

**Last updated:** 2 Nov 2026

---

## What this is

A single self-contained HTML tool (no build step, no server, works offline from disk) for building isiBheqe syllabeme cards with Latin transliteration underneath each glyph. Built for the songbook project's scribes, and since November also a source of clean glyph images for the isibheqe-chart page and later a book.

- **Live at:** barc-prog.github.io/isibheqe-card-builder/
- **Deployed via:** GitHub Desktop → push to `main` → GitHub Pages, root deploy, file is `index.html`. From November, changes also arrive as a branch + pull request that you merge on your phone, the same way the chart repo works.
- **Upload note:** GitHub web upload only *replaces* a file if the name matches exactly (`index.html`) — check the repo file list after uploading from mobile, since some mobile browsers rename downloads and silently create a second file instead of overwriting.
- **Caching note:** GitHub Pages serves with a **10-minute cache header**. If you upload a fix and reload the same tab within 10 min, you may see the old version with no indication it's stale. Open in an incognito/private tab to bypass this and confirm.
- **This doc lives in two places**: the repo (`card-builder-status.md`, next to `index.html`) and the Claude Project. They should be the same text. On 2 Nov the Project held two identical copies dated 25 Aug and the repo held an older 24 Aug copy; both were replaced by this one.

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
- **The layer mode is not part of the record.** It is a presentation choice made at the "show" selector; the same record makes all five images.

---

## Known bugs

### Confirmed, diagnosed, NOT YET FIXED

*(none open at present)*

### Reported, NOT reproduced -- needs a screenshot or exact phrase

**"Glyphs show up if I put two spaces between words."** Traced the full pipeline with an actual double-space phrase (`["a",["p"]]," "," ",["e",["t"]]]`) at every layer that touches it:
- Engine `sentence` array (`Context.addLetter("space")` called twice) -- two `space` sentinel objects pushed, no phantom glyph object.
- Canvas card layout (`computeLayout`/`cardOps`) -- exactly 2 glyph ops, gap width doubled correctly.
- SVG export (`buildCardSvg`) -- same `ops` list, same result.
- Raw-string reload's Latin carry-forward (`alignTokens`, the LCS aligner) -- stress-tested with a doubled space between two *identical* syllables (the case most likely to expose the ambiguity of `JSON.stringify(" ")` being the same key for every space) and an adversarial restructuring. Latin only ever attached to a real syllable token, never to a space, never to the wrong syllable.

Could not reproduce at any layer. The likely actual cause: the editor's space button was toggle-only until this session (see Fixed, below), so there was no way to *get* two consecutive spaces except hand-editing the raw code string -- whatever the person did as a workaround for that may be what they were actually seeing, not a rendering bug from the spaces themselves. Worth confirming the symptom is gone now that `+ space` exists, before spending more time hunting for it. If it persists, get the exact phrase or a screenshot from the live site -- guessing at test cases has a search space too large to be productive without one.

### Needs a real-device check after the layer-modes merge

- Mode 1 on a phone: the preview canvas changes shape to the cut-out, on a checkered backdrop. Confirm the preview pane still sits sensibly in portrait and landscape.
- **Share** and **Copy card** in mode 1, pasted into WhatsApp: should arrive on the cream/white card colour, not black.
- A mode-1 PNG and SVG dropped onto a coloured page: both should be transparent.

### Logged, not yet actioned

*(none open at present)*

## Fixed this project, worth knowing about (in case of regression)

- **Layer modes** (2 Nov) — a **show** selector at the start of the bottom bar, five modes, each dropping one layer from the bottom up:
  1. *amaBheqe only (no background)* — glyphs only, transparent, cut out tight.
  2. *amaBheqe + Latin syllables* — the Latin under each syllable.
  3. *+ full Latin* — adds the Full Latin line.
  4. *+ gloss* — adds the gloss.
  5. *full card (all details)* — the card as it always was, footer included. Still the default.
  - **Toggled on the page, not at export.** PNG, Copy and Share all read the live preview canvas, so the preview *is* the export. A switch that only applied at download would have shown one thing and saved another. `cardMode()` reads the selector, `layers(m)` turns it into four flags (`sylLatin`, `fullLatin`, `gloss`, `footer`), and `computeLayout`/`cardOps` skip what is switched off. Everything still comes from the one `cardOps` list, so the canvas and SVG cannot drift apart.
  - **Glyph size — grow to fill (modes 1–4).** Before this, the layout tried full size (`s = 1.0`) and stepped down until it fitted; it never grew. Modes 1–4 now start at the largest size one glyph could have on the card and step down 2% at a time until every row fits across and the stack fits down. A two-syllable word fills the card, and the cap note says "scaled up to 340% to fill". It has to be a search rather than a formula, because a bigger glyph can push a word onto a new row. The Latin under each syllable is sized as a share of its glyph (`30*s` px), so it grows with it on purpose.
  - **Freed space is real.** With no footer, the bottom margin becomes the same as the top (64px, `BOT` in `computeLayout`) instead of reserving the 156px footer band. With no text block, the 40px gap above it goes too. With nothing below the glyphs, the last row's padding isn't counted, so the block sits centred, not high. When a text block follows, that padding stays as the space between them. This was found from the first renders, where the full Latin line crowded the last row of syllables.
  - **Words wrap whole in modes 1–4.** The old wrap went syllable by syllable. That never showed at full size, but grow-to-fill wraps constantly, and the first renders split *sawubona* as *sa wu bo / na*. `wrapWords` keeps a word together. The size search runs in two passes: first the largest size at which no word breaks, and only if no size manages that (one very long word) a second pass that allows a break. An intact word slightly smaller reads better than a split one slightly bigger. Rows are also centred without the one trailing gap in modes 1–4.
  - **Mode 5 is left exactly as it was.** Same `s ≤ 1.0` loop, same syllable-level wrap, same centring. Checked against the 30 Aug live version in a real browser: every layout op identical. The SVG differs only by an `x="0" y="0"` on the background rect.
  - **Mode 1 is a cut-out of the card, not a separate layout.** It lays the phrase out exactly as on the card in the chosen shape (same rows), then takes the glyphs' own box plus half a glyph gap (`cardOps` returns it as the frame `F`). The canvas, the overlay and the SVG `viewBox` all take that frame, so the file is the writing and nothing else. No background `rect`, no `text`, no rule. Exported glyphs need **no pass through `svg-cropper-v2.html`**, and the loose-viewBox cause of "placeholder triangle shows through" doesn't arise for anything made here.
  - **Glyphs are now drawn as vectors on the card canvas** (`drawGlyphVector`) instead of being copied from the 5-unit syllable tiles. A tile stretched to 9× goes soft. The vector path uses the same engine calls, transform and stroke width as the SVG export, so the canvas and SVG still agree by construction. The strip and editor still use the tiles, which is fine at their size.
  - **WhatsApp flattening.** WhatsApp shows transparent pixels as black, and the ink is black. So in mode 1, **Share** and **Copy card** lay the image onto the card colour first (`shareBlob`). Downloads stay transparent, since they are for pages and print. This is the one place the export deliberately differs from the preview.
  - **File names say the mode**: `-glyphs`, `-syllables`, `-latin`, `-gloss`, nothing for mode 5. A glyphs-only file never overwrites the card.
  - **Selected syllable (SVG)**, under More: exports the syllable selected in the strip, transparent, cropped to its own ink, and named from its Latin (`na.svg`, falling back to `syllable-N.svg`). With nothing selected it refuses with a status message. It is **always at the card's full-size scale** (`K = BASE_GH/bodyHeight()`), not stretched to fill, so a plain *a* and a nasalised *a* exported separately have the same vowel size and only the marks add height. That consistency matters for chart cells placed side by side. In mode 1 it is the glyph alone; in modes 2–5 it adds that syllable's own Latin underneath. The Full Latin line, the gloss and the footer describe the whole phrase, so they never go on a single syllable.
  - **Supersedes the 31 Oct glyphs-only export.** A *Glyphs only (SVG)* / *Selected syllable only (SVG)* pair was built in the "Repository sharing with Pule" chat on 31 Oct, but it was never committed and its status entry never reached this doc. Mode 1 plus the selected-syllable button cover both jobs.
  - Verified in headless Chromium against the shipped page: all 5 modes × both shapes through PNG and SVG, looked at as images. Also: empty phrase, a single syllable, a line break, a 60-syllable phrase, one 30-syllable word, editor insert/space/delete while in mode 1, real downloads with file names, a transparent corner pixel on the mode-1 PNG, an opaque cream corner on the flattened Share image, the selection highlight aligned to the cut-out, and a 390×780 phone viewport. No page errors.
  - **Known limit:** the shapes are the engine's, i.e. the isibheqe.org.za demo shapes Pule flagged in the 9 Oct review. Because the record stays canonical, images made now can be regenerated when the new keyboard/font shapes arrive.

- **Ink cropping / loose card layout** (25 Aug) — the "too loose" spacing was never a CSS rule. It was the render frame. Measured analytically by running the drawing calls through a bounds-collecting target instead of a canvas: across all 9930 vowel × consonant × consonant combinations the keyboard allows, the furthest ink ever reaches is 36.7 × 33.7 units, and a plain vowel uses about 16 × 16 of a 40 × 46 frame. Every consumer was treating the whole frame as the glyph, so a vowel in the card's 150px slot was landing ~37px of ink and ~113px of air — glyphs had been rendering at roughly a third of their intended size since the frame was enlarged.
  - **Fix**: `inkBounds` measures a token from its drawing calls (not canvas pixels, so the canvas card and the SVG export measure identical geometry by construction). `phraseCrop` unions those boxes across the tokens present in the phrase and caches on the token set. **Union, not per-glyph** — cropping each syllable to its own ink puts every one on its own baseline and the row ripples.
  - **`bodyHeight()` is the fixed reference**, not the crop height. Scaling by the crop meant one nasal shrank every other syllable on the card to make room — the writing equivalent of dropping the x-height because one word carries an accent. Now a single scale factor `K = BASE_GH/bodyHeight()` maps glyph space onto the card, so a vowel body draws 92px in *every* phrase and a tall mark extends the row instead of compressing what's under it.
  - **`BASE_GW` is gone.** Slot width is the crop's own aspect, so a narrow phrase packs narrow. Constants are now `BASE_GH = 92` (plain vowel body height), `BASE_LAT = 40`, `BASE_GAP = 24`, `BASE_WGAP = 34`, `BASE_RPAD = 26`, and they mean literal ink measurements rather than frame measurements.
  - Measured result on a 1080 card: plain phrase 93px glyphs, 159px rows, 7 per row; with a raised nasal + tone, 128px glyphs, 194px rows, still 7 per row. Same density as before the frame was enlarged, with the ink about three times bigger.
  - **Four places now move together**, not three: `renderSyllable`, `croppedCanvas` (strip tiles + editor preview), `drawCardNow` (since 2 Nov via `drawGlyphVector`, a vector draw through the same crop), `buildCardSvg` (scales by crop height, translates back by crop origin). All go through `phraseCrop()`. **If card and strip ever disagree again, check they all still call it.**

- **Nasal/consonant clipping** — old glyph render frame (88×130 px at unit 5) clipped 320 of 983 possible vowel+consonant+modifier combinations, including plain `f` and `s` with no marks — not nasal-specific, a systemic frame-sizing bug. Fixed by sizing the frame from an exhaustive sweep (verified 0/983 overflow). **The frame constants (`GLYPH_FRAME_W_U`/`GLYPH_FRAME_H_U`/`GLYPH_OFFSET_X_U`/`GLYPH_OFFSET_Y_U`) still exist and still matter, but they are now only a safe canvas to draw into** — nothing is displayed at frame size any more, so the generous padding costs nothing. The three-way sync problem this entry used to warn about is superseded by the crop pass above.

- **Glyph-slot overlap** — the frame enlargement made drawn glyphs ~21% wider than their card slot, fixed at the time with an inscribe clamp in `cardOps` (`if(dw > gw){ dw = gw; dh = dw/AR; }`). **That clamp has been removed.** With the crop pass the slot *is* the ink box scaled up, so the glyph fills it exactly on both axes and there is nothing to inscribe. Don't reintroduce the clamp; if glyphs overlap again the cause is in `phraseCrop`, not here.

- **Colour view syllable miscount** (25 Aug) — `splitCodeUnits` stripped all whitespace (`replace(/\s+/g,"")`) before testing a unit against `'" "'`, destroying the one character the test was looking for, so no space ever matched and each was counted as a real glyph. Changed to `.trim()` — `splitCodeUnits` already slices quote-to-quote, so the ends were never dirty. Verified on `["a",["b"]," ","e"," ",["i",["n","~"]],"\n","o"]`: 5 glyphs, 3 breaks (the old code reported 8 glyphs, which is exactly the reported "12 for a 10-syllable phrase" symptom and the off-by-one on click).

- **Space insertion from the syllable editor** (25 Aug, revised 25 Aug) — was the raw-string-only workaround logged earlier; now a first-class editor action.
  - First version was a single toggle button ("Space after" / "Remove space after") gated on whether exactly one space already followed the syllable, re-keying the Latin map around the insertion or removal the same way `deleteSyllable` does. Re-verified against the real `Context.deserialize`/`serialize` round trip and checked out -- but this was a genuine limitation, not just an untested edge: clicking it a second time removed the space just added instead of adding another, so there was no way to build a wider gap from the editor at all. (First verification pass used an invalid consonant key and produced a false alarm -- worth remembering CONSONANTS is a fixed list, not "any Latin letter," when writing test tokens. **It happened again on 2 Nov**: `b` is not a key, so a test `ba` lost its consonant on the first rebuild. Both the live and new versions did the same, which is how it was ruled out.)
  - **Revised**: two buttons, `+ space` (always inserts one more) and `− space` (removes one, shown only when the run is non-empty), with a count between them once the run exceeds one. `spaceRunAfter(ti)` counts consecutive `" "` tokens after a given position; `insertSpaceAfter`/`removeSpaceAfter` both re-key the Latin map the same way the original did. `rebuildFromTokens` factors out the shared re-deserialize/redraw step both now call.
  - Verified against the real engine: `+ space` clicked three times then `− space` once on a two-syllable phrase, checking token array, card glyph count (held at 2 throughout), row gap width (widened/narrowed by exactly one unit per click), and Latin re-keying at every step. Last-syllable guard still fires.
  - Also stress-tested the raw-string-reload Latin aligner (`alignTokens`) against double spaces, since `JSON.stringify(" ")` is the one non-unique key it has to match on -- including a doubled space between two identical syllables. No misattribution found; see Known bugs for the unreproduced report this was investigating.

- **Entry Details fixes** (25 Aug):
  - **Language field** is now a text input backed by a `<datalist>` of the 9 official indigenous languages in endonymic form (isiNdebele, isiXhosa, isiZulu, Sepedi, Sesotho, Setswana, siSwati, Tshivenda, Xitsonga) -- offered as suggestions, not enforced, since the project already uses languages outside this list (siPhuthi, via Iris's work) and a locked dropdown would have blocked that.
  - **"seed" removed as the Theme default**, in all three places it was hiding: the HTML `value="seed"` attribute, the card footer's fallback text, and `buildRecord()`'s saved-JSON fallback (now `null`, matching how gloss/scribe/language already behave when blank). The word "seed" survives only as example text inside the placeholder, which is never submitted as data.
  - **Clear now resets the five Entry Details fields** (`f-lang`, `f-gloss`, `f-round`, `f-scribe`, `f-id`), not just the glyphs. This is the actual fix for the reported "Latin bleeds into Entry ID" -- traced every place any of these fields gets written and found nothing programmatically copies Latin into Entry ID; the real mechanism is that Entry ID is naturally filled by hand from the previous card's Latin-derived filename hint, and Clear silently left it (and the other four fields) untouched, so it carried into the next card looking like a leak.
  - **Manual-Latin note added**: a persistent line above the syllable strip ("type the Latin spelling yourself, it isn't guessed for you") plus the same point folded into the empty-state message. Hidden in the short-landscape phone layout to protect the vertical-space budget from the mobile-keyboard fix -- the empty-state version still covers it there.

- **Insert a syllable between two others** (25 Aug) -- editor gains `+ syllable before` / `+ syllable after`, both calling `insertSyllableAt(glyphNo, before)`. Same shape as delete and the space controls: splice `state.tokens` at a position, re-run the whole array through `Context.deserialize`/`serialize`, done. New syllable starts as a bare vowel (`"a"`) and the editor opens on it immediately for picking the real vowel/marks.
  - Composes with the space controls rather than special-casing "insert between two words" -- insert always lands directly adjacent to the selected syllable; add a space after with the existing `+ space` control if it should be its own word instead.
  - Latin re-keying uses a direct index formula (`i < at ? i : i+1`), not the incremental walk the space functions use -- that pattern doesn't split cleanly into "before" and "after" since the two cases open the gap at different points relative to the loop.
  - Verified against the real engine using the person's own reported phrase (a run of 7 syllables/marks with double-space word breaks): inserted after glyph 3 -- landed exactly between the run's last syllable and the following double space, glyph count 7→8, every Latin value re-keyed correctly. Also checked inserting before glyph 1, the one position with no "previous syllable" to select.

- **Latin carry-forward on code edits** — originally compared old/new tokens by array index, so inserting or deleting even one syllable while hand-editing shifted every position after it and silently wiped their Latin. Replaced with a proper LCS (longest common subsequence) alignment in `loadFromText`, which matches syllables by content and relative order rather than raw position — correctly handles duplicate syllables too (tested: three identical `["a",["k"]]` tokens with an insert between them each kept their own distinct Latin).
- **Stale syllable editor on file load** — `loadFromText` never called `closeSyllableEditor()`, so if the editor was open when a different file loaded, it kept a closure reference to the *old* token index — typing into it could silently corrupt the *new* phrase at the wrong position. Fixed: `loadFromText` now closes the editor first.
- **Portrait mobile layout** — was previously a forced "rotate your phone" gate; replaced with a genuine single-column portrait layout and a collapsible keyboard drawer (collapsed by default only on narrow screens, checked via `matchMedia`).
- **Portrait/landscape media query collision** — the new portrait rule (`max-width:900px`, caps keyboard at `46vh` for stacking) and an older, separate "compact landscape" rule (`max-height:560px`, two-column layout, built in an earlier session) both match on a real phone in landscape (e.g. 844×390 satisfies both thresholds simultaneously). The portrait height-cap was bleeding into the two-column layout, squashing the keyboard. Fixed by making the two modes explicitly mutually exclusive in both CSS and JS (`isPortraitDrawerMode()` checks width *and* height before engaging the drawer). **If touching either media query again, re-check this interaction** — it's an easy regression to reintroduce.
- **Descender clipping** (g/y/p tails cut off on card text) — caused by sub-pixel baseline coordinates with no explicit `textBaseline`; fixed by rounding coordinates and setting `textBaseline` explicitly.
- **Double-space bug** — was actually a leftover duplicate code block from an earlier edit registering every click listener twice, not an engine quirk as first assumed. Removed the duplicate.

---

## Design decisions worth NOT re-litigating

- **Square (1080×1080) is the default card shape**, better for WhatsApp Status (less letterboxing). **Landscape (1440×1080) is opt-in**, better for long phrases (measured: same real phrase needs 88% scale in square vs 100%/no scaling in landscape). Only width changes between shapes — height stays 1080 in both, since all vertical layout tuning (text wrap budget, footer clearance) was built against that. The card shape also decides how a mode-1 cut-out wraps, since mode 1 is cut from the card.
- **Layer modes (2 Nov):** toggled on the page so the preview is the export; mode 5 never grows above full size and keeps its old wrap; modes 1–4 grow to fill and keep words whole; mode 1 is tight and transparent, 2–5 keep the card shape; the selected-syllable export is always transparent and always at full-size scale; only Copy and Share flatten mode 1, for WhatsApp. All chosen by Bandile on 2 Nov.
- **Storage/export philosophy**: the *record* (code string + per-syllable Latin) is canonical; the *image* (PNG/SVG) is always derived from it, never the other way round. Don't build a workflow that treats the image as a source of truth.
- **No Google Drive / cloud folder for hand-off.** Considered and rejected — the tool has no server, so it can't *fetch* a file from Drive automatically, only a human downloading-then-re-uploading, which isn't actually a one-action improvement. Instead: **Share card** (native share sheet, feature-detected, image only) and **Copy record** (minified single-line JSON, meant for pasting straight into a WhatsApp message, not for reading).
- **Copyright/attribution**: any future Wikipedia-sourced reference material (e.g. the consonant chart) should be a static, hand-copied snippet with attribution — never a live iframe/fetch, which would reintroduce a network dependency this tool deliberately doesn't have.
- **Mobile keyboard broken in both orientations** (25 Aug) — the two phone media queries (`max-width:900px` for portrait, `max-height:560px` for landscape) were gated on size only, and a real phone in landscape (e.g. 844x390) satisfies both at once. The portrait rule's 46vh cap and drawer-collapse were landing on the same `#left-pane` as the landscape rule's fixed two-column grid, squashing the keyboard into a sliver regardless of how much width the sidebar had. The JS collapse decision had the same problem: it checked width alone, so a landscape phone loaded with the keyboard hidden behind a toggle button the landscape CSS never displays.
  - **Fix**: both queries gained an `orientation` condition (`and (orientation:portrait)` / `and (orientation:landscape)`), which is what actually makes them mutually exclusive — a phone is never both at once, size thresholds alone don't guarantee that. Each block is now self-contained rather than assuming the other left something compatible: the landscape block sets its own `#workspace` grid and explicitly unsets `#left-pane`'s max-height instead of relying on the portrait block to not be present.
  - JS: the initial-collapse check now uses the same `(max-width:900px) and (orientation:portrait)` query via `matchMedia`, and listens for that query's `change` event to uncollapse if the device rotates out of portrait-phone mode — previously a drawer left closed in portrait stayed marked closed after rotating to landscape (harmless under the CSS fix, since the collapsed rule is scoped inside the portrait query now, but left inconsistent internal state).
  - **A second, older bug in the same area, found from screenshots**: `#kb-toggle` lives inside `#left-pane`'s own `.pane-head`, and the collapse rule was `#left-pane.kb-collapsed{max-height:0;overflow:hidden}` — so collapsing the drawer hid the only control that reopens it. On a narrow screen the keyboard simply wasn't there, with nothing on screen naming it or offering it back. Now only `.kb`/`.kb-lbl` are hidden; the pane head, its label and its Show button stay. The hint text also swaps to "tap Show for the keys" instead of being display:none'd while collapsed.
  - The `change` listener now calls `setKbCollapsed(e.matches)` — following the query in **both** directions. Setting it only on exit meant re-entering portrait (rotating back, or un-expanding a preview pane) left the drawer open at full 46vh with the strip crushed underneath.
  - Portrait also gets `#toolbar{flex-wrap:wrap}` and an auto-height toolbar row, since at this width the toolbar was scrolling sideways and hiding Load/Open behind a scrollbar.
  - **The artifact preview pane is a portrait-phone viewport, not a desktop one** (roughly 760 wide and taller than wide, so `max-width:900px and (orientation:portrait)` matches). Useful as a free mobile-portrait test surface; misleading if read as "this is what desktop looks like." Expanding the pane to full screen is the desktop check.

  - **This is very close to a fix logged in an earlier status doc** ("Portrait/landscape media query collision," `isPortraitDrawerMode()`) that doesn't exist in the file as received this session — either it didn't survive an edit, or this file predates it. Worth checking the repo's deployed version against this one before assuming the bug is fresh.

- **Analytic bounds measurement beats pixel scanning.** Glyph geometry can be measured without a canvas at all, by passing the engine a target object that records coordinates instead of drawing them (`boundsTarget`). This is how the 9930-combination sweep runs in Node in under a second, and it's why the canvas and SVG paths are guaranteed to agree — they're measuring the same numbers, not two independent rasterisations. Reach for this before reaching for `getImageData`.

- **A real headless browser is now available** (from Nov, Chromium + Playwright come pre-installed in the build environment). Until August it wasn't (the Chrome download was blocked), so fixes were verified only through Node.js DOM/canvas stubs exercising the shipped functions. Those caught real bugs (e.g. functions accidentally deleted by earlier find-replace edits) but not visual ones. Now: load the actual `index.html`, render, save the PNG/SVG, and *look at it*. The layer-modes word-splitting and text-crowding problems were caught that way, not by any number. A real phone is still the final check for touch and WhatsApp behaviour.

---

## Testing approach (for whoever picks this up)

There's an informal regression suite as standalone `.js` files (not committed to the repo, live only in the working session). The older ones stub `document`/`canvas`/`navigator` well enough to `vm.runInContext` the actual app script and call its real functions; the newer ones drive the real page in headless Chromium. Re-run after any change:
- Edge cases (empty phrase, bare vowels, syllabic nasals, long phrases, unknown marks)
- Both card shapes against a real stress phrase
- Syllable editor operations (vowel swap, mark toggle, delete)
- LCS carry-forward (deletion, insertion, duplicate syllables)
- Keyboard collapse default-state across viewport sizes
- **Crop geometry** — `phraseCrop` on an all-vowel phrase, a phrase with a raised nasal, and an empty phrase; confirm `bodyHeight()` stays 89 and that glyph *width* is constant across phrases while *height* varies with the marks present
- **Layer modes** — all 5 modes × both shapes, saved as PNG and SVG and looked at side by side; mode 5 layout ops compared against `main` (must be identical); no word split unless one word is wider than a row; mode-1 PNG corner pixel transparent, flattened Share corner pixel opaque card colour; selected-syllable SVG with and without a selection
- Use real consonant keys in test phrases (`s`, `w`, `n`, `t`, `k`, `p`...) — `b` is not one, and an invalid key silently disappears on the first rebuild

If continuing in a fresh chat, these test files won't exist yet — regenerate the relevant one before trusting a fix, don't skip verification just because it's a "small" change. Several bugs in this build were introduced by supposedly-small edits.

---

## How to hand off to yet another new chat

1. Update this doc yourself (or ask Claude to update it) before switching.
2. Download the current `index.html` from the chat's outputs, or just point the new chat at the repo, which Claude can now clone and push a branch to.
3. Start the new chat **inside this Project** (keeps Songbook/Rest-of-Project docs and Project memory attached).
4. Upload both this doc and `index.html` (or name the repo), paste a one-line pointer to what you want done next.
