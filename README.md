# Rollin — Developer Notes

A knowledge dump for anyone (human or Claude) picking up `platen.html`. Read this before
changing behavior; several things look wrong until you understand *why* they're that way.

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

If you internalize only one thing, internalize this.

---

## 2. Layout of the DOM

- `.stage` — fixed full-viewport container. Owns the CSS `perspective` (fixed at 1400px) and
  `overflow:hidden`. All pointer/wheel handlers live here. `touch-action:none` so we drive
  touch scrolling ourselves.
- `.stage::before` — the top/bottom paper fade gradient. Its opacity is `var(--chrome)`
  (= `1 - flatAmt`) so it dissolves in flat mode.
- `.seam` — the faint center line marking where the current line rests. Also `var(--chrome)`.
- `.ghost` — the "begin typing" placeholder, centered, fades out once there's text.
- `.col` — the moving text column. `transform-style: preserve-3d`, `pointer-events:none` on the
  container but **`.band` children are `pointer-events:auto`** so they're hit-testable.
- `.band` — one per **visual row** (see §4), positioned entirely by JS transforms. Contains an
  optional `.hl` (selection rectangle) and a `.ink` span (the text + caret).
- `.input` — the textarea. `opacity:0; pointer-events:none;` but focused.
- `.wrapmirror` / `.xmirror` — two offscreen measuring twins (see §4).
- `.topbar` — holds the `roll`/`unroll` toggle and the `settings` gear.
- `.panel` — the settings sliders. `.count` — word count.

CSS variables that matter: `--step` (line pitch in px), `--font` (font size), `--chrome`
(curved-only UI opacity). `--step` and `--font` are set from JS in `applyFont()`.

---

## 3. Coordinate & geometry model

There is a single scalar **`center`** — a *fractional* row index — that means "the row sitting
at the vertical center of the screen." `target` is where `center` is easing toward.

`place(el, i)` positions band `i`. Let `s = i - center` (signed distance from center in rows):

- **Curved** contribution: the row lies on a cylinder of radius `R = STEP * size`.
  - angle `a = s * (STEP / R) = s / size`
  - `yC = R*sin(a)`, `zC = R*cos(a) - R` (recedes), `rxC = -a` (surface tilt)
  - opacity fades via `t = (cos(a) - cosMax)/(1 - cosMax)`, then `pow(t, opacityExp)`; blur rises
    as `(1 - t)`.
- **Flat** contribution: a plain linear stack — `yF = s * STEP`, no z, no rotation, opacity 1,
  no blur.
- The two are blended by **`flatAmt`** (0 = curved, 1 = flat): `y = yC + (yF - yC)*flatAmt`, etc.

**Both modes are center-anchored** (they share `center`). This is the key to the smooth
unroll: the centered row (`s = 0`) is at screen center in *both* layouts (`yC(0)=yF(0)=0`), so
during the morph it stays put and the arc just straightens by pivoting around it. Near-center
line spacing is ≈ `STEP` in both modes because `size*sin(1/size) ≈ 1`, so unrolling mostly
affects far-from-center lines.

Fade geometry insight: the number of visible lines is **derived**, not set directly.
`cosMax = cos(lines/size)` makes a line fully fade at `s = lines`. So "lines visible" and
"curvature (size)" together define the fade angle. `STEP` cancels out of that ratio, so font
size does not change how many lines are visible.

---

## 4. Visual rows and the two measuring mirrors

The text is split into **visual rows** — every soft-wrap *and* every hard newline starts a new
row. `rows` is an array of `{start, end}` character offsets (end exclusive, excluding the
trailing `\n`). This is what makes per-line curvature possible; a textarea can't bend line 3
differently from line 4.

We can't ask the browser where a textarea wraps, so we measure it ourselves with two hidden
twins styled *identically* to `.band`:

- **`wrapmirror`** (`white-space:pre-wrap`, same width as `.col`): `segment()` walks every
  character, reads `getBoundingClientRect().top` via a Range, and starts a new row wherever the
  top jumps by more than `TUNE.wrapDetect * STEP`. O(n) rects per segmentation, cached and only
  re-run when text or column width changes.
- **`xmirror`** (`white-space:pre`, no wrap): `xWidth(i, k)` measures the pixel x of caret
  offset `k` within row `i` by measuring the width of that row's first `k` characters.
  `offsetAtX(i, x)` binary-searches it. These power arrow navigation (goal column) and
  selection-rectangle geometry.

**Critical invariant:** the mirrors' font metrics (`font-size`, `line-height`, `letter-spacing`,
width) must match `.band` *exactly*, or wrapping and x-positions will disagree with what's
drawn. They're kept in sync in `applyFont()`. This is why the caret has zero layout width
(see §9) — otherwise the rendered active line wraps at a different point than the mirror
predicts.

