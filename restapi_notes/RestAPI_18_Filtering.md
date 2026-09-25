# REST API Session 18 — Filtering: `get_queryset()`, DjangoFilterBackend, SearchFilter & OrderingFilter

Source: `transcripts/restapi/REST API-18.txt`
Covers: this is DRF Session 18. The video opens by recapping the previous session's manual filtering (overriding `get_queryset()` and filtering by a query parameter, plus filtering a queryset down to just the logged-in user's own data), then spends the rest of the session on DRF's **declarative filtering backends** — `DjangoFilterBackend` (from the third-party `django-filter` package), `SearchFilter`, and `OrderingFilter` (both built into DRF itself) — including live install/debugging of `django-filter`, wiring each backend up both globally (`settings.py`) and per-view, and a real (if slightly confused, in the video) debugging session about sort order.

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped, rushed, or garbled something
- **[Researched]** — pulled from the official DRF docs (https://www.django-rest-framework.org/) or Django docs (https://docs.djangoproject.com/)
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. Recap — manual filtering with `get_queryset()` **[From video]**

> In the last session, we just started with the normal filtering by using the `get_queryset()` method. Today we will look into more filtering concepts.

The previous session (not covered by these notes, but referenced here) showed the most basic way to filter API results: instead of a generic view's `queryset` attribute being a fixed, unfiltered `Model.objects.all()`, you override the `get_queryset()` method and build the queryset dynamically, typically based on a query parameter pulled from the incoming request.

> **[Gap-filled] — what `get_queryset()` looks like, since the video only referred back to it.** A generic view (`generics.ListAPIView` and friends, introduced in earlier DRF sessions) normally reads its data source from a `queryset` class attribute. Overriding `get_queryset()` instead lets you compute that queryset at request time — reading data out of `self.request` (the current request object) — rather than using one fixed value for every request:

```python
# views.py
from rest_framework import generics
from .models import Student
from .serializers import StudentSerializer

class StudentList(generics.ListAPIView):
    serializer_class = StudentSerializer

    def get_queryset(self):
        # self.request is the incoming HTTP request DRF handed this view.
        # query_params is DRF's read-only wrapper around the URL's query
        # string (the ?key=value part) — e.g. GET /student-api/?address=hyd
        queryset = Student.objects.all()
        address = self.request.query_params.get('address')
        if address is not None:
            # only narrow the queryset if the caller actually supplied
            # ?address=... — otherwise fall through and return everything
            queryset = queryset.filter(address=address)
        return queryset
```

- `self.request.query_params` — DRF's equivalent of Django's `request.GET`; a dictionary-like object holding whatever was written after the `?` in the URL. Using `.get('address')` returns `None` if the key isn't present, instead of raising an error.
- `queryset.filter(address=address)` — the same `.filter()` method covered back in Lecture 37, now applied conditionally, only for requests that ask for it.
- Why override `get_queryset()` at all instead of just setting `queryset = Student.objects.all()`? Because a class attribute is evaluated **once**, when the class/module is loaded, and can't see anything about an individual request. `get_queryset()` is a method, called fresh on every request, so it can inspect `self.request` and return a different queryset each time.

## 2. Filtering against the current logged-in user **[From video]**

> Coming to this filtering against the current user — all we will also discuss. Currently, who is logged in, that user's data — you can get it by using `get_queryset()`.

A very common real-world need: an API where every user should only ever see **their own** records (their own orders, their own posts, their own profile data) — never anyone else's — even though the underlying database table holds rows for every user. The video points out that `get_queryset()` is exactly the tool for this, because (thanks to DRF's authentication classes, covered in REST API Sessions 15–17) `self.request.user` gives you the currently authenticated user for that request.

> **[Gap-filled] — the code for this, since the video described the idea without a full listing.**

```python
# views.py
from rest_framework import generics, permissions
from .models import Student
from .serializers import StudentSerializer

class MyStudentRecords(generics.ListAPIView):
    serializer_class = StudentSerializer
    permission_classes = [permissions.IsAuthenticated]  # from REST API Session 14

    def get_queryset(self):
        user = self.request.user       # the logged-in user, set by authentication
        return Student.objects.filter(owner=user)  # only rows this user owns
```

