# Lecture 26 — Template fragment cache, and introducing the Django authentication system

Source: `transcripts/Django26.txt`
Covers: the third and final cache scope (finishing the state-management arc from Lectures 21–25), then a hard pivot into a new major topic that will run for several lectures — Django's built-in authentication system, starting with the authentication-vs-authorization distinction and a hands-on tour of the admin panel's user/permission machinery.

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official Django docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. Template fragment cache **[From video]**

Recall from Lecture 25 that Django's cache mechanism has three scopes: per-site, per-view, and **template fragment** — caching only *part* of one page's HTML, leaving the rest to render fresh on every request. This lecture finally demonstrates it.

```html
<!-- cache2.html -->
{% load cache %}
<h1>Welcome</h1>

{% cache 40 mycache %}
  <h1>Hyderabad</h1>
  <h1>Myself</h1>
{% endcache %}
```

- `{% load cache %}` at the top of the template makes the `{% cache %}` tag available — without it, Django has no way to recognize this template uses fragment caching.
- `{% cache 40 mycache %}...{% endcache %}` wraps the portion of the HTML you want cached. `40` is the duration in seconds; `mycache` is an arbitrary name identifying this cached fragment (needed so Django can tell multiple cached fragments in the same project apart).
- Everything **inside** the tag is cached for 40 seconds; everything **outside** it (like the `<h1>Welcome</h1>` above) re-renders on every single request, completely unaffected by the cache.

The video confirms this by editing the template live: changes made to the `<h1>Welcome</h1>` line (outside the cache block) show up on the very next refresh; changes made to the `Hyderabad`/`Myself` lines (inside the cache block) are invisible until the 40-second window expires, at which point they suddenly appear all at once.

```python
# views.py
def v3(request):
    return render(request, 'cache2.html')
```

```python
# urls.py
path('v3/', views.v3),
```

> **[Gap-filled] — this closes a gap flagged in Lecture 25.** Lecture 25's notes had to reconstruct `{% cache %}` usage as a researched addition because the video named the concept without demonstrating it. This lecture is that demonstration, confirming the researched syntax from Lecture 25 was correct: `{% load cache %}` once at the top, then `{% cache <seconds> <name> %}...{% endcache %}` around the region to cache.

> **Industry best practice:** Reach for template fragment caching specifically when a page has one expensive-to-render, slow-changing section (e.g. a sidebar of trending items) sitting alongside content that must always be current (e.g. a live comment count) — it's the middle ground between "cache the whole page" (Lecture 25's per-site cache, too coarse when only part of the page is cacheable) and "cache nothing" (wasteful if that one section really is expensive).

## 2. Recap: why cache exists at all **[From video]**

The video re-states the core motivation before moving on: cache is temporary, server-side storage that avoids repeating expensive work. The first request for a given piece of data is processed normally (hits the database, does the real computation); the *result* is kept in cache for a set duration (300 seconds/5 minutes by default — Lecture 25); every request for that same data within that window is served straight from cache, without touching the database or re-running the computation, which reduces load on the server and gives a faster response to the visitor.

## 3. Authentication vs. authorization **[From video]**

With state management (cookies/sessions/cache) now complete, the video moves to a new topic that will span several lectures: the **Django authentication system**.

- **Authentication** — the process of checking whether a user's credentials (username + password) are correct. If they are, that person becomes an **authenticated** user.
- **Authorization** — the process of checking whether an *already-authenticated* user has permission to access specific content or perform specific actions.

> **[Gap-filled] — the distinction in one sentence.** Authentication answers "who are you?" (do your username and password actually match a real account?). Authorization answers "what are you allowed to do?" (now that we know who you are, can you view this, edit that, delete the other thing?). **Every authorized user is authenticated, but not every authenticated user is authorized** for everything — a logged-in regular user is authenticated, but isn't authorized to, say, delete other users' accounts the way an admin is. Lecture 26's own admin-panel walkthrough (below) makes this concrete: everyone with a login can authenticate, but only users granted specific permissions are authorized for specific actions.

## 4. The built-in apps behind Django's auth system **[From video]**

