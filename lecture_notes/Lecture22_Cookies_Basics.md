# Lecture 22 — Cookies: concept, creating, reading, and deleting

Source: `transcripts/Django22.txt`
Covers: the first state-management technique in detail — what a cookie actually is, its limits, the two cookie types (in-memory vs. persistent), and the practical Django mechanics for setting, reading, and deleting a cookie.

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official Django docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. What a cookie is **[From video]**

A **cookie** is a small piece of information — up to **4096 bytes (4 KB)** — stored as **plain text** (a string) in the browser. A cookie is *created by the server* but *maintained/stored by the client* (the browser): the server tells the browser "please hold onto this," and from then on the browser is the one physically keeping it.

Once created, a cookie **travels with every request/response** between that browser and that specific website — it's attached automatically, you don't have to manually resend it. Cookies are also **domain-based**: a cookie created while visiting `gmail.com` is only ever sent back to `gmail.com`, never to an unrelated site.

> **[Gap-filled] — a simple mental model.** Think of a cookie as a sticky note the server hands the browser: "when you talk to me again, show me this note so I remember who you are." The browser keeps the note (in its own storage) and attaches it to every future request to that same site. The server never has to keep asking "who are you again?" — it just reads the note.

## 2. Cookie limits **[From video]**

| Limit | Value | What happens if exceeded |
|---|---|---|
| Size per cookie | 4096 bytes (4 KB) | — |
| Cookies per website, per browser | 20 | The oldest cookie for that site is deleted to make room for the new one |
| Cookies across all websites, in one browser | 200 | The oldest cookie overall is deleted to make room |

> **[Researched] — these numbers are historical browser conventions, not an HTTP or Django rule.** Per-domain and total cookie limits are set by each browser, not by any web standard — different browsers have historically used slightly different numbers (many settled around 50+ cookies per domain and several thousand total in modern versions). The video's 20/200 figures reflect an older, commonly-cited rule of thumb rather than a value you'll find enforced identically in every current browser. The takeaway that matters in practice is the same regardless of the exact number: **cookies are for small amounts of non-critical data**, not a general-purpose storage mechanism — don't design a feature around "I'll just store a lot of small values in cookies."

## 3. Why cookies have no security **[From video]**

Because a cookie's value is stored as **plain text**, anyone with access to that browser (or anyone intercepting unencrypted traffic) can open the browser's cookie storage and read — or delete — the value directly. This is the core drawback: **cookies should never hold sensitive data** (passwords, tokens, personal details) in plain form.

> **[Gap-filled] — how this plays out in practice.** This is exactly why the video's later cookie examples never store a real password; and it's why, once sessions are introduced (next lecture), the general pattern becomes "put a random, meaningless session *ID* in the cookie, and keep the actual sensitive data on the server" — the cookie itself never carries anything worth stealing.

## 4. Why cookies are useful, despite that **[From video]**

Even with no built-in security, cookies solve a real problem: **recognizing a returning visitor without asking them to re-authenticate every single request.**

- Log into Gmail once (without explicitly signing out) → close the browser → reopen it later and revisit Gmail → you're still logged in. The first login stored a cookie; every later visit sends that cookie along, and the server recognizes "this is the same user I already authenticated," so it skips asking for credentials again.
- On a shopping site, moving from a product page → cart → checkout needs *some* mechanism to remember "which item is this checkout page even about" — a cookie (or, more robustly, a session tied to a cookie, covered next lecture) is one way to carry that forward.

## 5. Two types of cookies **[From video]**

| Type | Expiration | Behavior |
|---|---|---|
| **In-memory cookie** | None set | Lives only in browser memory; deleted automatically the moment the browser window for that site is closed. |
| **Persistent cookie** | A specific date/time | Survives closing the browser; stays until that date/time is reached (or is deleted manually/programmatically before then). |

This lecture works only with in-memory cookies (no expiration set); the next lecture (23) covers setting an actual expiration to make a cookie persistent.

## 6. Setting up a cookie app **[From video]**

```bash
python manage.py startapp cookie_app
```

```python
# settings.py
INSTALLED_APPS = [
    # ...
    'cookie_app',
]
```

Same pattern as every earlier app in this course: create it with `startapp`, then register it in `INSTALLED_APPS` so Django actually loads it.

## 7. Setting a cookie **[From video]**

```python
# cookie_app/views.py
from django.http import HttpResponse

def set_cookie(request):
    response = HttpResponse("Cookie is set")
    response.set_cookie('name', 'Durga')
    return response
```

- `HttpResponse(...)` creates the response object first.
- `.set_cookie(key, value)` is a **method on the response object** — this is the key structural point: you don't create a cookie directly; you call `set_cookie()` on whatever response you're about to send back, and Django attaches the `Set-Cookie` header to it before it goes to the browser.
- `key` is the cookie's name (used later to look the value back up); `value` is what gets stored.
- The view must `return` that same response object — the cookie only gets sent if the response carrying it actually gets returned.

```python
# cookie_app/urls.py
from django.urls import path
from cookie_app import views

urlpatterns = [
    path('set/', views.set_cookie),
]
```

```python
# project urls.py
from django.urls import path, include

urlpatterns = [
    # ...
    path('cookie_app/', include('cookie_app.urls')),
]
```

