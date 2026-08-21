# Lecture 9 — JavaScript basics: output, variables, functions, DOM, form validation

Source: `transcripts/Django9.txt`
Covers: what JavaScript is, comments, internal/external scripts, output methods, variables, a full even/odd example, functions triggered by `onclick`, DOM manipulation, and JavaScript form validation.

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official JS/MDN docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. What JavaScript actually is **[Gap-filled, drawn out from the video's intro]**

JavaScript is a lightweight, **interpreted** language (run line-by-line by the browser, like Python — not compiled ahead of time) and **dynamic** (variable types are figured out at runtime, not declared upfront). It was originally named **LiveScript**; the rename to "JavaScript" was a marketing decision to ride on Java's popularity at the time — the two languages are otherwise unrelated, despite the similar name.

In a Django project, JavaScript's main job is **client-side validation**: catching obviously-bad form input in the browser, before it's ever sent to the server. HTML provides structure, CSS provides styling, JavaScript provides behavior.

## 2. Where JavaScript code lives **[From video]**

Like CSS, JavaScript can be internal (inside a `<script>` tag on the page) or external (a separate `.js` file, linked in).

```html
<!-- internal -->
<script>
    console.log("Hello, Durga!");
</script>

<!-- external -->
<script src="script.js"></script>
```

Convention: place `<script>` tags in the `<head>`, same as CSS `<link>` tags — though for scripts that manipulate content on the page, placing them at the end of `<body>` is also common so the HTML loads first.

`src` on `<script>` works exactly like `href` did for a CSS `<link>` — it points to the external file. Unlike the CSS `rel` attribute (see Lecture 7), the `type="text/javascript"` attribute really *is* optional here: modern HTML defaults `<script>` to JavaScript, so omitting `type` works correctly in every current browser.

> **[Gap-filled]** Inside a Django project, create an external JavaScript file the same way as an HTML template or a CSS file: in the `templates` folder, right-click → **New** → **JavaScript File**, name it (e.g. `sample.js`). Link it into a page with `<script src="sample.js"></script>`.

## 3. Comments **[From video]**

```javascript
// single-line comment

/* multi-line
   comment */
```

Same purpose as HTML comments: notes for whoever reads the code later. The JavaScript interpreter skips comments entirely — they have no effect on how the script runs.

## 4. Displaying output — and which method to actually use **[From video]**

| Method | What it does |
|---|---|
| `console.log()` | Prints to the browser's developer console (not visible on the page) — the standard tool for debugging |
| `alert()` | Shows a popup dialog box the user must dismiss |
| `document.write()` | Writes directly into the page's HTML |

> **[Gap-filled]** The video shows all three as roughly interchangeable options. In real code, `document.write()` is avoided — if it runs after the page has finished loading, it wipes out the entire existing page content, which causes hard-to-debug bugs. `console.log()` is the standard for debugging; `alert()` is reserved for messages that genuinely need to interrupt the user.

## 5. Variables **[From video]**

```javascript
let name = "Durga";
console.log("Your name is " + name);

// getting input at runtime
let userName = prompt("Enter name");
let a = Number(prompt("Enter first number"));
let b = Number(prompt("Enter second number"));
console.log("Sum is " + (a + b));
```

`prompt()` is JavaScript's equivalent of Python's `input()` — but everything it returns is always text. That's why `Number(...)` wraps it above: without converting, `"10" + "20"` would produce the text `"1020"` (stuck together) instead of adding to `30`.

> **[Example] — even/odd checker.** Combining a runtime-entered number with an `if`/`else` and the remainder operator `%`:
> ```javascript
> let n = Number(prompt("Enter any number"));
>
> if (n % 2 == 0) {
>     console.log(n + " is even");
> } else {
>     console.log(n + " is odd");
> }
> ```
> `%` gives the remainder of a division — `n % 2` is `0` for any even number and `1` for any odd number, which is exactly what the condition checks. Input `7` → output `"7 is odd"`; input `10` → output `"10 is even"`.

> **[Gap-filled] — var vs. let vs. const.** The video only ever declares variables with `var`. Modern JavaScript (since 2015) prefers `let` (for values that change) and `const` (for values that never get reassigned). `var` has looser, more error-prone scoping rules that `let`/`const` were introduced specifically to fix. Current style guides, including MDN's, recommend avoiding `var` in new code.

## 6. Functions & the `onclick` event **[From video]**

A **function** is a named, reusable block of code — write it once, then run it whenever you need to by calling its name. In the browser, functions are commonly triggered by **events**: something the user does, like clicking a button.

```html
<script>
function sayHello() {
    alert("Hello!");
}
</script>

<input type="button" value="Click here" onclick="sayHello()">
```

`function sayHello() { ... }` defines the function but doesn't run it — nothing happens until it's called. `onclick="sayHello()"` on the button is that call: it wires up the click event so the function runs each time the button is clicked. Unlike Python, JavaScript doesn't use indentation to mark a function's body — the curly braces `{ }` do that job.

## 7. What "DOM" means **[From video]**

The **DOM** (Document Object Model) is the browser's live, in-memory representation of your HTML page — a structure JavaScript can read and change after the page has loaded, and the browser instantly reflects those changes on screen.

```html
<p id="msg">This is my first paragraph.</p>

<script>
    document.getElementById("msg").innerHTML = "Welcome, Durga!";
</script>
```

`document.getElementById("msg")` finds the one element whose `id` is `"msg"`; `.innerHTML` is the property holding everything inside that element, which you can read or overwrite.

## 8. Form validation **[From video]**

```html
<form name="myform" onsubmit="return validateForm()">
    Name: <input type="text" name="username"><br>
    Password: <input type="password" name="pwd"><br>
    <input type="submit" value="Login">
</form>

<script>
function validateForm() {
    let name = document.myform.username.value;
    let pwd  = document.myform.pwd.value;

    if (name == "") {
        alert("Name cannot be blank");
        return false;
    }
    if (pwd.length < 6) {
        alert("Password must be at least 6 characters long");
        return false;
    }
    return true; // only now does the form actually submit
}
</script>
```

The key mechanic: `onsubmit="return validateForm()"`. If `validateForm()` returns `false`, the browser cancels the submission entirely — the form never leaves the page. If it returns `true`, submission proceeds normally — to wherever the form's `action` attribute points (e.g. `<form action="https://example.com" ...>` would send the browser there once validation passes).

### Matching two password fields

```javascript
function matchPwd() {
    let p1 = document.myform.password1.value;
    let p2 = document.myform.password2.value;
    if (p1 != p2) {
        alert("Passwords must be the same");
        return false;
    }
    return true;
}
```

> **[Gap-filled] — this is an important limitation, not a footnote.** Client-side JavaScript validation only improves the user's experience (instant feedback, no round-trip to the server for an obvious mistake). It is **not** a security boundary — anyone can open their browser's developer tools, disable JavaScript, or send a request directly (bypassing the form entirely) and submit whatever data they want. The video actually mentions this: Django performs its own server-side validation regardless, and that server-side check is the one that actually protects your data. Treat JavaScript validation as a convenience layered on top of — never a replacement for — server-side validation (in Django: Django Forms/ModelForms, covered in later sessions).

---

## Wrap-up

- **From video:** script placement/internal vs. external, comments, the three output methods, variables + `prompt()` + `Number()` conversion, functions + `onclick`, DOM manipulation via `getElementById`/`innerHTML`, and full form validation (name-not-empty, password-length, password-match) using `onsubmit`.
- **Gap-filled:** JavaScript's background (interpreted/dynamic, LiveScript origin, unrelated to Java), the Django workflow for creating a `.js` file, `var` vs `let`/`const`, and the reminder that client-side validation is UX, not security.
- **Researched:** none needed this lecture beyond what's already gap-filled — nothing the video stated was factually wrong here (a contrast with Lecture 7's incorrect claim about `rel="stylesheet"` — this time the video's claim that `type="text/javascript"` is optional is actually correct).

Double-check: nothing flagged as incorrect this lecture — the main things worth re-verifying yourself are the `var`/`let`/`const` guidance and the DOM/`innerHTML` behavior, since those move fast in the video.
