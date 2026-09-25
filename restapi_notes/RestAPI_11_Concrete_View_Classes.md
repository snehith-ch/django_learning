# REST API Session 11 — Concrete View Classes (recap) and Working with ViewSets & Routers

Source: `transcripts/restapi/REST API-11.txt`
Covers: the session opens with a one-line recap of the *previous* session's topic — DRF's **concrete view classes**, the fully pre-built generic views that do CRUD with almost no code — then pivots immediately into this session's actual new topic: **ViewSets**, a way of combining several related views into one class, and **routers**, which auto-generate the URL configuration for a ViewSet instead of you writing `path()` entries by hand. The session is a live build: a new app (`restapp7`) reusing the `Customer` model from the previous app, a hand-written `ViewSet` class with `list`/`retrieve`/`create`/`update`/`destroy` methods, and a `DefaultRouter` wired up in the project's `urls.py`.

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official DRF/Django docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## A note on this lecture's title **[Gap-filled]**

This assignment's working title was "Concrete view classes," matching the DRF course's provisional running order. Having read the transcript in full, that title only fits the *opening thirty seconds*: the instructor briefly recaps that "we were discussing concrete view classes... in a simple way, we perform CRUD operations using all these concrete view classes" — a one-sentence callback to the previous session — and then says explicitly: **"in today's session, I'm going to discuss a new topic... that is called working with ViewSets."** Everything else in the transcript (the vast majority of the session) is building a hand-written `ViewSet` class and wiring up a `DefaultRouter`.

To stay honest to both the assignment and the actual content, this lecture does both:
1. A properly researched recap section on **concrete view classes** (Section 1) — since the transcript itself doesn't re-teach them, this section is built from DRF's official generic-views documentation, filling the gap the video skipped over.
2. Full, from-video notes on **ViewSets and routers** (Sections 2 onward) — the actual new material taught in this session.

