# Lecture 6 — HTML Basics

Source: `transcripts/Django6.txt`
Covers: HTML structure, comments, lists, text formatting, images, tables, hyperlinks, and forms — the front-end foundation needed before building Django templates.

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from Django/HTML official docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. What HTML is, and where it fits **[From video]**

**HTML** stands for **HyperText Markup Language**. It's the mandatory building block of every web page — you cannot build a web application without it.

- **HTML** — describes the *content and structure* of a page (headings, paragraphs, images, forms).
- **CSS** — styles that HTML (colors, fonts, layout). Covered in the next lecture.
- **JavaScript** — adds functionality/behavior to the page.
- **Django template tags** — later, once HTML files become Django templates, tags like `{% %}` and `{{ }}` are added on top of this HTML to make the content dynamic (e.g. showing database records).

Every HTML page has two parts:

| Part | Contains |
|---|---|
| `<head>` | Metadata — title, links to CSS/JS files. Not visible on the page itself. |
| `<body>` | The actual visible content. |

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Home Page</title>
</head>
<body>
    <h1>This is heading one</h1>
    <p>This is a paragraph.</p>
</body>
</html>
```

- `<!DOCTYPE html>` — tells the browser this is a modern HTML5 document.
- `<meta charset="UTF-8">` — sets the character encoding so text displays correctly.
- `<title>` — shown on the browser tab, not on the page.
- `<h1>`–`<h6>` — heading levels, largest/most important (`h1`) to smallest (`h6`).
- `<p>` — a paragraph of text.

## 2. Creating & previewing HTML inside a Django project **[From video]**

Outside Django, you'd open a plain text editor, hand-type the tags, save as `.html`, then double-click to open it in a browser.

Inside a Django project it's smoother: in the `templates` folder, right-click → **New** → **HTML File**, name it (e.g. `home.html`), and the editor auto-generates the boilerplate skeleton for you (`<!DOCTYPE html>`, `<html>`, `<head>` with charset meta tag, empty `<body>`).

To preview it, Django-aware editors show browser icons (Chrome, Firefox, Edge) next to the file — click one for a live preview, no separate "run" step needed.

> **[Gap-filled]** This in-editor preview is just a convenience for *looking at* static markup while learning HTML in isolation. Once this HTML becomes a real Django template, you won't open the `.html` file directly in a browser anymore — you'll visit a URL your Django dev server serves (e.g. `http://127.0.0.1:8000/`), which runs it through a view first. Opening the raw file skips Django entirely, and any `{{ }}` template tags would show up as literal text instead of rendering.

## 3. Comments **[From video]**

A comment is text the browser parses but never displays.

```html
<!-- This is a comment, it will not show up on the page -->
<h1>This is heading one</h1>
```

Comments start with `<!--` and end with `-->` — anything in between (even multiple lines) is ignored by the browser.

## 4. Lists **[From video]**

Two list types: `<ol>` (ordered — numbered) and `<ul>` (unordered — bulleted). Items inside either use `<li>` (list item).

```html
<ol>
    <li>C</li>
    <li>C++</li>
    <li>Python</li>
</ol>
```

By default `<ol>` numbers items `1, 2, 3…`. The `type` attribute changes the style:

| `type` value | Renders as |
|---|---|
| `1` | 1, 2, 3, 4 (default — can omit `type` entirely) |
| `A` | A, B, C, D |
| `a` | a, b, c, d |
| `I` | I, II, III, IV (uppercase Roman numerals) |
| `i` | i, ii, iii, iv (lowercase Roman numerals) |

> **[Example]** An ordered list lettered instead of numbered:
> ```html
> <ol type="A">
>     <li>Asha</li>
>     <li>Ravi</li>
>     <li>Meera</li>
> </ol>
> <!-- Renders as: A. Asha   B. Ravi   C. Meera -->
> ```

`<ul>` has no `type`-based numbering — always bullets, and item order has no visual effect.

## 5. Text formatting: HTML5 tags, not legacy ones **[From video]**

| Want | Legacy tag (avoid) | HTML5 tag (use this) |
|---|---|---|
| Bold | `<b>` | `<strong>` |
| Italic | `<i>` | `<em>` |

`<strong>` and `<em>` render the same visually (bold/italic) but also carry *meaning* — "this matters" / "this is emphasized" — which screen readers and search engines pick up on. `<b>` and `<i>` are purely visual with no semantic meaning.

## 6. Images **[From video]**

```html
<img src="flag.jpg" alt="Country flag">
```

- `<img>` is **self-closing** — no separate closing tag, unlike `<p>` or `<b>`.
- `src` — the image file's location.
- `alt` — fallback text shown if the image fails to load.

> **[Gap-filled]** The video frames `alt` purely as a "broken image" fallback. Its bigger purpose: screen readers read `alt` text aloud for visually impaired users, so every meaningful image needs a real, descriptive `alt` — a baseline accessibility requirement, not a nice-to-have.

## 7. Tables **[From video]**

```html
<table border="1">
    <tr>
        <th>Employee ID</th>
        <th>Name</th>
    </tr>
    <tr>
        <td>101</td>
        <td>Asha</td>
    </tr>
</table>
```

`<table>` is the container, `<tr>` is one table row, `<th>` is a header cell (bold, centered by default), `<td>` is a normal data cell.

To center content in a cell: `<td align="center">101</td>`.

> **[Gap-filled]** `align` is a presentational HTML attribute — old-style, like `<b>`/`<i>` were for bold/italic. Modern practice is CSS instead: `text-align: center;` on the cell (see the CSS lecture). `align` still works everywhere, so you'll see it in older tutorials, but avoid it in new projects.

