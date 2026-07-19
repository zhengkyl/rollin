# Rollin - a writing surface that ~~rules~~ rolls
<img width="450" height="350" alt="rollin_preview" src="https://github.com/user-attachments/assets/de27e943-bc32-45ad-bcc0-cdec66b96d36" />

## TODO
- double/triple click
- ime shows over invisible text area
- mobile drag select

Below is a knowledge dump for anyone (human or Claude).

---

## 1. The one idea that explains everything

**Rollin is a plain `<textarea>` with a hand-drawn visual projection floating on top of it.**

The textarea is the single source of truth. It holds the text, the caret position, the
selection range, undo history, IME composition, and the clipboard. It is made *visually*
invisible (`opacity: 0`) and *non-interactive to the pointer* (`pointer-events: none`), but
it stays **focused** so it keeps receiving keystrokes and owning the selection.

Everything you see — the curved lines, the caret, the selection highlight — is drawn by us in
a separate layer (`.col` full of `.band` divs) that *reads* state from the textarea and
renders a projection of it. We never store text ourselves.

Consequences that follow directly from this and recur throughout the code:

- We get typing, arrow keys, shortcuts, copy/cut/paste, undo/redo, and IME **for free**,
  as long as the textarea keeps focus and its selection mirrors what we draw.
- Anything the browser normally draws for a textarea (caret, selection highlight, spellcheck
  underlines, its own text) we must **suppress on the textarea and redraw ourselves**. This is
  the source of a whole family of bugs (see §9).
- Because we draw the caret/selection, we are responsible for the *visual-line* concept the
  textarea doesn't expose (wrapping, per-line geometry, caret affinity).
- Because the user never taps the real control, **every focus is programmatic** — which is why
  mobile keyboards are their own category of pain (§8).

---

## 2. The DOM, only where it's load-bearing

Read the file for the full tree. What matters:

- `.stage` — fixed full-viewport container. Owns the CSS `perspective` and `overflow:hidden`.
  All pointer/wheel handlers live here. `touch-action:none` so we drive touch scrolling ourselves.
- `.col` — the moving text column. `preserve-3d`, `pointer-events:none` on the container but
  **`.band` children are `pointer-events:auto`** so they're hit-testable.
- `.band` — one per **visual row** (§4), positioned entirely by JS transforms.
- `.input` — the textarea. `opacity:0; pointer-events:none;` but focused.
- `.wrapmirror` / `.xmirror` — two offscreen measuring twins (§4).

CSS variables that matter, **all written from JS** (`applyFont`) — don't tune them in the
stylesheet, the values there are only pre-JS fallbacks: `--step` (line pitch), `--font`,
`--anchor` (§3), and `--chrome` (= `1 - flatAmt`, the curved-only UI opacity, written per frame
in `draw`).

---

## 3. Coordinate & geometry model

A single scalar **`center`** — a *fractional* row index — means "the row sitting at the resting
line." `target` is where `center` is easing toward.

`place(el, i)` positions band `i`. Let `s = i - center` (signed distance from center in rows).
The curved layout lays the row on a cylinder of radius `R = STEP * size`; the flat layout is a
plain linear stack `y = s * STEP`. The two are blended by **`flatAmt`** (0 = curved, 1 = flat).
The formulas are short and live in `place()`; the things you can't read off them:

- **Both modes are center-anchored** (they share `center`). This is the key to the smooth
  unroll: the centered row (`s = 0`) is at the resting line in *both* layouts, so during the
  morph it stays put and the arc just straightens by pivoting around it. Near-center line
  spacing is ≈ `STEP` in both modes because `size*sin(1/size) ≈ 1`, so unrolling mostly affects
  far-from-center lines.
- **The number of visible lines is derived, not set.** `cosMax = cos(lines/size)` makes a line
  fully fade at `s = lines`. So "lines visible" and "curvature" together define the fade angle.
  `STEP` cancels out of that ratio, so font size does not change how many lines are visible.

### The anchor

The resting line is **not** screen center. `TUNE.anchor` (0.4 = 40% from the top) lifts it so a
mobile keyboard doesn't cover the line being written. Three things this touches:

1. It is applied **on the containers** (`.band`, `.seam`, `.ghost` are anchored by
   `top:var(--anchor)`), never inside `place()`. That way curved and flat shift by exactly the
   same amount and the center-anchored invariant above survives for free. Baking an offset into
   `place()`'s `y` means blending it through `flatAmt` too, and it will break the unroll.
2. **`perspective-origin` tracks it** (`50% var(--anchor)`). That's the viewer's vantage point;
   leave it at `50% 50%` while the content sits higher and you're viewing the drum from below —
   the cylinder renders visibly asymmetric, near rows skewed against far ones.