This assumes a `Student` model with an `owner = models.ForeignKey(User, on_delete=models.CASCADE)` field recording who each row belongs to. Note this is a **different kind of filtering** from the rest of the lecture: it's identity-based (who is asking?), not a field the client chooses via the query string, so it has to stay in `get_queryset()` — none of `DjangoFilterBackend`, `SearchFilter`, or `OrderingFilter` (below) can express "only rows belonging to whoever is logged in," because those backends only ever look at the URL's query parameters, never at `request.user`.

> **[Example] — combining both.** The two techniques from sections 1 and 2 combine naturally: scope to the current user first, *then* let the declarative backends (below) filter/search/order within that already-scoped queryset.
```python
class MyStudentRecords(generics.ListAPIView):
    serializer_class = StudentSerializer
    permission_classes = [permissions.IsAuthenticated]
    filter_backends = [filters.OrderingFilter]
    ordering_fields = ['name']

    def get_queryset(self):
        return Student.objects.filter(owner=self.request.user)
```
A request like `GET /my-students/?ordering=-name` while logged in as `raja` would return only `raja`'s own student records, sorted by name descending — `get_queryset()` handles the "whose data," `OrderingFilter` handles the "what order."

## 3. What "generic filtering backends" means in DRF **[From video]**

> REST framework also includes support for generic filtering backends that allow you to easily construct complex searches and filters. Generic filters can also present themselves as controls in the browsable API and the admin API.

`get_queryset()` overrides (sections 1–2) work, but writing one by hand for every field you want filterable, searchable, or sortable gets repetitive fast, and none of that hand-written logic is visible to DRF's **browsable API** (the human-friendly, clickable HTML view of your API, from REST API Session 4) — a visitor browsing the API in a browser has no way to discover which fields are filterable.

**[Gap-filled] — defining "filter backend."** A <dfn>filter backend</dfn> is a small, reusable class that DRF's generic views know how to call automatically: given a queryset and the incoming request, it returns a new, narrowed-down (or reordered) queryset. You attach one or more of them to a view via a `filter_backends` list, and DRF runs each backend on the queryset in turn, in order, before serializing the result. Because they're a documented, standard interface, DRF can also *introspect* them to render matching form controls (a search box, a set of filter dropdowns) directly inside the browsable API — which is what "generic filters can present themselves as controls" means.

DRF ships three commonly-used filter backends relevant to this lecture:

| Backend | Where it lives | What it does |
|---|---|---|
| `DjangoFilterBackend` | third-party `django-filter` package | exact-match (or custom) filtering on one or more named fields |
| `SearchFilter` | built into `rest_framework.filters` | a single free-text search box across one or more text fields |
| `OrderingFilter` | built into `rest_framework.filters` | lets the client choose which field(s) to sort results by |

