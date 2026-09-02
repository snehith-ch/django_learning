# Lecture 25 — Making sessions robust, and the cache mechanism

Source: `transcripts/Django25.txt`
Covers: fixing the previous lecture's session bug, checking whether a visitor's browser even supports cookies, a session-based page counter, properly expiring/clearing sessions (`flush()`, `clear_expired()`, `request.session.modified`), storing sessions in a file instead of the database, and — the third and final state-management technique — the cache mechanism, in all three of its storage forms and all three of its scopes.

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official Django docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. Resolving last lecture's session bug **[From video]**

The video opens by tracking down why `get_session` kept showing "no name" even right after `set_session`. The root cause: a leftover, inconsistent `SESSION_COOKIE_NAME` value in `settings.py` meant the browser was still sending the *old* session cookie under the *old* name, while the code expected the new one — so the session Django read back was a different (empty) session than the one just created.

> **[Gap-filled] — the actual lesson, cleaned up from a messy live-debugging segment.** The transcript for this part is largely the presenter narrating trial and error in real time rather than explaining a stable concept, so it's not reproduced verbatim here. The generalizable takeaway: if you change a session/cookie **name** setting (`SESSION_COOKIE_NAME`, or a cookie's own key) mid-development, browsers that already hold a cookie under the *old* name won't automatically pick up the new one — you may need to clear cookies for the site in your browser (Chrome: Settings → Privacy and security → Cookies and other site data → remove the site's entries) after renaming, or you'll keep reading a stale, empty session and wonder why your data "disappeared."

## 2. Checking whether the browser supports cookies at all **[From video]**

Since sessions are entirely dependent on cookies (Lecture 23 — the session ID has nowhere to live without one), it's useful to be able to detect, server-side, whether a visitor's browser is actually accepting cookies before relying on sessions for anything important. Django provides a dedicated three-method mechanism for this:

```python
def check_session(request):
    request.session.set_test_cookie()
    return render(request, 'check.html')

def confirm_session(request):
    if request.session.test_cookie_worked():
        request.session.delete_test_cookie()
        return HttpResponse("Cookies are working properly")
    else:
        return HttpResponse("Cookies are disabled")
```

- `request.session.set_test_cookie()` sends a throwaway test cookie to the browser.
- `request.session.test_cookie_worked()` — on a **later** request — checks whether that test cookie actually came back. If the browser accepts cookies, it will have; if the browser is blocking cookies, it won't.
- `request.session.delete_test_cookie()` cleans up the test cookie once you're done checking, so it doesn't linger.

The video demonstrates both outcomes directly: with cookies enabled in Chrome, `test_cookie_worked()` returns `True`. Then, after manually blocking all cookies for the site (Chrome Settings → Privacy and security → Cookies and other site data → **Block all cookies**), the same flow instead raises a `KeyError` on `delete_test_cookie()` (because no test cookie ever arrived to delete) — confirming that cookies genuinely aren't reaching the server. Re-enabling cookies in the browser restores the working behavior.

