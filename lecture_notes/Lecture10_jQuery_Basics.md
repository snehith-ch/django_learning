# Lecture 10 — Password match validation, and jQuery basics

Source: `transcripts/Django10.txt`
Covers: a re-enter-password validation example, then a full introduction to jQuery — what it is, how to add it (locally vs. CDN), its syntax, and worked hide/show examples.

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official jQuery/MDN docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. Password re-entry validation **[From video]**

A form with two password fields (`password1`, `password2`) that must match before submission is allowed to proceed:

```html
<form name="myform" action="home.html" onsubmit="return matchPwd()">
    Password: <input type="password" name="password1"><br>
    Re-enter password: <input type="password" name="password2"><br>
    <input type="submit" value="Submit">
</form>

<script>
function matchPwd() {
    let firstPassword = document.myform.password1.value;
    let secondPassword = document.myform.password2.value;

    if (firstPassword == secondPassword) {
        return true;
    } else {
        alert("Password must be same");
        return false;
    }
}
</script>
```

Same mechanic as the earlier form-validation example: `onsubmit="return matchPwd()"` blocks submission if the function returns `false`. On success here, the form's `action="home.html"` sends the browser to that page.

> **[Gap-filled]** The video also makes an important point at this stage: Django itself provides built-in server-side validation (through Django Forms), so you often don't need to hand-write JavaScript validation like this at all in a real Django app — it's mainly useful when Django's built-in validation doesn't cover something specific you need. This reinforces the earlier note that JS validation is a UX convenience layered on top of, not a replacement for, Django's own validation.

## 2. What jQuery is **[From video]**

jQuery is a **JavaScript library** — a pre-written bundle of code you call into instead of writing everything from scratch. It wraps a lot of common, repetitive JavaScript (selecting elements, showing/hiding things, handling events, and **AJAX** — Asynchronous JavaScript and XML, i.e. fetching data from a server without reloading the page) into short one-line methods.

jQuery was created by **John Resig in 2006**. Its selling points at the time: small, fast, and it smooths over inconsistencies between browsers — the same jQuery code reliably worked the same way across Chrome, Firefox, Internet Explorer, etc., which plain JavaScript of that era often didn't.

Slogan: **"write less, do more."** (For comparison, Django's own slogan is "don't repeat yourself" — DRY. Plain JavaScript is jokingly "write more, do more" by contrast.)

> **[Researched] — is jQuery still worth learning?** jQuery was essential a decade ago because plain JavaScript was verbose and inconsistent across browsers. Modern JavaScript has since added most of what jQuery offered directly into the language (e.g. `document.querySelector()`, `fetch()`), and modern browsers are far more consistent — so new projects increasingly skip jQuery entirely. It's still worth knowing because a huge number of existing/older sites (and some third-party front-end libraries) still depend on it, but for new Django projects specifically, plain JavaScript or a modern front-end approach is now more common than reaching for jQuery by default. Django itself has no dependency on jQuery at all.

## 3. Adding jQuery to a page: two ways **[From video]**

| Method | How | Needs internet each load? |
|---|---|---|
| Local | Download the `.js` file once, keep it in your project, link to it like any other script | No |
| CDN | Link directly to jQuery's file hosted on their servers | Yes |

```html
<!-- Local -->
<script src="jquery-3.6.0.min.js"></script>

<!-- CDN -->
<script src="https://code.jquery.com/jquery-3.6.0.min.js"></script>
```

jquery.com offers each release as either **compressed/production** (the `.min.js` file — all whitespace and comments stripped, one dense unreadable line, but small and fast to download) or **uncompressed/development** (the same code, readably formatted, larger). For a real site, use the minified build; the unminified one is only useful if you ever need to read jQuery's own source.

> **[Gap-filled]** For the local option, the downloaded `jquery-3.6.0.min.js` file goes wherever your other static files live — in this project, the `templates` folder, alongside your CSS/JS files. Then link it with a normal `<script src="jquery-3.6.0.min.js"></script>`, exactly like any other external JS file.

> **[Researched] — the `integrity` attribute on a CDN script.** jQuery's official CDN snippet includes an `integrity` attribute (a hash of the file) and `crossorigin="anonymous"`, e.g. `<script src="..." integrity="sha256-..." crossorigin="anonymous"></script>`. This is [Subresource Integrity (SRI)](https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity): the browser hashes the downloaded file and compares it to the expected hash, refusing to run the script if they don't match. It protects you if the CDN is ever compromised and starts serving tampered code. Always copy the full snippet jQuery's site gives you, integrity attribute included — don't strip it down to just `src`.

## 4. Basic syntax **[From video]**

```javascript
$(selector).action();

// example: hide every paragraph when a button is clicked
$(document).ready(function() {
    $("button").click(function() {
        $("p").hide();
    });
});
```

- `$` — shorthand for accessing jQuery.
- `selector` — which HTML element(s) to target (same idea as CSS selectors — element, `.class`, `#id`).
- `.action()` — what to do to them (`.hide()`, `.show()`, `.click()`, etc.).
- `$(document).ready(...)` — waits until the whole page has finished loading before running the code inside, so it never tries to act on an element that hasn't appeared yet.

> **[Example] — separate hide and show buttons.** Using ID selectors to wire up two different buttons to two different actions on the same paragraph:
> ```html
> <script>
> $(document).ready(function() {
>     $("#hide").click(function() {
>         $("p").hide();
>     });
>     $("#show").click(function() {
>         $("p").show();
>     });
> });
> </script>
>
> <p>This is my first paragraph.</p>
> <button id="hide">Click here to hide</button>
> <button id="show">Click here to show</button>
> ```
> `#hide` and `#show` match the buttons' `id` attributes, exactly like an ID selector in CSS. Each button's click handler only runs the action wired to *its own* id.

> **Common pitfall:** It's easy to swap the code inside two handlers by accident — e.g. writing `.hide()` inside the `#show` button's click handler (the video itself does exactly this at first) — and get no visible error, just a button that silently does the wrong thing (or nothing). If a jQuery action doesn't seem to fire, double-check that the selector in `$("#id")` actually matches the element you clicked, and that the action inside is the one you meant for that specific handler.

---

## Wrap-up

- **From video:** the password re-entry validation pattern, jQuery's purpose/history/slogan, the two ways to add it (local vs. CDN) with the compressed/uncompressed distinction, basic `$(selector).action()` syntax and `$(document).ready()`, and the hide/show button example.
- **Gap-filled:** Django's built-in server-side validation as the reason you often don't need hand-written JS validation, and the Django-project workflow for placing the downloaded jQuery file.
- **Researched:** what Subresource Integrity (the `integrity`/`crossorigin` attributes on CDN scripts) actually does, and a current-day take on whether jQuery is still worth learning for new projects.

Double-check: the SRI explanation and the "is jQuery still relevant" note are both added context beyond the video — worth a quick skim of MDN's SRI page if you want the full picture.
