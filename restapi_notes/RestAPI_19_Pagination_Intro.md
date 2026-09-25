# REST API Session 19 — Pagination in DRF: PageNumberPagination

Source: `transcripts/restapi/REST API-19.txt`
Covers: why large API responses need to be split into pages, the three pagination styles Django REST Framework ships with (`PageNumberPagination`, `LimitOffsetPagination`, `CursorPagination`), and a full build-along of the first style — `PageNumberPagination` — including global settings, per-view settings, a custom pagination class, and the settings that let a client control the page parameter name, the page size, and the maximum page size. The video explicitly works alongside the official DRF documentation page for pagination throughout. `LimitOffsetPagination` and `CursorPagination` are announced as "next session" material (REST API Session 20) and are only named here, not explained in depth.

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from the official DRF documentation (https://www.django-rest-framework.org/api-guide/pagination/) or Django docs
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. Why pagination exists **[From video]**

> Imagine that you have 100,000 records with you, and 1,000 records in one webpage — we cannot display [comfortably]. Yes, we can display, but we have to scroll up and down to see the records. But whenever you split 1,000 records into multiple pages, we can easily access them just by clicking on page numbers.

The instructor's framing: a **queryset** (the full set of database rows a view would otherwise return) can be very large. Returning every row in a single HTTP response is technically possible, but:

- The response becomes huge (slow to transfer, slow to parse on the client).
- A human looking at the browsable API (or any UI built on the API) would have to scroll through everything to find what they want.

**Pagination** is the general technique — not specific to DRF, or even to APIs — of splitting a large result set into fixed-size chunks ("pages") and letting the caller ask for one page at a time, typically by clicking page numbers (1, 2, 3, …) or passing a page number as a parameter.

> **[Gap-filled] — why this matters beyond "it's inconvenient to scroll."** For an API specifically (as opposed to a webpage a human scrolls), unpaginated responses cause real, measurable problems: every request re-fetches and re-serializes the *entire* table even if the client only needs the first 20 rows; response payload size grows without bound as the table grows, so an API that works fine in development with 50 rows can time out or exhaust memory in production with 500,000; and a single expensive query blocks the database connection for longer, hurting every other concurrent request. Pagination caps the cost of each request to a predictable, small size regardless of how large the underlying table gets.

## 2. DRF's own framing of pagination **[From video]**

The instructor reads this almost directly from the official docs page (`https://www.django-rest-framework.org/api-guide/pagination/`):

> REST framework includes support for customizable pagination styles. This allows you to modify how large result sets are split into individual pages of data... The pagination API can support pagination links that are provided as part of the page [content], or [as] response headers such as Content-Range or Link.

> **[Researched] — confirming and completing this against the actual DRF docs.** The official docs state it slightly more precisely: pagination links can be included **either as part of the page content itself** (e.g. `next`/`previous` URLs inside the JSON body — this is what `PageNumberPagination` and `LimitOffsetPagination` do by default) **or via response headers**, such as `Content-Range` or `Link` (an alternative style some APIs use, closer to how GitHub's REST API paginates). DRF's built-in pagination classes default to the body-based style because it keeps the browsable API self-contained and clickable without needing to inspect headers.

## 3. The three pagination styles in DRF **[From video]**

> You can see pagination can be done using page number pagination, limit-offset pagination, cursor pagination. There are three different types of pagination concepts. ... I am going to discuss page number pagination only [today]. In the next session, that will be Monday, we'll go for limit-offset pagination [and] cursor pagination.

DRF ships three built-in pagination classes, all importable from `rest_framework.pagination`:

| Class | How the client asks for a page | This lecture covers it? |
|---|---|---|
| `PageNumberPagination` | A page **number** in the query string, e.g. `?page=3` | Yes — full build-along |
| `LimitOffsetPagination` | A `limit` (how many rows) and an `offset` (how many rows to skip), e.g. `?limit=10&offset=20` | Named only — REST API Session 20 |
| `CursorPagination` | An opaque, encoded **cursor** token pointing at a position in the ordering, e.g. `?cursor=cD0yMDIx...` | Named only — REST API Session 20 |

> **[Researched] — a preview of the other two, since the video only names them.** Per the DRF docs:
> - **`LimitOffsetPagination`** mirrors the `LIMIT`/`OFFSET` keywords used directly in SQL — `limit` is the maximum number of items to return, `offset` is the starting position in the full result set. It's flexible (a client can ask for any window) but, like `PageNumberPagination`, it can show duplicate or skipped rows if the underlying data changes between requests (e.g. a new row inserted while a user is on page 2 shifts every later row by one).
> - **`CursorPagination`** replaces page numbers/offsets with an opaque, encoded token that records a position relative to an ordering (commonly a timestamp or ID). It only exposes "next" and "previous," never an arbitrary page number, which is a deliberate trade-off: it can't jump straight to "page 47," but it stays performant on very large tables (no `OFFSET`-style scanning) and doesn't show duplicates/gaps if rows are added or removed mid-pagination. It's the recommended choice for large, frequently-changing datasets (e.g. an infinite-scroll feed).

## 4. `PageNumberPagination`: the basic idea **[From video]**

> This pagination style accepts a single number — page number — in the request query parameters. Example: `page=4`, `page=5` — whichever page number you give, we get that page's records in the result set.

So, once `PageNumberPagination` is active for a view, hitting the endpoint with `?page=4` in the URL returns only the rows belonging to page 4, not the whole table. If no `page` parameter is given, it defaults to page 1.

## 5. Turning pagination on: two settings are always needed **[From video]**

The instructor is explicit that two things are required together: which **pagination class** to use, and what **page size** each page should have. He shows two ways to wire these in — **globally** (affects every view in the project) or **per view** (affects only one view).

### 5a. Global settings (`settings.py`) **[From video]**

> The pagination style may be set globally, using the `DEFAULT_PAGINATION_CLASS` and `PAGE_SIZE` setting keys... This should be in `settings.py`. `REST_FRAMEWORK` equals curly braces — in this we have to use `DEFAULT_PAGINATION_CLASS` colon, in single quotes, `rest_framework.pagination.PageNumberPagination`, comma, `PAGE_SIZE` — this is compulsory if you require pagination — `PAGE_SIZE` colon, I'm giving [it] three.

Cleaned up, the block added to `settings.py`:

```python
# settings.py

REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 3,   # the video uses 3 to make the demo's 12 records split into 4 pages
                       # (the official docs example uses 100 — page size is just a project choice)
}
```

- `REST_FRAMEWORK` is the single settings dictionary DRF looks at for **all** of its global configuration (not just pagination — the same dict holds authentication classes, permission classes, etc. from earlier lectures).
- `DEFAULT_PAGINATION_CLASS` is a string **import path** (not the class object itself) to the pagination class every view in the project should use, unless a view overrides it.
- `PAGE_SIZE` is how many records go on each page. The instructor is explicit that this key is **required** — without it, pagination doesn't actually take effect.

> **[Gap-filled] — why `PAGE_SIZE` is required, not optional.** `PageNumberPagination.page_size` defaults to `None` in DRF's own source. A pagination class with `page_size = None` performs **no pagination at all** — it just returns everything, silently. This is a common pitfall: setting `DEFAULT_PAGINATION_CLASS` alone, without also setting `PAGE_SIZE` (globally or on a custom subclass), looks like it should paginate but doesn't, and there's no error — the API just keeps returning the full table. Always set both together.

Setting `REST_FRAMEWORK['DEFAULT_PAGINATION_CLASS']` applies pagination to **every** view in the project that returns a list (list views specifically — pagination has no effect on a detail/single-object response).

### 5b. Per-view settings (`pagination_class` attribute) **[From video]**

> I don't want to do global settings — I want to do pagination for [a] particular view only, not global settings... We can set the pagination class on an individual view by using [the] `pagination_class` attribute.

```python
from rest_framework import generics
from rest_framework.pagination import PageNumberPagination

class StudentList(generics.ListAPIView):
    queryset = Student.objects.all()
    serializer_class = StudentSerializer
    pagination_class = PageNumberPagination   # pagination only for this view
```

- `pagination_class` is a class-level attribute on any generic view (or `ViewSet`) that, when set, **overrides** whatever `DEFAULT_PAGINATION_CLASS` is configured globally — for that view only. Every other view keeps using the global default (or no pagination, if none is set globally).
- This is the same "override per view, fall back to a project-wide default" pattern already used for `authentication_classes` and `permission_classes` in earlier lectures (REST API Sessions 13–17).

> **[Gap-filled] — when to reach for per-view instead of global.** Global (`DEFAULT_PAGINATION_CLASS`) is the right choice when you want consistent pagination behavior across the whole API — the common case. Per-view `pagination_class` is for the exception: maybe one endpoint returns a small, fixed list (e.g. "list of countries") that should never be paginated, or one endpoint needs a much larger/smaller page size than the rest of the API. Setting `pagination_class = None` directly on a view is also valid — it explicitly disables pagination for just that view even if a global default is set.

## 6. Build-along: a dedicated `paginationapp` **[From video]**

The instructor builds a small, self-contained demo app to show pagination end-to-end (separate from the student/product apps used in earlier DRF lectures).

### 6a. Creating and registering the app

```bash
python manage.py startapp paginationapp
```

```python
# settings.py
INSTALLED_APPS = [
    # ...
    'paginationapp',
]
```

### 6b. The model **[From video]**

> Name of the student, name of the students... `models.CharField(max_length=20)` — name and address... I'm taking age [as] `IntegerField`.

```python
# paginationapp/models.py
from django.db import models

class Student(models.Model):
    name = models.CharField(max_length=20)
    address = models.CharField(max_length=20)
    age = models.IntegerField()
```

Nothing pagination-specific here — it's a plain three-field model, deliberately simple so the video can add many rows quickly and demonstrate pages splitting cleanly.

### 6c. Registering with the admin **[From video]**

```python
# paginationapp/admin.py
from django.contrib import admin
from paginationapp.models import Student

@admin.register(Student)
class StudentAdmin(admin.ModelAdmin):
    list_display = ('id', 'name', 'address', 'age')
```

The `@admin.register(...)` decorator form (rather than a separate `admin.site.register(Student, StudentAdmin)` call) was already used in earlier lectures — same pattern, reused here just to get an admin screen quickly for typing in demo data.

### 6d. The serializer **[From video]**

```python
# paginationapp/serializers.py
from rest_framework import serializers
from paginationapp.models import Student

class StudentSerializer(serializers.ModelSerializer):
    class Meta:
        model = Student
        fields = ['id', 'name', 'address', 'age']
```

A standard `ModelSerializer` — nothing pagination-specific in the serializer itself. This matters: **pagination is applied by the view, not the serializer.** The serializer just knows how to turn one `Student` (or a list of them) into JSON; it's the view + pagination class that decides *which slice* of the queryset gets serialized per request.

### 6e. The view **[From video]**

```python
# paginationapp/views.py
from rest_framework import generics
from paginationapp.serializers import StudentSerializer
from paginationapp.models import Student

class StudentList(generics.ListAPIView):
    queryset = Student.objects.all()
    serializer_class = StudentSerializer
```

A plain `ListAPIView` (the "concrete view class" style from REST API Session 11) — `queryset` is the full, unfiltered set of students; `serializer_class` says how to represent each one. No `pagination_class` is set here yet, because the video applies pagination **globally** first (§5a).

### 6f. Wiring up the URL **[From video]**

```python
# urls.py (project-level, as this project keeps URLs at the project root)
from django.urls import path
from paginationapp.views import StudentList

urlpatterns = [
    # ...
    path('studentapi/', StudentList.as_view()),
]
```

> **[Gap-filled] — exact endpoint name reconstructed.** The transcript never dictates the literal URL string character-by-character (it's read aloud as "student API slash"), so the path name above (`studentapi/`) is a reasonable reconstruction of what's shown running in the browser, not a verbatim quote.

### 6g. Migrations, superuser, and demo data **[From video]**

```bash
python manage.py makemigrations
python manage.py migrate
python manage.py createsuperuser
python manage.py runserver
```

The instructor then logs into `/admin/` and manually adds **12 `Student` rows** through the admin's "Add student" form (name, address, age for each) — purely so there's enough data to visibly split across multiple pages.

> **[Gap-filled] — why 12 specifically.** 12 isn't a special number technically; the instructor picked it so that with `PAGE_SIZE = 3`, `12 ÷ 3 = 4` comes out to a clean 4 pages, which is easy to demonstrate and count on screen ("page 1, 2, 3, 4 — no page 5").

## 7. Seeing global pagination work **[From video]**

With `DEFAULT_PAGINATION_CLASS` + `PAGE_SIZE = 3` set globally (§5a) and 12 `Student` rows in the database, hitting `/studentapi/` in the browsable API shows:

> Now you can see our API is working, but now — pagination count is totally 12 records, but here only [the first] 3 records also there. Now you can see page numbers are clearly available here — second page, click on this page number 2, and page number 2 records — 4, 5, 6 records are coming. Third page we can click on it...

- Only the first `PAGE_SIZE` (3) records are returned per request.
- The browsable API renders clickable page-number controls (and "Previous"/"Next" style links) driven by the pagination metadata in the response.
- Navigating to `?page=2` returns rows 4–6, `?page=3` returns rows 7–9, `?page=4` returns rows 10–12.
- Requesting a page that doesn't exist (`?page=10` when there are only 4 pages) returns an **"Invalid page"** error instead of an empty or wrapped-around result.

> **[Researched] — the actual JSON shape behind that browsable-API display.** The video shows the browsable API's rendered HTML controls, not the raw JSON, but per the DRF docs, `PageNumberPagination`'s default response envelope looks like this for every paginated list endpoint:
> ```json
> {
>   "count": 12,
>   "next": "http://127.0.0.1:8000/studentapi/?page=2",
>   "previous": null,
>   "results": [
>     {"id": 1, "name": "Mohan", "address": "Hyderabad", "age": 22},
>     {"id": 2, "name": "Manoj", "address": "Hyderabad", "age": 25},
>     {"id": 3, "name": "Kiran", "address": "Hyderabad", "age": 26}
>   ]
> }
> ```
> `count` is the total number of rows across *all* pages (not just this page); `next`/`previous` are ready-to-use full URLs for the adjacent pages (`null` when there isn't one — e.g. `previous` is `null` on page 1); `results` is the actual, serialized slice of data for the requested page. A client (a JS frontend, a mobile app) is expected to read `next`/`previous` directly rather than construct page URLs by hand.

## 8. A custom, per-view pagination class **[From video]**

Instead of the global settings, the instructor then shows defining a **named, reusable pagination class** and attaching it to one view via `pagination_class` — the per-view approach from §5b, filled in with a concrete subclass.

> I'm going to import from `rest_framework.pagination` import `PageNumberPagination`... I'm going to create my own class — one class name is `MyPagination` — which is inherited from `PageNumberPagination`, and here `page_size` I'm giving equals to 3.

```python
# paginationapp/pagination.py
from rest_framework.pagination import PageNumberPagination

class MyPagination(PageNumberPagination):
    page_size = 3
```

```python
# paginationapp/views.py
from rest_framework import generics
from paginationapp.serializers import StudentSerializer
from paginationapp.models import Student
from paginationapp.pagination import MyPagination

class StudentList(generics.ListAPIView):
    queryset = Student.objects.all()
    serializer_class = StudentSerializer
    pagination_class = MyPagination
```

With this in place, the instructor **comments out** the `REST_FRAMEWORK` block in `settings.py` — the global setting is no longer needed, because the view now supplies its own pagination class directly. Hitting `/studentapi/` behaves identically (3 records per page, 4 pages) — because `MyPagination` still sets `page_size = 3` — but now it's scoped to this one view instead of the whole project.

> **[Gap-filled] — why bother with a subclass instead of just reusing `PageNumberPagination` directly?** As shown in §5b, `pagination_class = PageNumberPagination` (the bare class) works fine on its own *if* `PAGE_SIZE` is still set globally — the class itself has `page_size = None` by default and would fall back to reading it from settings, or paginate nothing if that's also unset. Subclassing lets a view carry its own page size (and, as shown next, its own parameter names) **without depending on any global setting at all** — genuinely self-contained, and the standard way DRF expects per-view pagination tuning to be done, per the official docs' own examples.

## 9. Renaming the page-number query parameter (`page_query_param`) **[From video]**

> One more important thing: [it is] compulsory the parameter name [be] `page` only... other than `page` [it] will not work. For example, `p=1` — is it going to work? No... but sir, I want to work with `p=3`, not `page=3` — how can I do [that]? ... `page_query_param` equal to [the] letter `p`.

By default, `PageNumberPagination` only recognizes a query parameter literally named `page` — `?p=2` is silently ignored and the API just returns page 1 again, with no error. To change which parameter name the class listens for, set `page_query_param`:

```python
class MyPagination(PageNumberPagination):
    page_size = 3
    page_query_param = 'p'   # now ?p=2 works; ?page=2 no longer does
```

> **[Gap-filled] — this is a genuinely easy trap.** Because an unrecognized query parameter causes *no error at all* (it's just ignored, defaulting to page 1), this is easy to misdiagnose as "pagination is broken" when really the client and the API just disagree on the parameter's name. Always confirm the exact parameter name a given endpoint expects — either from its docs or by checking `page_query_param` on its pagination class — rather than assuming `page` is universal.

## 10. Letting the client choose the page size (`page_size_query_param`) **[From video]**

> Page size query parameter... I am taking `records` — `records` equals to any number, you can [give] that many size will come... Page size query parameter, `records`, equals [to] four — four records... one, two, three, four records.

By default, the page size is fixed by `page_size` on the server; the client can't change it. Setting `page_size_query_param` opts into letting the client request a different page size per request, via a query parameter of that name:

```python
class MyPagination(PageNumberPagination):
    page_size = 3
    page_query_param = 'p'
    page_size_query_param = 'records'   # ?records=4 → 4 rows per page instead of 3
```

The video demonstrates `?p=1&records=4` returning 4 rows on page 1, then `?records=5` returning 5, then `?records=2` returning 2 — the client is now directly controlling how many rows come back per page.

## 11. Capping the client-requested page size (`max_page_size`) **[From video]**

> Max size — max page size — is only five, I'm taking... if I give six or seven, now you can see it will not come — only five, only maximum. Even though if I give `records=7`... five records only [are] coming, because we [have] limited the max size.

`page_size_query_param` alone would let a client ask for an unbounded number of rows per page (e.g. `?records=100000`), defeating the entire point of pagination. `max_page_size` caps how large a client-requested page size is allowed to be — requests above the cap are silently clamped down to the maximum, not rejected with an error:

```python
class MyPagination(PageNumberPagination):
    page_size = 3
    page_query_param = 'p'
    page_size_query_param = 'records'
    max_page_size = 5   # ?records=7 still only returns 5 rows
```

> **[Researched] — why `max_page_size` matters for more than tidiness.** Per the DRF docs, `page_size_query_param` is `None` by default specifically so a project has to *opt in* to client-controlled page sizes; once opted in, `max_page_size` is the recommended safeguard against a client (accidentally or deliberately) requesting a page size large enough to hurt server performance — effectively re-introducing the "return everything in one response" problem pagination exists to solve in the first place. Always set a `max_page_size` alongside `page_size_query_param` in a production API.

## 12. What's next **[From video]**

> I have done with page number pagination. In [the] next session, that will be Monday, we'll go for limit-offset pagination [and] cursor pagination — we'll discuss in detail.

REST API Session 20 picks up with `LimitOffsetPagination` and `CursorPagination` — the two styles named in §3 but not built out in this lecture.

---

## Example — a complete, standalone `PageNumberPagination` setup **[Example]**

To make the mechanics concrete outside the video's specific `Student` model, here's a minimal, self-contained pagination setup for a hypothetical `Book` API:

```python
# books/pagination.py
from rest_framework.pagination import PageNumberPagination

class BookPagination(PageNumberPagination):
    page_size = 5                     # 5 books per page by default
    page_query_param = 'page'         # ?page=2 (kept as the DRF default here)
    page_size_query_param = 'page_size'  # client may override with ?page_size=10
    max_page_size = 20                # but never more than 20 per page, regardless of request
```

```python
# books/views.py
from rest_framework import generics
from .models import Book
from .serializers import BookSerializer
from .pagination import BookPagination

class BookList(generics.ListAPIView):
    queryset = Book.objects.all().order_by('id')  # explicit ordering matters for stable pagination
    serializer_class = BookSerializer
    pagination_class = BookPagination
```

**Sample input → output**, assuming 12 `Book` rows exist:

| Request | Result |
|---|---|
| `GET /books/` | First 5 books; `"next"` points at `?page=2`; `"previous"` is `null` |
| `GET /books/?page=2` | Books 6–10; both `"next"` and `"previous"` are set |
| `GET /books/?page=3` | Books 11–12 (only 2 left); `"next"` is `null` |
| `GET /books/?page=4` | `404 Not Found` with `{"detail": "Invalid page."}` — no such page exists |
| `GET /books/?page_size=10` | First 10 books on page 1 (client-requested size, under the cap) |
| `GET /books/?page_size=100` | Clamped to 20 books (the `max_page_size` cap), not 100 |

> **[Gap-filled] — why `.order_by('id')` matters here, beyond the video's example.** `PageNumberPagination` (and `LimitOffsetPagination`) work by slicing a queryset — "give me rows 6 through 10." Without an explicit, stable ordering, the database is free to return rows in a different order on each query, which can make the *same* row appear on two different pages, or never appear at all, as a table is being paginated through. Always pair pagination with an explicit `.order_by(...)` (or a `Meta.ordering` on the model) — this is exactly the kind of gap the video's small, single-session demo (added, viewed, and paginated within one uninterrupted run) wouldn't naturally surface.

---

## Industry best practices & common pitfalls **[Gap-filled / Researched]**

- **Always set `PAGE_SIZE` (or a subclass's `page_size`) alongside `DEFAULT_PAGINATION_CLASS`.** The class alone does nothing without a size — see §5a.
- **Set a `max_page_size` whenever `page_size_query_param` is enabled**, so client-controlled page sizes can't defeat the purpose of pagination — see §11.
- **Pair pagination with an explicit ordering** on the queryset (`.order_by(...)`), so which rows land on which page is stable and repeatable — see the Example above.
- **Prefer `CursorPagination` for large, frequently-changing datasets** (e.g. a live feed) — `PageNumberPagination`/`LimitOffsetPagination` can show duplicate or skipped rows if data is inserted/deleted between page requests, since they're based on numeric position, not a stable cursor. (Covered properly in REST API Session 20.)
- **Document the pagination parameter names** your API actually uses (`page`, `p`, `records`, whatever a project has customized them to) — since an unrecognized parameter name is silently ignored rather than erroring, a mismatch here is a common, hard-to-notice bug for API consumers (see §9).
- **Decide global vs. per-view deliberately.** A single project-wide `DEFAULT_PAGINATION_CLASS` keeps every endpoint predictable for API consumers; reach for a per-view `pagination_class` (or `pagination_class = None`) only for genuine exceptions (a small, fixed lookup list; one endpoint needing a very different page size).
- **A `404`/"Invalid page" response is expected, not a bug**, when a client requests a page number beyond the last page — handle it in client code rather than assuming every page number succeeds.

---

## Wrap-up

- **From video:** why pagination is needed (large result sets are slow and unwieldy to return whole); DRF's three built-in pagination styles (`PageNumberPagination`, `LimitOffsetPagination`, `CursorPagination`), with this lecture focused entirely on the first; global pagination settings (`REST_FRAMEWORK['DEFAULT_PAGINATION_CLASS']` + `PAGE_SIZE`) versus per-view settings (`pagination_class` attribute); a full build-along in a new `paginationapp` (model, admin, serializer, `ListAPIView`, URL, 12 demo records) showing 12 records split into 4 pages of 3; a custom `MyPagination(PageNumberPagination)` subclass used per-view instead of global settings; three tunable attributes demonstrated live — `page_query_param` (rename the `page` parameter, e.g. to `p`), `page_size_query_param` (let the client choose page size, e.g. via `records`), and `max_page_size` (cap how large a client-requested page can be); `LimitOffsetPagination` and `CursorPagination` announced as next session's topic.
- **Gap-filled:** why unpaginated APIs cause real performance/scalability problems, not just inconvenience; why `PAGE_SIZE`/`page_size` being unset silently disables pagination; when to choose global vs. per-view pagination; why a bare `pagination_class = PageNumberPagination` still needs a page size from somewhere; why an unrecognized query-parameter name fails silently rather than erroring; why explicit `.order_by(...)` matters for stable pagination; the reconstructed URL name (`studentapi/`) since the video didn't dictate it character-by-character.
- **Researched:** the DRF docs' precise wording on header-based vs. body-based pagination links; a preview of how `LimitOffsetPagination` and `CursorPagination` actually work (ahead of REST API Session 20); the exact JSON response shape (`count`/`next`/`previous`/`results`) behind the browsable API's page controls; why `max_page_size` is the recommended safeguard once client-controlled page sizes are enabled.

Double-check against the completeness pass: why pagination exists — covered; DRF's doc framing (header vs. body links) — covered; all three pagination style names — covered (two previewed via research, one fully built); global settings — covered; per-view settings — covered; the full demo app (model/admin/serializer/view/URL/migrations/superuser/data) — covered; global pagination behavior in the browsable API, including the "Invalid page" case — covered; the custom `MyPagination` class — covered; `page_query_param`, `page_size_query_param`, `max_page_size` — each covered individually; the "next session" teaser — covered. Nothing from the transcript was left out.