3. The viewport is now split **unevenly**, so `spaceAbove()`/`spaceBelow()` replace naive
   `innerHeight/2` math: `draw()` sizes its flat-mode band window off the *longer* side (or it
   under-renders off the bottom), and `layout()` bounds the unroll caret margin by the *shorter*
   side (or the caret escapes past the top before triggering a scroll). Both reduce to their old
   values at `anchor: 0.5`.

---

## 4. Visual rows and the two measuring mirrors

The text is split into **visual rows** — every soft-wrap *and* every hard newline starts a new
row. `rows` is an array of `{start, end}` character offsets (end exclusive, excluding the
trailing `\n`). This is what makes per-line curvature possible; a textarea can't bend line 3
differently from line 4.

We can't ask the browser where a textarea wraps, so we measure it ourselves with two hidden
twins styled *identically* to `.band`:

- **`wrapmirror`** (`white-space:pre-wrap`, same width as `.col`): `segment()` walks every
  character, reads a Range's rect, and starts a new row wherever the top jumps by more than
  `TUNE.wrapDetect * STEP`. O(n) rects per segmentation, cached and only re-run when text or
  column width changes.
- **`xmirror`** (`white-space:pre`, no wrap): `xWidth(i, k)` measures the pixel x of caret
  offset `k` within row `i`. `offsetAtX(i, x)` binary-searches it. These power arrow navigation
  (goal column) and selection-rectangle geometry.

**Critical invariant:** the mirrors' font metrics (`font-size`, `line-height`, `letter-spacing`,
width) must match `.band` *exactly*, or wrapping and x-positions will disagree with what's
drawn. This is why the caret has zero layout width (§9) and why iOS text inflation had to be
turned off globally (§9). When the drum won't roll on a line that visibly wrapped, this
invariant is where to look first.

`segment()` reads `getClientRects()` and takes the character's own box rather than
`getBoundingClientRect()` — see §9, it is not a stylistic choice.

---

## 5. The animation loop

One `requestAnimationFrame` loop (`tick`) eases `center → target` and `flatAmt → flatTarget`,
calls `draw()` each frame, and **stops itself** when everything is within epsilon. `kick()`
restarts it; anything that changes a target calls `kick()`.

**There are no CSS transitions on bands.** Motion is entirely JS-driven per frame. An earlier
version used CSS transitions and it caused a "new row animates in from nowhere" desync on wrap;
removing them and driving everything through `center`/`flatAmt` is what fixed it and unified
wrap-roll, scroll, and the unroll morph into one mechanism.

---

## 6. Caret affinity (the subtle one)

At a **soft-wrap boundary**, the end of row *r* and the start of row *r+1* are the *same*
character offset. The textarea only stores the offset, so "which visual line is the caret on?"
is genuinely ambiguous and must be tracked as extra state: **`endAffinity`**.

- `endAffinity = true` → show the caret on the line that *ends* at that offset.
- `false` → show it on the line that *starts* there (the default; `rowOf` prefers the later row).

The rule lives in exactly one place: **`placeCaret(pos, row, anchor)`** sets
`endAffinity = affinityFor(pos, row)`. Every caret placement that knows its intended row (Home,
End, Up/Down, click, tap, drag) goes through it. Native horizontal moves and typing reset
`endAffinity = false`.

Lesson learned the hard way: affinity is **caret state set by the placement's intended row**,
not something a single operation (like End) can assume. The bug sequence was: End forced
end-affinity to find its *source* row, which broke "End from a line start." The fix was to
resolve the source row using the *current* affinity and set the new affinity from where it
lands. If you touch caret navigation, preserve this: source row uses current `endAffinity`,
destination affinity comes from `affinityFor`.

This ambiguity is not ours alone — the browser has it too, and resolves it against us in
`segment()` (§9).

---

## 7. Scroll model & modes

`mode` is one of `'write'`, `'scroll'`, `'select'`.

Caret-follow lives in `layout()` and is **margin-based**, not "always recenter":

- Roll mode: `margin = 0` → the caret is pinned to the resting line (typewriter scrolling).
- Unroll mode: `margin ≈ visibleLines/2 - scrolloff` → the caret roams freely on screen; the
  view only scrolls when the caret would cross an edge, and then by the minimum to keep it just
  inside. This is why moving the caret in flat mode usually doesn't move the page.

Explicit scrolling (wheel, touch drag) moves `target`/`center` directly with `mode='scroll'`
and never touches the caret ("look back"). Because both modes share `center`, one code path
serves scroll, snap, and caret-follow for both.