> **[Researched] — the video blurs an important distinction here.** Only `DjangoFilterBackend` requires installing the separate `django-filter` package. `SearchFilter` and `OrderingFilter` ship with DRF core (`rest_framework.filters`) and need no extra install — per the [DRF filtering docs](https://www.django-rest-framework.org/api-guide/filtering/). Near the end of the session the transcript says "you must and should install `pip install django-filter` ... then only we can able to work with all the filtering concepts" — that's only true for `DjangoFilterBackend`; `SearchFilter`/`OrderingFilter` work with a plain `pip install djangorestframework`.

## 4. Installing and enabling `django-filter` **[From video]**

> To use `DjangoFilterBackend`, first we need to install `django-filter`. Go to the terminal window: `pip install django-filter`. ... After installing, you can add `django_filters` into `INSTALLED_APPS` of your `settings.py`. Then only we can work with the `DjangoFilterBackend`.

Step by step, as demonstrated (the transcript garbles the package name as "Django iPhone filter" / "Django underscore filters" throughout — the real package is **`django-filter`**):

```bash
# in the terminal, inside the project's virtual environment
pip install django-filter
```

```python
# settings.py
INSTALLED_APPS = [
    # ...
    'rest_framework',
    'django_filters',   # note the underscore — the *import*/app name differs
                         # from the *pip package* name (django-filter, with a hyphen)
    # ...
]
```

> **[Gap-filled] — why the name changes.** This trips up a lot of beginners (and tripped up the presenter live in this video, who initially hit a `ModuleNotFoundError`). PyPI package names (what you type after `pip install`) and Python import/app names (what you type in code) aren't always identical. Here: `pip install django-filter` (hyphen) installs a package whose importable app name is `django_filters` (underscore, plural). Get either piece wrong — install the wrong thing, misspell the app name in `INSTALLED_APPS`, or forget the app-registration step entirely — and Django raises `ModuleNotFoundError: No module named 'django_filters'` the next time you run the server, exactly as happened in the video (`python manage.py runserver` → `ModuleNotFoundError`, traced back to the missing/misspelled `INSTALLED_APPS` entry).

With the package installed and registered, `DjangoFilterBackend` can be enabled two ways — **globally**, for every view in the project, or **per-view**, only where you actually add it. The video demonstrates both.

### Global setup (`settings.py`) **[From video]**

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_FILTER_BACKENDS': [
        'django_filters.rest_framework.DjangoFilterBackend',
    ]
}
```

- `REST_FRAMEWORK` — the dictionary DRF reads all of its project-wide settings from (already seen in earlier lectures for authentication/permission defaults).
- `'DEFAULT_FILTER_BACKENDS'` — the dictionary key; its value is a list of import paths (as strings) to the filter backend classes that should apply to *every* generic view in the project by default, unless a view overrides it.
- `'django_filters.rest_framework.DjangoFilterBackend'` — the full dotted import path to the class. Note it's `django_filters.rest_framework...`, not `rest_framework.django_filters...` — the video stumbles over this exact ordering more than once.

### Per-view setup **[From video]**

```python
# views.py
from django_filters.rest_framework import DjangoFilterBackend
from rest_framework import generics
from .models import Student
from .serializers import StudentSerializer

class StudentList(generics.ListAPIView):
    queryset = Student.objects.all()
    serializer_class = StudentSerializer
    filter_backends = [DjangoFilterBackend]   # only this view gets it
    filterset_fields = ['address']
```

The video explicitly does both in turn — first the global `settings.py` route, then removes it and shows the identical result configuring the same view directly — to demonstrate they're two ways to reach the same outcome. **Either is valid**; which one to pick is a project-wide-default-vs-one-off-view tradeoff (see the best-practices note in section 10).

## 5. `DjangoFilterBackend` & `filterset_fields` **[From video]**

Once `DjangoFilterBackend` is active on a view (globally or per-view), add a `filterset_fields` attribute listing which model field(s) API clients are allowed to filter on:

```python
# views.py
from django_filters.rest_framework import DjangoFilterBackend
from rest_framework import generics
from .models import Student
from .serializers import StudentSerializer

class StudentList(generics.ListAPIView):
    queryset = Student.objects.all()
    serializer_class = StudentSerializer
    filter_backends = [DjangoFilterBackend]
    filterset_fields = ['address']   # exact-match filtering on `address`
