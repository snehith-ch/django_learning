# Lecture 7 — More HTML formatting, the marquee tag, and CSS basics

Source: `transcripts/Django7.txt`
Covers: leftover HTML formatting elements from session 6, the `<marquee>` tag, and the full introduction to CSS — syntax, the three ways to apply it, and priority/cascade rules.

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from HTML/CSS official docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. More text-formatting elements **[From video]**

Beyond `<b>`/`<i>` vs. `<strong>`/`<em>` (covered in Lecture 6), HTML has more inline elements for special-meaning text:

| Tag | Effect | Status |
|---|---|---|
| `<u>` | Underline | Valid, but avoid — underlined text reads as a link to most users |
| `<sub>` | Subscript (small, below baseline — e.g. H₂O) | Valid |
| `<sup>` | Superscript (small, above baseline — e.g. x²) | Valid |
| `<mark>` | Highlighted text (yellow background by default) | Valid — HTML5 semantic tag |
| `<small>` | Smaller text — fine print, side comments | Valid |
| `<big>` | Larger text | Non-standard — not part of the HTML5 spec at all |
| `<del>` | Marks text as removed (strikethrough) | Valid — semantic replacement for `<strike>` |
| `<ins>` | Marks text as inserted (underline) | Valid |
| `<strike>` | Strikethrough | Obsolete legacy tag — use `<del>` or `<s>` instead |

```html
<p><b>Bold</b>, <i>italic</i>, <strong>strong</strong>, <em>emphasized</em>.</p>
<h2>My name is <small>Mohan</small> and I am from <ins>Hyderabad</ins></h2>
<p>This is <sub>subscripted</sub> and this is <sup>superscripted</sup>.</p>
<p>Please note: <mark>the office is closed on Monday</mark>.</p>
```

> **[Gap-filled] — `<mark>` vs. `<marquee>`.** These sound alike (and are easy to mix up from audio alone) but are two completely different tags. `<mark>` just highlights text in place. `<marquee>`, below, makes text scroll across the screen — a much older tag with a different purpose entirely.

> **[Researched] — `<big>` and `<strike>`.** Per [MDN](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/big), `<big>` was dropped from the HTML5 standard entirely (still renders in most browsers for backward compatibility, but never use it in new code — use CSS `font-size` instead). `<strike>` is officially obsolete too, superseded by `<del>` (semantically "removed") or `<s>` ("no longer accurate/relevant", no removal implied).

## 2. The `<marquee>` tag — and why to avoid it **[From video]**

`<marquee>` makes its text scroll across the page — historically used for banner-style alerts, e.g. a bank's website scrolling an important notice so it catches the customer's eye.

```html
<marquee>Welcome to Durga Sir</marquee>
<marquee bgcolor="red" direction="right">Welcome to Durga Sir</marquee>
```

- `bgcolor` — background color behind the scrolling text.
- `direction` — which way it scrolls (`left` by default, or `right`, `up`, `down`).

> **[Researched] — don't use `<marquee>` in real projects.** `<marquee>` was never part of any official HTML standard — [MDN marks it deprecated](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/marquee) and explicitly warns against using it, since browsers aren't required to keep supporting it. It's shown here because it still appears in a lot of tutorials and older sites, and it's useful to recognize. For a scrolling-banner effect in a real project, use a CSS animation instead — supported the same way everywhere, no reliance on a non-standard element.

## 3. CSS basics **[From video]**

**CSS** stands for **Cascading Style Sheets**. Its purpose: describe *how* HTML elements are displayed — colors, fonts, sizes, borders, spacing. HTML describes *content*; CSS describes *appearance*.

Basic syntax:

```css
selector {
    property: value;
    property: value;
}

/* example */
p {
    color: red;
    font-size: 25px;
}
```

A **selector** picks which HTML element(s) to style; everything inside `{ }` is a list of `property: value;` pairs.

> **[Example] — common properties together**
> ```css
> h1 {
>     color: crimson;
>     font-family: Georgia;
>     font-size: 40px;
> }
> ```
> `font-family` sets the typeface (browsers fall back to a default if the named font isn't installed — real projects usually list several as fallbacks, e.g. `font-family: Georgia, serif;`). `color` sets text color, `font-size` sets text size.

## 4. The three ways to apply CSS **[From video]**

| Type | How | Scope |
|---|---|---|
| **Inline** | `<p style="color:red;">` | Just that one element |
| **Internal** | `<style>...</style>` in `<head>` | Just that one HTML page |
| **External** | Separate `.css` file, linked in | Every page that links to it |

```html
<!-- linking an external stylesheet -->
<head>
    <link rel="stylesheet" href="style.css">
</head>
```

**Inline** — the `style` attribute directly on a tag. Useful for a one-off override on a single element; not recommended as a general approach.

**Internal** — a `<style>` block inside `<head>`. Applies only to that one HTML page — the same styling won't carry over to another page in the same project.

**External** — a separate `.css` file (only CSS rules inside, no HTML tags at all) linked via `<link>`. This is what real projects use almost exclusively: change one file, every linked page updates.

> **[Gap-filled]** Inside a Django project, an external stylesheet is created the same way as an HTML template: in the `templates` folder, right-click → **New** → **Style Sheet**, name it with a `.css` extension (e.g. `sample.css`). This file should contain *only* CSS rules — no `<html>`, `<head>`, or `<body>` tags.

> **[Researched] — the `rel` attribute is not optional.** The video calls `rel="stylesheet"` optional. That's incorrect for linking CSS: per the [HTML spec](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/link), `rel` tells the browser *how to treat* the linked file — without `rel="stylesheet"`, the browser has no reason to apply the file's contents as CSS at all, and the styles silently won't load. Always include it.

## 5. Which style wins? (priority) **[From video]**

If the same element gets conflicting styles from more than one place, CSS follows a clear order:

1. **Inline style always wins** over internal or external, no matter what.
2. Between internal and external: **whichever appears later** (further down) in the HTML file wins. If the `<link>` to the external file appears *before* an internal `<style>` block, the internal block wins (it's "later"). If the internal block is written first and the external `<link>` comes after it, the external file wins.

> **[Researched] — the real rule (specificity).** The "whichever comes later wins" rule the video gives is true only when the competing rules have equal *specificity* (roughly, how precisely a selector targets an element). CSS actually resolves conflicts using a specificity score first, and only falls back to "later wins" when scores are tied — ID selectors beat class selectors beat element selectors, regardless of order. The full topic (selectors, specificity scoring) is covered in the next lecture's notes.

---

## Wrap-up

- **From video:** the full list of HTML text-formatting elements (`u`, `sub`, `sup`, `mark`, `small`, `big`, `del`, `ins`, `strike`), the `<marquee>` tag and its `bgcolor`/`direction` attributes, CSS's purpose and basic syntax, the three ways to apply CSS (inline/internal/external), and the "later wins" priority rule between internal and external.
- **Gap-filled:** the mark-vs-marquee naming confusion, and the Django-specific workflow for creating an external `.css` file in the `templates` folder.
- **Researched:** `<big>` and `<strike>` are obsolete/non-standard; `<marquee>` was never an official standard and should be avoided in real projects; `rel="stylesheet"` is required, not optional, contrary to what the video says; and the "later wins" priority rule is really a fallback for equal-specificity ties, not the true CSS cascade rule.

Double-check: the video's claim that `rel="stylesheet"` is optional is flagged as incorrect above — worth re-testing yourself (try a `<link>` without `rel` and see the styles fail to apply) if you want to confirm it firsthand.