Mouse-wheel vs trackpad: **browsers don't tell you.** We infer: `deltaMode !== 0` or
`|deltaY| >= TUNE.wheelMouse` ⇒ mouse notch → step exactly one line (magnitude ignored so
high-DPI scaling can't inflate it). Otherwise trackpad → glide continuously, then snap to the
nearest line after `TUNE.snapDelay` ms of stillness. Touch is unambiguous and snaps on release.
This is heuristic and can misjudge a free-spinning high-res mouse wheel; known and accepted.

---

## 8. Pointer input & focus

All pointer logic is on `.stage`; the textarea is `pointer-events:none`. Branching:

- **mouse** press → place caret (and set anchor); drag past `TUNE.dragMouse` px → select;
  release without drag → click (recenters in roll, not in unroll).
- **touch** press → potential scroll; drag past `TUNE.dragTouch` px → scroll; tap → place caret
  and focus (which opens the mobile keyboard).

Point→offset uses `caretPositionFromPoint` (fallback `caretRangeFromPoint`) against the bands,
returning `{pos, row}` so the caller can set affinity from the clicked row. A geometric fallback
(nearest visible band by y, then `offsetAtX`) covers clicks in gaps. Both read live
`getBoundingClientRect()`s, so hit-testing follows the anchor automatically — §3 needed no
changes here.

**Focus is the fragile part**, because the user never taps the real textarea. Three separate
guards, all load-bearing, all of which have bitten:

- **`preventDefault()` on mouse `pointerdown`**, and a blanket `preventDefault()` on `mousedown`
  over `.stage`. Without them the browser's default mousedown moves focus off the
  (`pointer-events:none`) textarea right after we focus it, and typing silently stops. The
  `mousedown` one also catches the *synthesized* mouse events a touch tap generates — iOS Firefox
  showed this as the keyboard flashing up and immediately dismissing.
- **`focusInput()` forces a real focus transition** (blur, then focus) instead of calling
  `focus()` directly. Dismissing the iOS keyboard leaves the textarea as `document.activeElement`,
  so a plain `focus()` is a no-op and the keyboard can *never* be raised again.
- **Pointer capture is mouse-only.** Capturing a touch pointer suppresses WebKit's synthesized
  click/focus flow. `.stage` is `position:fixed; inset:0`, so touch moves can't leave it and
  capture bought nothing there anyway.

Focus happens on `pointerup` (tap confirmed), deliberately not `pointerdown`. `pointerdown` is a
stronger user-activation context and is the fallback if a browser still refuses to raise the
keyboard — but it pops the keyboard on every touch that turns into a scroll, which ruins the
"look back" gesture. Don't move it without weighing that.

---

## 9. Bugs we hit and the lessons (read before "fixing" something that looks odd)

- **Invisible textarea leaks.** `color:transparent` hides the text but *not* the native
  selection highlight, caret, or spellcheck underlines. The robust fix is `opacity:0` on the
  whole element (renders nothing, still focusable), **not** per-property transparency. Don't
  reintroduce `color:transparent` thinking it's equivalent.
- **Zero-width caret.** The caret is a 0-width element painted with `box-shadow`, not a 2px box.
  A 2px caret adds layout width to the active line, so it wraps one character earlier than the
  `wrapmirror` predicts → the line visibly wraps but the drum doesn't roll. Keep it layout-free.
- **Held arrow keys.** Re-render must be driven by `selectionchange` (fires on every caret
  move, including auto-repeat), not `keyup` (fires only on release). Keying off `keyup` makes
  the view freeze while a key is held.
- **`WINDOW` coupling.** `WINDOW` must be ≥ the max `lines`-visible option or edge rows get
  clipped. It's derived (`Math.max(...OPTS.lines) + 1`); keep it derived.
- **High-DPI wheel deltas** are non-integer; don't gate mouse-wheel detection on
  `Number.isInteger(deltaY)`.
- **`localStorage` in a sandboxed preview** can throw; all access is wrapped in try/catch. Keep
  it that way — the file must run both as a standalone download and inside a preview.
- **Caret only when focused.** The drawn caret's visibility/blink hang off a `.focused` class
  the textarea toggles; otherwise it blinks forever even when focus is elsewhere.

### iOS / WebKit

Everything here reproduced only on iOS and cost real time to find. Desktop is not a sufficient
test for anything touching measurement or focus.

- **iOS text auto-sizing broke wrapping.** iOS inflates the rendered font-size of blocks it
  heuristically decides are body text, and it did so *unevenly* between the visible `.band` and
  the offscreen `.wrapmirror`. On-screen text then wrapped a character early, the mirror never
  saw the wrap, and the drum didn't roll — a direct violation of §4's invariant. Fixed globally
  with `text-size-adjust:100%`. Don't remove it.
- **Range rects are ambiguous at a line boundary.** A Range starting exactly at a line boundary
  has the same ambiguity as the caret (§6), and WebKit resolves it to the end of the *previous*
  line: `getClientRects()` returns a zero-width rect there *plus* the character's real rect, and
  `getBoundingClientRect()` **unions them**, reporting the previous line's top. In `segment()`
  that seeded `curTop` a full `STEP` too high after a hard newline, so the second character after
  Enter looked like a wrap and every Enter produced a one-character line. Fix: take the
  character's own box from `getClientRects()`, skipping rects with no width (a rect with no width
  is a caret *position*, not a glyph — which also covers whitespace swallowed at a wrap point).
  A single character can't span lines, so exactly one rect survives the filter.