```

With this in place, a request to:

```
GET http://127.0.0.1:8000/student-api/?address=hyd
```

returns only `Student` rows whose `address` is exactly `hyd` — in the video's demo data, this returned the `raja` and `ramesh` records (both stored with address `hyd`), while a follow-up request for `?address=SR` returned only the one record whose address was `SR`.

- `?` — separates the URL path (`/student-api/`) from the **query string** (everything after it). This is standard HTTP/URL syntax, not DRF-specific.
- `address=hyd` — a **query string parameter**: a `key=value` pair. `DjangoFilterBackend` reads `filterset_fields` to know that `address` is a legal key, then filters the queryset with (effectively) `.filter(address='hyd')` — an **exact match**, not a partial/`__icontains` match (contrast with `SearchFilter` below, and with the `__icontains` lookup from Lecture 38).

### Filtering on multiple fields at once (AND logic) **[From video]**

```python
filterset_fields = ['address', 'branch']
```

> Both must and should be satisfied — the `and` means both conditions must be satisfied.

With two fields listed, a request like:

```
GET /student-api/?address=SR%20Nagar&branch=Mohan
```

only matches rows where **both** `address` equals `SR Nagar` **and** `branch` equals `Mohan` — it's an AND, not an OR. In the video's data, a `Mohan`-branch record existed for `Hyderabad` as well as one for `SR Nagar`; requesting `branch=Mohan` alone returned both, but adding `address=SR Nagar` narrowed it to just the one matching row, since a plain `Mohan` request without a matching address also pulled in the Hyderabad record. (The video's own field is literally called "branch" in this demo dataset — a general-purpose string field, not necessarily a bank/company branch — used the same way `address` is.)

> **[Example] — a small, standalone illustration independent of the video's data.** Given a `Student` model with `name`, `address`, and `branch`, and rows:

| name | address | branch |
|---|---|---|
| Asha | Pune | CSE |
| Neel | Pune | ECE |
| Divya | Mumbai | CSE |

with `filterset_fields = ['address', 'branch']`:

- `?address=Pune` → Asha, Neel (2 rows — address matches both)
- `?branch=CSE` → Asha, Divya (2 rows — branch matches both)
- `?address=Pune&branch=CSE` → Asha only (1 row — both conditions must hold)

> **[Example] — a realistic use case.** An e-commerce products API — `GET /products/?category=electronics&brand=sony` — where a storefront's category/brand filter sidebar builds exactly this kind of query string, and `filterset_fields = ['category', 'brand']` on the `ProductList` view does the rest with no custom view logic needed.

> **[Researched] — `filterset_fields` can do more than exact match.** Per the [django-filter docs](https://django-filter.readthedocs.io/en/stable/guide/rest_framework.html), instead of a plain list of field names, `filterset_fields` can be replaced with a full `django_filters.FilterSet` subclass for things the simple list can't express — ranges (`price__gte`, `price__lte`), case-insensitive contains lookups, or filtering on related-model fields. The plain-list form the video uses only supports exact matches.

## 6. `SearchFilter` & `search_fields` **[From video]**

> The `SearchFilter` class supports simple, single-query-parameter-based searching, and it is based on the Django admin's search functionality. The browsable API will include a search filter control. The `SearchFilter` class will only be applied if the view has `search_fields` set.

`SearchFilter` is a **different backend** from `DjangoFilterBackend` — built into DRF core, so it needs no extra package, only the `rest_framework.filters` import:

```python
# views.py
from rest_framework import generics, filters
from .models import Student
from .serializers import StudentSerializer

class StudentList(generics.ListAPIView):
    queryset = Student.objects.all()
    serializer_class = StudentSerializer
    filter_backends = [filters.SearchFilter]
    search_fields = ['address']   # note: search_fields, NOT filterset_fields
```

- `search_fields` — a list of the model's **text-type fields** (`CharField`, `TextField`) the search should scan. The video explicitly flags this distinction: `filterset_fields` is the `DjangoFilterBackend` attribute name; `SearchFilter` uses a differently-named attribute, `search_fields`, even though both configure "which field(s) am I acting on."
- The query parameter name defaults to **`search`**:

```
GET /student-api/?search=hyd
```

By default this behaves like Django admin search — a partial, case-insensitive match (comparable to `__icontains`, Lecture 38), so `?search=hyd` matches an address of `Hyderabad` too, not just an exact `hyd`.

### Renaming the search query parameter **[From video]**

> By default the search parameter is named `search` only. But you can override it in `settings.py`.

```python
# settings.py
REST_FRAMEWORK = {
    'SEARCH_PARAM': 's',   # dictionary value; now the query key is "s" instead of "search"
}
```

With `SEARCH_PARAM` set to `'s'`, the same lookup becomes:

```
GET /student-api/?s=hyd
```

and the plain `?search=hyd` form stops working — the param name is whatever `SEARCH_PARAM` says, not both.

> **[Example] — standalone illustration.** `search_fields = ['name', 'address']` (multiple fields at once is allowed — `SearchFilter` ORs across all of them): `?search=raj` would match a row named "Raja" *or* a row whose address contains "raj" (e.g. "Rajahmundry") — either field matching is enough, unlike `DjangoFilterBackend`'s multi-field AND behavior in section 5.

> **[Example] — a realistic use case.** A single search box at the top of a directory-style front end (à la the Teacher Directory project from Lecture 42) that searches across a teacher's first name, last name, *and* subjects in one go — exactly the kind of "one text box, several underlying fields" job `SearchFilter` is built for, versus `DjangoFilterBackend`'s per-field dropdown/filter-panel style UI.

> **[Researched] — search-field prefixes.** Per the [DRF docs](https://www.django-rest-framework.org/api-guide/filtering/#searchfilter), a field name in `search_fields` can be prefixed: `^field` matches only at the start of the field ("starts-with"), `=field` requires an exact match, `@field` does a full-text search (Postgres only), and `$field` treats the search term as a regex. With no prefix (as in this lecture), the match is a case-insensitive "contains" anywhere in the field.

## 7. `OrderingFilter` & `ordering_fields` **[From video]**

> Ordering filter is one more searching option. To work with ordering filters we have to import `OrderingFilter` in place of `SearchFilter`.

```python
# views.py
from rest_framework import generics, filters
from .models import Student
from .serializers import StudentSerializer

