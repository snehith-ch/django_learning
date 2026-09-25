# REST API Session 2 — Setting Up a DRF Project and Building the First Serializer-Backed API

Source: `transcripts/restapi/REST API-2.txt`

Covers: the first hands-on lecture of the Django REST Framework (DRF) unit. Picking up right after REST API Session 1's theory-only introduction, this lecture builds a small real project (`MyRestProject`, app `rest_app`) from the ground up: confirming `djangorestframework` is installed and registered, defining a plain Django model (`Employees`), registering it in the admin site, running migrations, creating a superuser, adding a few records through the Django admin — and then, for the first time, introducing **serializers**: a DRF-specific concept that converts database records into JSON so they can be returned directly from an API view, with no HTML template involved at all.

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official Django/DRF docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## A note on this transcript **[Gap-filled]**

This transcript is unusually garbled even by this course's standards — the auto-transcription renders "employees" as "implies"/"imply" throughout, "queryset" as "Coryset"/"choricit"/"kure set", "API"/"APIView" as "APA"/"APAV"/"KPA", "serializer" as "irrelevant"/"realizer", and "pip install" as "PAAP space install". The underlying lesson is clear and internally consistent once decoded, and the code shown on screen (model fields, serializer class, view, URLs) is reconstructed here from that decoding plus standard DRF conventions, not invented from scratch. Where a exact variable/class name was ambiguous, a conventional DRF-style name is used and called out.

---

## 1. Recap and where this lecture starts **[From video]**

> In the last session of REST API, we just discuss about the introduction to REST API. And in this session, I am going to start with the examples of REST API.

REST API Session 1 covered REST API theory only (what an API is, what REST means, why DRF exists on top of Django). This lecture is the first one that writes actual code. The instructor says a project has *already* been created before this recording: a Django project named **`MyRestProject`**, containing one app named **`rest_app`** — the video opens that existing project in PyCharm rather than running `django-admin startproject` on screen.

> **[Gap-filled] — recreating this from scratch.** If you're following along and don't already have this project, the setup is the same two commands used throughout this whole course:
> ```bash
> django-admin startproject MyRestProject
> cd MyRestProject
> python manage.py startapp rest_app
> ```
> Nothing about *creating* a Django project or app changes for DRF — DRF is not a separate framework with its own project scaffolding; it's a library that plugs into an ordinary Django project (this was the headline point of REST API Session 1).

## 2. Installing `djangorestframework` **[From video]**

> We just simply go to Terminal window and pip space install space Django REST framework. That's all. So that's going to install... Already have installed, you only [need] to install again after installation of completion.

Decoded: the install command is the standard `pip` install, run once per environment:

```bash
pip install djangorestframework
```

The instructor notes this was already covered conceptually in REST API Session 1 and treats it as a checkpoint rather than new material — before creating an API, first *verify* two things are true for the project:

1. The `djangorestframework` package is actually installed in the active Python environment.
2. `'rest_framework'` has been added to `INSTALLED_APPS` in the project's `settings.py`.

> **[Gap-filled] — the package name vs. the import name.** This trips people up: you `pip install` the package as **`djangorestframework`** (one word, matching PyPI's listing), but everywhere in Python code you `import` it as **`rest_framework`** (with an underscore). They are not typos of each other — it's simply that the installable package name and the importable module name inside it are different strings, which is common in the Python packaging ecosystem (another example: you `pip install beautifulsoup4` but `import bs4`).

## 3. Registering `rest_framework` in `INSTALLED_APPS` **[From video]**

> Go to settings.py of your project. You can include your application here. And also you have to include `rest_underscore_framework` here.

In `MyRestProject/settings.py`:

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',

    'rest_framework',   # DRF itself — must be listed to use any DRF class/feature
    'rest_app',         # this project's own app
]
```

> **[Gap-filled] — why this step is non-negotiable.** `INSTALLED_APPS` is how Django discovers what's part of a project — not just your own apps, but third-party packages like DRF that ship their own app config (templates for the browsable API, DRF-specific settings, management integration, etc.). Skip this line and importing `rest_framework.views` or `rest_framework.serializers` in your code will still *work* at the Python level, but DRF's own machinery (like the browsable API's HTML templates, covered later in this unit) won't be wired into the project, and some DRF features will misbehave or fail outright. This is the same `INSTALLED_APPS` mechanism used for every app in this course since Lecture 1 — DRF doesn't get special treatment here, it's added exactly like any other app.

## 4. Defining the `Employees` model **[From video]**

> First I have to go to models... Here I'm going to create one model for employees model... I'm creating imply ID. Employ ID equals to `models.IntegerField`... imply name equals to `models.CharField`... Employee address equals to `models.CharField`... email equals to `models.EmailField`... I'm giving max length equals to 20 for [the] name, max length equals to 20 [for] address, and email field max length equals to 30.

The instructor stresses this step is *identical* to plain Django model creation from earlier in the course — DRF changes nothing about how models are written. In `rest_app/models.py`:

```python
from django.db import models