---

## 5. The animation loop

One `requestAnimationFrame` loop (`tick`) eases `center → target` and `flatAmt → flatTarget`,
calls `draw()` each frame, and **stops itself** when everything is within epsilon (`running`
flag). `kick()` restarts it. Anything that changes a target calls `kick()`.

`draw()` renders a window of bands around `round(center)` (width `WINDOW`, widened to fill the
viewport in flat mode), creating/removing band elements as the window moves, and writes
`--chrome`. `place()` runs per band per frame.

**There are no CSS transitions on bands.** Motion is entirely JS-driven per frame. (An earlier
version used CSS transitions and it caused a "new row animates in from nowhere" desync on wrap;
removing them and driving everything through `center`/`flatAmt` is what fixed it and unified
wrap-roll, scroll, and the unroll morph into one mechanism.)

---

## 6. Caret affinity (the subtle one)

At a **soft-wrap boundary**, the end of row *r* and the start of row *r+1* are the *same*
character offset. The textarea only stores the offset, so "which visual line is the caret on?"
is genuinely ambiguous and must be tracked as extra state: **`endAffinity`**.

- `endAffinity = true` → show the caret on the line that *ends* at that offset.
- `false` → show it on the line that *starts* there (the default; `rowOf` prefers the later row).

The rule lives in exactly one place now: **`placeCaret(pos, row, anchor)`** sets
`endAffinity = affinityFor(pos, row)` — end-affinity iff `pos` is that row's wrap seam. Every
caret placement that knows its intended row (Home, End, Up/Down, click, tap, drag) goes through
`placeCaret`. Native horizontal moves and typing reset `endAffinity = false`.

Lesson learned the hard way: affinity is **caret state set by the placement's intended row**,
not something a single operation (like End) can assume. The bug sequence was: End forced
end-affinity to find its *source* row, which broke "End from a line start." The fix was to
resolve the source row using the *current* affinity and set the new affinity from where it
lands. If you touch caret navigation, preserve this: source row uses current `endAffinity`,
destination affinity comes from `affinityFor`.

---

## 7. Scroll model & modes

`mode` is one of `'write'`, `'scroll'`, `'select'`.

Caret-follow lives in `layout()` and is **margin-based**, not "always recenter":

- Roll mode: `margin = 0` → the caret is pinned to the center seam (typewriter scrolling).
- Unroll mode: `margin ≈ visibleLines/2 - scrolloff` → the caret roams freely on screen; the
  view only scrolls when the caret would cross an edge, and then by the minimum to keep it just
  inside. This is why moving the caret in flat mode usually doesn't move the page.

Explicit scrolling (`wheel`, touch drag) moves `target`/`center` directly with `mode='scroll'`
and never touches the caret ("look back"). Because both modes share `center`, one code path
serves scroll, snap, and caret-follow for both.