class StudentList(generics.ListAPIView):
    queryset = Student.objects.all()
    serializer_class = StudentSerializer
    filter_backends = [filters.OrderingFilter]
    ordering_fields = ['name']   # NOT filterset_fields, NOT search_fields
```

> **[Gap-filled] — the video's own mid-lesson mistake, kept here because it's a genuinely useful thing to notice.** In the transcript, the presenter first writes `filterset_fields = ['address']` while trying to demonstrate ordering, gets no effect, and only then catches the mistake out loud: *"ordering underscore fields, you have to use — set underscore fields is now for only for set filters, but we are working with ordering filters, compulsory ordering underscore fields only."* This is a real, easy trap: **three different backends, three differently-named attributes** (`filterset_fields` / `search_fields` / `ordering_fields`) that all superficially do "pick which field(s)" — mixing them up silently does nothing (Django doesn't error on an unused class attribute), which is exactly why the presenter's first attempt appeared to have no effect at all.

- Query parameter: **`ordering`** (default name, itself renameable the same way `SEARCH_PARAM` renames search — via an `ORDERING_PARAM` setting, per the DRF docs).
- Ascending order (the default — no sign needed):
  ```
  GET /student-api/?ordering=name
  ```
- Descending order — prefix the field with a minus sign:
  ```
  GET /student-api/?ordering=-name
  ```
- Multiple sort fields — comma-separate them; results are sorted by the first field, ties broken by the next, and so on:
  ```
  GET /student-api/?ordering=address,name
  ```

`ordering_fields` acts as an **allow-list**: only fields named in it are legal values for the `?ordering=` parameter — this prevents API clients from sorting by, say, a sensitive or unindexed field you never intended to expose. The video sets `ordering_fields = ['address']` in one pass and `ordering_fields = ['name']` in another, confirming both individually sort correctly.

> **[Example] — standalone illustration.** `ordering_fields = ['name', 'address']`, data:

| name | address |
|---|---|
| Ramesh | Hyderabad |
| Asha | Pune |
| Neel | Mumbai |

- `?ordering=name` → Asha, Neel, Ramesh (A → R)
- `?ordering=-name` → Ramesh, Neel, Asha (R → A)
- `?ordering=address` → Hyderabad, Mumbai, Pune

> **[Example] — a realistic use case.** A sortable results table on a front end, where clicking a column header sends `?ordering=<column>` (and clicking again toggles to `?ordering=-<column>`) — the exact interaction pattern that drives most "click a header to sort" admin/dashboard tables, backed by nothing more than this one DRF attribute.

## 8. The case-sensitive ordering "bug" the video ran into **[From video + Gap-filled]**

The last several minutes of the transcript are a live debugging session: the presenter adds a new `Student` row named `"Anwesh"` (transcribed variously as "unwaish"/"a and wish") and expects `?ordering=name` (ascending) to put it first alphabetically, and `?ordering=-name` (descending) to put it last — but the observed order looks wrong to them, and the video ends mid-investigation ("we will check it now, okay") without a clean resolution on camera.

> **[Gap-filled] — reconstructing what was actually going on, since the video doesn't land on the explanation.** The most likely cause, matching everything described (an inconsistent-looking sort that the presenter blames on "capital letters having the least value"), is **case-sensitive string ordering**: SQL's default `ORDER BY` on a text column sorts by the underlying character codes, and in the common ASCII/Unicode ordering, **all uppercase letters sort before all lowercase letters** (`'A'`–`'Z'` = codes 65–90, `'a'`–`'z'` = codes 97–122). So a dataset mixing consistently-capitalized names (`"Raja"`, `"Ramesh"`) with an inconsistently-entered one (say, a stray leading-lowercase or an extra space typed into the admin panel while adding the new row) will not sort the way a human expects "alphabetical order" to work, even though the database is doing exactly what it was told.

> **[Researched] — the standard fixes.** Per the [Django QuerySet reference](https://docs.djangoproject.com/en/stable/ref/models/querysets/#order-by), two common ways to get case-insensitive ordering:
> 1. Annotate with `Lower()` and order by that: `Student.objects.annotate(name_lower=Lower('name')).order_by('name_lower')`.
> 2. Configure the database column/collation to be case-insensitive at the schema level (backend-specific; e.g. `citext` in PostgreSQL).
> `OrderingFilter` itself doesn't change this — it just passes the client's chosen field straight to `.order_by()`, so whatever case-sensitivity behavior the database applies by default is what the client sees. It's worth treating this as a standing pitfall (see section 10) rather than a one-off bug in this particular video.

## 9. Combining filter backends **[Example]**

The video demonstrates `DjangoFilterBackend`, `SearchFilter`, and `OrderingFilter` **one at a time**, swapping `filter_backends` out for each demo. In real projects it's normal — and fully supported — to run **all three together** on the same view, since `filter_backends` accepts a list:

```python
# views.py
from django_filters.rest_framework import DjangoFilterBackend
from rest_framework import generics, filters
from .models import Student
from .serializers import StudentSerializer