- **The keyboard.** Three distinct failures, all covered in §8: focus stolen by synthesized mouse
  events, `focus()` no-op after keyboard dismissal, and touch pointer capture. iOS Chrome and
  Firefox are stricter than Safari here despite all being WebKit — they're WKWebView embedders.
  Test all three.

---

## 10. Settings & TUNE

- **`TUNE`** (top of the script) holds all *feel* constants — easing, scroll sensitivity, snap
  delay, thresholds, opacity falloff, wrap detection, font ratio, scrolloff, and `anchor` (§3).
  Tune feel here, in one place. `anchor` is the single source of truth for the resting line; the
  CSS `--anchor` is only a pre-JS fallback.
- **`OPTS`** are the user-facing curated slider values (`size`, `lines`, `blur`, `font`), each a
  short list; sliders are index-based over these lists. `DEFAULTS` picks one from each (font
  snaps to the nearest option to a screen-derived size). `flat` (roll/unroll) is persisted too.
- Settings apply **live** (on slider `input`), persist immediately, and hand focus back to the
  textarea on `change` so you can type-test with the panel open.
- Adding a knob: add to `OPTS`/`fmt`/`KEYS`, add a slider in the panel HTML, handle it in
  `applyOne`. Geometry knobs call `draw()`; ones that change wrapping (font) must invalidate
  `cacheVal` and call `layout()`.

---

## 11. Invariants — don't break these

1. The textarea stays focused whenever the user is editing; its selection always mirrors the
   drawn caret/selection.
2. Mirror font metrics == band font metrics (kept in `applyFont`, and don't let a platform
   inflate one and not the other).
3. The caret occupies zero layout width.
4. Both modes stay center-anchored on the shared `center` (or the smooth unroll breaks). The
   anchor offset is applied on the containers, never per-band in `place()`.
5. `perspective-origin` tracks `--anchor`.
6. Caret affinity is set by `placeCaret` from the intended row; native moves clear it.
7. No CSS transitions on bands — motion goes through the rAF loop.
8. `WINDOW` stays derived from `OPTS.lines`.
9. All `localStorage` stays inside try/catch.

---

## 12. Known limitations / intentionally not done

- **Touch text selection** (selection handles) isn't implemented; touch drag scrolls. Mouse
  drag and keyboard selection work.
- **Very large documents**: flat mode renders enough bands to fill the viewport, and selection
  re-measures per row on drag. Fine at prototype scale; would need memoization / virtualization
  to push hard.
- **Spellcheck** is off (its underlines can't be redrawn on the bands cheaply).
- **Mid-text typing that causes a downstream wrap** places the caret with start-affinity (a rare
  cosmetic edge); typing at the end (the common case) is always correct.
- **The anchor is static.** A `visualViewport`-driven lift would track the keyboard exactly
  instead of guessing 40%; deliberately not done, since a fixed anchor keeps the resting line in
  one place whether or not the keyboard is up.
- **Top-anchored flat editing** was deliberately dropped in favor of center-anchored, because a
  stable unroll transition + top-anchored editing is a genuinely harder combination (you'd have
  to animate the anchor point).

---

## 13. Working on it

- Manual test matrix worth running after any input/geometry change: type until it wraps; press
  Enter and type two characters; hold Up/Down across wrapped lines; Home/End on a soft-wrapped
  line and from a line start; click the tail of a wrapped line; drag-select across the edge
  (auto-scroll); mouse-wheel one notch = one line; trackpad glide + snap; toggle roll/unroll
  mid-edit and while scrolled away; every settings slider live; reset.
- **On iOS specifically** (§9): type past a wrap and confirm the drum rolls; press Enter and
  confirm the next line doesn't break after one character; dismiss the keyboard and tap to
  reopen it; confirm the resting line clears the keyboard. Do it in Safari, Chrome *and* Firefox.