Mouse-wheel vs trackpad: **browsers don't tell you.** We infer: `deltaMode !== 0` or
`|deltaY| >= TUNE.wheelMouse` (40) ⇒ treat as a mouse notch → step exactly one line
(`sign(deltaY)`, magnitude ignored so high-DPI scaling can't inflate it). Otherwise trackpad →
glide continuously, then snap to the nearest line after `TUNE.snapDelay` ms of stillness. Touch
is unambiguous (pointer events) and snaps on release. This is heuristic and can misjudge a
free-spinning high-res mouse wheel; that's a known, accepted limitation.

---

## 8. Pointer input

All pointer logic is on `.stage`; the textarea is `pointer-events:none`. Branching:

- **mouse** press → place caret (and set anchor); drag past `TUNE.dragMouse` px → select;
  release without drag → click (recenters in roll, not in unroll).
- **touch** press → potential scroll; drag past `TUNE.dragTouch` px → scroll; tap → place caret
  (and focus, which opens the mobile keyboard).

Point→offset uses `caretPositionFromPoint` (fallback `caretRangeFromPoint`) against the bands,
returning `{pos, row}` so the caller can set affinity from the clicked row. A geometric fallback
(nearest visible band by y, then `offsetAtX`) covers clicks in gaps.

**`e.preventDefault()` on the mouse `pointerdown` is load-bearing** — without it the browser's
default mousedown moves focus off the (pointer-events:none) textarea right after we focus it,
and typing silently stops. This bug has bitten twice.

---

## 9. Bugs we hit and the lessons (read before "fixing" something that looks odd)

- **Invisible textarea leaks.** `color:transparent` hides the text but *not* the native
  selection highlight, caret, or spellcheck underlines. The robust fix is `opacity:0` on the
  whole element (renders nothing, still focusable), **not** per-property transparency. Don't
  reintroduce `color:transparent` thinking it's equivalent.
- **Focus stealing** — see §8. Keep the `preventDefault` on mouse pointerdown.
- **Zero-width caret.** The caret is a 0-width element painted with `box-shadow`, not a 2px box.
  A 2px caret adds layout width to the active line, so it wraps one character earlier than the
  `wrapmirror` predicts → the line visibly wraps but the drum doesn't roll. Keep the caret
  layout-free.
- **Held arrow keys.** Re-render must be driven by `selectionchange` (fires on every caret
  move, including auto-repeat), not `keyup` (fires only on release). Keying off `keyup` makes
  the view freeze while a key is held.
- **`WINDOW` coupling.** `WINDOW` must be ≥ the max `lines`-visible option or edge rows get
  clipped. It's now derived (`Math.max(...OPTS.lines) + 1`); keep it derived.
- **High-DPI wheel deltas** are non-integer; don't gate mouse-wheel detection on
  `Number.isInteger(deltaY)`.
- **`localStorage` in the artifact sandbox** can throw; all access is wrapped in try/catch. Keep
  it that way — the file must run both as a standalone download and inside a preview.
- **Caret only when focused.** The drawn caret's visibility/blink hang off a `.focused` class
  the textarea toggles; otherwise it blinks forever even when focus is elsewhere.

---

## 10. Settings & TUNE

- **`TUNE`** (top of the script) holds all *feel* constants (easing, scroll sensitivity, snap
  delay, thresholds, opacity falloff, wrap detection, font ratio, scrolloff). Tune feel here,
  in one place.
- **`OPTS`** are the user-facing curated slider values (`size`, `lines`, `blur`, `font`), each a
  short list; sliders are index-based over these lists. `DEFAULTS` picks a value from each list
  (font default snaps to the nearest option to a screen-derived size). `flat` (roll/unroll
  state) is persisted too.
- Settings apply **live** (on slider `input`), persist immediately (`saveSettings`), and hand
  focus back to the textarea on `change` so you can type-test with the panel open. Reset returns
  to defaults and to rolled.
- Adding a knob: add to `OPTS`/`fmt`/`KEYS`, add a slider in the panel HTML, handle it in
  `applyOne`. Geometry knobs call `draw()`; ones that change wrapping (font) must invalidate
  `cacheVal` and call `layout()`.

---

## 11. Invariants — don't break these

1. The textarea stays focused whenever the user is editing; its selection always mirrors the
   drawn caret/selection.
2. Mirror font metrics == band font metrics (kept in `applyFont`).
3. The caret occupies zero layout width.
4. Both modes stay center-anchored on the shared `center` (or the smooth unroll breaks).
5. Caret affinity is set by `placeCaret` from the intended row; native moves clear it.
6. No CSS transitions on bands — motion goes through the rAF loop.
7. `WINDOW` stays derived from `OPTS.lines`.
8. All `localStorage` stays inside try/catch.

---

## 12. Known limitations / intentionally not done

- **Touch text selection** (selection handles) isn't implemented; touch drag scrolls. Mouse
  drag and keyboard selection work.
- **Clicking off-screen** can't happen, so click-to-place is bounded to visible rows; far jumps
  use keyboard.
- **Very large documents**: flat mode renders enough bands to fill the viewport, and selection
  re-measures per row on drag. Fine at prototype scale; would need memoization / virtualization
  to push hard.
- **Spellcheck** is off (its underlines can't be redrawn on the bands cheaply).
- **Mid-text typing that causes a downstream wrap** places the caret with start-affinity (a rare
  cosmetic edge); typing at the end (the common case) is always correct.
- **Top-anchored flat editing** was deliberately dropped in favor of center-anchored, because a
  stable unroll transition + top-anchored editing is a genuinely harder combination (you'd have
  to animate the anchor point).

---

## 13. Working on it

- It's a single self-contained HTML file on purpose (portable, "just open it"). Don't split CSS/JS
  into separate files or add a build step without a strong reason.
- Sanity check after edits: extract the script and run `node --check` on it (catches syntax
  errors; it won't catch layout/runtime issues, which need a browser).
- Manual test matrix worth running after any input/geometry change: type until it wraps; hold
  Up/Down across wrapped lines; Home/End on a soft-wrapped line and from a line start; click the
  tail of a wrapped line; drag-select across the edge (auto-scroll); mouse-wheel one notch =
  one line; trackpad glide + snap; toggle roll/unroll mid-edit and while scrolled away; every
  settings slider live; reset.
- The code is one IIFE grouped by `// ----` sections: state → settings → geometry/measurement →
  rendering → animation loop → caret/selection → layout → point-hit → keyboard → pointer →
  wheel → panel/toggle → init. Keep new code in the matching section.
