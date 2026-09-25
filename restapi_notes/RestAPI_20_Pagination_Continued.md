# REST API Session 20 — Pagination continued: Limit/Offset & Cursor Pagination, and HyperlinkedModelSerializer

Source: `transcripts/restapi/REST API-20.txt`
Covers: a direct continuation of REST API Session 19's pagination topic. REST API Session 19 covered **page number pagination**; this session opens with a quick recap of it, then walks through DRF's other two built-in pagination styles — **limit/offset pagination** and **cursor pagination** — building and testing both live against the same `pagination_app`/`Student` project used in REST API Session 19. The session then pivots to a related but distinct new topic: **`HyperlinkedModelSerializer`**, built from scratch in a brand-new app (`myapp1`) using a `ModelViewSet` and a `DefaultRouter`.

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official Django/DRF docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. Recap: page number pagination from REST API Session 19 **[From video]**

The session opens by briefly re-showing last session's setup before moving on: a `Student` model (registered in the admin panel), a `StudentSerializer` (a `ModelSerializer`), and a `ListAPIView` in `views.py` using a custom pagination class:

```python
# pagination_app/pagination.py  (from REST API Session 19)
from rest_framework.pagination import PageNumberPagination

class MyPagination(PageNumberPagination):
    page_size = 3                     # 3 records per page
    page_query_param = 'p'            # rename ?page= to ?p=
    page_size_query_param = 'records' # lets the client request a different page size via ?records=
    max_page_size = 5                 # even if the client asks for more, never return more than 5
```