Visiting `/cookie_app/set/` runs the view, and Chrome's cookie inspector (Settings → Privacy and security → Cookies and other site data → See all cookies and site data → the `127.0.0.1` entry) now shows a cookie named `name` with value `Durga`, no expiration date set ("When the browsing session ends" — confirming it's an in-memory cookie). Closing the browser and reopening on a fresh window shows the cookie gone, exactly as the in-memory behavior predicts.

## 8. Reading a cookie **[From video]**

```python
def get_cookie(request):
    nm = request.COOKIES['name']
    return HttpResponse("Your name is " + nm)
```

`request.COOKIES` is a dictionary-like object holding every cookie the browser sent along with this request; `request.COOKIES['name']` looks up the cookie by the same key used in `set_cookie()`. Note the asymmetry: you *set* a cookie on the **response**, but you *read* a cookie from the **request** — makes sense once you notice a cookie only becomes visible to your code the next time the browser sends it back, i.e. on a subsequent incoming request.

> **Industry best practice / [Researched] — always prefer `.get()` over `[key]` for reading a cookie.** `request.COOKIES['name']` raises a `KeyError` (crashing the view with a 500 error) if that cookie doesn't exist — for instance if it was deleted, expired, or simply never set. `request.COOKIES.get('name', 'default value')` returns the default instead of crashing:
> ```python
> def get_cookie(request):
>     nm = request.COOKIES.get('name', 'Cookie not set')
>     return HttpResponse("Your name is " + nm)
> ```
> This is exactly the fix the video applies mid-lecture after hitting the `KeyError` live — it's a real, common Django gotcha, not an edge case you can safely ignore. Per the [Django docs](https://docs.djangoproject.com/en/stable/ref/request-response/#django.http.HttpRequest.COOKIES), `request.COOKIES` is a standard Python dict, so this is just the normal dict-safety pattern (`.get(key, default)` vs. `[key]`) applied to cookies specifically.

## 9. Deleting a cookie **[From video]**

```python
def del_cookie(request):
    response = HttpResponse("Cookie is deleted")
    response.delete_cookie('name')
    return response
```

`.delete_cookie(key)` is, like `set_cookie()`, a method called on the **response** object — it tells the browser to remove that cookie. After this view runs, a subsequent `get_cookie` request (using the safe `.get()` version above) correctly reports "Cookie not set" instead of crashing.

## 10. Rendering cookie views through templates instead of raw text **[From video]**

The video also shows the same three operations wired to actual HTML templates via `render()`, rather than returning bare `HttpResponse` text — useful once a cookie action needs to show more than one line:

```python
from django.shortcuts import render

def set_cookie(request):
    response = render(request, 'set.html')
    response.set_cookie('name', 'Durga')
    return response

def get_cookie(request):
    name = request.COOKIES.get('name', 'Cookie is deleted')
    return render(request, 'get.html', {'name': name})

def del_cookie(request):
    response = render(request, 'del.html')
    response.delete_cookie('name')
    return response
```

```html
<!-- set.html --><h1>Cookie is set</h1>
<!-- get.html --><h1>Cookie value: {{ name }}</h1>
<!-- del.html --><h1>Cookie deleted</h1>
```

The important structural detail here: `render()` *also* returns a full response object (an `HttpResponse` under the hood), so `.set_cookie()` / `.delete_cookie()` work on it exactly the same way as on a plain `HttpResponse` — you just call `render()` first, store what it returns, call the cookie method on that, then return it.

> **[Example]** A standalone illustration of the "set on response, read from request" split, outside the cookie app: a simple "remember the visitor's favorite color" pair of views.
> ```python
> def set_color(request, color):
>     response = HttpResponse(f"Favorite color set to {color}")
>     response.set_cookie('fav_color', color, max_age=None)  # in-memory, no expiry
>     return response
>
> def show_color(request):
>     color = request.COOKIES.get('fav_color', 'not set yet')
>     return HttpResponse(f"Your favorite color is: {color}")
> ```
> `GET /set-color/blue/` → `GET /show-color/` now returns "Your favorite color is: blue"; visiting `/show-color/` first, before ever setting one, safely returns "Your favorite color is: not set yet" instead of crashing.

---

## Wrap-up

- **From video:** what a cookie is (small, plain-text, client-stored, domain-scoped, travels with every request); the 4 KB / 20-per-site / 200-total limits; why cookies have no real security; why they're still useful (recognizing returning visitors); in-memory vs. persistent cookies; creating the `cookie_app`; `response.set_cookie(key, value)`, `request.COOKIES[key]`, `response.delete_cookie(key)`; wiring all three through both raw `HttpResponse` and `render()`.
- **Gap-filled:** the "sticky note" mental model for what a cookie is; why cookies never carry sensitive data directly (foreshadowing sessions); the request-vs-response asymmetry between setting and reading a cookie.
- **Researched:** a caveat that the 20/200 cookie-count limits are browser conventions, not a fixed web standard, and the general "cookies are for small, non-critical data" principle that follows from that; the `.get()`-over-`[key]` best practice for reading `request.COOKIES` safely.

Double-check: the browser-limit numbers (20/200) are worth treating as "the video's rule of thumb," not gospel — the durable lesson is *why* cookies are limited-purpose, not the exact count.
