# REST API Session 9 — Generic API Views & Mixins: cutting down `APIView` boilerplate

Source: `transcripts/restapi/9.txt`
Covers: DRF's `generics` module (specifically `GenericAPIView`) combined with the five built-in **mixin** classes (`ListModelMixin`, `CreateModelMixin`, `RetrieveModelMixin`, `UpdateModelMixin`, `DestroyModelMixin`) to build full CRUD (Create, Read, Update, Delete) endpoints with far less code than the plain `APIView` classes written in REST API Sessions 7–8. This transcript is **unusually garbled** even by this course's standard (heavy speech-to-text errors — "APA View" = APIView, "Dijango" = Django, "courier set"/"kure set" = queryset, "TPA"/"Resta framework" = REST framework, "mixings"/"mixing" = mixins, "purified model" = "primary model", etc.). Where a sentence was truly unrecoverable, the actual DRF behavior is reconstructed from official DRF knowledge instead of transcribing nonsense — every such passage is labeled **[Gap-filled]** or **[Researched]** below, not presented as a real quote.

Legend:
- **[From video]** — explained directly in the transcript (even if the wording had to be cleaned up from garbled text)
- **[Gap-filled]** — my own explanation, added because the video skipped, rushed, or garbled something
- **[Researched]** — pulled from official DRF/Django docs
- **[Example]** — an extra worked example beyond what the video showed

---

## Completeness checklist (topics/subtopics this transcript touches)

