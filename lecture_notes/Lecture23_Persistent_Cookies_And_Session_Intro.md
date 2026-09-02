# Lecture 23 — Persistent cookies, a cookie-based page counter, and introducing sessions

Source: `transcripts/Django23.txt`
Covers: giving a cookie an actual expiration date (`max_age` and `expires`), what happens when you re-set a cookie's value vs. its key, a small "page view counter" built entirely with cookies, and the move to the second state-management technique — sessions.

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official Django docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. Making a cookie persistent with `max_age` **[From video]**

Every cookie set so far (Lecture 22) was an *in-memory* cookie — no expiration, gone the moment the browser window closes. Adding `max_age` (in seconds) turns it into a **persistent cookie**: it survives closing the browser and only disappears once that many seconds have actually elapsed.

```python
def set_cookie(request):
    response = HttpResponse("Cookie is set")
    response.set_cookie('name', 'Durga', max_age=60)   # persists for 60 seconds
    return response
```

Setting `max_age=60` and checking Chrome's cookie inspector shows a real expiration timestamp (roughly "now + 60 seconds") instead of "When the browsing session ends." Closing and reopening the browser within that minute still shows the cookie present; checking again after the minute has passed shows it gone.

## 2. Making a cookie persistent with `expires` **[From video]**

`max_age` is a relative duration; `expires` sets an absolute date/time instead, using Python's `datetime` module:

```python
from datetime import datetime, timedelta

def set_cookie(request):
    response = HttpResponse("Cookie is set")
    response.set_cookie(
        'name', 'Durga',
        expires=datetime.utcnow() + timedelta(days=3)
    )
    return response
```

- `datetime.utcnow()` gives the current date/time in **UTC** (Coordinated Universal Time — the time-zone-independent reference time computers use internally, so that "3 days from now" means the same instant no matter what time zone the server happens to be running in).
- `timedelta(days=3)` is a Python object representing "a span of 3 days" — added to a `datetime`, it produces a new `datetime` exactly 3 days later. `timedelta` also accepts `hours=`, `minutes=`, `seconds=`, etc., for finer control (the video demonstrates swapping `days=3` for `hours=3` and getting a 3-hour-lifetime cookie instead).
- The resulting `expires` value is that computed future date/time — the cookie survives until then, browser-close or not.