- `page_size` — how many records appear on one page by default.
- `page_query_param` — the URL query string key used to request a specific page (renamed here from the default `page` to `p`).
- `page_size_query_param` — the URL query string key that lets the *client* override `page_size` per request (renamed here to `records`); this is `None` by default in DRF (meaning clients normally can't change page size at all unless you opt in like this).
- `max_page_size` — a hard ceiling on `page_size_query_param`, so a client can't ask for an unreasonably large page (e.g. `?records=1000`) and get the whole table back at once.

> **[Gap-filled] — why this recap matters for the rest of the lecture.** Every pagination class DRF ships works the same way structurally: you subclass a built-in pagination class, override a handful of class attributes to customize its behavior, then point a view's `pagination_class` attribute at your subclass. REST API Session 19 taught this pattern using `PageNumberPagination`; this lecture reuses the *exact same pattern* twice more, just swapping in `LimitOffsetPagination` and `CursorPagination` as the parent class. If the shape of the code below looks repetitive, that's intentional — the point of this lecture is that once you understand the pattern, adding a different pagination style is a small, mechanical change.

## 2. The three pagination styles DRF offers **[From video / Researched]**

The instructor pulls up DRF's official pagination documentation page live and reads from it:

> Pagination in DRF can be done in three different ways: **page number pagination**, **limit/offset pagination**, and **cursor pagination**.

| Style | DRF class | Client controls | Best for |
|---|---|---|---|
| Page number pagination | `PageNumberPagination` | Which numbered page to view (`?page=2`) | Simple lists where "go to page 5" makes sense |
| Limit/offset pagination | `LimitOffsetPagination` | How many records, and from which position (`?limit=4&offset=8`) | SQL-style slicing; mirrors `LIMIT`/`OFFSET` in a database query |
| Cursor pagination | `CursorPagination` | Only "next"/"previous" — no arbitrary jump | Large or frequently-changing datasets (feeds, logs) |

> **[Researched]** Per the [DRF pagination docs](https://www.django-rest-framework.org/api-guide/pagination/), these are indeed the three pagination classes DRF ships out of the box, all living in `rest_framework.pagination`. All three also share one global settings hook — the `PAGE_SIZE` setting in `settings.py` acts as the fallback default for `page_size` (`PageNumberPagination`), `default_limit` (`LimitOffsetPagination`), and `page_size` (`CursorPagination`) alike, if a subclass doesn't set its own value.

## 3. Where the pagination class is configured: global vs. per-view **[From video]**

The video repeats a detail from REST API Session 19: a pagination class can be wired in two different places.

**Globally**, in `settings.py` — applies to *every* view in the project unless a view overrides it:

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.LimitOffsetPagination',
}
```

**Locally**, per view — set `pagination_class` directly on the view/viewset:

```python
# views.py
from .pagination import MyPagination

class StudentList(generics.ListAPIView):
    queryset = Student.objects.all()
    serializer_class = StudentSerializer
    pagination_class = MyPagination
```

The instructor deliberately chooses the **local** route for this lecture's demos ("I'm not going to give globally, in the local only I'm going to use") so that each pagination style can be swapped in and tested one at a time without affecting other views in the project.

> **Industry best practice.** Setting `DEFAULT_PAGINATION_CLASS` globally is convenient when an entire API should behave consistently (e.g. every list endpoint paginates the same way), but it's easy to forget that a *global* default silently affects views you didn't intend to paginate — including ones added later by teammates. A common real-world pattern is: set a sensible global default (usually `PageNumberPagination` with a modest `PAGE_SIZE`), then override `pagination_class` only on the specific views that need something different (like a `CursorPagination`-backed activity feed).

## 4. Limit/offset pagination — what it is **[From video]**

The instructor reads directly from the DRF documentation on screen, matching it closely:

> This pagination style mirrors the syntax used when looking up multiple database records. The client includes both a `limit` and an `offset` query parameter. The limit indicates the maximum number of items to return. The offset indicates the starting position of the query in relation to the complete set of un-paginated items.

In plain language:

- **`limit`** — how many records you want back in one response (like SQL's `LIMIT`).
- **`offset`** — how many records to *skip* before starting to collect results (like SQL's `OFFSET`). "Offset 7" means "skip the first 7 records, then start counting."

So `?limit=4&offset=7` reads as: "skip the first 7 records, then give me the next 4" — i.e. records #8, #9, #10, #11 (1-indexed).

> **[Example]** Given a table of 12 students ordered `1..12`:
>
> | Request | Records returned |
> |---|---|
> | `?limit=4&offset=0` | 1, 2, 3, 4 |
> | `?limit=4&offset=7` | 8, 9, 10, 11 |
> | `?limit=5&offset=10` | 11, 12 *(only 2 — see note below)* |
>
> The last row demonstrates a real edge case shown in the video: with 12 total records, requesting `limit=5` starting at `offset=10` can only return records 11 and 12 — there simply aren't 5 records left after skipping the first 10. DRF doesn't error here; it just returns whatever remains, which is a smaller batch than `limit` asked for.

## 5. Building limit/offset pagination **[From video]**

The instructor reuses the same `pagination_app` project from REST API Session 19, swapping the pagination class:

```python
# pagination_app/pagination.py
from rest_framework.pagination import LimitOffsetPagination

class MyPagination(LimitOffsetPagination):
    pass   # start with every default DRF setting, unchanged
```

```python
# pagination_app/views.py
from .pagination import MyPagination

class StudentList(generics.ListAPIView):
    queryset = Student.objects.all()
    serializer_class = StudentSerializer
    pagination_class = MyPagination
```

After `python manage.py makemigrations` / `migrate` (run "for clarity," even though no model fields changed) and restarting the server, hitting `/studentapi/` in the browsable API with the plain (unmodified) class returns **all 12 records with no pagination at all** — because an empty `LimitOffsetPagination` subclass still needs at least one attribute set (`default_limit`, or DRF's global `PAGE_SIZE`) before it will actually start splitting results into pages.

> **[Gap-filled] — why `pass` alone doesn't paginate anything.** `LimitOffsetPagination.default_limit` defaults to whatever `PAGE_SIZE` is set to in `settings.py` — and if `PAGE_SIZE` was never set (as in this project), `default_limit` stays `None`. With no default limit, `LimitOffsetPagination` effectively can't decide how many records belong on a page unless the client supplies `?limit=` explicitly, so an unqualified request appears "unpaginated." This is what the video demonstrates before setting `default_limit` in the next section.

## 6. `default_limit` and clicking through pages **[From video]**

Setting `default_limit` fixes the above:

```python
class MyPagination(LimitOffsetPagination):
    default_limit = 5   # 5 records per page if the client doesn't specify ?limit=
```

Refreshing `/studentapi/` now shows exactly 5 of the 12 student records. The browsable API renders **Next**/**Previous** buttons; clicking **Next** appends `?limit=5&offset=5` to the URL automatically (then `offset=10` on the following click) — DRF's browsable API calculates and inserts the next `offset` value for you, the same way it calculated `?page=2` for `PageNumberPagination` in REST API Session 19.

> **[Gap-filled] — the `count`/`next`/`previous`/`results` response envelope.** Whichever built-in pagination class is active, the JSON response is wrapped in the same four-key shape:
> ```json
> {
>   "count": 12,
>   "next": "http://127.0.0.1:8000/studentapi/?limit=5&offset=5",
>   "previous": null,
>   "results": [ { "...": "5 student records here" } ]
> }
> ```
> `count` is the total number of un-paginated records, `next`/`previous` are ready-to-click full URLs for the adjacent page (or `null` if there isn't one), and `results` holds just the current page's records. This shape is identical across `PageNumberPagination`, `LimitOffsetPagination`, and `CursorPagination` — only how `next`/`previous` are computed differs.

## 7. Custom query parameter names: `limit_query_param` / `offset_query_param` **[From video]**

Just like `PageNumberPagination`'s `page_query_param` in REST API Session 19, the URL parameter names for limit/offset can be renamed:

```python
class MyPagination(LimitOffsetPagination):
    default_limit = 5
    limit_query_param = 'page_limit'    # the video's garbled "pays limit"
    offset_query_param = 'page_offset'  # the video's garbled "pays offset"
```

Once these are renamed, the instructor demonstrates that the **old default names stop working entirely** — visiting `?limit=5&offset=2` after this change no longer slices the data at all (it's silently ignored, since DRF is now only looking for `page_limit`/`page_offset`). Only `?page_limit=5&page_offset=2` behaves correctly, returning records 3 through 7.

> **[Gap-filled]** — the transcript's "pays limit"/"pays offset" is the instructor's accented pronunciation of **"page limit"** and **"page offset"** — the actual custom parameter names typed on screen. This is purely a naming choice for the URL; it doesn't change any underlying pagination behavior, only what the client has to type in the query string.

## 8. `max_limit` — capping the client's request **[From video]**

```python
class MyPagination(LimitOffsetPagination):
    default_limit = 5
    max_limit = 4   # hard ceiling — no request can ever get more than 4 records
```

With `max_limit = 4` set, even an explicit `?limit=5` request is clamped down to **4** records returned, not 5. The instructor demonstrates this directly: asking for 5 records still yields only 4, "because we are giving max limit equals to four only."

> **Industry best practice.** `max_limit` (and `max_page_size` for `PageNumberPagination`, seen in REST API Session 19) exists specifically to stop a client — malicious or just careless — from requesting an enormous page (`?limit=100000`) and forcing the server to serialize and return the entire table in one response, which can spike memory/CPU usage and slow down the whole API for everyone. Always set a `max_limit`/`max_page_size` in production APIs, even if your default page size is generous.

## 9. Cursor pagination — what it is and why it's different **[From video / Researched]**

Again read closely from the official docs on screen:

> The cursor-based pagination presents an opaque "cursor" indicator that the client may use to page through the result set. This pagination style only presents forward and reverse controls — there are no page numbers.

The key distinguishing idea: instead of a page number or a numeric offset, `CursorPagination` gives the client an **opaque cursor token** — an encoded string that doesn't mean anything to the client, but that the server can decode to know exactly where the previous page left off. The browsable API for a cursor-paginated view therefore shows only **Next** and **Previous** buttons — never a numbered page list, and never a raw `?page=3` you could type in by hand to jump around.

> **[Researched] — why cursor pagination exists at all.** Per the [DRF docs](https://www.django-rest-framework.org/api-guide/pagination/#cursorpagination), offset-based pagination styles (`LimitOffsetPagination`, and to a lesser extent `PageNumberPagination`) have a real weakness on data that changes between requests: if a new row is inserted near the start of the ordering while a client is paging through results, every subsequent `offset` shifts by one, causing the client to see a duplicate record or skip one entirely. `CursorPagination` avoids this because the cursor encodes a position *relative to a specific record* (per the `ordering` field) rather than a raw numeric position — so insertions/deletions elsewhere in the table don't desynchronize the client's paging. This is exactly why cursor pagination is the standard choice for things like social media feeds and activity logs, where new rows are constantly being added while users scroll.

## 10. Implementing cursor pagination — the `ordering` requirement **[From video]**

Swapping in `CursorPagination`, initially left as `pass`:

```python
# pagination_app/pagination.py
from rest_framework.pagination import CursorPagination

class MyPagination(CursorPagination):
    pass
```

Running the server and hitting `/studentapi/` shows **no pagination at all** (all 12 records returned) — the same "an empty subclass doesn't actually paginate" issue seen with `LimitOffsetPagination` in Section 5. Setting `page_size = 3` produces a real error instead of silence:

```
FieldError at /studentapi/
Cannot resolve keyword 'created' into field. Choices are: address, id, name, ...
```

The instructor identifies the fix: `CursorPagination` requires an explicit `ordering` attribute naming a real field on the model, because cursors are computed *relative to a sort order* — without a stable ordering, there's no consistent way to say "the next record after this cursor."

```python
class MyPagination(CursorPagination):
    page_size = 3
    ordering = 'name'   # must be an actual field on the Student model
```

> **[Researched] — exactly why the error mentions `'created'`.** Per DRF's source and docs, `CursorPagination.ordering` defaults to `'-created'` (descending by a field literally named `created`) — a reasonable default *if* your model has a `created`/`created_at` timestamp field, which is a common convention for tracking insertion order. This project's `Student` model has no such field, so leaving `ordering` unset (or, as here, effectively unset via `pass`) makes DRF try to sort by a field that doesn't exist, producing exactly the `FieldError` shown in the transcript. Setting `ordering = 'name'` explicitly — pointing at a field that *does* exist — is the correct fix, matching what the video does.

> **[Gap-filled] — a caveat the video doesn't mention.** The DRF docs recommend ordering by a field that is both unique and effectively immutable — a creation timestamp or auto-incrementing ID are ideal, because they guarantee a strict, stable order with no ties. Ordering by `name` (as this demo does, for simplicity, since the `Student` model has no timestamp field) can be unreliable in a real project if two students share the same name — the cursor position could become ambiguous between tied rows. For a production API, prefer `ordering = '-id'` or an actual `created_at` field over a non-unique text field like `name`.

## 11. `page_size` and `cursor_query_param` in action **[From video]**

With `ordering = 'name'` in place, refreshing `/studentapi/` now correctly shows **3 records per page**, sorted alphabetically by `name` (the video's example: "Ganesh," then "G...", "H...", "I...", "J...", "K..." — i.e. students appear in alphabetical order rather than database-insertion order). Clicking **Next** repeatedly advances alphabetically through the remaining records; clicking **Previous** goes back the same way. The URL bar shows a parameter like `?cursor=cD0z...` — a long, encoded, non-human-readable string, confirming the "opaque cursor" concept from Section 9.

Just like `limit_query_param`/`offset_query_param`, the cursor's own query parameter name is customizable:

```python
class MyPagination(CursorPagination):
    page_size = 3
    ordering = 'name'
    cursor_query_param = 'cursor'   # this is already the default; shown for completeness
```

> **[Gap-filled] — a practical limitation worth knowing.** Because there's no numeric page number under the hood, cursor pagination cannot support "jump to page 7" style UI — only "next"/"previous." If a project's UI genuinely needs numbered page links (common in admin dashboards), `PageNumberPagination` or `LimitOffsetPagination` are better fits; `CursorPagination` trades that convenience for stability on frequently-changing data, per Section 9.

## 12. Comparing all three pagination styles **[Gap-filled / Researched]**

| | `PageNumberPagination` | `LimitOffsetPagination` | `CursorPagination` |
|---|---|---|---|
| Client asks for | A page number | A count + starting position | An opaque token (next/previous only) |
| Can jump to an arbitrary page? | Yes (`?page=7`) | Yes (`?offset=35`) | No — forward/backward only |
| Stable if rows are inserted/deleted mid-browsing? | Not guaranteed | Not guaranteed | Yes, by design |
| Typical use case | Simple admin-style lists, search results | SQL-like slicing, data exports | Feeds, logs, large or live-changing datasets |
| Key customizable attributes | `page_size`, `page_query_param`, `page_size_query_param`, `max_page_size` | `default_limit`, `limit_query_param`, `offset_query_param`, `max_limit` | `page_size`, `ordering`, `cursor_query_param` |

> **Industry best practice.** Choosing a pagination style is a real design decision, not just a style preference: an admin panel where staff routinely need to "jump to page 12" should use `PageNumberPagination`; a public-facing infinite-scroll feed with new posts arriving constantly should use `CursorPagination` to avoid the duplicate/skipped-row problem from Section 9; a data-export or reporting API that needs precise numeric slicing (e.g. "give me records 500–600") fits `LimitOffsetPagination` best.

---

## 13. New topic: `HyperlinkedModelSerializer` — motivation **[From video]**

With pagination finished, the instructor moves to a related but separate concept:

> Hyperlink model serializer means: you have to provide a link, [and] when the user clicks on that link, then automatically they will access the API. So far, whatever we discussed in the browsable API, records are coming record by record — but [now] I have to click on the link, [and] through the link we'll be able to access [the record]. That is called hyperlink model serializer.

In plain language: a normal `ModelSerializer` (used everywhere earlier in this course) represents each record's identity with its raw primary key (`"id": 3`). A **`HyperlinkedModelSerializer`** instead represents each record's identity as a clickable **URL** pointing straight to that record's own detail endpoint (`"url": "http://127.0.0.1:8000/studentapi/3/"`) — clicking it in the browsable API navigates directly to that one record, where it can be viewed, updated, or deleted through the browsable API's own forms.

> **[Gap-filled] — why this matters beyond "it looks nicer."** This is a small taste of a REST design principle called **HATEOAS** (Hypermedia As The Engine Of Application State) — the idea that an API response shouldn't just hand back raw data and IDs, but should also tell the client *where to go next* via embedded links, the same way a web page's `<a href>` links tell a human browser where to go next. A client consuming a hyperlinked API doesn't need to know your URL-building conventions in advance (e.g. "detail URLs are `/studentapi/<id>/`") — it can just follow the `url` field it was given.

## 14. Building a fresh app for the demo: model, admin **[From video]**

Rather than reuse `pagination_app`, the instructor creates a brand-new app to keep the two concepts separate:

```bash
python manage.py startapp myapp1
```

```python
# settings.py
INSTALLED_APPS = [
    # ...
    'rest_framework',
    'myapp1',
]
```

```python
# myapp1/models.py
from django.db import models

class Student(models.Model):
    s_id = models.IntegerField()
    name = models.CharField(max_length=100)
    address = models.CharField(max_length=200)
```

```python
# myapp1/admin.py
from django.contrib import admin
from .models import Student

admin.site.register(Student)
```

## 15. The serializer — `HyperlinkedModelSerializer` and the required `url` field **[From video / Researched]**

```python
# myapp1/serializers.py
from rest_framework import serializers
from .models import Student

class StudentSerializer(serializers.HyperlinkedModelSerializer):
    class Meta:
        model = Student
        fields = ['url', 'id', 's_id', 'name', 'address']
```

The video runs into an error here first: the initial `Meta.fields` list was written *without* `'url'` in it, which produced no hyperlink at all in the response ("class Meta: serializer hyperlink is required... we have not seen hyperlink anywhere"). Adding `'url'` explicitly to `fields` is what actually makes the clickable link appear.

> **[Researched] — why `'url'` has to be listed explicitly.** Per the [DRF docs on `HyperlinkedModelSerializer`](https://www.django-rest-framework.org/api-guide/serializers/#hyperlinkedmodelserializer), this serializer class behaves like an ordinary `ModelSerializer` except that record relationships (and the record's own identity) are represented with hyperlinks instead of primary keys — by default it adds a `url` field, built from a `HyperlinkedIdentityField`, in place of (or alongside) the model's real primary key. But `url` is **not an actual field on the `Student` model** — it's a serializer-only, computed field. Whenever `Meta.fields` is set to an explicit list (rather than left out to mean "every model field"), Django REST Framework only includes fields that are in that list — so if `'url'` isn't spelled out, it's silently dropped, exactly matching the bug shown in the video.

> **[Gap-filled] — how DRF builds that URL.** The `url` field is a `HyperlinkedIdentityField` under the hood, which reverses a named URL pattern using Django's normal `reverse()` mechanism — by convention, a pattern named `'<basename>-detail'` (e.g. `student-detail`) that accepts the record's primary key. This is exactly the URL name that a `DefaultRouter` (next section) generates automatically once a `ModelViewSet` is registered — which is why the two pieces (serializer + router) have to agree on the same `basename`.

## 16. The view: `ModelViewSet`, and wiring with `DefaultRouter` **[From video]**

```python
# myapp1/views.py
from rest_framework import viewsets
from .models import Student
from .serializers import StudentSerializer

class StudentModelViewSet(viewsets.ModelViewSet):
    queryset = Student.objects.all()
    serializer_class = StudentSerializer
```

```python
# myapp1/urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from . import views

router = DefaultRouter()
router.register('studentapi', views.StudentModelViewSet, basename='student')

urlpatterns = [
    path('', include(router.urls)),
]
```

```python
# project urls.py
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('myapp1.urls')),
]
```

This is the same `ViewSets`/`DefaultRouter` pattern from REST API Session 12, applied here specifically so that the router auto-generates both the list URL (`/studentapi/`) *and* the per-record detail URL (`/studentapi/<pk>/`, named `student-detail`) that the serializer's `url` field needs to resolve.

After `makemigrations`/`migrate`, logging into the admin panel, and manually adding three `Student` records (`s_id` 101/102/103 with names and addresses), running the server and visiting `/studentapi/` shows each record with a clickable `url` field. Clicking it opens that one record's own page in the browsable API — where the instructor demonstrates editing a student's `name` directly through the generated HTML form and saving the change, confirming the update.

> **[Gap-filled] — connecting `basename` back to the `url` field.** `router.register('studentapi', views.StudentModelViewSet, basename='student')` tells the router to name the generated URL patterns `student-list` and `student-detail`. Since the serializer's `url` field (via `HyperlinkedIdentityField`) looks for a `student-detail` pattern by default (derived from the model name, lowercase), these two independently-declared pieces have to end up agreeing on the same name — if `basename` were changed to something else without updating the serializer, the `url` field would raise a `NoReverseMatch` error trying to build the link.

---

## 17. Extra standalone example: choosing a pagination style **[Example]**

Suppose a `BlogPost` model with hundreds of thousands of rows, ordered by `-published_at`, powering a mobile app's infinite-scroll feed:

```python
# blog/pagination.py
from rest_framework.pagination import CursorPagination