class StudentList(generics.ListAPIView):
    queryset = Student.objects.all()
    serializer_class = StudentSerializer
    filter_backends = [DjangoFilterBackend, filters.SearchFilter, filters.OrderingFilter]
    filterset_fields = ['branch']       # exact-match dropdown-style filtering
    search_fields = ['name', 'address'] # free-text search box
    ordering_fields = ['name', 'address']  # sortable columns
```

DRF applies each backend to the queryset in the order listed, each narrowing/reordering what the previous one produced. A single request can now combine all three:

```
GET /student-api/?branch=CSE&search=hyd&ordering=-name
```

**Sample input → output:** given the earlier example table plus a `branch` column (`Asha`/Pune/CSE, `Neel`/Pune/ECE, `Divya`/Mumbai/CSE), that request would: keep only `branch=CSE` rows (Asha, Divya) → keep only those whose name or address contains "hyd" (none, in this particular sample data — illustrating that a combined query can also legitimately return zero rows) → order the remainder descending by name. Swap `search=hyd` for `search=pune` instead and it narrows to Asha, then orders that one row — output: `["Asha"]`.

## 10. Best practices & common pitfalls **[Gap-filled / Researched]**

> **Industry best practice — prefer declarative filter backends over hand-rolled `get_queryset()` filtering, except where identity is involved.** `DjangoFilterBackend`/`SearchFilter`/`OrderingFilter` are self-documenting (they show up as controls in the browsable API), consistent across every view that uses them, and far less code than writing equivalent `if`/`.filter()` chains by hand in every `get_queryset()`. Reserve manual `get_queryset()` overrides for logic the declarative backends genuinely can't express — like "only this user's own rows" (section 2), which depends on `request.user`, not on anything in the query string.

> **Industry best practice — index columns you let clients filter or order by.** Every field listed in `filterset_fields`, `search_fields`, or `ordering_fields` is a column the database may need to filter or sort on for arbitrary client-supplied requests. On a small SQLite demo table this is invisible; on a real production table it's a common source of slow queries if there's no database index on that column. Per the [Django docs on `Field.db_index`](https://docs.djangoproject.com/en/stable/ref/models/fields/#db-index), adding `db_index=True` to frequently filtered/ordered fields is a standard, cheap fix.

> **Pitfall — mixing up the three attribute names.** As section 7 shows the video itself doing: `filterset_fields` (DjangoFilterBackend), `search_fields` (SearchFilter), and `ordering_fields` (OrderingFilter) are three unrelated attribute names for three unrelated backends. Setting the wrong one is not an error — Django/DRF simply ignores an attribute a given backend doesn't look for — so the symptom is silent "nothing happens," which is harder to debug than a loud error would be.

> **Pitfall — forgetting the `django-filter` install/registration steps.** As section 4 covers, `DjangoFilterBackend` needs both `pip install django-filter` *and* `'django_filters'` added to `INSTALLED_APPS` — miss either and the result is a `ModuleNotFoundError` at server start, exactly as happened live in this video. `SearchFilter`/`OrderingFilter` need neither step, since they ship with `djangorestframework` itself.

> **Pitfall — exact match vs. partial match confusion.** `filterset_fields`'s plain-list form does **exact** matching (`address=hyd` only matches a row whose address is precisely `"hyd"`), while `SearchFilter` does a **partial, case-insensitive contains** match by default. Using the wrong tool for the job — reaching for `DjangoFilterBackend` when you actually wanted "contains" behavior — is an easy mistake; either use a `FilterSet` with an explicit `icontains` lookup (section 5's researched note), or use `SearchFilter` instead.

> **Pitfall — case-sensitive ordering surprises.** Covered in depth in section 8 — don't assume `.order_by('name')` (or `?ordering=name`) sorts the way a human reads alphabetical order if the underlying data has inconsistent capitalization.

## 11. What's next **[From video]**

The video closes by previewing the remaining topics in the DRF unit: **pagination** (with three named styles — page number pagination, limit/offset pagination, and cursor pagination — covered "tomorrow"), **hyperlinked model serializers**, and **throttling** — with a separate, later "project session" (multiple full DRF projects) planned for a following Sunday session. These match REST API Sessions 19–21 in this notes series (Pagination intro/continued, Throttling); hyperlinked model serializers and the project session aren't yet covered by a dedicated lecture number in this list and may appear folded into a nearby lecture.

---

## Wrap-up

- **From video:** the recap of manual `get_queryset()` filtering and filtering to the current logged-in user; the concept of DRF "generic filtering backends" and their browsable-API integration; installing and registering `django-filter`; `DjangoFilterBackend` configured both globally (`settings.py`) and per-view, with `filterset_fields` (including multi-field AND behavior); `SearchFilter` with `search_fields`, the default `search` query param, and renaming it via `SEARCH_PARAM`; `OrderingFilter` with `ordering_fields`, ascending/descending (`-field`) and multi-field ordering via `?ordering=`; a live debugging session around unexpected sort order; a preview of upcoming pagination/throttling/hyperlinked-serializer/project-session topics.
- **Gap-filled:** full code for the `get_queryset()` recap and the current-user-filtering pattern (the video described these without a complete listing); why the PyPI package name (`django-filter`) and the Python app name (`django_filters`) differ, and how that mismatch causes the `ModuleNotFoundError` seen live in the video; an explicit table of which attribute name (`filterset_fields`/`search_fields`/`ordering_fields`) belongs to which backend, prompted by the video's own mid-lesson mix-up; a reconstruction of the likely cause behind the video's unresolved ordering "bug" (case-sensitive sorting); a combined-backends example; a best-practices/pitfalls section.
- **Researched:** the DRF filtering docs' clarification that only `DjangoFilterBackend` needs the separate `django-filter` package (`SearchFilter`/`OrderingFilter` ship with DRF core); `django-filter`'s `FilterSet`-class alternative to plain `filterset_fields` for range/contains/related-field filtering; `SearchFilter`'s `^`/`=`/`@`/`$` field prefixes; Django's `Lower()`-annotation and collation-based fixes for case-insensitive ordering; `Field.db_index` for keeping filtered/ordered columns fast at scale.

Double-check against the completeness checklist drawn up before writing: recap of `get_queryset()` ✓, current-user filtering ✓, generic filtering backend concept ✓, `django-filter` install + `INSTALLED_APPS` ✓, `DjangoFilterBackend` global vs. per-view setup ✓, `filterset_fields` incl. multi-field AND ✓, `SearchFilter`/`search_fields`/default+custom search param ✓, `OrderingFilter`/`ordering_fields`/ascending-descending/multi-field ✓, the case-sensitive-ordering debugging moment ✓, the "next topics" preview (pagination styles, hyperlinked serializers, throttling, project session) ✓ — nothing from the transcript appears to have been left out.