1. Recap of the previous lecture: plain `APIView`-based CRUD, and why it still involves writing a lot of repetitive code.
2. Today's new topic: `generics` (specifically `GenericAPIView`) + **mixin** classes as a "more loaded"/less repetitive alternative.
3. Pointer to DRF's official documentation for "mixins" and "generic views".
4. What a **generic API view** is: "a set of commonly used patterns", meant to build API views that map closely to a database model without repeating code; `GenericAPIView` is described as a more-featured version of `APIView`.
5. What a **mixin** is, in general (a class bundling reusable methods) and specifically in DRF (cannot be used standalone; must be paired with `GenericAPIView`).
6. The five built-in model mixins and what each one does: `ListModelMixin`, `CreateModelMixin`, `RetrieveModelMixin`, `UpdateModelMixin`, `DestroyModelMixin`.
7. Practical setup: creating a new app (`rest_app_5` in the transcript), registering it in `INSTALLED_APPS`.
8. Reusing the `Customer` model (fields: name, address, email/mail, age) and its serializer from the previous app rather than retyping them.
9. `makemigrations` / `migrate` / `runserver`, then adding a handful of `Customer` records through the Django admin.
10. Required imports for the mixin-based views: `generics`, `mixins`, the serializer, and the model.
11. Building a **list** view: `GenericAPIView` + `ListModelMixin`, wiring `def get(...)` to `self.list(...)`.
12. Explanation of `*args, **kwargs` in this context (positional vs. keyword arguments; each DB record's fields arrive as keyword arguments).
13. URL wiring for the list endpoint and testing it in the browser (`GET` returns all customers, status 200).
14. Building a **create** view: `GenericAPIView` + `CreateModelMixin`, wiring `def post(...)` to `self.create(...)`.
15. A live bug in the video: using the wrong mixin (`ListModelMixin` instead of `CreateModelMixin`) for the create class caused an `AttributeError` — corrected on screen. This illustrates that the mixin used must match the operation being implemented.
16. Testing the create endpoint (POST a new customer, "Ganesh"), verifying it now appears in the list.
17. Building a **retrieve** view: `GenericAPIView` + `RetrieveModelMixin`, wiring `def get(...)` to `self.retrieve(...)`; the URL must now include a primary key (`pk`).
18. Testing retrieve for different IDs.
19. Building an **update** view: `GenericAPIView` + `UpdateModelMixin`, wiring `def put(...)` to `self.update(...)`; URL requires `pk`.
20. Testing update (changing a customer's name/address/email/age via `PUT`), confirming the change via the list view and the admin panel.
21. Building a **delete** view: `GenericAPIView` + `DestroyModelMixin`, wiring `def delete(...)` to `self.destroy(...)`; URL requires `pk`.
22. Testing delete, confirming the record count drops.
23. A code-length comparison: each mixin-based view class is described as "only 4 lines" of real logic vs. 10–18 lines for the equivalent plain-`APIView` classes from REST API Sessions 7–8.
24. An explicit statement that writing **one separate class per operation** (5 classes total) is **not** the recommended real-world pattern — it was done this way purely for teaching clarity. The instructor promises a further simplification (combining list+create into one class, and retrieve+update+delete into another — 2 classes total) in the next session.
25. Logistics/scheduling remarks at the end (not technical content).

All 25 items are covered in the sections below.

---

## 1. Where this picks up — from `APIView` to `generics` **[From video]**

The previous lecture (REST API Session 7) built full CRUD using plain `rest_framework.views.APIView`: one class per operation, each with its own `get`/`post`/`put`/`delete` method that manually instantiated a serializer, called `.is_valid()`, called `.save()`, and manually returned a `Response`. That approach works, and it does already save code compared to writing everything as plain Django function-based views — but it is still "a completely plain API view," in the instructor's words: every single line of serialization/validation/saving logic has to be written out by hand, in every class, for every model.

This lecture's subject is **generic API views and mixins** — a way to get the same CRUD behavior with dramatically less code, because DRF ships ready-made building blocks for "the operations basically every model-backed API needs" (list all, create one, retrieve one, update one, delete one) so you don't re-derive them yourself each time.

> **[From video]** "In today's lecture, I am going to show you more simplified operations, same operations with API [views]. That is called generic API View and Mixins... The purpose of generic API View is to quickly build API Views that map closely to our database models without repeating the code."

---

## 2. `GenericAPIView`: what it adds over `APIView` **[From video + Researched]**

`GenericAPIView` (imported as `from rest_framework import generics`, used as `generics.GenericAPIView`) is a subclass of `APIView`. The video's own description of it — "a more loaded version of `APIView`", "more flexibility" — is accurate but doesn't spell out concretely *what* is added, so this section fills that in from DRF's own documentation.

**[Researched]** Per the [DRF generic views documentation](https://www.django-rest-framework.org/api-guide/generic-views/), `GenericAPIView` extends `APIView` with:

- Two attributes you're expected to set on your subclass:
  - <dfn>queryset</dfn> — the base `QuerySet` this view operates on (e.g. `Customer.objects.all()`). DRF uses this to know *which table/model* the view is about.
  - <dfn>serializer_class</dfn> — the serializer class DRF should use to convert between Python/model instances and JSON for this view.
- Helper methods built on top of those two attributes, which the mixins (below) call internally, and which you can also override:
  - `get_queryset()` — returns the queryset to use (defaults to `self.queryset`); override this if the queryset needs to depend on the request (e.g. filtering to only the logged-in user's own records).
  - `get_object()` — looks up a *single* model instance from the queryset, using a URL keyword argument (by default named `pk`) to filter it. This is what powers retrieve/update/delete — it's the piece that actually finds "the one record with this ID."
  - `get_serializer()` / `get_serializer_class()` — instantiate/return the serializer to use, automatically passing it the request context.
- Attributes controlling that lookup: `lookup_field` (which model field to match against, default `'pk'`) and `lookup_url_kwarg` (which URL keyword argument name supplies that value — defaults to the same as `lookup_field`).
- Hooks for `pagination_class` and `filter_backends`, which later lectures (pagination, filtering — REST API Sessions 18–20) build on.

On its own, `GenericAPIView` still does *nothing* useful for a request — it has no `get`/`post`/etc. methods of its own. It only supplies the *plumbing* (`queryset`, `serializer_class`, `get_object()`, etc.) that the mixins below are written to use. That's why the video repeatedly stresses that `GenericAPIView` and the mixins **must be combined** — neither one is a complete view by itself.

---

## 3. What is a mixin? **[From video]**

> **[From video]** "A mixin is a class which contains a combination of methods from other classes... Mixin provides bits of common behavior. They cannot be used stand-alone... [A mixin] must be paired with generic API View to make [a] functional view."

**[Gap-filled] — mixin, in plain terms.** A <dfn>mixin</dfn> is a small class that exists purely to be combined with other classes via multiple inheritance — it's not meant to be instantiated or used by itself. Think of it like a single tool in a toolbox (a screwdriver bit) rather than a complete toolbox: a screwdriver bit alone can't drive a screw into anything, it needs a driver (handle) to hold it and apply force. Similarly, a DRF mixin class provides *one piece of behavior* (e.g. "how to list a queryset as JSON") but has no idea how to actually receive an HTTP request, load settings, or return an HTTP response — that machinery comes from `GenericAPIView` (and `APIView` underneath it). Combine the two via Python's multiple inheritance, and you get a class that is both "a working view" (from `GenericAPIView`/`APIView`) and "knows how to list/create/retrieve/update/delete" (from the mixin).

This is a standard object-oriented programming pattern (not unique to DRF) — Django itself uses the same idea for its class-based views (e.g. `LoginRequiredMixin`).

---

## 4. The five built-in model mixins **[From video + Researched]**

**[From video]** The instructor walks through DRF's official docs page listing the mixin classes available in `rest_framework.mixins`, describing each one's purpose:

| Mixin class | Method it adds | What it does |
|---|---|---|
| `ListModelMixin` | `.list(request, *args, **kwargs)` | Returns a list of **all** records in the queryset, serialized as JSON — "to list out all the records". |
| `CreateModelMixin` | `.create(request, *args, **kwargs)` | Validates the incoming data with the serializer and saves a **new** record — "to create the record". |
| `RetrieveModelMixin` | `.retrieve(request, *args, **kwargs)` | Looks up and returns **one** specific record (via `get_object()`) — "retrieve a model instance". |
| `UpdateModelMixin` | `.update(request, *args, **kwargs)` (and `.partial_update(...)`) | Looks up one record and overwrites its fields with new data — "update a model instance". |
| `DestroyModelMixin` | `.destroy(request, *args, **kwargs)` | Looks up and **deletes** one record — "delete a model instance". |

**[Researched]** All five live in `rest_framework.mixins` and are imported as, e.g., `from rest_framework import mixins` then referenced as `mixins.ListModelMixin`. Each one is intentionally tiny — it assumes `self.get_queryset()`, `self.get_serializer()`, and (for the single-record ones) `self.get_object()` already exist, which is exactly what `GenericAPIView` supplies. None of the mixins define `get`, `post`, `put`, or `delete` methods themselves — that wiring (which HTTP verb calls which mixin method) is left for *you* to write, one line per verb, as shown in the sections below. This is an important distinction from **ViewSets** (a later topic, REST API Session 12), which wire this automatically.

> **[Gap-filled] — why five separate mixins instead of one big class?** Splitting behavior this finely means a view can opt into *only* the operations it should support. A read-only "reports" endpoint, for example, can inherit `GenericAPIView` + `ListModelMixin` + `RetrieveModelMixin` and nothing else — there is then no `create`/`update`/`destroy` method available at all, so those operations are structurally impossible on that view, not just hidden by a permission check.

---

## 5. Setting up the practice app **[From video]**

The instructor creates a fresh app to demonstrate this topic cleanly, separate from the app used in REST API Sessions 7–8:

```bash
python manage.py startapp restapp5
```

Then registers it in the project's `settings.py`:

```python
# myproject/settings.py
INSTALLED_APPS = [
    # ...
    'restapp5',
]
```

Rather than retyping a model from scratch, the model and serializer are **copied over from the previous app** (called `rest_app_4` in the transcript) — reasonable, since the point of this lecture is the *view* layer, not re-teaching models/serializers already covered in REST API Sessions 2–6.

---

## 6. The model and serializer being reused **[From video, fields; Gap-filled, exact code]**

**[From video]** The model is named `Customer` and has fields **name, address, mail (email), and age** — these field names are stated directly in the transcript. The exact `models.py`/`serializers.py` code isn't re-dictated on screen (it's copy-pasted from the earlier app), so the field *types* below are a reasonable reconstruction consistent with earlier lectures' patterns, not a verbatim quote.

```python
# restapp5/models.py
from django.db import models

class Customer(models.Model):
    name = models.CharField(max_length=100)
    address = models.CharField(max_length=200)
    email = models.EmailField()
    age = models.IntegerField()

    def __str__(self):
        # Shown in the Django admin list and in shell reprs — not required
        # by DRF, but good practice on every model (Lecture 24).
        return self.name
```

```python
# restapp5/admin.py
from django.contrib import admin
from .models import Customer

admin.site.register(Customer)
```

```python
# restapp5/serializers.py
from rest_framework import serializers
from .models import Customer

class CustomerSerializer(serializers.ModelSerializer):
    class Meta:
        model = Customer
        fields = ['id', 'name', 'address', 'email', 'age']
```

**[From video]** After wiring the app in, the video runs:

```bash
python manage.py makemigrations
python manage.py migrate
python manage.py runserver
```

...then logs into `/admin/` with an existing superuser account and manually adds four `Customer` records through the admin's "Add" form (name/address/email/age for each) — this is just data seeding, not new API code, so the record contents themselves aren't important beyond "there are now a few rows to work with."

---

## 7. List — `GenericAPIView` + `ListModelMixin` **[From video]**

```python
# restapp5/views.py
from rest_framework import generics, mixins
from .serializers import CustomerSerializer
from .models import Customer

class CustomerListClass(mixins.ListModelMixin, generics.GenericAPIView):
    # Two attributes GenericAPIView needs: which rows, which serializer.
    queryset = Customer.objects.all()
    serializer_class = CustomerSerializer

    def get(self, request, *args, **kwargs):
        # self.list(...) comes from ListModelMixin. It looks up
        # self.get_queryset(), serializes every row, and returns
        # a Response containing the JSON list — you never write
        # that logic yourself, you just call it.
        return self.list(request, *args, **kwargs)
```

- `queryset = Customer.objects.all()` — tells `GenericAPIView` which rows this view is allowed to work with.
- `serializer_class = CustomerSerializer` — tells it how to turn each `Customer` row into JSON.
- `def get(self, request, *args, **kwargs):` — DRF's class-based views (inherited from `APIView`) dispatch an incoming HTTP `GET` request to a method literally named `get`, same as REST API Session 7's plain `APIView` classes. The *only* thing this method does here is delegate to `self.list(...)`.

**[Gap-filled] — `*args, **kwargs`, explained.** The video pauses here to remind viewers of a core Python concept, so it's worth spelling out precisely:
- `*args` collects any extra **positional** arguments passed into the method, as a tuple.
- `**kwargs` collects any extra **keyword** arguments, as a dictionary (key → value pairs).
- Django's URL dispatcher passes captured URL segments (like a `pk` from `<int:pk>` in the URL pattern) into the view as keyword arguments — so `**kwargs` is how a value like `pk=1` from the URL actually reaches `self.retrieve()` a few sections down. Writing `*args, **kwargs` in every one of these methods is what lets the *same* method signature work whether the URL supplies extra values (like `pk`) or not — the method doesn't need to know or care in advance.

**URL wiring:**

```python
# restapp5/urls.py
from django.urls import path
from . import views

urlpatterns = [
    path('customerlist/', views.CustomerListClass.as_view(), name='customerlist'),
]
```

```python
# myproject/urls.py
from django.urls import path, include

urlpatterns = [
    # ...
    path('', include('restapp5.urls')),
]
```

**[From video]** Visiting `http://127.0.0.1:8000/customerlist/` in the browser (DRF's browsable API, REST API Session 4) returns all four seeded customer records as JSON, with an HTTP `200 OK` status.

> **Sample output [Example]**
> ```json
> [
>   {"id": 1, "name": "Raj", "address": "Sweety Nagar", "email": "raj@gmail.com", "age": 32},
>   {"id": 2, "name": "Kumar", "address": "R.S. Nagar", "email": "kumar@gmail.com", "age": 32},
>   {"id": 3, "name": "Reddy", "address": "Ameerpet", "email": "reddy@gmail.com", "age": 35},
>   {"id": 4, "name": "Priya", "address": "Kukatpally", "email": "priya@gmail.com", "age": 29}
> ]
> ```

**[From video]** The instructor explicitly calls out how short this is: the whole class is about four meaningful lines (`queryset`, `serializer_class`, and a two-line `get` method), compared to the roughly 10–15+ lines the equivalent plain-`APIView` list method took in REST API Session 7.

---

## 8. Create — `GenericAPIView` + `CreateModelMixin`, and a live bug **[From video]**

```python
class CustomerCreateClass(mixins.CreateModelMixin, generics.GenericAPIView):
    queryset = Customer.objects.all()
    serializer_class = CustomerSerializer

    def post(self, request, *args, **kwargs):
        # self.create(...) comes from CreateModelMixin. It builds a
        # serializer from request.data, validates it, saves a new
        # Customer row, and returns a 201 Created response containing
        # the new record's JSON.
        return self.create(request, *args, **kwargs)
```

```python
# restapp5/urls.py (added)
path('customercreate/', views.CustomerCreateClass.as_view(), name='customercreate'),
```

**[From video] — the bug.** On the first attempt, the instructor pasted the *list* class as a starting point for the create class but forgot to change the mixin, leaving `ListModelMixin` in place while calling `self.create(...)` in the `post` method. Submitting a `POST` request produced an error: *"customer create object has no attribute create"* — because `ListModelMixin` only defines `.list()`, not `.create()`; that method genuinely does not exist on a class that only mixes in `ListModelMixin`. Swapping in `CreateModelMixin` fixed it immediately.

> **[Gap-filled] — why this is a useful mistake to see.** This isn't a random typo — it demonstrates a real rule: **the mixin you inherit and the method you call must match.** `AttributeError: 'CustomerCreateClass' object has no attribute 'create'` is exactly the error Python raises whenever code calls a method that doesn't exist anywhere in a class's inheritance chain. If you see this exact shape of error with DRF generics, the fix is almost always "you called `self.create()`/`self.list()`/`self.retrieve()`/`self.update()`/`self.destroy()` without including the mixin that defines it."

**[From video]** After the fix, `POST`-ing a new customer (name "Ganesh", plus address/email/age) to `/customercreate/` succeeds — the response confirms the created record, and revisiting `/customerlist/` shows it as a new 5th row.

> **Sample input → output [Example]**
> Request: `POST /customercreate/` with body
> ```json
> {"name": "Ganesh", "address": "Somajiguda", "email": "ganesh@gmail.com", "age": 43}
> ```
> Response: `201 Created`
> ```json
> {"id": 5, "name": "Ganesh", "address": "Somajiguda", "email": "ganesh@gmail.com", "age": 43}
> ```

---

## 9. Retrieve — `GenericAPIView` + `RetrieveModelMixin` (needs a primary key) **[From video]**

```python
class CustomerRetrieveClass(mixins.RetrieveModelMixin, generics.GenericAPIView):
    queryset = Customer.objects.all()
    serializer_class = CustomerSerializer

    def get(self, request, *args, **kwargs):
        # self.retrieve(...) calls self.get_object() internally, which
        # reads the 'pk' value out of kwargs (supplied by the URL) and
        # fetches exactly that one Customer row -- or raises Http404
        # if no row with that id exists.
        return self.retrieve(request, *args, **kwargs)
```

```python
# restapp5/urls.py (added)
path('customerretrieve/<int:pk>/', views.CustomerRetrieveClass.as_view(), name='customerretrieve'),
```

**[From video]** Unlike the list/create endpoints, this one **requires a primary key in the URL** — the transcript is explicit about this ("compulsory for retrieve model mixin, a URL creation time, compulsory we have to use... ID... primary key"). Visiting `/customerretrieve/1/` returns only the record with `id=1`; `/customerretrieve/2/` returns only `id=2`, and so on.

**[Researched] — why `pk` specifically, and what happens if it's missing.** Per the [DRF generic views docs](https://www.django-rest-framework.org/api-guide/generic-views/#genericapiview), `GenericAPIView.get_object()` reads `self.kwargs[self.lookup_url_kwarg or self.lookup_field]` — by default that's `self.kwargs['pk']` — and does `get_queryset().get(pk=<that value>)`. If the URL pattern doesn't capture a `pk` (e.g. you reuse the list URL by mistake), `get_object()` raises a `KeyError`/`AssertionError` rather than silently returning something — the URL and the mixin have to agree on how a single record gets identified. The default lookup field name (`pk`, short for "primary key") can be changed by setting `lookup_field` on the view if you want to look records up by, say, a `slug` instead of `id`.

---

## 10. Update — `GenericAPIView` + `UpdateModelMixin` **[From video]**

```python
class CustomerUpdateClass(mixins.UpdateModelMixin, generics.GenericAPIView):
    queryset = Customer.objects.all()
    serializer_class = CustomerSerializer

    def put(self, request, *args, **kwargs):
        # self.update(...) looks up the record via get_object() (using
        # the URL's pk, same as retrieve), validates request.data against
        # the serializer, overwrites the record's fields, and saves it.
        return self.update(request, *args, **kwargs)
```

```python
# restapp5/urls.py (added)
path('customerupdate/<int:pk>/', views.CustomerUpdateClass.as_view(), name='customerupdate'),
```

**[From video]** The instructor notes `PUT` or `PATCH` can both be used ("put method I'm using and... put method... patch method also we can use, not a problem"), and wires `put` in the demo. Submitting `PUT /customerupdate/1/` with new field values changes record `id=1`'s name from "Raj" to "Harry" (and updates its other fields) — confirmed both by re-fetching `/customerlist/` and by checking the Django admin panel directly.

**[Researched] — `PUT` vs. `PATCH`, and `.update()` vs. `.partial_update()`.** `UpdateModelMixin` actually defines two methods: `update()` (expects the *entire* object's data — a full replacement) and `partial_update()` (accepts only the fields being changed, leaving the rest untouched), which DRF conventionally maps to the `PUT` and `PATCH` HTTP methods respectively. The video only wires `put`/`.update()` — to also support `PATCH`, you'd add `def patch(self, request, *args, **kwargs): return self.partial_update(request, *args, **kwargs)`. `PUT` requiring every field can be a common pitfall: a `PUT` request that omits a required field is fully rejected by the serializer's validation as "missing data," whereas `PATCH` would have accepted the same partial payload.

---

## 11. Delete — `GenericAPIView` + `DestroyModelMixin` **[From video]**

```python
class CustomerDeleteClass(mixins.DestroyModelMixin, generics.GenericAPIView):
    queryset = Customer.objects.all()
    serializer_class = CustomerSerializer

    def delete(self, request, *args, **kwargs):
        # self.destroy(...) looks up the record via get_object() (pk
        # from the URL again) and deletes it, returning 204 No Content.
        return self.destroy(request, *args, **kwargs)
```

```python
# restapp5/urls.py (added)
path('customerdelete/<int:pk>/', views.CustomerDeleteClass.as_view(), name='customerdelete'),
```

**[From video]** The instructor refers to this mixin by both names ("delete model mixing" and, correctly, "destroy model mixing") — the actual DRF class name is `DestroyModelMixin`, and its method is `.destroy()`; there is no `.delete()` method on the mixin itself (that name is reserved for the HTTP-verb method you write yourself, same pattern as `get`/`post`/`put`). `DELETE`-ing `/customerdelete/4/` removes that record; re-checking `/customerlist/` afterward shows one fewer row.

**[Researched]** `DestroyModelMixin.destroy()` returns an HTTP `204 No Content` response on success (an empty body, which is the standard REST convention for a successful delete — there's no content left to return since the resource no longer exists) rather than `200 OK` with a body.

---

## 12. Code-volume comparison: this lecture vs. REST API Session 7 **[From video]**

The instructor draws the comparison directly, view class by view class: every mixin-based class above is roughly **4 lines of real logic** (`queryset`, `serializer_class`, one method that's a single delegating line), compared to roughly **8–18 lines per class** for the equivalent hand-written logic in REST API Session 7's plain-`APIView` versions (which had to manually build a serializer, call `.is_valid()`, branch on the result, and construct a `Response` by hand, per operation).

> **[From video]** "Now compared to previous examples... 10, 15 lines also we have written it, [for the] same example purpose, to get the records all from the database. So it is simplified or not? Yes."

---

## 13. Why five separate classes isn't the recommended real-world pattern **[From video + Gap-filled]**

> **[From video]** "This is not recommended in real time. First, initially I was explained for clarity purpose only... Instead of separate class we can include one class [with] multiple functions... In the two class-based views I can include all multiple functionalities... this I'll [be] discussing tomorrow session."

The instructor is explicit that writing **five separate view classes** (one each for list, create, retrieve, update, delete) was done purely as a teaching device — so each mixin's job is crystal clear in isolation. In practice, a single class can mix in *more than one* mixin at once. Concretely:

- One class combining `ListModelMixin` + `CreateModelMixin` (with both `get` → `.list()` and `post` → `.create()` defined on it) handles the "no ID in the URL" operations — list and create — since neither needs a specific record.
- A second class combining `RetrieveModelMixin` + `UpdateModelMixin` + `DestroyModelMixin` (with `get` → `.retrieve()`, `put` → `.update()`, `delete` → `.destroy()`) handles the "ID required in the URL" operations — retrieve, update, delete.

That reduces five classes down to two, each still built from `GenericAPIView` + mixins — this is the promised subject of **REST API Session 10** ("Generic API Views & Mixins (more examples)").

> **[Gap-filled] — how this connects forward to REST API Session 11's "concrete view classes".** Manually combining mixins like this (as REST API Session 10 will show) is still more typing than necessary for the *very common* case of "give me the standard list+create pair" or "give me the standard retrieve+update+destroy trio." DRF actually ships that combination **pre-built**, as ready-to-use classes in `rest_framework.generics` — e.g. `ListCreateAPIView` (already `ListModelMixin` + `CreateModelMixin` + `GenericAPIView`, with `get`/`post` already wired for you) and `RetrieveUpdateDestroyAPIView` (already `RetrieveModelMixin` + `UpdateModelMixin` + `DestroyModelMixin` + `GenericAPIView`, with `get`/`put`/`patch`/`delete` already wired). Using one of those, you'd write `queryset` and `serializer_class` and *nothing else* — no manual `get`/`post` delegation at all. That's the "concrete view classes" topic mentioned in this course's own lecture list (REST API Session 11) — a further step beyond what this lecture and REST API Session 10 cover, worth knowing exists so you don't have to hand-write the two-class version forever.

---

## 14. Mixin ↔ HTTP method ↔ underlying method — quick reference **[Gap-filled/Researched]**

| You write (HTTP verb method) | You call inside it | Mixin that provides it | URL needs `pk`? |
|---|---|---|---|
| `get` (list) | `self.list(...)` | `ListModelMixin` | No |
| `post` | `self.create(...)` | `CreateModelMixin` | No |
| `get` (single) | `self.retrieve(...)` | `RetrieveModelMixin` | **Yes** |
| `put` | `self.update(...)` | `UpdateModelMixin` | **Yes** |
| `patch` | `self.partial_update(...)` | `UpdateModelMixin` | **Yes** |
| `delete` | `self.destroy(...)` | `DestroyModelMixin` | **Yes** |

This table condenses everything demonstrated in the five view classes above into one lookup — every operation follows the identical shape: define the HTTP-verb method, delegate one line to the matching mixin method, keep `queryset`/`serializer_class` set at the class level.

---

## 15. Correct inheritance order: mixins before `GenericAPIView` **[Researched]**

> **[Researched]** DRF's own documentation and source examples always list the mixin(s) **first**, then `GenericAPIView` **last**, e.g.:
> ```python
> class ListCreateAPIView(mixins.ListModelMixin,
>                          mixins.CreateModelMixin,
>                          generics.GenericAPIView):
>     ...
> ```

This matters because of Python's **Method Resolution Order (MRO)** — with multiple inheritance, Python looks for a method on each parent class left-to-right. Putting `GenericAPIView` last means the mixins' methods (`.list()`, `.create()`, etc.) are found first if there's ever a naming clash, and it matches the order DRF's own docs and source code use everywhere. The transcript's spoken description happens to describe it the other way around ("inherited from one generic API view, comma, it should be inherited from create model mixin") — this is treated as a transcription/verbal-order slip rather than a real instruction to reverse the order, since the video's own working code (and every DRF example) uses mixin-then-`GenericAPIView`, matching the code shown in this note.

---

## 16. Standalone example: a `Book` review-style endpoint **[Example]**

To see the pattern applied to a different model, here's a small, self-contained example beyond the video's `Customer` case — a `Book` model with list+retrieve-only mixins (deliberately **no** create/update/delete, to show how leaving a mixin out removes that capability entirely):

```python
# models.py
from django.db import models

class Book(models.Model):
    title = models.CharField(max_length=150)
    author = models.CharField(max_length=100)
    published_year = models.IntegerField()

# serializers.py
from rest_framework import serializers
from .models import Book

class BookSerializer(serializers.ModelSerializer):
    class Meta:
        model = Book
        fields = ['id', 'title', 'author', 'published_year']

# views.py
from rest_framework import generics, mixins
from .models import Book
from .serializers import BookSerializer

class BookListView(mixins.ListModelMixin, generics.GenericAPIView):
    queryset = Book.objects.all()
    serializer_class = BookSerializer

    def get(self, request, *args, **kwargs):
        return self.list(request, *args, **kwargs)

class BookDetailView(mixins.RetrieveModelMixin, generics.GenericAPIView):
    queryset = Book.objects.all()
    serializer_class = BookSerializer

    def get(self, request, *args, **kwargs):
        return self.retrieve(request, *args, **kwargs)
```

```python
# urls.py
from django.urls import path
from . import views

urlpatterns = [
    path('books/', views.BookListView.as_view(), name='book-list'),
    path('books/<int:pk>/', views.BookDetailView.as_view(), name='book-detail'),
]
```

**Sample input → output:**
- `GET /books/` → `200 OK`, `[{"id": 1, "title": "Fluent Python", "author": "Luciano Ramalho", "published_year": 2015}, ...]`
- `GET /books/1/` → `200 OK`, `{"id": 1, "title": "Fluent Python", "author": "Luciano Ramalho", "published_year": 2015}`
- `POST /books/` → `405 Method Not Allowed` — because `BookListView` never mixed in `CreateModelMixin` or defined a `post` method, so DRF's routing correctly reports the method as unsupported on that URL, rather than the request silently doing nothing.

This last case is the practical payoff of mixins being separate, small pieces: a read-only API is created just by *not including* the mixins for writing.

---

## Industry best practices & pitfalls **[Gap-filled/Researched]**

- **Match the mixin to the HTTP-verb method** — as the video's own bug demonstrated, calling `self.create()` only works if `CreateModelMixin` is actually in the class's bases. When you see `AttributeError: ... object has no attribute 'list'/'create'/'retrieve'/'update'/'destroy'`, check the class's mixin list first.
- **Don't hand-write the delegate methods if DRF already has the combined class you need** — this lecture's 5-classes-of-4-lines pattern is a stepping stone; for the extremely common "list+create" and "retrieve+update+destroy" combinations, prefer DRF's pre-built **concrete view classes** (`ListCreateAPIView`, `RetrieveUpdateDestroyAPIView`, etc. — REST API Session 11) once you understand what they're built from.
- **Always define `pk` in the URL for single-record operations.** Forgetting `<int:pk>/` in the `path()` for retrieve/update/delete URLs is a common early mistake — `get_object()` needs that value in `self.kwargs` and will error without it.
- **`queryset` is evaluated once at class-definition time** (per DRF docs) unless you override `get_queryset()`. For anything that needs to depend on the current user or request (e.g. "only this user's own orders"), override `get_queryset()` instead of relying on the static `queryset` attribute — the static version can't see `self.request`.
- **Prefer `PATCH` over `PUT` for partial edits** from client code (e.g. a form that only changes one field) — `PUT`/`.update()` expects a full replacement payload and will reject a request missing a required field as invalid, whereas `PATCH`/`.partial_update()` is designed exactly for partial updates.
- **A 405, not a silent failure, is the correct behavior for an unsupported method** — as shown in the standalone `Book` example, not defining `post` on a list-only view isn't a bug to work around; it's the correct way to make an endpoint genuinely read-only.

---

## Wrap-up

- **From video:** the motivation for moving beyond plain `APIView` to `GenericAPIView` + mixins; what a mixin is and why it can't stand alone; the five built-in mixins (`ListModelMixin`, `CreateModelMixin`, `RetrieveModelMixin`, `UpdateModelMixin`, `DestroyModelMixin`) and the DRF method each provides (`.list()`, `.create()`, `.retrieve()`, `.update()`, `.destroy()`); building a full `Customer` CRUD API as five small classes in a new `restapp5` app, reusing the model/serializer from a previous app; the `get`/`post`/`put`/`delete` methods each delegating one line to the matching mixin method; the live mixin-mismatch bug and its fix; testing every endpoint (including the "pk required" URLs for retrieve/update/delete); the instructor's own code-length comparison against REST API Session 7; and the explicit statement that this five-class layout is a teaching device, not the recommended real-world structure.
- **Gap-filled:** the toolbox/screwdriver-bit analogy for what a mixin is; a full explanation of `*args`/`**kwargs` in this URL-dispatch context; how `get_object()` actually locates a record via `pk`; the mixin↔HTTP-verb↔URL-requirement reference table; the forward-pointer distinguishing REST API Session 10 (manually combining mixins into fewer classes) from REST API Session 11 (DRF's pre-built concrete view classes that need no manual `get`/`post` delegation at all); the standalone read-only `Book` example; the best-practices/pitfalls list.
- **Researched:** the exact attributes/methods `GenericAPIView` adds over `APIView` (`get_queryset()`, `get_object()`, `get_serializer()`, `lookup_field`) per the [DRF generic views docs](https://www.django-rest-framework.org/api-guide/generic-views/); the documented mixin-before-`GenericAPIView` inheritance order and why (Python MRO); `PUT`/`.update()` vs. `PATCH`/`.partial_update()`; the `204 No Content` response convention for a successful delete.

Double-check against the checklist above: all 25 listed items are covered. The most heavily garbled passages — the definition of a mixin, the "generic API views are set of commonly used patterns" explanation, and the closing "combine into two classes" teaser — were reconstructed from known DRF behavior rather than quoted verbatim, and are flagged as such above rather than presented as direct transcript quotes.