If a later lecture's notes also cover "ViewSets" from scratch, it's likely because the *next* session (per this video's own closing line, "there is some `ModelViewSet` concept also there... how to work with it I'm going to discuss tomorrow") goes further into `ModelViewSet` — the fully-automatic ViewSet variant, analogous to how concrete view classes are the fully-automatic version of the generic mixin views from the previous two sessions.

---

## 1. Recap — concrete view classes **[Researched]**

> "We were discussing about concrete view classes. And in a simple way, we perform CRUD operations using all these concrete view classes."

That's the transcript's entire mention of the topic — a callback, not a lesson. Here's what concrete view classes actually are, reconstructed from the [DRF generic views documentation](https://www.django-rest-framework.org/api-guide/generic-views/), to properly close the loop the previous two sessions opened.

The last two lectures (generic API views & mixins) showed that DRF's CRUD building blocks come in layers:

1. **`APIView`** (REST API Sessions 7–8) — the base class-based view. You write every `get()`/`post()`/`put()`/`delete()` method yourself, by hand, including all the serializer/queryset logic inside each one.
2. **`GenericAPIView` + mixins** (REST API Sessions 9–10) — `GenericAPIView` handles the common plumbing (`queryset`, `serializer_class`, `get_object()`, pagination), and separate mixin classes (`ListModelMixin`, `CreateModelMixin`, `RetrieveModelMixin`, `UpdateModelMixin`, `DestroyModelMixin`) each supply *one* action's logic as a method you still have to call from your own `get()`/`post()`/etc. You still write a small amount of boilerplate wiring the mixin methods to HTTP verbs.
3. **Concrete view classes** — the topic name-checked at the top of this transcript — go one step further: DRF ships classes that have **already combined `GenericAPIView` with the right mixins**, and already wired the mixin methods to the correct HTTP verbs internally. You don't write `get()`/`post()` at all. You just set two class attributes — `queryset` and `serializer_class` — and the class handles the rest.

<dfn>Concrete view class</dfn> — a ready-made DRF view class that is `GenericAPIView` plus one or more mixins, pre-assembled, so it needs (in the simplest case) zero custom methods — only `queryset` and `serializer_class`.

The full set of concrete view classes DRF ships:

| Class | HTTP methods it handles | Built from |
|---|---|---|
| `CreateAPIView` | POST | `GenericAPIView` + `CreateModelMixin` |
| `ListAPIView` | GET (collection) | `GenericAPIView` + `ListModelMixin` |
| `RetrieveAPIView` | GET (single item) | `GenericAPIView` + `RetrieveModelMixin` |
| `DestroyAPIView` | DELETE | `GenericAPIView` + `DestroyModelMixin` |
| `UpdateAPIView` | PUT, PATCH | `GenericAPIView` + `UpdateModelMixin` |
| `ListCreateAPIView` | GET (collection), POST | `GenericAPIView` + `ListModelMixin` + `CreateModelMixin` |
| `RetrieveUpdateAPIView` | GET, PUT, PATCH | `GenericAPIView` + `RetrieveModelMixin` + `UpdateModelMixin` |
| `RetrieveDestroyAPIView` | GET, DELETE | `GenericAPIView` + `RetrieveModelMixin` + `DestroyModelMixin` |
| `RetrieveUpdateDestroyAPIView` | GET, PUT, PATCH, DELETE | `GenericAPIView` + `RetrieveModelMixin` + `UpdateModelMixin` + `DestroyModelMixin` |

```python
# customer/views.py
from rest_framework import generics
from .models import Customer
from .serializers import CustomerSerializer

# Handles GET (list all customers) and POST (create a customer).
# No list()/create() methods needed — ListCreateAPIView already has them.
class CustomerListCreate(generics.ListCreateAPIView):
    queryset = Customer.objects.all()
    serializer_class = CustomerSerializer

# Handles GET (one customer), PUT/PATCH (update), DELETE — all on one URL like /customers/3/
class CustomerDetail(generics.RetrieveUpdateDestroyAPIView):
    queryset = Customer.objects.all()
    serializer_class = CustomerSerializer
```

```python
# customer/urls.py
from django.urls import path
from .views import CustomerListCreate, CustomerDetail

urlpatterns = [
    path('customers/', CustomerListCreate.as_view()),
    path('customers/<int:pk>/', CustomerDetail.as_view()),
]
```

That's the entire CRUD API for a model — two classes, four lines of attributes total, still needing explicit `path()` entries. That last point matters: **concrete view classes remove the need to write view logic, but you still write the URLs by hand.** This session's actual topic, ViewSets + routers, removes *that* remaining piece too — see Section 3.

> **[Researched] — when to reach for a concrete view class vs. a mixin combination.** Per the DRF docs, most real APIs can use the concrete classes directly — they cover the common CRUD shapes. Drop back down to `GenericAPIView` + individual mixins only when you need to *override* one action's behavior (e.g. a `create()` that also sends a welcome email) while leaving the others as pre-built — a concrete class can still be subclassed and have one method overridden, so this isn't an either/or; concrete classes are the default, mixins are for the exceptions.

---

## 2. What is a ViewSet? **[From video]**

> "Django REST Framework allows you to combine the logic for a set of related views into a single view class — that is called a ViewSet."

<dfn>ViewSet</dfn> — a class-based view that groups the logic for *all* of a resource's related operations (list, retrieve, create, update, delete) into **one class**, instead of spreading them across separate view classes the way `APIView` and the concrete/generic views do.

The instructor draws a direct comparison to other web frameworks:

> "In other frameworks, you may also find a conceptually similar implementation — something like resources or controllers. But in Python [DRF], we have ViewSets."

This is a close paraphrase of DRF's own documentation, which opens its ViewSets guide with almost this exact framing — so it's fair to treat it as accurate, standard terminology rather than the instructor's own invention.

A crucial distinction the video makes carefully:

> "A ViewSet class is simply a type of class-based view that does **not** provide method handlers such as a `get()` method or `post()` method — instead it provides *actions*, such as `list()` and `create()` methods."

**[Gap-filled] — why this distinction matters.** With `APIView` (and even with the mixins), your methods are named after HTTP verbs: `get()`, `post()`, `put()`, `delete()`. A ViewSet flips this: its methods are named after what they *do* — `list()`, `retrieve()`, `create()`, `update()`, `partial_update()`, `destroy()` — not after the HTTP verb that triggers them. The mapping from "HTTP verb + URL shape" to "which action method runs" isn't hardcoded on the class itself; it's decided later, by whatever wires the class into a URL. That's the router's job (Section 3).

> "The method handlers for a ViewSet are only bound to the corresponding actions at the point of finalizing the view — using the `.as_view()` method — typically by explicitly registering the views in a ViewSet with a router class."

**[Gap-filled] — unpacking that sentence.** A plain `APIView.as_view()` call always produces the same view function, because `get()` always means GET and `post()` always means POST — the binding is fixed by convention. A `ViewSet`, by contrast, doesn't have that fixed convention, so *someone* has to say "for this URL, GET should call `list()`, but for that URL, GET should call `retrieve()`." A router does exactly this — for each URL it generates, it calls `.as_view({'get': 'list', 'post': 'create'})` (and similar dictionaries for the other URLs) behind the scenes, explicitly mapping each HTTP verb to the ViewSet action method that should run. You can technically do this `.as_view({...})` call by hand without a router at all, but the video (and DRF in practice) always pairs ViewSets with routers because writing that mapping manually for every URL defeats the purpose.

---

## 3. Why routers exist **[From video]**

> "Django REST Framework allows a router class to automatically determine the URL configuration for your request... Routers are used with ViewSets in Django REST Framework to auto-configure the URLs. Routers provide a simple, quick, and consistent way of wiring ViewSet logic to a set of URLs. A router automatically maps the incoming request to the proper ViewSet action based on the request method — GET or POST."

<dfn>Router</dfn> — a DRF class that, given a ViewSet, automatically generates the full set of URL patterns that resource needs (list, detail, create, update, delete) — without you writing individual `path()` lines for each one.

**[Gap-filled] — the contrast this is making, spelled out.** Every earlier lecture in this course (plain Django and DRF alike) has you writing `urls.py` by hand: one `path()` per view, one name per path. That's fine when each view only does one thing. A ViewSet, by design, does *several* things (list, retrieve, create, update, destroy) in a single class, and each of those five actions normally needs its own URL shape:

| Action | Typical URL | HTTP verb |
|---|---|---|
| `list` | `/customers/` | GET |
| `create` | `/customers/` | POST |
| `retrieve` | `/customers/3/` | GET |
| `update` | `/customers/3/` | PUT |
| `partial_update` | `/customers/3/` | PATCH |
| `destroy` | `/customers/3/` | DELETE |

Writing all of that by hand for every ViewSet, every time, is exactly the kind of repetitive boilerplate a router exists to eliminate — you register the ViewSet once, and the router generates all of the above automatically.

```python
# project-level urls.py
from rest_framework.routers import DefaultRouter
from restapp7 import views

router = DefaultRouter()
router.register('customer', views.CustomerViewSet, basename='customer')

urlpatterns = [
    # ... other paths ...
    path('', include(router.urls)),
]
```

`router.register('customer', views.CustomerViewSet, basename='customer')` is the one line that replaces the whole table above.

> **[Researched] — `DefaultRouter` vs. `SimpleRouter`.** The video only uses `DefaultRouter`, from `rest_framework.routers`. Per the [DRF docs](https://www.django-rest-framework.org/api-guide/routers/), DRF actually ships two router classes: `SimpleRouter` generates just the list/detail URLs described above, while `DefaultRouter` does everything `SimpleRouter` does *plus* adds an automatically-generated **API root view** — a browsable index page listing links to every registered ViewSet — and supports an optional `.json` format suffix on URLs (e.g. `/customers.json`). That root/index page is exactly what the video shows appearing when the server starts: a page listing a "customer view set" link before anything is clicked into.

---

## 4. ViewSet attributes: `basename`, `action`, `detail`, `name` **[From video]**

The instructor lists four attributes every ViewSet action carries, and demonstrates printing them to the terminal from inside the `list()` method to show their live values.

| Attribute | Meaning (as given in the video) |
|---|---|
| `basename` | "The base to use for the URL names that are created." If you don't set it explicitly when calling `router.register()`, DRF auto-generates one from the ViewSet's `queryset` attribute. **If the ViewSet has no `queryset` attribute at all, you must set `basename` yourself** — otherwise the router has nothing to derive a name from and registration fails. |
| `action` | "The name of the current action" — i.e., which ViewSet method is running for this particular request: `list`, `create`, `retrieve`, `update`, `partial_update`, or `destroy`. |
| `detail` | A boolean. `True` means the current action operates on a *single* object (retrieve/update/destroy); `False` means it operates on the *collection* (list/create). |
| `name` | A display name for the view, used in things like the browsable API's page headings. |

```python
# Printed inside the ViewSet, purely to demonstrate these attributes live:
class CustomerViewSet(viewsets.ViewSet):
    def list(self, request):
        print(self.basename)   # e.g. "customer"
        print(self.action)     # "list"
        print(self.detail)     # False — list() acts on the whole collection
        print(self.name)       # a display name DRF derives, e.g. "Customer List"
        ...
```

> **[Gap-filled] — why `basename` specifically matters so much.** The router uses `basename` to build the *name* of each generated URL (e.g. `customer-list`, `customer-detail`) — the same kind of name you'd pass to `{% url %}` or `reverse()` (Lecture 21's named-URL pattern, now generated automatically instead of typed). Get the `basename` wrong or leave it unset with no `queryset` on the class, and the router literally cannot build those names, which is why the video calls it out as something you "compulsorily" must set in that situation.

---

## 5. Building it — project setup **[From video]**

The instructor builds a brand-new app for this demo rather than reusing the previous one, but deliberately reuses that previous app's model and admin registration rather than retyping them.

```bash
# 1. Create a new app dedicated to this ViewSets demo
python manage.py startapp restapp7
```

```python
# settings.py — register the new app
INSTALLED_APPS = [
    # ...
    'restapp7',
]
```

```python
# restapp7/models.py — copied over unchanged from the previous app (restapp6)
from django.db import models

class Customer(models.Model):
    name = models.CharField(max_length=100)
    address = models.CharField(max_length=200)
    email = models.EmailField()
    age = models.IntegerField()
```

```python
# restapp7/admin.py — same registration pattern as before
from django.contrib import admin
from .models import Customer

admin.site.register(Customer)
```

```bash
python manage.py makemigrations
python manage.py migrate
python manage.py createsuperuser   # only if one doesn't already exist
python manage.py runserver
```

The instructor then logs into `/admin/`, and manually inserts three test records (name, address, email, age) directly through the Django admin, purely so there's data to exercise once the ViewSet is wired up.

<div class="code-note">This step is a good moment to remember why the admin exists at all (Lecture 24): it's a free, auto-generated data-entry UI Django builds from your models — perfect for exactly this kind of "I just need some rows to test against" situation, with zero extra code.</div>

---

## 6. The serializer **[From video]**

Reused directly from the previous app, unchanged:

```python
# restapp7/serializers.py
from rest_framework import serializers
from .models import Customer

class CustomerSerializer(serializers.ModelSerializer):
    class Meta:
        model = Customer
        fields = ['id', 'name', 'address', 'email', 'age']
```

The video doesn't re-explain `ModelSerializer` here since it was covered in earlier sessions (REST API Sessions 2–6) — worth a quick reminder: `ModelSerializer` auto-generates the serializer's fields from the model, the same time-saving relationship `ModelForm` has to a plain `Form` in ordinary Django.

---

## 7. The `CustomerViewSet` class **[From video]**

```python
# restapp7/views.py
from rest_framework.response import Response
from restapp7.models import Customer
from restapp7.serializers import CustomerSerializer
from rest_framework import status
from rest_framework import viewsets


class CustomerViewSet(viewsets.ViewSet):
    """
    A hand-written ViewSet: every action method below is written out
    explicitly (unlike ModelViewSet, next session, which provides all
    of these automatically). Compare each method here to the matching
    mixin from REST API Sessions 9–10 — the logic is identical; only the
    method's *name* (an action, not an HTTP verb) has changed.
    """

    def list(self, request):
        # GET /customer/  — return every row
        queryset = Customer.objects.all()
        serializer = CustomerSerializer(queryset, many=True)  # many=True: serializing a list of objects, not one
        return Response(serializer.data)

    def retrieve(self, request, pk=None):
        # GET /customer/<pk>/  — return a single row by primary key
        customer = Customer.objects.get(id=pk)
        serializer = CustomerSerializer(customer)
        return Response(serializer.data)

    def create(self, request):
        # POST /customer/  — validate incoming data and insert a new row
        serializer = CustomerSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response({"message": "data inserted"}, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    def update(self, request, pk=None):
        # PUT /customer/<pk>/  — replace an existing row's data
        customer = Customer.objects.get(id=pk)
        serializer = CustomerSerializer(customer, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response({"message": "updated"})
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    def destroy(self, request, pk=None):
        # DELETE /customer/<pk>/  — remove a row
        customer = Customer.objects.get(id=pk)
        customer.delete()
        return Response({"message": "record deleted"})
```

Line-by-line notes on what's new here versus the mixins from REST API Sessions 9–10:

- **`viewsets.ViewSet`** is the base class being inherited from — the plainest ViewSet base, providing *no* automatic behavior at all (contrast with `ModelViewSet`, previewed at the end of the video for next session, which provides every one of these five methods automatically from just `queryset` + `serializer_class`, the same relationship concrete view classes have to plain mixins).
- Method **names** (`list`, `retrieve`, `create`, `update`, `destroy`) are fixed, conventional action names DRF and its routers recognize — naming them anything else means a router can't automatically wire them up.
- `pk=None` as the default for `retrieve`/`update`/`destroy` — these three operate on one specific object, so they need a primary key from the URL; `list`/`create` don't take a `pk` at all since they act on the whole collection.
- The video also mentions `partial_update` as an available action ("if you want to go for partial update, you can go for partial update also") for PATCH-style partial edits, alongside `update` for full PUT-style replacement — the same PUT vs. PATCH distinction from earlier `UpdateModelMixin` coverage — but doesn't write its body out in the demo, since it's identical to `update()` in shape.
- The 201/400 status-code pattern (`status.HTTP_201_CREATED` on success, `status.HTTP_400_BAD_REQUEST` with `serializer.errors` on failure) is unchanged from the `CreateModelMixin` pattern in REST API Session 9/52 — a ViewSet's `create()` isn't a new concept, just the same logic under a new method name.

> **[Gap-filled] — a real bug hiding in this code.** `Customer.objects.get(id=pk)` (used in `retrieve`, `update`, and `destroy`) raises `Customer.DoesNotExist` — an unhandled Python exception — if no row with that `pk` exists, which Django turns into an ugly, generic **500 Internal Server Error** rather than a clean "not found" response. The video doesn't demonstrate this failure case. DRF's own generic views avoid it by using `get_object_or_404()` internally, which raises `Http404` instead — DRF automatically converts that into a proper `404 Not Found` JSON response. The fix, applied consistently to every method that looks an object up by `pk`:
> ```python
> from django.shortcuts import get_object_or_404
>
> def retrieve(self, request, pk=None):
>     customer = get_object_or_404(Customer, id=pk)
>     serializer = CustomerSerializer(customer)
>     return Response(serializer.data)
> ```
> This is a common beginner pitfall with any hand-written `retrieve`/`update`/`destroy` — always assume a client might request a `pk` that doesn't exist, and handle it explicitly rather than letting the ORM's exception bubble up unhandled.

---

## 8. Wiring the router into `urls.py` **[From video]**

The instructor deliberately does **not** create an app-level `urls.py` for `restapp7` — everything is configured directly at the project level, since the router already produces every URL the ViewSet needs.

```python
# project/urls.py
from django.contrib import admin
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from restapp7 import views

router = DefaultRouter()
router.register('customer', views.CustomerViewSet, basename='customer')

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include(router.urls)),   # every URL for CustomerViewSet, generated automatically
]
```

- `DefaultRouter()` — creates the router object.
- `router.register('customer', views.CustomerViewSet, basename='customer')` — registers the ViewSet under the URL prefix `customer`, with `basename='customer'` used to name the generated URLs (`customer-list`, `customer-detail`, etc., per Section 4).
- `path('', include(router.urls))` — `router.urls` is a ready-made list of URL patterns (the router built it from the registration above); `include()` (Lecture 7's app-URL pattern, here applied to a router instead of an app's `urls.py`) drops that whole list into the project's `urlpatterns` at the root path.

> **[Gap-filled] — could this have used an app-level `urls.py` instead?** Yes — the video says so directly ("this router configuration... you can do it into the application-level URLs also, no problem"). Registering the router inside `restapp7/urls.py` and then `include()`-ing *that* file from the project level (the standard per-app pattern from Lecture 7 onward) works identically; the demo just skips that extra layer for a single-app project. For a real multi-app project, keeping each app's router in its own `urls.py` (mirroring how ordinary app URLs are organized) is the more scalable, conventional choice — a project-level `urls.py` that directly imports every app's views, as this demo does, doesn't scale past a handful of apps.

---

## 9. Testing it in the browsable API **[From video]**

With the server running, the instructor demonstrates the full CRUD cycle through DRF's browsable API (first introduced in REST API Session 4):

1. **Root listing** — visiting the base URL shows a page (generated by `DefaultRouter`'s API-root feature) with a link to the registered `customer` ViewSet.
2. **List** — clicking through shows every customer record, confirming `list()` and the router's GET-to-`list` mapping both work.
3. **Detail** — clicking into one record's URL calls `retrieve()`, returning just that one row.
4. **Create — failure case** — POSTing a malformed record (the demo deliberately omits required comma/formatting in the browsable form) fails JSON parsing before it ever reaches `create()`. The response is an **HTTP 400 Bad Request** with a "JSON parse error" message and a line number. The instructor also points out that the same terminal printouts from Section 4 confirm what actually happened: `detail: False`, `action: list` (i.e., the request never got past initial parsing to reach the `create` action at all) — a useful debugging habit: **when a request behaves unexpectedly, check which action and detail value it actually triggered**, since that alone can reveal the request never reached the code you expected it to.
5. **Create — success case** — POSTing a correctly-formatted record succeeds, returns `{"message": "data inserted"}` with **201 Created**, and the new row is confirmed both via the ViewSet's own list view and independently in the Django admin (Lecture 24) — a nice sanity check, since both are reading the same underlying database table.
6. **Delete** — deleting a record by its detail URL removes it; confirmed again both through the ViewSet's list view and the admin.
7. **Update** — PUT-ing new values onto an existing record's detail URL (renaming and changing other fields) succeeds and is reflected in both places.

> **Industry best practice.** Testing every CRUD path through the browsable API — not just the "happy path" — is exactly what the instructor does here by deliberately breaking the POST request once. Confirming error handling (a bad request returns a proper 400 with a useful message, not a raw 500 crash) is just as important as confirming the successful cases, and DRF's browsable API makes this easy to do without any separate tooling like Postman or curl.

---

## 10. What's next — `ModelViewSet` and authentication **[From video]**

The instructor closes by naming the next two topics directly:

> "This is about model ViewSets... how to work with it, in tomorrow's lecture I'm going to discuss — and also authentication and permissions in Django REST Framework."

> **[Researched] — a preview of `ModelViewSet`, for continuity.** Per the [DRF docs](https://www.django-rest-framework.org/api-guide/viewsets/#modelviewset), `ModelViewSet` is to plain `ViewSet` what a concrete view class (Section 1) is to `GenericAPIView` + mixins: it already implements `list`, `retrieve`, `create`, `update`, `partial_update`, and `destroy` for you, so the entire `CustomerViewSet` written by hand in Section 7 could instead be written as:
> ```python
> class CustomerViewSet(viewsets.ModelViewSet):
>     queryset = Customer.objects.all()
>     serializer_class = CustomerSerializer
> ```
> — five class-based methods replaced by two class attributes, the exact same trajectory Section 1 describes DRF taking from `APIView` all the way down to concrete view classes, now applied to ViewSets instead of single-purpose views.

---

## [Example] A second, standalone worked example: a `Book` ViewSet with a custom action

The video's demo only shows the five standard CRUD actions. A common next question once you understand basic ViewSets is: *can a ViewSet expose an endpoint that isn't one of the five standard actions?* Yes — via the `@action` decorator, which the video does not mention but is directly relevant once you're comfortable with everything above.

```python
# books/models.py
from django.db import models

class Book(models.Model):
    title = models.CharField(max_length=200)
    author = models.CharField(max_length=100)
    is_available = models.BooleanField(default=True)
```

```python
# books/views.py
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.response import Response
from .models import Book
from .serializers import BookSerializer


class BookViewSet(viewsets.ViewSet):
    def list(self, request):
        books = Book.objects.all()
        serializer = BookSerializer(books, many=True)
        return Response(serializer.data)

    def retrieve(self, request, pk=None):
        book = Book.objects.get(pk=pk)
        return Response(BookSerializer(book).data)

    # A CUSTOM action — not one of list/retrieve/create/update/destroy.
    # detail=True means this acts on ONE book (so the URL needs a pk),
    # like retrieve/update/destroy, not like list/create.
    @action(detail=True, methods=['post'])
    def mark_borrowed(self, request, pk=None):
        book = Book.objects.get(pk=pk)
        book.is_available = False
        book.save()
        return Response({"message": f"'{book.title}' marked as borrowed"}, status=status.HTTP_200_OK)
```

```python
# project urls.py
router.register('books', BookViewSet, basename='book')
# router.urls now automatically includes an extra route:
#   POST /books/<pk>/mark_borrowed/   ->  BookViewSet.mark_borrowed
```

**Sample request → response:**

```http
POST /books/3/mark_borrowed/
```
```json
{
  "message": "'Clean Code' marked as borrowed"
}
```

**Realistic use case:** any action that's clearly *about* one resource but isn't a plain create/read/update/delete — approving an order, resetting a user's password, marking a book borrowed/returned, publishing a draft post. `@action` lets that logic live right alongside the rest of the resource's ViewSet, and the router picks up the extra URL automatically, with zero manual `path()` entries — the same automatic-URL benefit Section 3 describes, extended to custom behavior.

---

## Wrap-up

- **From video:** the previous session's "concrete view classes" recap line; the definition of a ViewSet and how it differs from `APIView`/mixins (actions, not HTTP-verb methods); why routers exist and what `DefaultRouter` does; the four ViewSet attributes (`basename`, `action`, `detail`, `name`); building a full `restapp7` demo app reusing the `Customer` model; a hand-written `CustomerViewSet` with `list`/`retrieve`/`create`/`update`/`destroy`; wiring it up with `router.register()` + `include(router.urls)` at the project level with no app-level `urls.py`; testing every CRUD path (including a deliberate error case) through the browsable API; the announced next topics, `ModelViewSet` and authentication/permissions.
- **Gap-filled:** an explicit note on why this lecture's title had to be adjusted from the assignment's working title; unpacking what "method handlers are only bound to actions via `.as_view()`" actually means; the `basename` auto-generation logic; why an app-level `urls.py` was skipped and when you'd still want one; a real unhandled-exception bug in the video's `retrieve`/`update`/`destroy` code (`.get()` vs. `get_object_or_404()`) with a fix.
- **Researched:** the full concrete-view-classes table and code (Section 1) — the actual topic named at the top of the transcript, but never re-taught in it; `SimpleRouter` vs. `DefaultRouter`; a `ModelViewSet` preview; the `@action` decorator with a complete standalone `Book` example.

Double-check against the transcript: every attribute mentioned (`basename`, `action`, `detail`, `name`), every ViewSet method written (`list`, `retrieve`, `create`, `update`, `destroy`, plus the passing mention of `partial_update`), the full app-setup sequence (new app, reused model/admin, migrations, superuser, admin data entry), the router registration and `include()`, and every browsable-API test step (list, detail, failed POST, successful POST, delete, update) are all covered above. Nothing in the transcript was skipped.