Django ships authentication support as two built-in apps, already present in `INSTALLED_APPS` in a freshly generated project (confirmed back in Lecture 23's researched note):

- **`django.contrib.auth`** — the authentication system itself (users, permissions, groups, login/logout, forms).
- **`django.contrib.contenttypes`** — used internally *by* `django.contrib.auth` to keep track of which models/modules are installed in the project's database.

## 5. Where Django's auth system physically lives **[From video]**

The video locates the actual source files behind `django.contrib.auth` on the presenter's machine, to demystify "where does all this built-in behavior actually come from":

```
C:\Users\<username>\AppData\Local\Programs\Python\Python310\Lib\site-packages\django\contrib\auth\
```

- `AppData` is a normally **hidden** folder on Windows — visible only after enabling "Hidden items" in File Explorer's View menu.
- Inside `django/contrib/auth/`, the video points out `forms.py` (the built-in authentication-related form classes — covered in depth next lecture), `models.py` (built-in models like `User`), and `views.py` (built-in views like login/logout) — the actual Python source code Django runs whenever any project uses `django.contrib.auth`.

> **[Researched] — this file-hunting is optional, but the underlying idea generalizes.** Per the [Django docs](https://docs.djangoproject.com/en/stable/ref/contrib/auth/), everything `django.contrib.auth` provides — models, forms, views, middleware — is genuinely just ordinary Python/Django code, installed as a package the same way any third-party library is. Nothing about it is magic; it's readable, and (as this lecture shows) literally sitting on disk in `site-packages` like any other installed package. On macOS/Linux, the equivalent path is typically inside your virtual environment's `lib/pythonX.Y/site-packages/django/contrib/auth/` rather than under `AppData`.

## 6. Admin-panel authentication and authorization, hands-on **[From video]**

Before writing any authentication code of their own, the video spends the rest of this lecture exploring how authentication/authorization already work in the **admin panel** the course has been using since Lecture 8 — because the same underlying mechanism (the `auth_user` table and its permission flags) is what custom authentication code will build on top of next lecture.

### The `auth_user` table

Opening the project's database in DB Browser for SQLite and inspecting the `auth_user` table (Django's built-in user table, created automatically the moment `django.contrib.auth` is installed and migrated) shows columns including:

| Column | Meaning |
|---|---|
| `username` | The login name |
| `password` | Stored **encrypted/hashed**, never as plain text |
| `first_name`, `last_name`, `email` | Optional profile fields |
| `is_superuser` | `1` if this user has *every* permission automatically; `0` otherwise |
| `is_staff` | `1` if this user is even allowed to log into the `/admin/` panel at all; `0` otherwise |
| `is_active` | `1` if the account is enabled; `0` would mean it's disabled |
| `last_login`, `date_joined` | Timestamps |

> **[Gap-filled] — why `password` looking encrypted is exactly right.** This directly continues the "cookies store data in plain text; sessions and this user table don't" theme from earlier lectures. Django never stores a raw password anywhere — what's in that column is a one-way hash (per the [Django docs](https://docs.djangoproject.com/en/stable/topics/auth/passwords/), PBKDF2 by default), so even someone with direct database access can't read out an actual password.

### `is_staff` vs. `is_superuser` — two different gates

The video demonstrates, by creating a second user through the admin panel and trying to log that user in, that these two flags control genuinely different things:

- A user with `is_staff = 0` **cannot log into `/admin/` at all**, even with a completely correct username and password — attempting it produces "Please enter the correct username and password for a staff account." Being staff is the minimum requirement just to *open the door*.
- A user with `is_staff = 1` but `is_superuser = 0` **can log in**, but sees "You don't have permission to view or edit anything" — being staff gets you in the door, but grants **no permissions** by itself.
- A user with `is_superuser = 1` has **every permission automatically** and needs nothing else granted.

### Granting specific permissions

A superuser can grant a staff (non-superuser) user specific, individual permissions — e.g. "can view user," "can add user," "can change user," "can delete user" — through that user's admin edit page. The video walks through this concretely: a staff user granted only "can view user" can see the user list and open individual records, but has no edit controls anywhere; granted "can add/change/delete user" as well, the same account can then create, modify, and remove other users.

> **[Gap-filled] — this is authorization, made visible.** This whole walkthrough is Section 3's authentication/authorization distinction turned into something you can watch happen: every one of these accounts *authenticates* successfully (correct username + password), but each one is *authorized* for a different, deliberately different set of actions depending on `is_staff`, `is_superuser`, and the individual permission checkboxes.

> **Industry best practice:** Follow the **principle of least privilege** — grant a user only the specific permissions their role actually requires, not superuser status as a default convenience. The video's own example (a staff member granted only "view," not "add/change/delete") is exactly this pattern in practice: broader access should be a deliberate, specific grant, not the default.

## 7. What's coming next **[From video]**

The video previews the arc for the next several lectures: building real sign-up, login, logout, profile, and password-change/reset forms **using Django's built-in authentication form classes and views** rather than writing that logic from scratch — the same `django.contrib.auth` machinery just explored in the admin panel, now wired into a custom app.

---

## Wrap-up

- **From video:** template fragment caching (`{% load cache %}`, `{% cache seconds name %}...{% endcache %}`), confirming which page regions are/aren't affected by the cache window; a recap of why cache exists; the authentication-vs-authorization distinction; the two built-in apps (`django.contrib.auth`, `django.contrib.contenttypes`) and where their source files physically live; a full hands-on tour of `auth_user`, `is_staff` vs. `is_superuser`, and granting individual permissions through the admin panel.
- **Gap-filled:** a one-sentence framing of authentication vs. authorization; why the `password` column being unreadable is consistent with earlier state-management lessons; framing the admin-panel walkthrough explicitly as "authorization made visible."
- **Researched:** confirmation that `django.contrib.auth`'s built-in behavior is ordinary, readable Python code (with a note on the equivalent macOS/Linux path, since the video only shows Windows); the password-hashing mechanism behind why `auth_user.password` looks encrypted.

Double-check: the `{% cache %}` tag syntax researched (as a gap-fill) back in Lecture 25 is confirmed correct by this lecture's actual demonstration — worth a quick cross-check against Lecture 25's notes if you want to see the "researched, then confirmed" arc across two lectures.