> **[Researched] — `max_age` vs. `expires`, and which to prefer.** Per the [Django docs](https://docs.djangoproject.com/en/stable/ref/request-response/#django.http.HttpResponse.set_cookie), `set_cookie()` accepts both, and if both are given `max_age` takes precedence. In modern code, `max_age` (a plain integer number of seconds) is generally simpler and less error-prone than building a `datetime` + `timedelta` by hand for `expires` — reach for `expires` only when you specifically need an absolute calendar date/time rather than a relative duration.

## 3. Overriding a cookie's value vs. creating a new cookie **[From video]**

This is a subtle but important distinction the video verifies by direct experiment:

- **Changing the *value*** passed to `set_cookie()` while keeping the same **key** → the existing cookie is **overwritten in place**. Re-running `set_cookie('name', 'Durga', ...)` and then `set_cookie('name', 'Sonu', ...)` results in exactly one cookie named `name`, now holding `Sonu`.
- **Changing the *key*** → a **brand-new, separate cookie** is created alongside the old one. `set_cookie('nickname', 'Sonu', ...)` after `set_cookie('name', 'Durga', ...)` leaves two cookies in the browser, `name` and `nickname`, each independent.

> **[Gap-filled] — why this matters in practice.** This is really just "a cookie is identified by its key, like a dictionary key" — but it's worth stating explicitly because it explains a common point of confusion: if a cookie "isn't updating" the way you expect, check whether your code is accidentally using a slightly different key string each time (a typo, a different variable) rather than genuinely reusing the same one.

## 4. A page-view counter using cookies **[Example — from video, with cleanup]**

The video's practical "why would I actually use a cookie" example: a page that counts how many times *this browser* has visited it, using nothing but a cookie to remember the running total across otherwise-stateless requests.

```python
def count_view(request):
    count = request.COOKIES.get('count')
    if count:
        new_count = int(count) + 1
    else:
        new_count = 1

    response = render(request, 'count.html', {'count': new_count})
    response.set_cookie('count', new_count)
    return response
```

```html
<!-- count.html -->
<h1>Page count: {{ count }}</h1>
```

- First visit: no `count` cookie exists yet, so `new_count` starts at `1`.
- Every later visit: the previous count is read back out of the cookie, incremented by one, shown on the page, and written back into the cookie for next time.
- Without the cookie, every refresh would show `1` forever — HTTP's statelessness means the server has no memory of the previous request on its own; the cookie is what carries the running total across requests.

> **[Gap-filled] — a `request.COOKIES.get('count')` subtlety worth flagging.** Cookie values are always stored and read back as **strings** (recall: "plain text format" from Lecture 22) — that's why the code does `int(count) + 1` rather than `count + 1`. Forgetting the `int()` conversion here is a realistic bug: `'3' + 1` would raise a `TypeError` in Python, since you can't add an `int` to a `str` directly.

## 5. Cookie vs. session — why a second technique is needed **[From video]**

Cookies solve "remember something across requests," but their two weaknesses (no security, since it's plain text in the browser; and a small size cap) make them a poor fit for anything sensitive or substantial. That's the motivation for **sessions**, the server-side counterpart:

| | Cookie | Session |
|---|---|---|
| Category | Client-side | Server-side |
| Where the actual data lives | In the browser | On the server (in the database, by default) |
| What the browser holds | The data itself | Only a **session ID** — a reference, not the data |
| Security of the data | None (plain text, readable/editable by anyone with browser access) | Data stays server-side, out of the browser entirely |

> **Cookies contain only the session ID — never the session data itself.** This is the single most important fact the video repeats about how sessions and cookies relate to each other: a session is not an *alternative* to cookies, it's *built on top of* a cookie. The cookie's only job, once sessions are in play, is to carry a small, meaningless-looking ID; the real data that ID points to lives entirely on the server.

## 6. How a session works, step by step **[From video]**

1. The client sends a request to the server.
2. If the server wants to remember something about this client for later, it creates a **session object** and stores the relevant data inside it, server-side.
3. That session object is automatically given a unique identifier: the **session ID**.
4. The server's response includes that session ID.
5. The browser stores the session ID **in a cookie** (this is the "sessions are cookie-based" point above).
6. On every later request, the browser automatically sends that cookie back; the server reads the session ID out of it, looks up the matching session object, and instantly has all the data it stored earlier — without the client having to resend anything, and without that data ever having been exposed to the browser.

> **[Gap-filled] — a mental model for the ID vs. data distinction.** Picture a cloakroom at an event: you hand over your coat (the *data*) and get a numbered ticket (the *session ID*) in return. You only carry the ticket around — the coat itself stays safely behind the counter. Show the ticket later and you get the coat back. If someone steals your ticket, they can only claim the coat by presenting it — they never had physical access to the coat itself just by seeing you walk around. That's the security improvement sessions provide over storing the actual data in a cookie.

## 7. Where session data actually lives, and why migrations matter **[From video]**

**By default, Django stores session data in the database** — specifically, in a table Django creates for this purpose. Because this is a real database table, it has to go through the same `makemigrations` / `migrate` cycle as any model:

```bash
python manage.py makemigrations
python manage.py migrate
```

The table this creates is `django_session` — one of the built-in tables Django ships migrations for automatically (you don't write a model for it yourself; it comes from `django.contrib.sessions`, one of the apps already listed in `INSTALLED_APPS` by default in a new project). Each row holds a `session_key`, an `expire_date`, and an encrypted blob of `session_data` — the video confirms this by opening the project's SQLite database directly in DB Browser for SQLite and inspecting the `django_session` table's contents after setting a session in the browser.

> **[Researched] — `django.contrib.sessions` and the session middleware.** Per the [Django docs](https://docs.djangoproject.com/en/stable/topics/http/sessions/), session support is itself implemented as an installed app (`django.contrib.sessions`) plus a piece of middleware (`django.contrib.sessions.middleware.SessionMiddleware`) — both present by default in `INSTALLED_APPS` / `MIDDLEWARE` in a freshly generated `settings.py`. That's *why* `request.session` is simply available in every view without any extra setup in this course: the scaffolding was already in place from `startproject`, and the only missing piece was running `migrate` to actually create the storage table.

## 8. The default session lifetime **[From video]**

**A Django session, by default, lasts 14 days** (two weeks) from creation. The video confirms this by inspecting a freshly created session's cookie expiration timestamp in Chrome and noting it's exactly 14 days after the creation timestamp.

## 9. Core session methods **[From video]**

| Task | Syntax |
|---|---|
| Store a value in the session | `request.session['key'] = value` |
| Read a value from the session | `value = request.session['key']` |
| Set how long the session should last (seconds) | `request.session.set_expiry(seconds)` |
| Get the session's remaining lifetime, in seconds | `request.session.get_expiry_age()` |
| Get the exact date/time the session will expire | `request.session.get_expiry_date()` |

`request.session` behaves like a Python dictionary — you set and get values with the same `[key]` syntax used for any dict, and (as with `request.COOKIES`) `.get(key, default)` is the safer alternative to `[key]` when a key might not exist yet. The next lecture (24) puts all of these to work in a real `session_app`.

> **[Example]** A standalone illustration of `set_expiry`, distinct from the video's: a "remember me for just this shopping trip" session that intentionally expires fast.
> ```python
> def start_quick_session(request):
>     request.session['cart_id'] = 'abc123'
>     request.session.set_expiry(300)  # this session forgets everything after 5 minutes
>     return HttpResponse("Session started — expires in 5 minutes")
> ```
> Contrast this with the *default* case (no `set_expiry()` call at all) which, per the table above, would instead last the full 14 days.

---

## Wrap-up

- **From video:** `max_age` and `expires` (with `datetime.utcnow() + timedelta(...)`) for persistent cookies; overriding a cookie's value (same key) vs. creating a new one (different key); a cookie-based page-view counter; the cookie-vs-session comparison (client-side + no security vs. server-side + only an ID in the cookie); the full request→session-object→session-ID→cookie→next-request flow; sessions stored in the database by default, requiring `makemigrations`/`migrate`; the default 14-day session lifetime; the core session methods (`request.session[key]`, `set_expiry`, `get_expiry_age`, `get_expiry_date`).
- **Gap-filled:** why value-vs-key changes behave differently; the `int()` conversion subtlety in the cookie-counter example; the cloakroom analogy for session-ID-vs-session-data; an extra `set_expiry()` example.
- **Researched:** `max_age` vs. `expires` precedence and which to prefer day-to-day; the role of `django.contrib.sessions` and `SessionMiddleware` in making `request.session` available with no extra setup.

Double-check: the "cookie holds only the session ID, never the session data" point is the single most load-bearing fact from this lecture — worth re-reading if anything about sessions feels unclear later.
