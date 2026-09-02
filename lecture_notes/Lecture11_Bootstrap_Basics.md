# Lecture 11 — Bootstrap basics

Source: `transcripts/Django11.txt`
Covers: what Bootstrap is, responsive design, the two ways to add it to a page, and a tour of common prebuilt components.

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official Bootstrap docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. What Bootstrap is, and why "responsive design" matters **[From video]**

Bootstrap is a free front-end **framework**: a large pre-built set of CSS classes (and some JavaScript) for common UI needs — buttons, tables, navigation bars, forms, spinners, pagination — applied just by adding a class name, instead of writing that CSS yourself.

Bootstrap's headline feature is **responsive design**: the same page automatically rearranges itself to fit the screen it's viewed on, from a wide desktop monitor down to a narrow phone screen — without writing separate mobile and desktop versions.

> **[Gap-filled] — why this used to be a bigger deal.** Before responsive frameworks were standard, sites were often built for desktop only, and mobile visitors got a broken, oversized layout — or the site offered a separate link to switch to a stripped-down "mobile view." Bootstrap's grid and components use CSS that adapts continuously to the available width, so one HTML page serves every screen size correctly by default. This is also why Bootstrap saves real development time and gives visual consistency across pages — you're reusing tested components instead of rebuilding the same button or nav bar with custom CSS on every page.

## 2. Adding Bootstrap to a page: two ways **[From video]**

| Method | How | Needs internet each load? |
|---|---|---|
| Local | Download the compiled CSS + JS bundle from [getbootstrap.com](https://getbootstrap.com), extract it, keep the files in your project | No |
| CDN | Link directly to Bootstrap's file hosted on their servers | Yes |

```html
<!-- Local, in <head> -->
<link rel="stylesheet" href="bootstrap.css">
<script src="bootstrap.js"></script>

<!-- CDN, in <head> -->
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/css/bootstrap.min.css">
<script src="https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/js/bootstrap.bundle.min.js"></script>
```

Exactly the same pattern as jQuery: a `<link>` for the CSS, a `<script>` for the JS. The download from getbootstrap.com gives you the "compiled and minified" bundle — the same production-ready idea as jQuery's `.min.js`.

> **[Researched] — Bootstrap 5 dropped the jQuery dependency.** Versions of Bootstrap before 5 required jQuery to be loaded for interactive components (dropdowns, modals, etc.) to work. [Bootstrap 5](https://getbootstrap.com/docs/5.1/migration/) (used in this video) rewrote its JavaScript in plain JS and no longer needs jQuery at all. If you're following an older Bootstrap tutorial that includes a jQuery `<script>` tag, that's a sign it targets Bootstrap 3 or 4 — not required with Bootstrap 5+.

## 3. Turning a plain table into a Bootstrap table **[From video]**

```html
<div class="container">
    <h2>Basic table</h2>
    <table class="table table-striped table-hover">
        <tr><th>First name</th><th>Last name</th><th>Email</th></tr>
        <tr><td>Mohan</td><td>Rao</td><td>mohan@example.com</td></tr>
    </table>
</div>
```

No new HTML tags — the same plain `<table>`/`<tr>`/`<th>`/`<td>` from Lecture 6, just with Bootstrap classes added: `container` centers and constrains page width, `table` applies Bootstrap's base table styling, `table-hover` highlights a row under the mouse, and (common, though not shown above) `table-responsive` on a wrapping `<div>` adds horizontal scrolling on narrow screens instead of squashing the table.

## 4. Other prebuilt components **[From video]**

- **Navbar** — a responsive navigation menu that collapses into a "hamburger" icon on small screens.
- **Spinners** — animated loading indicators (`spinner-border`, `spinner-grow`), in variants like `text-primary`, `text-success`, `text-danger`.
- **Pagination** — numbered page links (1, 2, 3…) for splitting a long list of records across pages. This becomes directly relevant later, when Django (and Django REST Framework) views return large querysets that need paging.

> **[Gap-filled]** The exact class names for every component (which the video mostly copy-pastes from prepared examples rather than typing from memory) live in Bootstrap's own docs at [getbootstrap.com/docs](https://getbootstrap.com/docs). In real work, you don't memorize these — you look up the component you need, copy its example markup, and adjust the content. That's the normal, expected workflow, not a shortcut.

> **Industry best practice:** CDN is convenient for quick experiments, but for a real Django project, download Bootstrap and serve it as a Django static file (Lecture 13's topic) rather than dropping it loose in the `templates` folder — that keeps it working even with no internet access and fits Django's standard way of organizing CSS/JS/images.

---

## Wrap-up

- **From video:** what Bootstrap is and why responsive design matters, the local vs. CDN setup (mirroring jQuery), a basic-table example with `container`/`table`/`table-hover`, and a tour of navbar/spinner/pagination components.
- **Gap-filled:** the pre-responsive-design history, and that looking up class names in the docs (rather than memorizing them) is the normal workflow.
- **Researched:** Bootstrap 5 no longer depends on jQuery, unlike earlier versions.

Double-check: nothing flagged as incorrect this lecture — the jQuery-dependency note is useful context if you ever follow an older Bootstrap tutorial and wonder why it also loads jQuery.
