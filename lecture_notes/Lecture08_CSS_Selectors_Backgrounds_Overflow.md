# Lecture 8 — CSS selectors, pseudo-elements, backgrounds, div/span, overflow

Source: `transcripts/Django8.txt`
Covers: the four CSS selector types, selector priority/specificity, pseudo-elements and link pseudo-classes, background-image styling, `<div>` vs `<span>`, and the `overflow` property.

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official CSS docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. The main selector types **[From video]**

A **selector** picks which HTML element(s) a CSS rule applies to.

| Selector | Syntax | Targets |
|---|---|---|
| Element | `p { }` | Every `<p>` on the page |
| Class | `.center { }` | Any element with `class="center"` |
| ID | `#main { }` | The one element with `id="main"` (IDs must be unique per page) |
| Universal | `* { }` | Every element on the page |
| Grouped | `h1, h2, p { }` | All listed selectors, sharing one rule block |

```css
.center {
    text-align: center;
}

p.center {
    /* only paragraphs with class="center" — narrower than .center alone */
    color: purple;
}
```

## 2. Selector priority — and the real rule **[From video + Researched]**

The video's rule of thumb: **ID beats class beats element**. That's directionally correct, but the precise version (from any CSS reference) is a specificity score:

1. **Inline styles** — always highest.
2. **ID selectors** (`#main`) — very high.
3. **Class, attribute, and pseudo-class selectors** (`.center`, `:hover`) — medium.
4. **Element and pseudo-element selectors** (`p`, `::first-letter`) — lowest.

> **[Researched]** Only when two rules have the *exact same* specificity score does "whichever is defined later wins" come into play. Full reference: [MDN — CSS Specificity](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_cascade/Specificity).

> **Industry best practice:** Prefer class selectors over ID selectors for styling in real projects. IDs have such high specificity that they become hard to override later, and since an ID must be unique per page, class-based styles are far more reusable across multiple elements.

## 3. Pseudo-elements & link states **[From video]**

**Pseudo-elements** style a specific part of an element rather than the whole thing:

```css
p::first-letter { color: blue; }
p::first-line   { color: red; }
```

Links have four special states, styled with **pseudo-classes** — and their order in your CSS file matters:

```css
a:link    { color: red; }    /* unvisited */
a:visited { color: green; }  /* already visited */
a:hover   { color: yellow; } /* mouse over it */
a:active  { color: blue; }   /* being clicked, right now */
```

Remember the order with the mnemonic **LoVe HAte** — Link, Visited, Hover, Active. Writing them in a different order can make `:hover` or `:active` silently fail to apply.

## 4. Background images **[From video]**

```css
body {
    background-image: url('flag.jpg');
    background-repeat: no-repeat;
    background-position: right top;
    background-attachment: fixed;
    background-size: cover;
}
```

- `background-repeat: no-repeat` — stops the image tiling across the whole page.
- `background-position` — where the image sits (e.g. `right top`, `center`).
- `background-attachment: fixed` — the image stays in place even as the page scrolls.
- `background-size: cover` — stretches the image to fill the entire element, cropping if needed.

> **[Example] — shorthand.** All five properties above can be combined into a single `background` declaration, in the order `image → repeat → position → attachment`:
> ```css
> body {
>     background: url('flag.jpg') no-repeat right top fixed;
> }
> ```

> **[Gap-filled] — `body` vs. `html` for a true full-page background.** Setting the background on `body` only covers the area the page's content actually occupies — on a short page, the color/image stops where the content ends, leaving a gap below. Setting it on `html` instead covers the entire browser viewport, since `<html>` is the outermost element and always spans the full window height. For a background guaranteed to fill the whole visible page, target `html` (or set `min-height: 100%` on `body` as an alternative).

## 5. `<div>` vs. `<span>` **[From video]**

Both are generic containers with no visual style of their own — they exist purely to group content so you can apply CSS or JavaScript to that group.

- `<div>` — a **block-level** element (takes up the full width available, starts on a new line). Used to group larger sections — a header, a sidebar, a card.
- `<span>` — an **inline** element (only takes up as much width as its content, stays in the flow of surrounding text). Used to style a small piece of text inside a sentence without breaking the line.

```html
<style>
    div { color: red; background-color: lightblue; }
    span { color: green; font-size: 20px; font-family: Georgia; }
</style>

<div>
    <h1>This is heading one</h1>
    <h2>This is heading two</h2>
</div>

<p><b>Welcome to</b> <span>Durga's class</span></p>
```

Everything inside the `<div>` (both headings) gets red text on a light-blue background, and starts on its own new block. Only the word wrapped in `<span>` gets its own styling — the rest of that paragraph's text flows normally around it, on the same line.

## 6. Overflow **[From video]**

When content is bigger than the box you've given it (via fixed `width`/`height`), `overflow` decides what happens to the extra content:

| Value | Effect |
|---|---|
| `visible` | Extra content spills outside the box (the default) |
| `hidden` | Extra content is clipped and simply not shown |
| `scroll` | Scrollbars appear so the user can scroll to see the rest |
| `auto` | Scrollbars appear only if needed |

```css
p {
    height: 200px;
    width: 400px;
    border: 3px solid red;
    overflow: scroll;
}
```

Here the paragraph's box is fixed at 400×200 pixels. If the text inside is longer than that box, `overflow: scroll` keeps it readable by adding scrollbars, rather than letting it spill past the red border (`visible`) or silently cutting it off (`hidden`).

> **[Researched] — centering content isn't just "align".** The video reaches for a generic "align" property to center content inside a box and finds it doesn't work directly — that's expected. `text-align: center` centers inline content (text, images) *horizontally* within its container. `align-content`/`align-items` only have an effect inside a `display: flex` or `display: grid` container — applying them to an ordinary block element (like a plain `<div>` or `<p>`) does nothing, which is exactly the confusion in the video. To center a block element itself (not just its content) horizontally, the standard approach is `margin: 0 auto;` with a fixed `width`.

> **Industry best practice:** Prefer `overflow: auto` over `scroll` in real layouts — `auto` only shows scrollbars when content actually overflows, while `scroll` shows them permanently even when there's nothing to scroll, which looks broken to users.

---

## Wrap-up

- **From video:** the four selector types (element/class/ID/universal) plus grouping, the simplified ID-beats-class-beats-element priority rule, pseudo-elements (`::first-letter`/`::first-line`), the LVHA link pseudo-class order, all background-image properties, `<div>` vs `<span>`, and `overflow` values.
- **Gap-filled:** the `body` vs `html` distinction for a truly full-page background.
- **Researched:** the precise CSS specificity hierarchy (inline > ID > class/pseudo-class > element), and why `align`/`align-content` didn't work in the video's demo (they only apply inside flex/grid containers) — with `margin: 0 auto` given as the real way to center a block.

Double-check: the specificity hierarchy and the align-content explanation are the two spots I'd re-verify against MDN yourself if you want to be fully confident before relying on them.