class Employees(models.Model):
    emp_id = models.IntegerField()
    emp_name = models.CharField(max_length=20)
    emp_address = models.CharField(max_length=20)
    email = models.EmailField(max_length=30)

    def __str__(self):
        # Controls how each record displays as text (e.g. in the admin list) —
        # here, every Employees instance shows as its name rather than "Employees object (1)"
        return self.emp_name
```

- `IntegerField()` — a whole-number column (used here for `emp_id`).
- `CharField(max_length=20)` — a short text column; `max_length` is **required** for `CharField` (it becomes the database column's size limit and a validation rule).
- `EmailField(max_length=30)` — a `CharField` subclass that additionally validates the value looks like a valid email address, used for `email`.
- `__str__` — a Python "dunder" (double-underscore) method that returns a human-readable string for an object; Django calls it whenever an instance needs to display as text (Django admin list views, `print()`, `{{ obj }}` in a template). This is the same pattern from Lecture 24's `ModelAdmin`/`__str__` coverage, reused here without any DRF-specific change.

> **[Gap-filled] — model naming: `Employees` vs `Employee`.** Django's own style guide and most real-world codebases name models in the **singular** (`Employee`, not `Employees`) — Django itself pluralizes automatically wherever it needs to (e.g. the default related-manager name, or `Employee.objects` reads naturally as "the employees manager"). The video's model is named `Employees` (plural); it still works correctly — Django doesn't enforce singular naming — but singular is the more common convention and reads better in generated code like `Employees.objects.all()` vs. the more idiomatic `Employee.objects.all()`. This note is kept because the transcript's naming is used consistently through the rest of this lecture and the ones that follow it in this unit.

> **[Gap-filled] — why no `id` field was declared.** Only `emp_id`, `emp_name`, `emp_address`, and `email` are explicitly defined, yet every Django model gets a primary key automatically. Per Django's docs, if a model doesn't declare its own primary-key field, Django **automatically adds an `id` field** (an auto-incrementing integer, `AutoField`) as the primary key. So this table actually has *two* integer-ish identity fields: the auto-added `id` (the real primary key) and the separately-declared `emp_id` (just an ordinary integer column with no special database role, presumably an internal "employee ID" business field). This is a common beginner trap: don't assume a custom `..._id` field you defined yourself is the primary key unless you explicitly set `primary_key=True` on it.

## 5. Registering the model in the Django admin **[From video]**

> Let's register into admin. So I'm including this model into admin panel... `admin.site.register` — your model is `Employees`. If you include this [in the] normal way, every record ... is considered as one [employee] object ... I want to represent every model instance in the Django admin panel in the string format ... [using] the `__str__` function.

In `rest_app/admin.py`:

```python
from django.contrib import admin
from .models import Employees

