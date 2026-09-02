# Lecture 24 — Sessions in practice: a session_app, and session settings

Source: `transcripts/Django24.txt`
Covers: building a working `session_app` with set/get/delete views, confirming session data actually lands in the database, and the `settings.py` options that customize session behavior (cookie age, cookie name, cookie path).

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official Django docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. Setting up the session app **[From video]**

```bash
python manage.py startapp session_app
```

```python
# settings.py
INSTALLED_APPS = [
    # ...
    'session_app',
]
```

Same registration pattern as every app before it — no session-specific setup needed here, because (as covered in Lecture 23's researched note) `django.contrib.sessions` and its middleware are already present in a default project.

## 2. Setting a session value **[From video]**

```python
# session_app/views.py
from django.shortcuts import render

def set_session(request):
    request.session['name'] = 'Durga'
    return render(request, 'set.html')
```

```html
<!-- set.html --><h1>Session was set</h1>
```

`request.session['name'] = 'Durga'` stores the value under the key `'name'` in the session. Visiting the URL for this view and then inspecting Chrome's cookies (Settings → Privacy and security → Cookies and other site data) shows exactly one cookie: named `sessionid` (Django's default session-cookie name), whose *value* is an opaque-looking token — this is the **session ID**, not the word `"Durga"`. The actual value `'Durga'` never appears in the browser at all, confirming Lecture 23's "cookie holds only the ID" point directly.

## 3. Reading a session value **[From video]**

```python
def get_session(request):
    name = request.session['name']
    return render(request, 'get.html', {'name': name})
```

```html
<!-- get.html --><h1>Session value: {{ name }}</h1>
```

Visiting `set_session` then `get_session` correctly displays `Durga`. But `request.session['name']` has the exact same fragility as `request.COOKIES['name']` did back in Lecture 22: if the session's `name` key was never set (or the cookie carrying the session ID was deleted from the browser), this raises a `KeyError` and crashes the view with a 500 error. The video hits this live by manually deleting the session cookie from Chrome's dev tools and then reloading `get_session`.

> **Industry best practice — use `.get()`, exactly as with cookies.**
> ```python
> def get_session(request):
>     name = request.session.get('name', 'No name set')
>     return render(request, 'get.html', {'name': name})
> ```
> `request.session` behaves like a dictionary (Lecture 23), so the same `.get(key, default)` safety pattern applies here that applied to `request.COOKIES`. This isn't a one-off fix for a single demo bug — it's the general rule for reading *any* dictionary-like value (session, cookies, a plain dict, `request.GET`, `request.POST`) whenever the key isn't guaranteed to exist.

## 4. Deleting a session value **[From video]**

```python
def del_session(request):
    if 'name' in request.session:
        del request.session['name']
    return render(request, 'del.html')
```

```html
<!-- del.html --><h1>Session deleted</h1>
```

The `if 'name' in request.session:` check guards against the same problem as above: attempting `del request.session['name']` when that key doesn't exist would raise a `KeyError`. Checking membership first (`in`) before deleting is the safe pattern — conceptually identical to checking a plain dict before calling `del my_dict['key']`.

## 5. Wiring up the URLs **[From video]**

```python
# session_app/urls.py
from django.urls import path
from session_app import views

urlpatterns = [
    path('set/', views.set_session),
    path('get/', views.get_session),
    path('del/', views.del_session),
]
```

```python
# project urls.py
from django.urls import path, include

urlpatterns = [
    # ...
    path('session_app/', include('session_app.urls')),
]
```

Same include-based, app-owns-its-URLs pattern used throughout the course — nothing session-specific here.

## 6. Confirming sessions land in the database **[From video]**

Because Django stores sessions in the database by default (Lecture 23), the video opens the project's `db.sqlite3` file in **DB Browser for SQLite** and inspects the `django_session` table directly, confirming:

- A row exists with a `session_key` that matches the value shown as the `sessionid` cookie in Chrome.
- The row's `session_data` column holds the actual stored value (`'Durga'`) — but **encrypted**, not as visible plain text, for security. This is the concrete difference from a cookie's plain-text storage.
- The row's `expire_date` is roughly 14 days out from creation, matching the default session lifetime from Lecture 23.

