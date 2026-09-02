# Lecture 21 — Dynamic URLs, and introducing state management

Source: `transcripts/Django21.txt`
Covers: named URL patterns and dynamic (runtime) URL segments — the mechanism behind "click a link, get a different record" — plus the start of a new topic that runs through the next several lectures: state management (why HTTP forgets everything, and the three techniques Django gives you to make it remember).

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official Django docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. Named URLs **[From video]**

Every `path()` in `urls.py` can carry an optional `name=` argument. That name becomes a stable label for the URL that you can reference elsewhere in your code (templates, redirects) instead of hardcoding the literal path string.

```python
# db_app/urls.py
from django.urls import path
from db_app import views

urlpatterns = [
    path('home/', views.home, name='home'),
]
```

```html
<!-- home.html -->
<a href="{% url 'home' %}">Back to home</a>
```

`{% url 'home' %}` looks up the URL pattern named `'home'` and inserts its actual path (`/db_app/home/`, or wherever it's mounted) into the `href`. The template never hardcodes the address itself.

> **[Gap-filled] — why this indirection matters.** If you hardcode `href="/db_app/home/"` in twenty templates and later move that view to a different path, all twenty links silently break. If every link instead uses `{% url 'home' %}`, changing the `path()` in `urls.py` is the *only* place you need to edit — every template that references the name picks up the new address automatically. This is the same reasoning covered when named URLs were first mentioned back in the URL-routing topic; this lecture is where it's actually put to use.

## 2. Dynamic URL segments **[From video]**

A "dynamic URL" is a URL that carries a value as part of the path itself — the same view responds to `/emp/1/`, `/emp/2/`, `/emp/6/`, etc., and receives whichever number was actually requested as a normal Python argument.

```python
# db_app/urls.py
from django.urls import path
from db_app import views

urlpatterns = [
    path('emp/<int:emp_id>/', views.emp_details, name='details'),
    path('home/', views.home, name='home'),
]
```

```python
# db_app/views.py
from django.shortcuts import render

def emp_details(request, emp_id):
    m = {'id': emp_id}
    return render(request, 'display.html', m)
```

```html
<!-- display.html -->
<h1>Display Employee Template</h1>
<h1>ID: {{ id }}</h1>
```

Visiting `/db_app/emp/1/` renders "ID: 1"; visiting `/db_app/emp/6/` renders "ID: 6" — same view, same template, different value, because `<int:emp_id>` captures whatever appears in that segment of the path and hands it to the view as the `emp_id` parameter.

`<int:emp_id>` is a **path converter**: `int` says "only match digits, and convert them to a Python `int` before calling the view"; `emp_id` is the parameter name the view function must accept. The video runs into exactly this mismatch as a live bug — a view was written to accept a parameter with a different name than the one used in the URL pattern, producing `emp_view() got an unexpected keyword argument`. The names in `<converter:name>` and the view's parameter list have to match exactly.

> **[Researched] — the full set of built-in path converters.** Per the [Django docs](https://docs.djangoproject.com/en/stable/topics/http/urls/#path-converters), Django ships five built-in converters:
>
> | Converter | Matches | Example |
> |---|---|---|
> | `str` | Any non-empty string, excluding `/` (the default if you omit a converter) | `<str:username>` |
> | `int` | One or more digits, converted to a Python `int` | `<int:emp_id>` |
> | `slug` | Letters, numbers, hyphens, underscores (a "slug" — URL-friendly text) | `<slug:article-slug>` |
> | `uuid` | A formatted UUID string, converted to a `UUID` object | `<uuid:id>` |
> | `path` | Any non-empty string, *including* `/` (matches multiple path segments) | `<path:subpath>` |
>
> Using `int` instead of the default `str` isn't just a style choice — it makes Django itself reject a request like `/emp/abc/` with a 404 before your view code ever runs, instead of your view having to defensively check "is this actually a number?" by hand.

## 3. Linking to a dynamic URL from a template **[From video]**

`{% url %}` also accepts arguments, so a template can build a link to a dynamic URL without ever writing the literal path:

```html
<!-- home.html -->
<a href="{% url 'details' 1 %}">Employee 1</a><br>
<a href="{% url 'details' 2 %}">Employee 2</a><br>
<a href="{% url 'details' 3 %}">Employee 3</a><br>
<a href="{% url 'details' 4 %}">Employee 4</a><br>
```

Each `{% url 'details' N %}` resolves to `/db_app/emp/N/` — the `1`, `2`, `3`, `4` after the URL name are positional arguments that get substituted into the `<int:emp_id>` slot of the `details` pattern.

The video builds this into a small working example: a `home` view renders `home.html` with four "Employee N" links; clicking one calls `emp_details` with that employee's ID, looks the ID up against a small in-code lookup of ID → name, and renders the matching name on `display.html`, which itself links `{% url 'home' %}` back to the list.

```python
# db_app/views.py
def emp_details(request, emp_id):
    names = {1: 'Sonu', 2: 'Sai', 3: 'Manoj', 4: 'Durga'}
    m = {'id': emp_id, 'name': names.get(emp_id, 'Unknown')}
    return render(request, 'display.html', m)

def home(request):
    return render(request, 'home.html')
```

```html
<!-- display.html -->
<h1>Display Employee Template</h1>
<h1>ID: {{ id }}</h1>
<h1>Name: {{ name }}</h1>
<a href="{% url 'home' %}">Back to home</a>
```

> **[Gap-filled]** The video's audio garbles the exact employee names used on screen (they come through in the transcript as fragments like "move on," "sign," "monos"). The mechanism shown above — a dictionary keyed by ID, looked up in the view — is reconstructed cleanly from the described logic (an `if ID == 1` / `elif ID == 2` chain assigning a different name per ID); the specific names are illustrative rather than a verbatim transcript quote.

> **[Example]** A second, standalone illustration of the same mechanism — a tiny "view a blog post by number" URL:
> ```python
> # urls.py
> path('post/<int:post_id>/', views.post_detail, name='post-detail'),
> ```
> ```python
> # views.py
> posts = {1: 'Hello World', 2: 'Django Basics', 3: 'Dynamic URLs'}
>
> def post_detail(request, post_id):
>     title = posts.get(post_id, 'Post not found')
>     return render(request, 'post_detail.html', {'post_id': post_id, 'title': title})
> ```
> `{% url 'post-detail' 2 %}` in any template now links straight to "Django Basics" — the pattern generalizes to any "list page links to individual detail pages by ID" use case, which is one of the most common URL shapes in real Django apps (blog posts, product pages, user profiles).

> **Industry best practice:** Always give a URL pattern a `name=` if anything will ever link to it — which in practice is almost every pattern except the admin's own catch-all. Untitled, unnamed URL patterns that get hardcoded into multiple templates are a recurring source of "I changed the URL and now half the site 404s" bugs.

## 4. Introducing state management **[From video]**

Django, like every web framework, runs on **client–server architecture**: the browser (client) sends a request, the server processes it and sends back a response, and — critically — once that response is delivered, **the server does not remember anything about that page or that user**. The next request, even from the same browser a second later, is treated as a completely new, unrelated request.

This is because HTTP (HyperText Transfer Protocol — the language the browser and server use to talk to each other) is a **stateless protocol**: it has no built-in memory of previous requests. `state` here means "held-onto information about a user or a page, carried across multiple requests" — and by default, there is none.

> **[Gap-filled] — plain-language definition.** Think of state management as the answer to: *when you click "Buy Now" on Flipkart, add a delivery address, then reach the payment page, how does the payment page still know which item you're paying for?* By default, it wouldn't — each of those pages is its own separate request, and HTTP forgets everything between requests. Something extra has to carry that information forward. That "something extra" is what state management provides.

The video walks through several everyday examples to build intuition for why this matters:

- **Flipkart checkout** — you move from a product page → login → delivery address → order summary → payment. Each of those is a separate page/request, yet the site "remembers" which item you're buying all the way through. Without state management, the order summary page would have no idea what the address page decided.
- **Gmail staying logged in** — log into Gmail once, close the browser without signing out, reopen it later, and it's still logged in. Something remembered your login between browser sessions.
- **Exam results sites on result day** — the very first time you check your result, there's a long delay (the server treats it as a brand-new request and does real work). Refresh minutes later out of curiosity, and the same result appears almost instantly — the server recognized it had already done this work and served the saved answer instead of redoing it.
- **Live cricket score sites** — the page seems to "auto-update" the score without you manually reloading constantly; behind the scenes this relies on the same kind of state-management mechanism (here, specifically caching) rather than hammering the server with fresh requests each time.
- **Online banking session timeouts** — take too long between steps of a money transfer and you get "Your session has expired." That message is state management (specifically, a *session*) actively working — deliberately forgetting your progress after a timeout, for security.

## 5. The three state management techniques **[From video]**

Django (and, per the video, every mainstream web framework — ASP.NET, JSP, PHP, etc., since this is an HTTP-level problem, not a Django-specific one) offers three techniques for carrying information across otherwise-stateless requests:

| Technique | Where it lives | Category |
|---|---|---|
| **Cookies** | Stored in the browser, on the client's machine | Client-side |
| **Sessions** | Stored on the server, with only an ID kept on the client | Server-side |
| **Cache** | Temporary storage on the server for data already computed once | Server-side |

The next several lectures cover each in turn: cookies first, then sessions, then cache.

> **[Gap-filled] — client-side vs. server-side, at a glance.** "Client-side" storage means the data itself sits in the visitor's own browser — convenient, but visible and editable by that visitor, and not something you'd trust with anything sensitive. "Server-side" storage means the actual data stays under your control on the server; the browser only ever holds a small reference (an ID) pointing at it. This distinction is why cookies are used for small, low-stakes convenience data, while sessions are the default choice for things like "is this user logged in."

The video previews where cookies physically live in the browser — in Chrome, via the three-dot menu → **Settings** → **Privacy and security** → **Cookies and other site data** → **See all cookies and site data**, filtered to the current site's domain. The next lecture uses this same location constantly to verify cookies are actually being created.

---

## Wrap-up

- **From video:** named URL patterns (`name=` in `path()` and `{% url 'name' %}` in templates); dynamic URL segments via path converters (`<int:emp_id>`); passing arguments through `{% url 'name' arg %}`; the state-management problem (HTTP is stateless), illustrated with Flipkart/Gmail/exam-results/cricket-score/banking-session examples; the three techniques (cookies, session, cache) and their client-side/server-side split; where cookies live in Chrome's settings.
- **Gap-filled:** why named URLs avoid hardcoding brittle path strings; a note on the video's garbled employee-name example; the client-side vs. server-side distinction spelled out plainly.
- **Researched:** the full table of Django's five built-in path converters (`str`, `int`, `slug`, `uuid`, `path`) and what each one actually validates/converts.

Double-check: if you're ever unsure which path converter to reach for on a new URL, the table above is worth bookmarking — `int` vs. `str` vs. `slug` covers the vast majority of real cases.