> **[Researched] — why this needs two separate requests.** Per the [Django docs](https://docs.djangoproject.com/en/stable/ref/request-response/#django.contrib.sessions.backends.base.SessionBase.test_cookie_worked), this only works as a *set on one request, check on the next* pair — you can't call `set_test_cookie()` and `test_cookie_worked()` in the same request and get a meaningful answer, because the browser hasn't had a chance to send the cookie back yet. This mirrors the general "set on response, read from request" pattern from Lecture 22 — cookie effects are only ever observable on a subsequent request.

> **Industry best practice:** This check matters more than it might seem — a small but real fraction of real visitors browse with cookies blocked (privacy settings, corporate policies, some browser extensions). A production app that depends on sessions for anything important (login, a shopping cart) should have a plan for what happens when `test_cookie_worked()` comes back `False`, rather than silently failing in confusing ways.

## 3. A page-view counter using sessions **[From video]**

The session-based counterpart to Lecture 23's cookie-based page counter:

```python
def page_count_view(request):
    count = request.session.get('count', 0)
    new_count = count + 1
    request.session['count'] = new_count
    return render(request, 'count.html', {'count': new_count})
```

```html
<!-- count.html -->
<style> span { font-size: 200px; } </style>
<h1>Page count: <span>{{ count }}</span></h1>
```

Structurally identical logic to the cookie version, but using `request.session` instead of `request.COOKIES` — and notably, **no manual `int()` conversion is needed here**, because (unlike a cookie, which only ever stores plain text) session values keep their real Python type. `request.session.get('count', 0)` returns an actual `int` `0` on the first visit, not the string `'0'`.

> **[Gap-filled] — the type-safety difference is worth calling out explicitly.** This is a genuine, practical advantage of sessions over cookies beyond the security angle already covered: a cookie can only ever store text, so any non-string data (numbers, lists, dictionaries) has to be manually serialized and parsed back (as Lecture 23's `int(count)` had to). A Django session, because the data lives server-side and Django handles the encoding for you, can transparently store whatever Python (JSON-serializable) types you assign to it — a list, a dict, a number — without you writing conversion code by hand.

## 4. Deleting session data properly: `flush()` **[From video]**

```python
def del_session(request):
    request.session.flush()
    return render(request, 'del.html')
```

`request.session.flush()` deletes **both** the current session's data from the database **and** its cookie from the browser, and generates a fresh, different session key for any future session activity from this same browser. This is a more thorough cleanup than manually deleting individual keys (Lecture 24's `del request.session['name']`), which only removes that one piece of data and leaves the session itself (and its ID) intact.

> **[Researched] — `flush()` is what Django's own logout view uses.** Per the [Django docs](https://docs.djangoproject.com/en/stable/topics/http/sessions/#django.contrib.sessions.backends.base.SessionBase.flush), this is exactly the method `django.contrib.auth.logout()` calls internally. That's the concrete real-world use case for `flush()`: logging a user out should not just clear "are they logged in" but genuinely invalidate the whole session, so that a stolen/reused old session ID can't be replayed afterward.

## 5. Removing already-expired sessions from storage: `clear_expired()` **[From video]**

```python
request.session.clear_expired()
```

Unlike `flush()` (which acts on the *current* session), `clear_expired()` is a maintenance operation: it removes session rows from the session store (the database, by default) that have **already passed their expiration date** — essentially garbage-collecting old, dead session rows so they don't accumulate indefinitely.

> **[Researched] — this is normally run as a scheduled command, not from inside a view.** Per the [Django docs](https://docs.djangoproject.com/en/stable/topics/http/sessions/#clearing-the-session-store), the standard way to do this in a real project is the management command `python manage.py clearsessions`, typically wired up as a periodic cron job — not called manually inside a view the way the video demonstrates it here for teaching purposes. Expired session rows aren't automatically deleted just because they're expired (Django simply treats them as invalid when encountered); without occasional cleanup, the `django_session` table would grow forever.

## 6. Keeping an active session from expiring: `request.session.modified = True` **[From video]**

By default, Django only extends a session's expiration when its *data* changes (e.g. a key is set or deleted). A view that only *reads* from the session — like repeatedly calling `get_session` — does **not**, by default, push the expiration further out, even if the visitor keeps actively using the site. The video demonstrates this concretely: with `SESSION_COOKIE_AGE` set to a short 30 seconds, repeatedly refreshing a read-only `get_session` view still expires the session exactly 30 seconds after it was first *set* — refreshing doesn't buy any extra time, which is the wrong behavior for something like an actively-used banking session.

The fix:

```python
def get_session(request):
    name = request.session.get('name', 'No name set')
    request.session.modified = True   # treat this request as activity, extend expiry
    return render(request, 'get.html', {'name': name})
```

Setting `request.session.modified = True` explicitly tells Django "treat this as a change, even though no key was actually added or removed" — which causes the session's expiration to be recalculated forward from *this* request, not just the original one. With this line added, the video shows the same 30-second session now genuinely staying alive indefinitely as long as `get_session` keeps being visited at least once every 30 seconds — matching real-world "your session stays alive while you're actively using the site" behavior (and correctly expiring only once activity actually stops).

> **[Gap-filled] — why this matters for the banking-session example from Lecture 21.** This is the exact mechanism behind "keep interacting and you won't get logged out, but go idle and you will." Without `request.session.modified = True` on every request that should count as activity, a session would expire on a fixed timer from its *creation*, regardless of how actively a user was using the site in the meantime — a bad experience (and arguably a security-relevant bug, since it means "still actively logged in" and "silently expired mid-task" become indistinguishable to the user).

## 7. File-based sessions instead of the database **[From video]**

Sessions are stored in the database by default, but Django supports storing them in the filesystem instead, via `SESSION_ENGINE`:

```python
# settings.py
SESSION_ENGINE = 'django.contrib.sessions.backends.file'
SESSION_FILE_PATH = os.path.join(BASE_DIR, 'session')
```

- `SESSION_ENGINE` swaps out *how* Django stores session data — `'django.contrib.sessions.backends.file'` tells it to use plain files on disk instead of a database table.
- `SESSION_FILE_PATH` says *where*: `os.path.join(BASE_DIR, 'session')` builds an absolute path to a folder named `session` inside the project's base directory (`BASE_DIR` is a variable already defined near the top of every generated `settings.py`, pointing at the project's root folder).
- That folder (here, literally named `session`) must be created manually first — Django won't create it for you.

Once set, each new session creates a file inside that folder (named after the session key) instead of a `django_session` row — the video confirms this by watching the folder in a file explorer and seeing a new file appear the moment `set_session` runs, with content matching what was previously visible in the database table.

> **[Researched] — when file-based sessions are actually worth using.** Per the [Django docs](https://docs.djangoproject.com/en/stable/topics/http/sessions/#using-file-based-sessions), the database backend is the default and is fine for the overwhelming majority of projects. File-based sessions are a niche option, occasionally used when a project deliberately avoids hitting the database for session reads/writes (for raw performance reasons on very high-traffic sites), or in constrained environments without an easy database setup. For a typical learning project or small-to-medium production app, there's no compelling reason to switch away from the database default.

## 8. Introducing the cache mechanism **[From video]**

**Cache** is the third and final state-management technique — like sessions, it's **server-side**. Where a session's purpose is remembering *a specific user's* data, cache's purpose is different: **avoiding repeated, expensive work by temporarily saving the *result* of a request**, so that repeat requests (from anyone, not just the original visitor) can be served instantly from the saved copy instead of being recomputed from scratch.

The video's examples for building intuition:

- **University exam-result sites** (also used in Lecture 21 for state management generally): the very first request for a given result is slow (the server does real work); every later request for that *same* result — even from a different curious refresh by the same student — is near-instant, because the computed page was cached rather than regenerated.
- **A phone/computer's cache folder**: recently opened images, audio, video, and documents are kept in a cache folder so that reopening them shortly after is instant, instead of the operating system re-reading and re-processing the file from scratch each time. Clearing that cache folder (something some users do, thinking it's "junk") means the *next* open of each file goes back to being slow, since the system has to redo that work.

> **[Gap-filled] — cache vs. session, side by side.** It's easy to conflate the two since both are server-side. The distinguishing question: is the data *specific to one visitor* (their name, their cart, their login state) — that's a **session**. Or is it *the same for anyone who asks* (a rendered page, a computed report) and just expensive to redo — that's a **cache**. A session remembers *who you are*; a cache remembers *what the answer was*, regardless of who's asking.

## 9. Three storage options for cache data **[From video]**

Exactly parallel to the cookie/session storage question, cache data itself has to live somewhere. Django supports three backends (configured the same way regardless of which cache *scope*, below, you're using):

| Backend | Where cache data lives | `BACKEND` setting value |
|---|---|---|
| **Database caching** | A dedicated database table | `django.core.cache.backends.db.DatabaseCache` |
| **File system caching** | Plain files in a folder you choose | `django.core.cache.backends.filebased.FileBasedCache` |
| **Local-memory caching** | The server process's own RAM | `django.core.cache.backends.locmem.LocMemCache` |

## 10. Three scopes of caching **[From video]**

Independently of *where* cache data is stored, Django lets you choose *how much* of a page (or site) gets cached:

| Scope | What gets cached |
|---|---|
| **Per-site cache** | The entire website — every view's response, for a set duration |
| **Per-view cache** | Only specific view(s) you explicitly mark, leaving others always freshly computed |
| **Template fragment cache** | Only a specific *part* of one template (e.g. a sidebar), leaving the rest of that same page always freshly rendered |

The video demonstrates per-site and per-view caching hands-on (each with all three storage backends, for per-site); it does not build a working template-fragment-cache example in this lecture, only naming the concept — flagged below as a gap.

> **[Gap-filled] — template fragment cache wasn't demonstrated.** The video names this as the third scope and gives a plain-language description ("some part of the page should be cached, the rest shouldn't") but does not show working code for it in this transcript. Per the [Django docs](https://docs.djangoproject.com/en/stable/topics/cache/#template-fragment-caching), the mechanism is the `{% load cache %}` tag plus `{% cache %}`/`{% endcache %}` around the portion of a template you want cached:
> ```html
> {% load cache %}
> {% cache 60 sidebar_fragment %}
>   <!-- this part of the page is cached for 60 seconds -->
>   {{ some_expensive_sidebar_content }}
> {% endcache %}
> ```
> `60` is the cache duration in seconds, and `sidebar_fragment` is an arbitrary name identifying this cached fragment (needed in case a template has more than one cached region). This is included here as a researched addition since it's directly relevant to a concept the video introduced but didn't finish demonstrating.

## 11. Per-site cache, with database caching **[From video]**

Per-site database caching needs settings changes plus a real database table for the cache data, plus two pieces of middleware in a specific order.

```python
# settings.py
CACHE_MIDDLEWARE_SECONDS = 60   # how long cached pages stay valid (default: 300 = 5 minutes)

CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.db.DatabaseCache',
        'LOCATION': 'cache_app_cache',   # the database table name to use
    }
}

MIDDLEWARE = [
    # ...
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.cache.UpdateCacheMiddleware',    # must come right after SessionMiddleware
    # ...
    'django.middleware.common.CommonMiddleware',
    'django.middleware.cache.FetchFromCacheMiddleware',  # must come right after CommonMiddleware
]
```

```bash
python manage.py createcachetable
```

- `CACHE_MIDDLEWARE_SECONDS` sets the cache duration, in seconds, the same way `max_age`/`SESSION_COOKIE_AGE` did for cookies/sessions — the default, if this is omitted, is **300 seconds (5 minutes)**.
- `CACHES['default']['LOCATION']` for the database backend is the **table name** the cached data will be stored under. If unspecified, Django defaults it to `<app_name>_cache`.
- `python manage.py createcachetable` is a dedicated management command that creates that table — it does **not** happen through `makemigrations`/`migrate` the way session/model tables do; caching has its own setup command.
- The two middleware entries (`UpdateCacheMiddleware` and `FetchFromCacheMiddleware`) do the actual work of intercepting every request/response to check/store the cache — and **their position in the `MIDDLEWARE` list matters**: `UpdateCacheMiddleware` must be placed near the *top* of the list (immediately after `SessionMiddleware`, in the video's setup), and `FetchFromCacheMiddleware` must be placed near the *bottom* (immediately after `CommonMiddleware`).

With this in place, any view's rendered output is cached for the configured duration — the video demonstrates this by editing a template's text and refreshing repeatedly within the 60-second window (the old, pre-edit content keeps showing, proving it's being served from cache, not freshly rendered), then waiting past 60 seconds and refreshing again (the new, edited content finally appears, because the cache expired and the page was genuinely re-rendered).

> **[Researched] — why middleware order matters here specifically.** Per the [Django docs](https://docs.djangoproject.com/en/stable/topics/cache/#the-per-site-cache), `UpdateCacheMiddleware` runs during the *response* phase and needs to run *last* on the way out (so it sees the final response to cache) while running *first* on the way in relative to most other middleware — which is why Django's own documentation places it near the top of the list. `FetchFromCacheMiddleware`, conversely, needs other middleware (like `CommonMiddleware`, which can modify the request path) to have already run before it decides whether a cached response exists — hence its position near the bottom. Getting this order wrong doesn't raise an error; it silently produces caching that doesn't actually work, which is why the video stresses double-checking it carefully.

## 12. Per-site cache, with file-system caching **[From video]**

Same middleware and `CACHE_MIDDLEWARE_SECONDS` as above; only the `CACHES` block's backend and location change:

```python
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.filebased.FileBasedCache',
        'LOCATION': '/absolute/path/to/your/project/cache',   # a folder you create manually
    }
}
```

`LOCATION` here is a real folder path (the video creates a folder literally named `cache` inside the project directory, mirroring the file-based-sessions setup from earlier in this lecture) rather than a table name — Django writes one file per cached response into that folder. No `createcachetable` step is needed for this backend, since there's no database table involved.

## 13. Per-site cache, with local-memory caching **[From video]**

```python
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.locmem.LocMemCache',
        'LOCATION': '',   # not applicable — lives in server RAM, no path/table needed
    }
}
```

No visible file or database table to inspect this time — the cached data lives directly in the running server process's memory. Functionally, the caching behavior (edits not showing up until the cache duration passes) is identical to the other two backends; only where the bytes physically sit differs.

> **[Researched] — a practical tradeoff worth knowing about local-memory caching.** Per the [Django docs](https://docs.djangoproject.com/en/stable/topics/cache/#local-memory-caching), `LocMemCache` is fast (no database or file I/O) but is **private to a single server process** — in a production deployment running multiple server processes (very common for handling real traffic), each process has its own separate cache, so a cache entry created in response to one request might not be found by the next request if it happens to land on a different process. Database or file-based caching (or, in serious production setups, a dedicated cache server like Redis or Memcached) avoids this by centralizing the cache somewhere every process can share.

## 14. Per-view cache **[From video]**

Rather than caching an entire site, `@cache_page(seconds)` marks just one specific view as cached:

```python
# views.py
from django.shortcuts import render
from django.views.decorators.cache import cache_page

@cache_page(30)
def v1(request):
    return render(request, 'cache.html')

def v2(request):
    return render(request, 'cache_one.html')   # not decorated — always freshly rendered
```

- `@cache_page(30)` is a **decorator** (a function that wraps another function to add behavior — here, imported from `django.views.decorators.cache`) placed directly above the view it should apply to. `30` is the cache duration in seconds for *this view only*.
- `v1` (decorated) behaves exactly like the per-site cache examples: edits to `cache.html` don't show up until 30 seconds pass.
- `v2` (not decorated) always shows live, freshly-rendered content on every request — proving the caching is scoped to only the marked view.

> **Critical setup difference from per-site caching:** per-view caching **must not** have `UpdateCacheMiddleware` / `FetchFromCacheMiddleware` in `MIDDLEWARE` — those two middleware entries implement *per-site* caching specifically, and leaving them in place while trying to use `@cache_page` produces confusing, incorrect results. The `CACHES` dictionary (with whichever backend — database/file/local-memory) is still needed, since `@cache_page` still needs somewhere to store its cached data; only the two middleware lines are per-site-only and must be removed for per-view caching to work correctly.

> **[Example]** A standalone illustration of when per-view caching earns its keep over per-site caching: an app with one expensive, rarely-changing report page alongside several cheap, always-fresh pages.
> ```python
> @cache_page(600)   # 10 minutes — this report is expensive to generate and rarely changes
> def sales_report(request):
>     # ...expensive database aggregation...
>     return render(request, 'sales_report.html', {'totals': totals})
>
> def live_notifications(request):
>     # cheap, and must always be fresh — never cached
>     return render(request, 'notifications.html', {'items': get_latest(request.user)})
> ```
> Caching the entire site here would be wrong — it would make `live_notifications` show stale data. Per-view caching lets the expensive, slow-changing page get the performance benefit without affecting the page that specifically needs to always be current.

---

## Wrap-up

- **From video:** fixing the prior lecture's session-name bug; browser-cookie-support checking (`set_test_cookie`, `test_cookie_worked`, `delete_test_cookie`); a session-based page counter (and its type-safety advantage over the cookie version); `flush()` vs. `clear_expired()`; `request.session.modified = True` to extend an active session's expiry on read-only requests; file-based sessions (`SESSION_ENGINE`, `SESSION_FILE_PATH`); the cache mechanism's purpose and its distinction from sessions; three cache storage backends (database, file system, local memory); three cache scopes (per-site, per-view, template fragment); full per-site cache setup for all three backends (`CACHE_MIDDLEWARE_SECONDS`, `CACHES`, `createcachetable`, and the required `UpdateCacheMiddleware`/`FetchFromCacheMiddleware` ordering); per-view caching with `@cache_page`, including the requirement to *remove* the per-site middleware for it to work.
- **Gap-filled:** the messy live-debugging segment condensed into its general lesson (renaming a session/cookie setting can leave stale cookies behind); a cache-vs-session comparison; why `request.session.modified` matters for realistic "stay logged in while active" behavior.
- **Researched:** why `test_cookie_worked()` needs two separate requests; `flush()`'s relationship to Django's built-in logout; `clearsessions` as the real-world way to run `clear_expired()`-style cleanup on a schedule; when file-based sessions are actually worth choosing; the `{% cache %}` template tag for fragment caching (named but not demonstrated in the video); why cache middleware order matters mechanically; the multi-process caveat with local-memory caching.

Double-check: template fragment caching was named by the video but never shown working — the `{% cache %}` tag example above is a researched fill-in for that gap, not a transcript-verified walkthrough, so treat it as a starting point to test yourself rather than a guaranteed-correct recipe.
