# REST API Session 12 — ViewSets: ModelViewSet, ReadOnlyModelViewSet & Routers

Source: `transcripts/restapi/REST API-12.txt`
Covers: a short recap of the plain `ViewSet` + `DefaultRouter` example from the previous session, then the main topic — `ModelViewSet` (a ViewSet that auto-generates all the CRUD actions for a model in about two lines of code) and `ReadOnlyModelViewSet` (the same idea, but restricted to `list`/`retrieve` only), built and tested end-to-end with a new `restapp8` app and a `Manager` model.

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official DRF/Django docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. Recap: plain `ViewSet` + `Router` from the previous session **[From video]**

The instructor opens by summarizing what the *previous* lecture (REST API Session 11, concrete view classes — continuing into a first look at `ViewSet`) had already built: a hand-written `CustomerViewSet` where each action (`list`, `create`, `retrieve`, `update`, `destroy`) was a method the developer wrote out explicitly, similar in spirit to the `APIView`/generic-view classes from earlier lectures, but grouped into **one class** instead of one class per HTTP verb.

> So, Viewset example, we will discuss in the last session... Now, we can see we create a View here. Create a customer for a Viewset. This is the list of values. Create a date. Destroy like this.

That class was wired up to a URL not with a single `path()` per action, but through a **router**:

> Viewsets actually use router configuration in the URL section. So, I included here default router I imported and router object I created. And I registered my Viewset into router. So, `router.register(...)` — customer Viewset, `Views.CustomerViewSet`, base name equals to customer. And automatically when we send the request `router.urls` will execute. Router will queue [route] the response from the Viewset automatically here.