admin.site.register(Employees)   # plain registration — no custom ModelAdmin/ModelForm here
```

The instructor deliberately contrasts this with a customized registration (a `ModelAdmin` subclass with `list_display`, covered in earlier Django lectures) and says that's deferred for later — for now, a plain `admin.site.register(Employees)` is enough, and each record shows up in the admin's object list using whatever `__str__` returns (here, the employee's name) rather than Django's generic `"Employees object (1)"` default.

## 6. Migrations, superuser, and adding data through the admin **[From video]**

> Let's go to terminal window, make migrations and migrate ... `python manage.py makemigrations` ... `python manage.py migrate` ... Next ... create one super user ... `python manage.py createsuperuser`.

```bash
python manage.py makemigrations   # generates a migration file describing the new Employees table
python manage.py migrate          # applies it — creates the actual table in the database
python manage.py createsuperuser  # creates an admin login (username/email/password)
```

After running the server and logging into `/admin/` with the new superuser account, the instructor adds a handful of `Employees` records directly through the Django admin UI (the transcript garbles the actual sample names/addresses beyond recognition — decoded only as roughly three records, one apparently named "Rajesh"). Because the admin list displays each row via `__str__`, the object list shows employee names rather than raw IDs.

> **[Gap-filled] — this is the same admin workflow from Lecture 8 and Lecture 24.** `createsuperuser` and logging into `/admin/` to manage rows by hand were already covered when this course first introduced the Django admin — nothing here is DRF-specific. The point of redoing it in this lecture is just to get a handful of real rows into the database *before* writing any API code, since an API with no data to return isn't a useful demo.

## 7. Why there's no template this time **[From video]**

> Whatever the data is there in database that we retrieve and, after retrieval, we are rendering that into template only. So here, template is not there... Without template I am rendering all the employee details into ... API format.

This is the conceptual pivot point of the lecture. In every plain-Django view from earlier in the course, the pattern was: query the database → pass the result into a template's context → the template renders HTML → that HTML is the response. Here, there is **no template file created at all** for this view. The API is going to hand back the employee data as raw **JSON** text instead of an HTML page — the client (a browser, a mobile app, another server, `curl`, anything) receives structured data it can parse itself, rather than a finished web page meant only for human eyes.

> **[Gap-filled] — why this matters.** This is the core reason REST APIs exist as a separate thing from ordinary Django views (the theory REST API Session 1 introduced): an HTML page is only useful to a browser rendering it for a human. JSON is useful to *any* program — a mobile app, a JavaScript frontend, another backend service — because JSON is a generic, open, language-independent data format rather than a page layout. The "no template" observation in this lecture is the concrete, hands-on version of that idea.

## 8. Serialization and deserialization — the core new concept **[From video]**

> There is a concept called, in Python or in `rest_...`, serializers. What is serializers we will discuss now. Serializers allow complex data — such as querysets, model instances — to be converted into native Python data types. [That] can then easily be rendered into JSON format or XML format or other content types also.

Breaking this down term by term, exactly as the instructor builds it up:

- A **queryset** is what you get back from a call like `Employees.objects.all()` — a collection representing (potentially) many database rows. This is the same `QuerySet` object from Lecture 16 and Lecture 37's field-lookup coverage; nothing new about it here.
- A **model instance** is a single database row, represented as one Python object (e.g. one `Employees` row).
- Both a queryset and a model instance are described in this lecture as **complex data** — meaning data shaped in a way that's specific to Django's ORM, not something a generic external client (or even plain Python's `json` module) knows how to turn into text directly.
- **Native Python data types** means the built-in, ordinary Python types everyone already knows — `dict`, `list`, `str`, `int`, `float`, `bool` — the types Python's own `json` module *can* directly convert to JSON text.
- **Serialization**, per the instructor's definition, is the process of taking that complex data (querysets/model instances) and converting it into native Python types, so it can then be easily rendered into JSON (or XML, or other formats).
- **Deserialization** is described as the reverse direction: taking parsed/incoming data (e.g. JSON sent by a client) and converting it back into complex data (a model instance / something that can be saved to the database).

> Serialization means complex data — model instances or database records — will be converted into Python data types [and] this Python data types [is] easily rendered into JSON format... deserialization means this is Python data types, JSON representation, and converted back into ... complex data.

The instructor is explicit that this lecture only implements the **serialization** direction (reading data out of the database and returning it as JSON) — deserialization (accepting incoming JSON and saving it, i.e. handling `POST` requests) is deferred to a later lecture in this unit.

> **[Researched] — serializers, precisely, per the DRF docs.** Per the [official DRF documentation on serializers](https://www.django-rest-framework.org/api-guide/serializers/): "Serializers allow complex data such as querysets and model instances to be converted to native Python datatypes that can then be easily rendered into JSON, XML or other content types. Serializers also provide deserialization, allowing parsed data to be converted back into complex types, after first validating the incoming data." This is close to a verbatim match of what the instructor says — this section of the lecture is essentially teaching the DRF docs' own opening description of serializers, decoded from the transcript's noise. Note the docs' extra detail the video doesn't dwell on: deserialization also involves *validating* the incoming data before it's converted back into a complex type — validation is the mechanism that rejects malformed or missing data on writes (covered when this unit reaches `POST`/inserting data).

> **[Example] — serialization in one sentence, with a concrete before/after.**
> - **Before (complex data):** an `Employees` model instance living in the database, e.g. conceptually `Employees(emp_id=101, emp_name="Rajesh", emp_address="Hyderabad", email="rajesh@example.com")`.
> - **After serialization (native Python / JSON):**
>   ```json
>   {"id": 1, "emp_id": 101, "emp_name": "Rajesh", "emp_address": "Hyderabad", "email": "rajesh@example.com"}
>   ```
> The "complex" object (tied to Django's ORM, database rows, model methods) became a plain dictionary of strings/numbers — something any client, in any language, can parse.

## 9. Creating `serializers.py` and the `ModelSerializer` class **[From video]**

> Right now into your project, or into your application, we have to add a new file with the name called `serializer.py`. The name `serializers.py` is not compulsory, but better to give the proper name... First, how to import — from `rest_framework` import `serializers`... Next I'm going to convert our model instances — which model? [It's] coming from your application, that is `rest_app`... dot `models`... the `employees` model. Now let's create serializer class... `EmpSerializer` — this serializer is inherited from `serializers.ModelSerializer`.

The instructor explicitly compares this to `ModelForm` from earlier in the course: just as a `ModelForm` is a form class built automatically from a model's fields, a **`ModelSerializer`** is a serializer class built automatically from a model's fields — same underlying idea, applied to converting data to JSON instead of rendering an HTML form.

In `rest_app/serializers.py`:

```python
from rest_framework import serializers
from .models import Employees