### Tables with images

A table cell can hold any HTML, including an `<img>` — useful for a "country + flag" list:

```html
<table border="1">
    <tr>
        <th>#</th>
        <th>Country</th>
        <th>Flag</th>
    </tr>
    <tr>
        <td align="center">1</td>
        <td align="center">India</td>
        <td align="center"><img src="india.jpg" alt="Flag of India"></td>
    </tr>
    <tr>
        <td align="center">2</td>
        <td align="center">USA</td>
        <td align="center"><img src="usa.png" alt="Flag of USA"></td>
    </tr>
</table>
```

Each `<img>` just goes inside its own `<td>`, exactly like anywhere else on the page.

> **[Gap-filled] — HTML is a markup language, not a programming language.** Every row above was typed by hand. In a real Django app, rows usually come from a database — e.g. one row per employee — and the count isn't known ahead of time. You can't write a loop in plain HTML to generate rows dynamically: HTML has no logic, no variables, no `if`/`for`, no classes or objects. It only describes structure. Repeating rows for database records is done with the **Django Template Language (DTL)** — e.g. `{% for employee in employees %}` — covered once templates are connected to views and models.

## 8. Hyperlinks **[From video]**

```html
<a href="https://www.djangoproject.com">Go to Django's site</a>
```

`<a>` is the anchor tag; `href` (hyperlink reference) is the destination — either another page on your own site (`href="index.html"`) or a full external URL.

> **Industry best practice:** Use headings (`h1`–`h6`) in order to reflect document structure, not just for font size — jumping from `h1` straight to `h4` for a smaller look is a common beginner mistake that hurts accessibility and SEO. Use CSS to control size instead.

## 9. HTML forms **[From video]**

Forms collect input from a user — logins, registrations, search boxes. Every form is a `<form>` wrapping one or more `<input>` elements.

```html
<form action="index.html" method="post">
    <label for="uname">Enter name</label>
    <input type="text" id="uname" name="username" placeholder="Enter name" required>
    <br>
    <input type="password" name="pwd">
    <br>
    <input type="submit" value="Submit">
</form>
```

The pieces:

- `action` — where the form's data gets sent when submitted.
- `method` — how it's sent (`post` for data that changes something, `get` for simple lookups).
- `type` on `<input>` — `text`, `password` (hides typed characters), `email`, `checkbox`, `radio`, `submit`, and more.
- `name` — the identifier used to read this field's value once submitted. Without a `name`, a field's value is never sent, even if filled in.
- `value` — meaning depends on the input type. For `text`/`email`/`password`, it's a *pre-filled default* the user can overwrite (usually left blank). For `checkbox`/`radio`/`submit`, it's not shown to the user — it's the identifier sent to the server for *that option* when checked/clicked.
- `placeholder` — greyed-out hint text inside an empty field; disappears once typed.
- `required` — the browser won't submit until this field has a value.
- `<label for="uname">` — links descriptive text to the input whose `id="uname"` matches. Clicking the label focuses the input, and screen readers announce it when the field is focused.

> **[Example] — grouping checkboxes and radio buttons.** Giving multiple inputs the *same* `name` groups them. For checkboxes, any number can be checked and each checked one's `value` is sent. For radio buttons, the shared `name` makes them mutually exclusive.
> ```html
> <p>Pick your courses:</p>
> <input type="checkbox" id="py" name="course" value="python">
> <label for="py">Python</label>
> <input type="checkbox" id="cs" name="course" value="csharp">
> <label for="cs">C#</label>
>
> <p>Preferred contact method:</p>
> <input type="radio" id="em" name="contact" value="email" checked>
> <label for="em">Email</label>
> <input type="radio" id="ph" name="contact" value="phone">
> <label for="ph">Phone</label>
> ```
> If both checkboxes are checked, the server receives `course=python` and `course=csharp`. For the radio group, only the selected one is sent — here `contact=email` by default via `checked`.

> **Industry best practice:** Always pair every input with a `<label>` (via matching `for`/`id`), not just a nearby paragraph of text — a paragraph has no programmatic connection to the field, so assistive technology can't associate the two.

> **[Researched] — CSRF protection.** These forms are plain HTML with no Django involved yet, so they work as shown. The moment a form like this `action`s to a real Django view with `method="post"`, Django's [CSRF protection](https://docs.djangoproject.com/en/stable/ref/csrf/) will reject the submission unless the form includes `{% csrf_token %}` inside it. This is a security feature (prevents other sites from silently submitting forms to your app on a logged-in user's behalf) — expect a 403 Forbidden error the first time you wire a POST form into Django and forget it.

---

## Wrap-up

- **From video:** HTML structure (`<head>`/`<body>`), headings, paragraphs, comments, ordered/unordered lists, legacy vs. HTML5 formatting tags, images (`src`/`alt`), tables (`<table>`/`<tr>`/`<th>`/`<td>`/`border`/`align`), tables with embedded images, hyperlinks (`<a>`/`href`), and forms (`<form>`, `<input>` types, `name`, `value`, `placeholder`, `required`, `<label>`).
- **Gap-filled:** the accessibility purpose of `alt` text; why `align` is legacy and CSS `text-align` is preferred; why HTML can't loop over database rows and how DTL solves that; the full meaning of `value` across input types; the in-editor HTML preview vs. actually being served by Django.
- **Researched:** CSRF token requirement once these forms POST to a real Django view.

Double-check: the CSRF note is forward-looking to when forms connect to Django — nothing to verify yet, just keep it in mind for later lectures.