**[Gap-filled]** — what this means in plain terms: a `ViewSet` groups related view logic (all the operations you'd perform on `/customers/` and `/customers/<id>/`) into a single class, and a **router** is a helper object that looks at that class and automatically generates the URL patterns for it — you never hand-write `path('customers/', ...)`, `path('customers/<id>/', ...)` etc. yourself. This is the foundation the rest of this lecture builds on, so it's worth restating clearly before moving to `ModelViewSet`:

```python
# restapp7/views.py (recap of the PREVIOUS lecture's plain ViewSet)
from rest_framework import viewsets
from rest_framework.response import Response
from .models import Customer
from .serializers import CustomerSerializer

class CustomerViewSet(viewsets.ViewSet):
    # each action is a normal method you write yourself
    def list(self, request):
        queryset = Customer.objects.all()
        serializer = CustomerSerializer(queryset, many=True)
        return Response(serializer.data)

    def create(self, request):
        serializer = CustomerSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        serializer.save()
        return Response(serializer.data)

    def retrieve(self, request, pk=None):
        customer = Customer.objects.get(pk=pk)
        serializer = CustomerSerializer(customer)
        return Response(serializer.data)

    def update(self, request, pk=None):
        customer = Customer.objects.get(pk=pk)
        serializer = CustomerSerializer(customer, data=request.data)
        serializer.is_valid(raise_exception=True)
        serializer.save()
        return Response(serializer.data)

    def destroy(self, request, pk=None):
        customer = Customer.objects.get(pk=pk)
        customer.delete()
        return Response(status=204)
```

```python
# project urls.py — the router recap
from rest_framework.routers import DefaultRouter
from restapp7 import views

router = DefaultRouter()
router.register('customer', views.CustomerViewSet, basename='customer')

urlpatterns = [
    # ... other paths ...
    path('', include(router.urls)),
]
```

`router.register(prefix, viewset, basename)`:
- `prefix` — the URL segment (`'customer'` → requests go to `/customer/` and `/customer/<pk>/`).
- `viewset` — the `ViewSet`/`ModelViewSet` class to route to.
- `basename` — the root used to build the internal URL *names* (`'customer-list'`, `'customer-detail'`) used by `reverse()`; DRF can normally infer this from `queryset.model.__name__` if you don't pass it, but it's good practice to always pass it explicitly, and it's **required** if your view doesn't define a static `queryset` attribute (e.g. it only overrides `get_queryset()`).

## 2. What a `ViewSet` actually is, and why `ModelViewSet` exists **[Gap-filled]**

A plain `ViewSet` (used above) still makes you write every action method by hand — it saves you from repeating the URL-routing boilerplate of one `path()` per verb, but not from repeating the actual CRUD logic. Every single "customer" or "manager" or "product" ViewSet you write ends up with nearly identical `list`/`create`/`retrieve`/`update`/`destroy` bodies that just swap the model and serializer.

DRF's `ModelViewSet` removes *that* remaining duplication too. It's the natural next step after the `mixins`/generic view classes covered in REST API Sessions 9–11 (`ListCreateAPIView`, `RetrieveUpdateDestroyAPIView`, etc.) — except instead of picking two separate classes (one for the "list" URL, one for the "detail" URL) and wiring each to its own `path()`, `ModelViewSet` bundles **all five actions into one class**, and a router turns that one class into **both** URLs automatically.

> **Analogy:** think of the generic-view classes from REST API Sessions 9–11 as pre-built "kits" you still have to assemble two of (one kit per URL) and connect with wires (`path()` entries) yourself. A `ModelViewSet` + `Router` is the same kit, fully assembled, that plugs itself into the wall.

## 3. `ModelViewSet` — actions and required attributes **[From video]**

> Model Viewset. Working with model Viewset. See, the model Viewset class inherits from generic APA View. And includes implementation for various actions by mixing in the behavior of the various mixing classes.

**[From video, reconstructed from "generic APA View" / "mixing classes"]** — "generic APA View" is the transcript's garbling of **`GenericAPIView`**, and "mixing classes" means **mixin classes**. So, in plain terms: `ModelViewSet` is built on top of `GenericAPIView` (the same base class the generic views from REST API Session 9 use) plus a set of **mixin classes**, each of which contributes one action's logic.

> So, the actions provided by the model Viewset class are list, retrieve, create, update, [partial] update, destroy... Because model Viewset extends generic APA View, you will normally need to provide at least the queryset and serializer class attribute. Two things [that are] compel[so]ry we have to provide.

**Actions provided by `ModelViewSet`:**

| Action | HTTP verb (via router) | What it does |
|---|---|---|
| `list` | `GET /manager/` | Return all records |
| `retrieve` | `GET /manager/<pk>/` | Return one record |
| `create` | `POST /manager/` | Create a new record |
| `update` | `PUT /manager/<pk>/` | Replace a record's fields entirely |
| `partial_update` | `PATCH /manager/<pk>/` | Update only the fields sent |
| `destroy` | `DELETE /manager/<pk>/` | Delete a record |

**Required attributes:** only two —
- `queryset` — which records the view operates on (a built-in `QuerySet` object, e.g. `Manager.objects.all()`).
- `serializer_class` — which serializer converts those records to/from JSON.

> Look at this in the previous example. We have seen Views section, in the Views section we have written lot of code... And these two [are] compulsory [and] required. Remaining things not required in the context of model Viewsets only.

This is the headline benefit the instructor keeps returning to: compare this to the `APIView` class from REST API Sessions 7–8, where every one of `list`/`create`/`retrieve`/`update`/`destroy` needed its own hand-written method (with its own `queryset`, its own serializer instantiation, its own `Response`) — `ModelViewSet` collapses all of that into two class attributes.

**[Researched]** — per the [DRF ViewSets docs](https://www.django-rest-framework.org/api-guide/viewsets/#modelviewset), the actual class hierarchy is:

```python
# (simplified, from DRF's own source — rest_framework/viewsets.py)
class ModelViewSet(mixins.CreateModelMixin,
                    mixins.RetrieveModelMixin,
                    mixins.UpdateModelMixin,
                    mixins.DestroyModelMixin,
                    mixins.ListModelMixin,
                    GenericViewSet):
    """
    A viewset that provides default create(), retrieve(), update(),
    partial_update(), destroy() and list() actions.
    """
    pass
```

...and `GenericViewSet` itself is `class GenericViewSet(ViewSetMixin, generics.GenericAPIView)`. So the transcript's "inherits from GenericAPIView and mixes in various mixin classes" is accurate — `ModelViewSet` is literally an empty class body (`pass`) that just combines the same `CreateModelMixin` / `RetrieveModelMixin` / `UpdateModelMixin` / `DestroyModelMixin` / `ListModelMixin` mixins used by the generic views in REST API Session 9, with `ViewSetMixin` layered on top to make the whole thing routable by a `Router` instead of needing individual `path()` entries.

## 4. `ReadOnlyModelViewSet` **[From video]**

> Read only model Viewset means we can only retrieve the records. We cannot able to do any actions. The read only model Viewset class also inherits from generic APA View. As with the model Viewset, it also includes implementation of various actions. But unlike model Viewset, only provides the read only actions like list and retrieve... These methods [update, partial update, destroy] will not be available in this context of read only model Viewset.

`ReadOnlyModelViewSet` is the same idea as `ModelViewSet`, but only mixes in `ListModelMixin` and `RetrieveModelMixin` — so a router built from it only ever exposes `GET` (list and detail). There is no way to `POST`, `PUT`, `PATCH`, or `DELETE` through it — those HTTP methods simply aren't wired up, so calling them returns `405 Method Not Allowed`.

**[Researched]** — matches the [DRF source](https://www.django-rest-framework.org/api-guide/viewsets/#readonlymodelviewset):

```python
class ReadOnlyModelViewSet(mixins.RetrieveModelMixin,
                            mixins.ListModelMixin,
                            GenericViewSet):
    """
    A viewset that provides default list() and retrieve() actions.
    """
    pass
```

This is genuinely useful (not just a teaching toy) for **read-only public endpoints** — e.g. exposing a product catalog or a list of published articles to anonymous clients, where you want browsing but never want the API itself to accept writes.

## 5. Building the example: a new `restapp8` app **[From video]**

The instructor deliberately starts a **brand-new app** for this demo ("if you create a new application, it will be more clear what we are doing") rather than reusing the previous lecture's `restapp7`.

```bash
python manage.py startapp restapp8
```

**[Gap-filled]** — this needs to be registered in the project's `settings.py` before Django will recognize it, exactly like every app since Lecture 2:

```python
# settings.py
INSTALLED_APPS = [
    # ...
    'restapp8',
]
```

### The `Manager` model

> Manager model I am creating, same like what we have created in the [previous] app... Name address made page. That's all.

**[Gap-filled] — reconstructing a garbled line.** "Name address made page" is not a real phrase; cross-referencing it against later parts of the transcript where the instructor actually enters data ("I address is a... so ID mail... age is like 30... RAK... gmail.com... 23") makes clear the intended fields are **name, address, mail (email), and age** — "made page" is a mis-transcription of "mail, age". The model is reconstructed as:

```python
# restapp8/models.py
from django.db import models

class Manager(models.Model):
    name = models.CharField(max_length=100)
    address = models.CharField(max_length=255)
    mail = models.EmailField()
    age = models.IntegerField()

    def __str__(self):
        return self.name
```

### Registering with the admin site

```python
# restapp8/admin.py
from django.contrib import admin
from .models import Manager

admin.site.register(Manager)
```

> admin.py I am preparing... this is common code... it is manager model, and I'm taking manager admin.

### Migrations and seeding via the admin panel

```bash
python manage.py makemigrations
python manage.py migrate
```

> Now let's [start] the server, admin panel it will go... I would include some records. I'm including two. Two is enough.

The instructor logs into `/admin/`, opens the `Manager` table, and manually adds two `Manager` records through the Django admin UI (the same superuser-login-then-add-row workflow from Lecture 24) purely to have data to test against once the API is wired up — **not** by posting through the API yet.

## 6. The serializer — `ManagerSerializer` **[From video]**

> Serializer.py file is responsible to convert from model instances into JSON format only. Even it is going to take responsible[for] Python data type[s] which [are] taken from user, from front end, and that will be converted into JSON — JSON to model instances. It's a mediator, [the] actual serializer file.

**[Gap-filled]** — this is a restatement of what serializers do (covered in depth in REST API Sessions 2–6): they're the two-way translator between Django model instances (Python objects) and JSON (what an HTTP client sends/receives) — "serialization" is model → JSON, "deserialization" is JSON → model.

```python
# restapp8/serializers.py
from rest_framework import serializers
from .models import Manager

class ManagerSerializer(serializers.ModelSerializer):
    class Meta:
        model = Manager
        fields = '__all__'   # include every field on the Manager model
```

> If you include model serializer then we can create only model. Preparing model based serializer class... No need to include all the fields inside this... fields equals to all I'm taking.

Using `ModelSerializer` (rather than a plain `Serializer` with every field typed out by hand, from REST API Session 2) means DRF inspects the `Manager` model and auto-generates matching serializer fields — `fields = '__all__'` tells it to include every model field rather than listing them individually.

## 7. The view — `ManagerModelViewSet` **[From video]**

> View section is the main view section because we are working with actually model view sets only here... it is generated from generic KPV [GenericAPIView] only, but some less code is required to use it here. Only two lines of code is enough in the context of model view sets only.

```python
# restapp8/views.py
from rest_framework import viewsets
from .models import Manager
from .serializers import ManagerSerializer

class ManagerModelViewSet(viewsets.ModelViewSet):
    queryset = Manager.objects.all()          # "correct set object" — which records this view works on
    serializer_class = ManagerSerializer       # which serializer converts them to/from JSON
```

> Two things are required: one is queryset — [a] built-in object. Manager dot objects dot [all] — so all records I want to read. Next... serializer class is equals to the manager serializer class I'm supply[ing] — that's all needs to be [done].

The instructor explicitly contrasts this with the earlier `restapp7` `ViewSet` example: *"look at this — implies rest app 7 how the views logic is available... this is ViewSet, ViewSet only here, it's not a model view set... now this time we can observe here ViewSet dot model view set."* — i.e. same import (`from rest_framework import viewsets`), but swapping `viewsets.ViewSet` (hand-written actions) for `viewsets.ModelViewSet` (auto-generated actions) is the entire difference.

## 8. Wiring it up with the router **[From video]**

> We have to include what already we have included — that is router... I need to go with rest app 8 directly, project level, [where the] router configuration [is] already [placed]... this view is coming from... not [the] customer view set — [it's now the] manager model view set. So in the URL, manager model view set... base name is what actually manager.

The router lives at the **project level** (same `urls.py` used for the `restapp7` example in the previous lecture) — only the registered app/view/basename change:

```python
# project urls.py
from django.contrib import admin
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from restapp8 import views

router = DefaultRouter()
router.register('manager', views.ManagerModelViewSet, basename='manager')

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include(router.urls)),
]
```

**[Gap-filled]** — nothing about the *router itself* changes between a `ViewSet` and a `ModelViewSet`; `router.register()` doesn't care which flavor of ViewSet it's pointed at, because both expose the same action methods (`list`, `create`, `retrieve`, `update`, `partial_update`, `destroy`) under the hood — `ModelViewSet` just auto-implements them instead of requiring you to write them.

## 9. Testing it — the browsable API **[From video]**

> When I click on this link, now you can see automatically we are reading this... we are reading [record] ID number one... we can also post the record... let's post... [give a] name... address... mail... age is 30... when I try to post, now you can see we have posted the record. Again, if you want to see this model view set, one to three records are available here — even you can go back to database admin panel and check this... it is creating and listing also.

The instructor runs `python manage.py runserver` and visits the router-generated URL directly in the browser (DRF's **browsable API**, from REST API Session 4), demonstrating that *all five actions work with zero view code beyond the two attributes*:

- **`GET /manager/`** → lists the two seeded records automatically.
- **`POST /manager/`** (via the auto-generated HTML form in the browsable API) → adds a third record (name, address, mail, age = 30) — confirmed both in the API response and by checking the same row in `/admin/`.
- **`PUT /manager/<pk>/`** on one record → the instructor changes the name to "Rock"/"RAK", the email, and the age to 23; the updated record is reflected both via the API and in the admin panel.
- **`DELETE /manager/<pk>/`** on the third record → it's removed; `GET /manager/` afterward shows only the original two records again.

> If you want to update or partial update... based [on] record I need to use... then one record I'm using... this record — do you want to delete, you can delete, and do you want to update, you can update it here.

**[Gap-filled]** — the browsable API auto-generates an HTML `<form>` for `POST` (on the list page) and separately for `PUT`/`DELETE` (on the detail page) purely because `ModelViewSet` exposes those actions — this is the exact same automatic-form behavior covered for `APIView`/generic views in REST API Session 8, it's just now available for *five* actions from *one* class instead of needing separate view classes per action.

> **[Example]** — sample request/response for the `POST` shown above:

```http
POST /manager/ HTTP/1.1
Content-Type: application/json

{
    "name": "Sagar",
    "address": "Hyderabad",
    "mail": "sagar@example.com",
    "age": 30
}
```

```json
HTTP/1.1 201 Created
Content-Type: application/json

{
    "id": 3,
    "name": "Sagar",
    "address": "Hyderabad",
    "mail": "sagar@example.com",
    "age": 30
}
```

## 10. Switching to `ReadOnlyModelViewSet` **[From video]**

> Let's go to view section and we have to include read only model view set... class name is manager model view set — same thing — which is generated from `views.viewsets.read_only_model_view_set`... queryset is common and serializer class is common. So now, for [the] time being, I'm commenting this one and I'm going to use only this view set.

```python
# restapp8/views.py — swapped to read-only
from rest_framework import viewsets
from .models import Manager
from .serializers import ManagerSerializer

class ManagerModelViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = Manager.objects.all()
    serializer_class = ManagerSerializer
```

> Now we can see we are using read only view set — only [it] makes you only reading, list out and retrieve it. So we cannot able to do operations like create, update, delete — all these things — only read only... there is no option like post method with the text box... only [you can] read all records or read [a] particular record — you cannot able to write it.

Only the class the `ManagerModelViewSet` inherits from changes (`viewsets.ModelViewSet` → `viewsets.ReadOnlyModelViewSet`); the `queryset` and `serializer_class` attributes, and the router registration, stay identical. Restarting the server and revisiting the same URL, the instructor shows the `POST` form is now gone from the list page and the detail page no longer offers update/delete — only `GET` (list and detail) still works.

## 11. `ViewSet` family — quick comparison **[Gap-filled / Researched]**

| Class | Actions available | You write |
|---|---|---|
| `APIView` (REST API Sessions 7–8) | whichever HTTP methods you define (`get`, `post`, ...) | full logic for each method, per class |
| Generic + mixin views (REST API Session 9) | whichever mixins you combine, split across 2 classes/URLs | `queryset`, `serializer_class` per class; still 1 `path()` per class |
| `ViewSet` (recap above) | whichever actions you define (`list`, `create`, ...) | full logic for each action, but one class handles both URLs via a router |
| `GenericViewSet` | none by default — a base to mix your own mixins into | pick your own mixins + `queryset`/`serializer_class` |
| `ModelViewSet` | all of `list`, `retrieve`, `create`, `update`, `partial_update`, `destroy` | just `queryset` + `serializer_class` |
| `ReadOnlyModelViewSet` | only `list`, `retrieve` | just `queryset` + `serializer_class` |

**[Researched]** — per the [DRF routers docs](https://www.django-rest-framework.org/api-guide/routers/), `DefaultRouter` (used throughout this lecture and the previous one) also automatically adds a browsable **API root view** listing every registered ViewSet, plus optional `.json`/`.api` format suffixes on each URL — a plain `SimpleRouter` gives you the generated URLs without that root view. Since none of that distinction was mentioned in the video, it's worth knowing `DefaultRouter` is the more "batteries-included" of DRF's two built-in router classes.

## 12. Industry best practices & pitfalls **[Researched]**

> **Best practice — don't reach for `ModelViewSet` automatically.** It's the fastest way to expose full CRUD for a model that genuinely needs it, but if an endpoint should never support one of the actions (e.g. records should never be deleted through the API), prefer `GenericViewSet` combined with only the specific mixins you want, or `ReadOnlyModelViewSet`, rather than shipping a `ModelViewSet` and relying on permission checks alone to hide the unwanted actions. Fewer exposed actions is one less thing that can be misused or need securing.

> **Pitfall — forgetting `basename`.** If your view overrides `get_queryset()` instead of setting a static `queryset` attribute, `router.register()` will raise an error unless you pass `basename` explicitly, since DRF can no longer infer a model name to derive it from.

> **Pitfall — `update` vs `partial_update` confusion.** A `PUT` request (`update`) is expected to include *every* required field (it conceptually replaces the whole resource); `PATCH` (`partial_update`) only requires the fields being changed. Sending a `PUT` with only some fields will fail validation against a `ModelSerializer`'s required fields unless those fields are optional on the model.

> **Pitfall — accidentally exposing writes.** Because `ModelViewSet` enables all five actions the instant you write two lines, it's easy to forget that anonymous, unauthenticated users can `POST`/`PUT`/`DELETE` through it too, unless permission/authentication classes (the next lecture's topic) are explicitly configured. This lecture's example has no permission classes set at all — fine for a local demo, unsafe for anything public.

## 13. Standalone extra example **[Example]**

A `ModelViewSet` for a simple `Book` model, showing the same two-attribute pattern applied to a different domain, plus how it looks wired to a router:

```python
# models.py
class Book(models.Model):
    title = models.CharField(max_length=200)
    author = models.CharField(max_length=100)
    price = models.DecimalField(max_digits=6, decimal_places=2)

# serializers.py
class BookSerializer(serializers.ModelSerializer):
    class Meta:
        model = Book
        fields = '__all__'

# views.py
class BookModelViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer

# urls.py
router = DefaultRouter()
router.register('books', BookModelViewSet, basename='book')
urlpatterns = [path('', include(router.urls))]
```

This automatically produces:

| URL | Methods | Maps to |
|---|---|---|
| `/books/` | `GET`, `POST` | `list`, `create` |
| `/books/<pk>/` | `GET`, `PUT`, `PATCH`, `DELETE` | `retrieve`, `update`, `partial_update`, `destroy` |

Switching `viewsets.ModelViewSet` to `viewsets.ReadOnlyModelViewSet` here would immediately drop `POST`, `PUT`, `PATCH`, and `DELETE` from that table, leaving only the two `GET` rows — exactly the behavior demonstrated in section 10 above, applied to a catalog-style read-only "browse the books" endpoint.

## 14. What's next **[From video]**

> In the next session will go for the new topic — that is authentication and permissions... that is important in REST — [like] basic authentication, token authentication, session authentication, remote user authentication, custom authentication... those applications will discuss on Monday session, mostly next week.

The video closes noting this was "a very small topic" (ModelViewSet and ReadOnlyModelViewSet), and that the course moves next into **authentication and permissions** — basic authentication first, then token, session, remote-user, and custom authentication over the following sessions (matching REST API Sessions 13–17 in this unit).

---

## Wrap-up

- **From video:** recap of the plain `ViewSet` + `DefaultRouter` pattern from the previous session; `ModelViewSet` — its inheritance from `GenericAPIView` via mixin classes, its six actions (list/retrieve/create/update/partial_update/destroy), and its two required attributes (`queryset`, `serializer_class`); `ReadOnlyModelViewSet` and its restriction to list/retrieve only; a full build of a new `restapp8` app (model, admin, migrations, serializer, view, router) and a live demo of every CRUD action through the browsable API, followed by swapping to read-only and confirming writes are blocked; the announced move to authentication & permissions next.
- **Gap-filled:** the plain-language distinction between `ViewSet`/`GenericViewSet`/`ModelViewSet`/`ReadOnlyModelViewSet`; reconstruction of the garbled `Manager` model fields (name, address, mail, age) from later transcript context; an explanation of `router.register()`'s `basename` parameter; a comparison table across all the view styles covered since REST API Session 7; common pitfalls (forgetting `basename`, `PUT` vs `PATCH`, unguarded writes with no permission classes yet).
- **Researched:** DRF's actual `ModelViewSet`/`ReadOnlyModelViewSet` source (confirming the mixin composition described in the video), the difference between `DefaultRouter` and `SimpleRouter`, and best-practice guidance on choosing the narrowest ViewSet/mixin combination an endpoint actually needs.

Double-check against the completeness pass: recap of Session-53 `ViewSet`+`Router` ✓; `ModelViewSet` definition, actions, required attributes ✓; `ReadOnlyModelViewSet` ✓; new app/model/admin/migrations walkthrough ✓; serializer ✓; view ✓; router/URL wiring ✓; live list/create/retrieve/update/delete demo via browsable API ✓; read-only swap demo ✓; next-session preview (authentication & permissions) ✓. Nothing from the transcript appears to have been left out.