class EmpSerializer(serializers.ModelSerializer):
    class Meta:
        model = Employees   # which model this serializer represents
        fields = '__all__'  # include every field defined on Employees
```

- `serializers.ModelSerializer` — a DRF base class that, similar to `ModelForm`, inspects a model and auto-generates the corresponding serializer fields, so you don't have to write out `emp_id = serializers.IntegerField()` etc. by hand for every column.
- The inner `class Meta:` — same pattern as `ModelForm`'s `Meta` class: a small nested class that tells the parent class *which* model to build from and *which* fields to include, without being itself a "real" field.
- `fields = '__all__'` — a shortcut meaning "include every field the model has." The alternative is an explicit list, e.g. `fields = ['emp_id', 'emp_name', 'email']`, to include only some fields.

> The transcript also notes an IDE warning: *"class name should be [in] camelCase... if your class name is two words, next-word starting letter is capital."* This is PyCharm flagging that the serializer class name should follow **PascalCase/CamelCase** convention (each word capitalized, no underscores) — standard Python class-naming style (PEP 8) — which `EmpSerializer` satisfies.

> **[Gap-filled] — `many=True` gets used one step later (in the view), but is set up right here conceptually.** A single `EmpSerializer` instance can serialize either **one** model instance (a single employee) or **many** at once (a whole queryset) — which mode it's in is controlled by a `many=True` argument passed when the serializer object is *created*, not by anything in the serializer class itself. This is covered concretely in the next section (the view), where `Employees.objects.all()` returns *multiple* rows, so `many=True` is required.

> **[Researched] — `fields = '__all__'` vs. an explicit list, and why it matters.** Per the [DRF `ModelSerializer` docs](https://www.django-rest-framework.org/api-guide/serializers/#specifying-fields-explicitly), `fields = '__all__'` is convenient for quick prototyping (like this lecture's first example) but is called out by DRF's own docs as something to be careful with: if the model later gains a sensitive field (a password hash, an internal flag, a cost field not meant for public API consumers), `'__all__'` will silently expose it through the API the moment it's added to the model, with no serializer change required. **Industry best practice** is to list fields explicitly once a project moves past a first prototype, e.g. `fields = ['id', 'emp_name', 'email']`, so adding a new model field never automatically exposes it through an API without a deliberate decision.

## 10. Creating the view — `APIView` and `Response` **[From video]**

> How to create a view ... I'm going to use `APIView` class — our view should be inherited from `APIView` only... `from rest_framework.views import APIView`... after that, I have to include the model also — `from rest_app.models import Employees`... `serializers.py`, the `EmpSerializer` class, is responsible to take model instances and render into JSON format. So in the view section... `from rest_app.serializers import EmpSerializer`.

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from .models import Employees
from .serializers import EmpSerializer

class EmpDetails(APIView):
    def get(self, request):
        emp = Employees.objects.all()             # QuerySet: all Employees rows, "complex data"
        serializer = EmpSerializer(emp, many=True) # many=True: serializing MULTIPLE records
        return Response(serializer.data)           # serializer.data is JSON-ready native Python data
```