> **[Gap-filled] — why `makemigrations`/`migrate` mattered here.** If this table doesn't exist yet (a fresh project where `migrate` was never run against the built-in apps), setting a session would fail outright, because Django has nowhere to write the session row. This is the practical payoff of running `migrate` early in a new project even before you've written a single model of your own — several of Django's built-in apps (sessions among them) need their own tables created the same way.

## 7. Session settings in `settings.py` **[From video]**

Three settings customize session-cookie behavior, all placed in `settings.py`:

```python
# settings.py
SESSION_COOKIE_AGE = 300          # seconds the session cookie should live (300 = 5 minutes)
SESSION_COOKIE_NAME = 'mysession' # override the default cookie name ('sessionid')
SESSION_COOKIE_PATH = '/login'    # restrict which URL path(s) the cookie is sent on
```

| Setting | What it controls | Default if omitted |
|---|---|---|
| `SESSION_COOKIE_AGE` | How many seconds the session cookie lasts | `1209600` (14 days, in seconds) |
| `SESSION_COOKIE_NAME` | The name the session cookie is stored under in the browser | `'sessionid'` |
| `SESSION_COOKIE_PATH` | Which URL path prefix the browser will actually attach the cookie to | `'/'` (sent on every request to the site) |

After setting these and re-running the server, Chrome's cookie inspector confirms the change directly: the cookie is now named `mysession` instead of `sessionid`, its expiration is ~5 minutes out instead of 14 days, and its `Path` column shows `/login` instead of `/`.

> **[Researched] — `SESSION_COOKIE_PATH` in plain terms.** Per the [Django docs](https://docs.djangoproject.com/en/stable/ref/settings/#session-cookie-path), setting this restricts the cookie to only be sent back on requests whose URL starts with that path. It exists for advanced multi-application setups (e.g. serving several independent Django apps under the same domain, each on its own path, without their session cookies leaking across each other) — for a normal single-app project, the default `'/'` (send on every request) is what you want, and the video's own choice of `/login` here is illustrative of the setting rather than something to copy into a real project without a specific reason.

## 8. Printing session debug info to the terminal **[From video]**

```python
def get_session(request):
    name = request.session.get('name', 'No name set')
    print(request.session.get_expiry_age())   # seconds remaining
    print(request.session.get_expiry_date())  # exact date/time of expiry
    return render(request, 'get.html', {'name': name})
```

Adding `print()` statements inside a view sends that output to the **terminal window running `runserver`**, not to the browser — a quick way to inspect internal values (like the session's expiry info from Lecture 23's method table) while debugging, without building a whole page around them.

> **[Example]** The same debug-printing technique applied to a different, very common situation — checking what's actually in `request.session` at any given point, without guessing key names:
> ```python
> def debug_session(request):
>     print(dict(request.session))  # every key/value currently in this session
>     return HttpResponse("Check the terminal")
> ```
> `dict(request.session)` converts the session object into a plain Python dictionary for a quick, readable printout — genuinely useful the moment a project has more than one or two session keys and you've lost track of what's actually stored.

> **Industry best practice:** Never leave debug `print()` statements in views deployed to production — they clutter server logs and can leak information. Use Python's built-in `logging` module (with an appropriate log level, e.g. `logger.debug(...)`) for anything meant to persist past local debugging.

---

## Wrap-up

- **From video:** building `session_app` with `set_session`/`get_session`/`del_session` views; `request.session[key] = value` to set, `request.session[key]` to get, `if key in request.session:` before `del request.session[key]`; confirming session storage in the `django_session` database table (encrypted `session_data`, 14-day `expire_date`); `SESSION_COOKIE_AGE`, `SESSION_COOKIE_NAME`, `SESSION_COOKIE_PATH` settings; printing session debug info to the terminal with `print()`.
- **Gap-filled:** why `migrate` is a prerequisite for sessions to work at all; a general note on `print()`-based view debugging.
- **Researched:** the practical meaning and intended use case of `SESSION_COOKIE_PATH`; the production best practice of `logging` over stray `print()` calls.

Double-check: this lecture's live debugging session (chasing why a session value wasn't showing up) carries over unresolved into the next lecture — Lecture 25 opens by tracking down and fixing that exact bug, so if something here felt incomplete, that's expected and gets closed out next.