class FeedCursorPagination(CursorPagination):
    page_size = 20
    ordering = '-published_at'   # newest first, and a good, effectively-unique sort key
    cursor_query_param = 'cursor'
```

```python
# blog/views.py
from rest_framework import generics
from .models import BlogPost
from .serializers import BlogPostSerializer
from .pagination import FeedCursorPagination

class FeedList(generics.ListAPIView):
    queryset = BlogPost.objects.all()
    serializer_class = BlogPostSerializer
    pagination_class = FeedCursorPagination
```

Sample request/response:

```
GET /api/feed/
```
```json
{
  "next": "http://api.example.com/api/feed/?cursor=cD0yMDI1LTA5LTAx",
  "previous": null,
  "results": [
    {"id": 501, "title": "New feature launch", "published_at": "2025-09-08T10:00:00Z"},
    {"id": 500, "title": "Weekly roundup", "published_at": "2025-09-07T09:00:00Z"}
  ]
}
```

If a brand-new post (`id: 502`) is published by another user *while* this client is still scrolling and later follows the `next` cursor, it will **not** see `id: 502` shift the rest of the list or cause a duplicate — because the cursor was computed relative to `published_at` of the last-seen post, not a raw numeric offset that a new insertion would shift. This is the concrete payoff of Section 9's stability argument for cursor pagination on live-changing data, applied to a realistic use case rather than the small, static 12-row demo table from the video.

---

## Wrap-up

- **From video:** a recap of REST API Session 19's `PageNumberPagination` setup; the three DRF pagination styles named directly from the official docs; configuring `pagination_class` locally per view vs. globally via `DEFAULT_PAGINATION_CLASS`; building and testing `LimitOffsetPagination` (`default_limit`, custom `limit_query_param`/`offset_query_param`, `max_limit`, and the "fewer records than requested near the end of the table" edge case); building and testing `CursorPagination` (the `ordering`-required error and its fix, `page_size`, `cursor_query_param`, next/previous-only navigation); and a full new demo of `HyperlinkedModelSerializer` in a fresh `myapp1` app — model, admin, serializer (including the `Meta.fields` bug where a missing `'url'` entry hid the hyperlink), `ModelViewSet`, `DefaultRouter`, and editing a record through the generated hyperlink.
- **Gap-filled:** why an empty `LimitOffsetPagination`/`CursorPagination` subclass doesn't paginate anything by itself; the `count`/`next`/`previous`/`results` response envelope shared by all three pagination classes; why the custom "pays limit"/"pays offset" parameter names replace (not add to) the defaults; the ambiguity risk of ordering `CursorPagination` by a non-unique field like `name`; a side-by-side comparison table of all three pagination styles and when to reach for each; why `HyperlinkedModelSerializer`'s `url` field must be explicitly listed in `Meta.fields`; and how `basename` in the router ties back to the `-detail` URL name the `url` field resolves.
- **Researched:** DRF's `LimitOffsetPagination`/`CursorPagination` class attributes and the shared global `PAGE_SIZE` settings fallback; the documented reason `CursorPagination` requires an `ordering` attribute (and why its unset default references a `created` field, exactly matching the transcript's error); the HATEOAS-style motivation behind hyperlinked serializers; and the DRF docs' explanation of how `HyperlinkedIdentityField` resolves the `url` field via a named `<basename>-detail` URL pattern.

Double-check against the completeness checklist: page-number-pagination recap ✓, three pagination styles named ✓, local vs. global pagination configuration ✓, limit/offset concept ✓, `default_limit` ✓, next/previous browsable-API navigation ✓, manual limit/offset arithmetic and the "fewer records left" edge case ✓, custom `limit_query_param`/`offset_query_param` ✓, `max_limit` ✓, cursor pagination concept ✓, the `ordering`-required error and its fix ✓, `page_size` and `cursor_query_param` for cursor pagination ✓, comparison across all three styles ✓, `HyperlinkedModelSerializer` motivation ✓, new app/model/admin for the demo ✓, the `Meta.fields`-missing-`'url'` bug ✓, `ModelViewSet` + `DefaultRouter` wiring ✓, editing a record via the generated hyperlink ✓. The session's closing off-topic Q&A (job-application/resume advice, and a brief forward-reference to exporting API data to CSV/Excel in a future session) is not technical DRF content and is intentionally omitted from these notes.