Walking through each piece as the video introduces it:

- **`APIView`** — DRF's own base class for class-based views, imported from `rest_framework.views`. The instructor stresses this is DRF's equivalent of Django's own `View` class (from the class-based-view material back in Lecture 30–31): it supports `get()`, `post()`, `put()`, `patch()`, `delete()` methods the same way Django's CBVs do, and DRF dispatches an incoming request to whichever method matches the HTTP verb used.
- **`def get(self, request):`** — only `GET` is implemented in this lecture, because this view only *reads* data. The instructor explicitly says a `post()` method isn't written yet because that would mean accepting data *from* a client and saving it — that's deserialization, deferred to a later lecture.
- **`Employees.objects.all()`** — the same `.objects.all()` queryset call from Lecture 16 onward; nothing DRF-specific here. This is the "complex data" that needs serializing.
- **`EmpSerializer(emp, many=True)`** — constructs a serializer instance, handing it the queryset as the data to convert. `many=True` is required whenever the thing being serialized is a *collection* of several model instances (a queryset here) rather than a single instance — without it, DRF would try (and fail) to treat the whole queryset as one record.
- **`serializer.data`** — after construction, this attribute holds the converted, JSON-ready native Python data (the instructor demonstrates in the terminal that this is actually of type `ReturnList`, DRF's own list subclass — see the debugging section below).
- **`Response(...)`** — imported from `rest_framework.response`, **not** Django's own `HttpResponse` or `JsonResponse`. The instructor is explicit about this distinction: *"this is not a Django response, it is a serializer[/DRF] response."* DRF's `Response` object is content-negotiation-aware — in the browser it renders as DRF's browsable HTML API; requested via `curl`/`Accept: application/json`, it comes back as plain JSON.

> **[Gap-filled] — why `APIView` instead of Django's plain `View`.** Nothing stops you from writing an API-returning view using Django's ordinary `View` class and `JsonResponse` — but `APIView` (and DRF generally) provides things a plain Django view doesn't have to reimplement each time: automatic parsing of incoming request bodies in multiple formats, DRF's own request/response objects, browsable-API rendering in a browser for free, and (covered later in this unit) built-in hooks for authentication and permission checks. `APIView` is the DRF-native entry point that unlocks the rest of the framework's features; a plain Django `View` returning `JsonResponse` would work for this one simple `GET` example but would need to reinvent everything DRF adds as the project grows (auth, pagination, throttling — all future lectures in this unit).

## 11. Wiring up the URLs **[From video]**

> We don't have any URLs under the `rest_app`. Let's come to create one `urls.py` under the `rest_app`... `from django.urls import path`... `from rest_app import views`... `urlpatterns` equals to ... `path`... I'm not giving any name... `EmpDetails.as_view()`... this is a class-based view... as-view function is required.

`rest_app/urls.py`:

```python
from django.urls import path
from . import views

urlpatterns = [
    path('empdetails/', views.EmpDetails.as_view()),
]
```

`MyRestProject/urls.py` (project-level):

```python
from django.urls import path, include

urlpatterns = [
    path('', include('rest_app.urls')),
]
```

> **[Gap-filled] — why `.as_view()` is required here.** This is the exact same rule already covered for Django's own class-based views back in Lecture 30: a URL pattern's second argument must be a plain *function* that Django/DRF can call with a request, not a class. `EmpDetails` is a class, so `EmpDetails.as_view()` is called to produce that dispatching function — `.as_view()` is a `classmethod` that returns a function which, when invoked, creates an instance of `EmpDetails` and routes the request to the right method (`get`, `post`, etc.) based on the HTTP verb. Forgetting `.as_view()` and writing `path('empdetails/', views.EmpDetails)` is a common mistake that raises an error, because Django can't call a class directly the way it calls a view function.

## 12. Running the server and seeing the result **[From video]**

> Now you can see, this is JSON format... application type is what type? Content type is `application/json` type only, this is JSON format. Now this model instance type is converted into JSON format... Django REST framework window is opening. From there also we can post the details.

Running the dev server and visiting the URL (e.g. `http://127.0.0.1:8000/empdetails/`) in a browser shows DRF's **browsable API** — not raw JSON text, but a styled HTML page DRF generates automatically, showing the JSON response nicely formatted, the HTTP methods available (`GET`, `POST`, `HEAD`, `OPTIONS`, as noted in the transcript), and (for endpoints that support it) a form to submit new data directly from that page. The instructor notes this browsable view is a convenience for development/testing and defers a full walkthrough of it — along with actually testing `POST`/insert/update/delete through it — to a later lecture in this unit.

> **[Researched] — how DRF decides browsable HTML vs. raw JSON.** Per the [DRF Renderers documentation](https://www.django-rest-framework.org/api-guide/renderers/), DRF picks a response format through **content negotiation**: it looks at the incoming request's `Accept` header (and DRF's default renderer settings) to decide how to format the same underlying `Response` data. A normal web browser sends an `Accept` header that prefers HTML, so DRF's `BrowsableAPIRenderer` kicks in and shows the friendly HTML page seen in this lecture. A tool like `curl` (with no special headers) or a mobile app requesting `application/json` gets the plain JSON body instead — the *same* view and the *same* `Response(serializer.data)` call serve both, with no extra code required.

## 13. Under the hood: `QuerySet` vs. `ReturnList` **[From video]**

> I want to print this ... `print(emp)` ... what is `type(emp)` ... you can see this is `QuerySet` type ... `django.db.models.query.QuerySet` ... then ... `print(type(serializer.data))` ... this is `rest_framework.utils.serializer.helpers.ReturnList` ... this will be converted into JSON format type only.

As a live debugging demonstration, the instructor temporarily adds `print()` statements inside the `get()` method to show the actual Python types involved at each stage of the pipeline:

```python
def get(self, request):
    emp = Employees.objects.all()
    print(emp)                       # e.g. <QuerySet [<Employees: Rajesh>, ...]>
    print(type(emp))                 # <class 'django.db.models.query.QuerySet'>

    serializer = EmpSerializer(emp, many=True)
    print(type(serializer.data))     # <class 'rest_framework.utils.serializer_helpers.ReturnList'>

    return Response(serializer.data)
```

This confirms, concretely, the serialization pipeline described in section 8: `Employees.objects.all()` is a Django `QuerySet` (complex, ORM-specific data) → after passing through `EmpSerializer`, `serializer.data` is a `ReturnList` (a DRF-specific `list` subclass holding plain dictionaries) → DRF's `Response` renders *that* into the actual JSON text sent to the client.

> **[Researched] — what `ReturnList` actually is.** `ReturnList` (and its sibling `ReturnDict`, used for a single non-`many` instance) are thin subclasses of Python's built-in `list`/`dict` that DRF's serializers return, defined in `rest_framework.utils.serializer_helpers`. They behave exactly like ordinary Python lists/dicts for nearly every purpose — indexing, iterating, `len()` — but additionally carry a reference back to the serializer that produced them (used internally, e.g. by the browsable API to know how to re-render a form). This is why the instructor's demonstration is reassuring rather than alarming: despite the unfamiliar class name, `serializer.data` behaves like "just a list of dicts" for all practical purposes, and Python's `json` module (which DRF's renderer uses internally) can serialize it exactly as if it were one.

## 14. What's deferred to future lectures **[From video]**

The instructor closes by listing what's intentionally left out of this first lecture, to be picked up later in this unit:

- A `test.py` (sic — Django's actual convention is `tests.py`) file already exists in the app for writing API test cases — not used yet, but mentioned as the place API tests will be written in an upcoming lecture.
- Rendering the same JSON data into an HTML **template** instead of returning it raw — mentioned as possible via "JSON renderer methods," to be covered later.
- **Deserialization** — accepting data sent *to* the API (`POST`) and saving it back into the database — explicitly deferred, with the instructor previewing that both serialization and deserialization will eventually live in the same app.
- Testing `POST`, `PUT`, `PATCH`, and `DELETE` through the browsable API's built-in forms — deferred.

> **[Gap-filled] — `test.py` vs. `tests.py`.** Django's `startapp` command scaffolds a file named **`tests.py`** (plural) by default, and Django's own test-discovery tooling (`python manage.py test`) looks for test code by that convention. If the instructor's project genuinely has a file named `test.py` (singular), it would still work for manually-run test code but wouldn't be picked up automatically the same way — worth double-checking the actual filename in your own project rather than assuming from the transcript's spoken audio alone, since "test.py" and "tests.py" sound nearly identical spoken aloud.

## 15. A second worked example: `Book` model → serializer → API **[Example]**

To make the serialization pipeline concrete beyond the video's `Employees` example, here's the same pattern applied to a different, common beginner model — a `Book`:

**`models.py`:**
```python
from django.db import models

class Book(models.Model):
    title = models.CharField(max_length=100)
    author = models.CharField(max_length=100)
    price = models.DecimalField(max_digits=6, decimal_places=2)
    published = models.BooleanField(default=True)

    def __str__(self):
        return self.title
```

**`serializers.py`:**
```python
from rest_framework import serializers
from .models import Book

class BookSerializer(serializers.ModelSerializer):
    class Meta:
        model = Book
        fields = ['id', 'title', 'author', 'price', 'published']  # explicit, not '__all__'
```

**`views.py`:**
```python
from rest_framework.views import APIView
from rest_framework.response import Response
from .models import Book
from .serializers import BookSerializer

class BookListAPIView(APIView):
    def get(self, request):
        books = Book.objects.all()
        serializer = BookSerializer(books, many=True)
        return Response(serializer.data)
```

**Sample input (database rows, conceptually):**

| id | title | author | price | published |
|----|-------|--------|-------|-----------|
| 1 | Clean Code | Robert C. Martin | 34.99 | True |
| 2 | Deep Work | Cal Newport | 18.50 | True |

**Output — `GET /books/`:**
```json
[
  {"id": 1, "title": "Clean Code", "author": "Robert C. Martin", "price": "34.99", "published": true},
  {"id": 2, "title": "Deep Work", "author": "Cal Newport", "price": "18.50", "published": true}
]
```

**Realistic use case:** a mobile reading-tracker app calls `GET /books/` on page load to populate its book list — the same JSON travels to an iOS app, an Android app, and a web frontend without any of them needing Django template rendering, because DRF handed back generic, language-agnostic data instead of HTML built for one specific screen.

---

## Best practices and common pitfalls for this lecture's topics **[Gap-filled / Researched]**

- **Prefer explicit `fields` over `fields = '__all__'`** once past initial prototyping (see section 9) — avoids silently exposing new model fields through the API.
- **Only implement the HTTP methods a view actually needs.** This lecture's `EmpDetails` only defines `get()` — DRF's `APIView` responds `405 Method Not Allowed` automatically for verbs you haven't implemented, which is the correct, safe default rather than manually rejecting unsupported methods.
- **Don't forget `.as_view()`** on class-based view URL patterns — a very common early mistake, same as with plain Django CBVs.
- **Remember `many=True`** whenever serializing a queryset/list of instances rather than a single instance — omitting it is a very common source of confusing errors when a serializer receives multiple records but is only configured to expect one.
- **Use singular model names** (`Employee`, not `Employees`) as the more conventional style, even though Django doesn't enforce it.
- **`ModelSerializer` vs. plain `Serializer`:** this lecture only shows `ModelSerializer` (auto-derived from a model, like `ModelForm`). DRF also has a base `serializers.Serializer` class for writing every field by hand — useful when the data being serialized doesn't map 1:1 to a single model (e.g. combining fields from two models, or data that isn't stored in the database at all). Not covered in this lecture, but worth knowing it exists.
- **`rest_framework.response.Response`, not Django's `HttpResponse`/`JsonResponse`**, inside any DRF view — mixing them up (e.g. trying to return a plain Django `JsonResponse` from inside `APIView`) loses DRF's content negotiation and browsable-API behavior.

---

## Wrap-up

- **From video:** confirming `djangorestframework` is installed and `'rest_framework'` is in `INSTALLED_APPS`; creating the `Employees` model (`emp_id`, `emp_name`, `emp_address`, `email`) with a `__str__` method; registering it plainly in the admin; running `makemigrations`/`migrate`/`createsuperuser` and adding a few records through the admin UI; the conceptual pivot to "no template — return JSON instead"; the serialization/deserialization concept in DRF's own words (complex data ↔ native Python types ↔ JSON); creating `serializers.py` with a `ModelSerializer` subclass (`EmpSerializer`, `Meta.model`, `fields = '__all__'`); creating the view (`EmpDetails(APIView)`, `get()`, `Employees.objects.all()`, `EmpSerializer(emp, many=True)`, `Response(serializer.data)`); wiring app-level and project-level `urls.py` with `.as_view()`; viewing the result via DRF's browsable API; a live `print(type(...))` demonstration showing `QuerySet` → `ReturnList`; and a preview of what's deferred (tests, template rendering of JSON, deserialization/`POST`).
- **Gap-filled:** the `pip install djangorestframework` vs. `import rest_framework` naming distinction; why `INSTALLED_APPS` registration matters; the auto-added `id` primary key vs. the custom `emp_id` field; model naming convention (singular vs. plural); why `APIView` is preferred over a plain Django `View` for API work; why `.as_view()` is required for class-based view URLs; the `test.py`/`tests.py` naming note; and a full second worked example (`Book`/`BookSerializer`/`BookListAPIView`) with sample input→output JSON.
- **Researched:** DRF's own docs definition of serializers/deserialization (near-verbatim match to the video's phrasing); `fields = '__all__'` vs. explicit fields as a security/best-practice concern; how DRF's content negotiation picks browsable HTML vs. raw JSON based on the `Accept` header; and what `ReturnList`/`ReturnDict` actually are under the hood.

Double-check against the completeness checklist: project/app context (`MyRestProject`/`rest_app`) ✓, installing and registering `djangorestframework` ✓, the `Employees` model and its fields ✓, admin registration and `__str__` ✓, migrations/superuser/adding data ✓, the "no template" pivot ✓, serialization vs. deserialization definitions ✓, `serializers.py`/`ModelSerializer`/`Meta`/`fields` ✓, the `EmpDetails` `APIView`/`get()`/`many=True`/`Response` view ✓, app-level and project-level URL wiring with `.as_view()` ✓, running the server and the browsable API ✓, the `QuerySet`-vs-`ReturnList` type demonstration ✓, and the closing preview of `test.py`, template rendering, and deserialization as future topics ✓ — nothing from the transcript appears to have been left out.
