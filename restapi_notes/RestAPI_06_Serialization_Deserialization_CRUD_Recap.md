# REST API Session 6 — Full CRUD recap: serialization vs. deserialization, now with a class-based view

Source: `transcripts/restapi/REST API-6 (1).txt`
Covers: a quick recap of REST API Session 5's full CRUD (Create, Read, Update, Delete) API built with **function-based views**, then the same Employee CRUD API rebuilt from scratch — new app, same model, same serializer — using a **class-based view** (Django's own `View` base class, not yet DRF's `APIView`), so the two styles can be compared side by side. Ends with the instructor's own live debugging of two real URL-routing bugs, and a forward look at how much shorter this will get once DRF's generic/concrete view classes and ViewSets are introduced.

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official DRF/Django docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. Recap: serialization and deserialization, and last session's function-based CRUD **[From video]**

> In the last session we covered all CRUD operations using the serialization/deserialization mechanism.

The video opens by restating two terms from REST API Session 5 (and earlier):

- **Serialization** — converting a "complex" Python object (a queryset or a model instance, i.e. a row or rows straight out of the database) into a plain Python native type (dicts/lists), and then into **JSON**, so it can be sent back to the end user as an HTTP response body.
- **Deserialization** — the reverse trip: JSON data arrives from a client (in the request body), gets converted into plain Python native types, and then into a "complex" type — a validated, savable model instance — so it can actually be written to the database.

> Same CRUD operations using function-based views: only one function, `def employee_view(request)`. If `request.method == 'POST'`, ...; `PUT` means updating the data; `DELETE` means deleting the data; `GET` means reading the data.

REST API Session 5's whole API lived in a **single view function**, with one big `if/elif` chain branching on `request.method` to decide whether the incoming request was a create, read, update, or delete.

> **[Gap-filled] — the mental model, in one line.** Serialization always flows *out* of Django (database → Python → JSON → the client); deserialization always flows *in* (client → JSON → Python → the database). A useful analogy: serialization is like translating a local document into a foreign language before mailing it out; deserialization is translating an incoming foreign-language letter back into your own language before filing it. Every CRUD operation this lecture builds is just some combination of those two translations, wired to one of the four HTTP methods.

## 2. Today's plan: the same CRUD API, but as a class-based view **[From video]**

> Same CRUD API view using class-based view — how to do it, we will look into that today, so you get a clear idea: if the class-based view is there, how it will be, and if the function-based view is there, how it will be.

The stated goal for this session is a direct **comparison**: rebuild the exact same Employee CRUD API from REST API Session 5, but organize the GET/POST/PUT/DELETE logic as **methods on a class** instead of `if/elif` branches inside one function. To keep the comparison clean, the instructor builds it in a **brand-new app** rather than editing the old one.

> **[Gap-filled] — why this matters, and a naming note for later lectures.** This lecture's class-based view inherits from Django's own built-in `View` class (`django.views.View`) — the same generic base class used for ordinary Django class-based views since earlier in the course — *not* DRF's `rest_framework.views.APIView`. That distinction matters for reading these notes alongside the rest of the unit: REST API Session 7 introduces `APIView`, which is purpose-built for API views and removes a lot of the manual plumbing (parsing, rendering, CSRF handling) this lecture still does by hand. Think of this lecture as the missing middle step: function-based view (REST API Session 5) → **hand-rolled class-based view using plain Django `View`** (this lecture) → DRF's dedicated `APIView` (REST API Session 7) → generic/concrete views and ViewSets (later in this unit), each step removing more boilerplate than the last.

## 3. Setting up a fresh app: `restapp3` **[From video]**

> To create a new application: `python manage.py startapp restapp3`. Because `restapp2` already exists, this is the next one — I'm working with CRUD API using a class-based view, so I created `restapp3`.

```bash
python manage.py startapp restapp3
```

The new app is then registered in the project's `settings.py`, alongside the reminder that `rest_framework` itself must also be listed:

```python
# settings.py
INSTALLED_APPS = [
    # ... Django's built-in apps ...
    'rest_framework',   # required by every app that uses DRF features
    'restapp2',
    'restapp3',
]
```

> `rest_framework` is compulsory, because you're working with a REST app.

## 4. Reusing the Employee model and admin registration **[From video]**

The model itself is **not new** — it's copied over unchanged from `restapp2`, just to have something to build the class-based CRUD API against:

```python
# restapp3/models.py
from django.db import models

class Employee(models.Model):
    name = models.CharField(max_length=100)
    address = models.CharField(max_length=200)
    email = models.EmailField()
    age = models.IntegerField()

    def __str__(self):
        return self.name
```

`admin.py` is copied the same way, with only the import path changed to point at `restapp3`'s own model:

```python
# restapp3/admin.py
from django.contrib import admin
from restapp3.models import Employee

admin.site.register(Employee)
```

Then the usual migration/verification steps: `python manage.py makemigrations`, `python manage.py migrate`, and `python manage.py runserver`, followed by logging into `/admin/` (with the superuser account already created earlier in the course) and confirming the new `Employee` table exists under `restapp3` — empty, with no records yet, ready to be filled in through the API being built.

## 5. The serializer — still hand-written, `ModelSerializer` coming soon **[From video]**

> In the previous application, `create` method is there for insertion, `update` method is there for updating — same code here too, no changes. This is not a model serializer — that's what you'll see next session. In the model I included name, address, mail, age — against `serializers.py` I have to include that many fields too. Next session I'll introduce model serialization, where you don't need to list every field by hand — instead `model = Employee` sets it up automatically.

`serializers.py` is copied over from `restapp2` verbatim — no changes are needed, because the underlying `Employee` model and its fields haven't changed. It's still a plain `serializers.Serializer` subclass (not the shortcut `ModelSerializer`, which the video explicitly says is coming in a future session), meaning every field is listed by hand, and `create()`/`update()` are written out manually:

```python
# restapp3/serializers.py
from rest_framework import serializers
from restapp3.models import Employee

class EmployeeSerializer(serializers.Serializer):
    name = serializers.CharField(max_length=100)
    address = serializers.CharField(max_length=200)
    email = serializers.EmailField()
    age = serializers.IntegerField()

    def create(self, validated_data):
        # Called by serializer.save() when the serializer was built
        # WITHOUT an existing instance (i.e. serializer = EmployeeSerializer(data=py_data)).
        return Employee.objects.create(**validated_data)

    def update(self, instance, validated_data):
        # Called by serializer.save() when the serializer WAS built with an
        # existing instance (i.e. serializer = EmployeeSerializer(emp, data=py_data)).
        instance.name = validated_data.get('name', instance.name)
        instance.address = validated_data.get('address', instance.address)
        instance.email = validated_data.get('email', instance.email)
        instance.age = validated_data.get('age', instance.age)
        instance.save()
        return instance
```

> **[Gap-filled] — reconstructing this code block.** The transcript doesn't re-dictate `serializers.py` line by line here — the instructor literally copy-pastes the file from `restapp2` and says "no changes." The field list (`name`, `address`, `email`/`mail`, `age`) and the fact that `create`/`update` exist are confirmed directly by the video; the exact field-type choices (`CharField`, `EmailField`, `IntegerField`) and the body of `create`/`update` above follow the standard DRF `Serializer` pattern already established earlier in this unit, reconstructed here rather than invented from nothing.

> **[Gap-filled] — a pitfall hiding in "just these four fields."** Notice the serializer above never declares an `id` field. Because a plain `Serializer` only ever outputs the fields it explicitly lists, a GET response built from this serializer will **not include each employee's database ID** — even though the view still needs that ID internally to fetch, update, or delete a specific row. In practice that means a client reading the "list all employees" response has no reliable way to know which ID to send back for a later PUT or DELETE, short of checking the admin panel directly. The fix is simple — add `id = serializers.IntegerField(read_only=True)` to the serializer — but it's a good example of why hand-listing fields on a plain `Serializer` is easy to get subtly wrong, and part of the motivation for `ModelSerializer` (which includes the primary key automatically via `fields = '__all__'`).

## 6. The class-based view: imports and the `csrf_exempt` + `method_decorator` pattern **[From video]**

> These libraries are required: JSON parser, HTTP response, `JSONRenderer`... but here, this was function-based before — now we're working with class-based view, so your class should be inherited from this class, the built-in `View` class. That's the reason I additionally included `from django.views import View`.

```python
# restapp3/views.py — imports
import io
from django.http import HttpResponse
from django.views import View
from django.views.decorators.csrf import csrf_exempt
from django.utils.decorators import method_decorator
from rest_framework.parsers import JSONParser
from rest_framework.renderers import JSONRenderer

from restapp3.models import Employee
from restapp3.serializers import EmployeeSerializer
```

> On top of this class we have to compulsorily include a `method_decorator`. Inside this, `csrf_exempt` is required, and `name='dispatch'` — meaning CSRF exemption is going to be dispatched whenever this class-based view is hit.

```python
@method_decorator(csrf_exempt, name='dispatch')
class EmployeeView(View):
    ...
```

> **[Gap-filled] — why a plain `@csrf_exempt` above the class doesn't work here.** Django's CSRF protection normally requires a special token on any "unsafe" request (POST/PUT/DELETE) — a safeguard against a malicious site quietly submitting forms on a logged-in user's behalf. In REST API Session 5's function-based view, `@csrf_exempt` could be slapped directly above the view function, because a plain function is exactly the kind of thing that decorator is designed to wrap. A class isn't a function, though — `@csrf_exempt` can't sensibly decorate a class body the same way. Every request to a class-based view actually enters through one specific method, `dispatch()`, which looks at `request.method` and routes the call to the matching `get()`/`post()`/`put()`/`delete()` method on the class. `django.utils.decorators.method_decorator` is a small adapter built specifically to let an ordinary function decorator (like `csrf_exempt`) be applied to *one method* of a class instead of a whole function; passing `name='dispatch'` tells it "apply this to the `dispatch` method specifically" — which, because every request passes through `dispatch` first, exempts the entire class from CSRF checks in one shot, no matter which HTTP method is actually being handled.

> **[Researched]** — this exact pattern (`method_decorator(csrf_exempt, name='dispatch')` on a class-based view) is the officially documented way to apply a function-based decorator to a Django class-based view; see the ["Decorating class-based views"](https://docs.djangoproject.com/en/stable/topics/class-based-views/intro/#decorating-class-based-views) section of the Django docs.

## 7. GET — reading one record, or all of them **[From video]**

> Inside the `get` method, this is a class-based Django method — it's not `if request.method`, because you're already inside `get`. I want to read the data: `json_data = request.body`. Next, `stream = io.BytesIO(...)` — this converts it to binary form. This binary stream we push into `JSONParser().parse()`. Finally, to get the record: `id = py_data.get('id')` — sometimes `id` might be `None`. If `id` is not `None`, get the object by that ID and build the serializer without `many`. If it is `None`, get all the records and build the serializer with `many=True`.

```python
class EmployeeView(View):

    def get(self, request, *args, **kwargs):
        stream = io.BytesIO(request.body)
        py_data = JSONParser().parse(stream) if request.body else {}
        emp_id = py_data.get('id', None)

        if emp_id is not None:
            emp = Employee.objects.get(id=emp_id)
            serializer = EmployeeSerializer(emp)             # single instance
        else:
            emp = Employee.objects.all()
            serializer = EmployeeSerializer(emp, many=True)  # queryset -> a list

        json_data = JSONRenderer().render(serializer.data)
        return HttpResponse(json_data, content_type='application/json')
```

- `*args, **kwargs`: `get`, `post`, `put`, and `delete` on a Django `View` subclass always take these two, in addition to `self` and `request` — `*args` catches any positional URL-pattern captures, `**kwargs` any named ones, even when (as here) the URL pattern doesn't actually use them.
- `many=True` (established in earlier lectures) tells the serializer it's serializing a **queryset** (many rows), not a single model instance — without it, DRF would try (and fail) to treat a queryset as one object.

> **[Gap-filled] — why sending a body with a `GET` request is a shaky pattern.** By HTTP convention, `GET` requests are meant to be **safe and idempotent reads** with **no body** — many HTTP clients, caching proxies, and load balancers are free to ignore or strip a request body on a GET, which makes relying on one fragile. This lecture reuses the same "read the JSON body" trick for GET that it uses for POST/PUT/DELETE purely for consistency with the rest of the code, but a more conventional and robust design would take the record's ID from the **URL path** instead — e.g. a route like `path('emp/<int:id>/', EmployeeView.as_view())`, with `id` arriving as a URL keyword argument the `get()` method can read straight from `kwargs['id']`, or from a query string (`?id=3`) via `request.GET.get('id')`. Both of those are guaranteed to survive across any HTTP client or proxy, unlike a GET body.

## 8. POST — creating a record **[From video]**

> Post method: same JSON body, JSON parser, stream — up to here, same as before. Then I create a serializer object: `serializer = EmployeeSerializer(data=py_data)` — because we're posting data, Python data type to complex type, it has to be converted. Now check if it's valid: `serializer.is_valid()`. If valid, `serializer.save()`. After save, result: message, "data is inserted into database." We have to render — not `serializer.data`, `result` — because we're finally sending that to the end user. If not valid, go to the else block: `serializer.errors`.

```python
    def post(self, request, *args, **kwargs):
        stream = io.BytesIO(request.body)
        py_data = JSONParser().parse(stream)
        serializer = EmployeeSerializer(data=py_data)

        if serializer.is_valid():
            serializer.save()                                # -> serializer.create()
            result = {'msg': 'Data inserted into database'}
        else:
            result = serializer.errors

        json_data = JSONRenderer().render(result)
        return HttpResponse(json_data, content_type='application/json')
```

Note the response for a successful POST is a small **status dict** (`{'msg': '...'}`), not the newly created record itself — a deliberate (if minimal) choice in this lecture's code, not something DRF requires.

## 9. PUT — updating an existing record **[From video]**

> Copy and paste the same code; only the method changes — `put` instead of `post`. Because it's ID-based, `id = py_data.get('id')` is compulsory here, since we need it to know which record to update. Next: `emp = Employee.objects.get(id=id)` — this is the one I want to update. Then create the serializer: `EmployeeSerializer(emp, data=py_data)`. `serializer.is_valid()`, `serializer.save()` — message: "data is updated into database," not "inserted." Everything else stays the same.

```python
    def put(self, request, *args, **kwargs):
        stream = io.BytesIO(request.body)
        py_data = JSONParser().parse(stream)
        emp_id = py_data.get('id')

        emp = Employee.objects.get(id=emp_id)
        serializer = EmployeeSerializer(emp, data=py_data)   # instance + data -> update
        if serializer.is_valid():
            serializer.save()                                # -> serializer.update()
            result = {'msg': 'Data updated in database'}
        else:
            result = serializer.errors

        json_data = JSONRenderer().render(result)
        return HttpResponse(json_data, content_type='application/json')
```

> **[Gap-filled] — the single line that decides `create` vs. `update`.** This is the same rule the course established when the plain `Serializer` class's `create`/`update` methods were first introduced: constructing the serializer as `EmployeeSerializer(data=py_data)` (no positional instance argument) means `.save()` will call `create()`; constructing it as `EmployeeSerializer(emp, data=py_data)` (an existing model instance passed first) means `.save()` will call `update(instance, validated_data)` instead. The POST view above uses the first form, the PUT view uses the second — that one difference in how the serializer object is built is what makes an otherwise near-identical block of code insert in one case and update in the other.

## 10. DELETE — removing a record **[From video]**

> Same thing for deleting: request object, `*args`, `**kwargs`. `if request.method == 'DELETE'`. JSON body, same parsing, up to here — get the ID the same way. `emp = Employee.objects.get(id=id)`. `emp.delete()` — call the delete method on it. Result: "data is deleted from database." Return that.

```python
    def delete(self, request, *args, **kwargs):
        stream = io.BytesIO(request.body)
        py_data = JSONParser().parse(stream)
        emp_id = py_data.get('id')

        emp = Employee.objects.get(id=emp_id)
        emp.delete()

        result = {'msg': 'Data deleted from database'}
        json_data = JSONRenderer().render(result)
        return HttpResponse(json_data, content_type='application/json')
```

Delete doesn't go through a serializer at all — there's nothing left to validate or transform once the target row has been found, so `emp.delete()` (a plain Django model method, not a DRF one) is enough on its own.

> **[Gap-filled/Researched] — a real gap in this lecture's code: no error handling around `.objects.get()`.** Every one of the four methods above calls `Employee.objects.get(id=...)` with no `try`/`except` around it. If a client sends an ID that doesn't exist in the table, Django raises `Employee.DoesNotExist`, which — left unhandled — becomes an unhandled server error (an HTTP 500) rather than a clean, informative response. The transcript doesn't mention this being addressed. A safer, standard version would be:
> ```python
> try:
>     emp = Employee.objects.get(id=emp_id)
> except Employee.DoesNotExist:
>     result = {'error': 'No employee found with that id.'}
>     return HttpResponse(
>         JSONRenderer().render(result), status=404,
>         content_type='application/json',
>     )
> ```
> The DRF-specific shortcut for this exact situation, `django.shortcuts.get_object_or_404`, is the more idiomatic fix and will appear once the course moves to DRF's dedicated view classes.

## 11. A missing piece: no HTTP status codes anywhere **[Gap-filled]**

Looking at every `HttpResponse(...)` returned above, none of them ever specifies a `status=` argument — meaning every single response, whether it's a successful GET, a successful POST, a validation failure, or (per the point above) an unhandled crash, comes back with Django's default **`200 OK`**. That's a real gap worth calling out on its own, separate from the missing-`try/except` issue above, because it affects every method, even ones that already run without error.

> **Industry best practice / [Researched]:** A well-behaved REST API is expected to return status codes that actually describe what happened, not always `200`. DRF ships a `status` module with readable constants for exactly this — `rest_framework.status.HTTP_200_OK`, `HTTP_201_CREATED` (a resource was successfully created — the conventional code for a successful POST), `HTTP_400_BAD_REQUEST` (the serializer's `is_valid()` came back `False`), and `HTTP_404_NOT_FOUND` (a requested ID doesn't exist). Every response above could be improved by passing the matching one, e.g. `HttpResponse(json_data, status=201, content_type='application/json')` on a successful POST. This is exactly the kind of boilerplate DRF's `Response` object (used with `APIView` from REST API Session 7 onward) makes far less tedious to get right — its status codes are usually just an integer or `status.HTTP_xxx` constant passed straight to the constructor, and `Response` also picks the right content type automatically.

## 12. Wiring up URLs — `.as_view()`, and a routing bug fixed live **[From video]**

> `urls.py` is copied directly from `restapp2` into `restapp3`. The URL name is `emp`, and the view name is `EmployeeView`. Because this is a class-based view, `.as_view()` is compulsory — it's a must for a class-based view.

```python
# restapp3/urls.py
from django.urls import path
from restapp3.views import EmployeeView

urlpatterns = [
    path('emp/', EmployeeView.as_view(), name='emp'),
]
```

`restapp3.urls` is then wired into the project-level `urls.py` — and this is where the video hits a real, on-screen bug:

> I understand the problem — in the project-level `urls.py`, we have to remove `restapp3` [as a prefix]. It's unable to find `restapp3` directly; in the API we haven't given `restapp3` [in the test script's URL] either. Now this time it will work for sure — we have to make this empty.

```python
# rest_project/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('restapp3.urls')),   # empty prefix — NOT path('restapp3/', ...)
]
```

> **[Gap-filled] — why the prefix mismatch broke things.** `path('restapp3/', include('restapp3.urls'))` would make the real, reachable URL `/restapp3/emp/`. The test script (see below), however, was written to call `http://127.0.0.1:8000/emp/` — with no `restapp3/` in it at all. Django's URL resolver had no route matching `/emp/` on its own, so every request came back as a **404 "page not found"**, even though the view, serializer, and model were all correct. The fix wasn't in the view code at all — it was making the project-level include use an **empty prefix** (`''`) so the app's own `emp/` pattern becomes the site's real, top-level `/emp/`. This is a good example of a bug that looks like a Python problem but is actually a routing/configuration mismatch — worth checking URL patterns end-to-end (project-level prefix + app-level pattern + whatever URL a client is actually calling) before assuming the view logic itself is broken.

## 13. Testing with a standalone script (`requests` + `json`, not `TestCase`) **[From video]**

> Test cases: `import requests`, `import json`. First set the URL — but before that, we have to start the server. `python manage.py runserver`. Now copy the server address, and here write the URL configuration... This API endpoint we can test now.

Rather than Django's built-in `TestCase` framework, this lecture (like REST API Session 5 before it) tests the live API with a small **standalone Python script** run separately, using the third-party `requests` library to make real HTTP calls against the running dev server:

```python
# restapp3/test.py
import json
import requests

url = 'http://127.0.0.1:8000/emp/'


def get_record(emp_id=None):
    data = {'id': emp_id} if emp_id is not None else {}
    json_data = json.dumps(data)
    r = requests.get(url, data=json_data)
    data = r.json()
    print(data)


def post_record():
    data = {
        'name': 'Raj',
        'address': 'Hyderabad',
        'email': 'raj@gmail.com',
        'age': 34,
    }
    json_data = json.dumps(data)
    r = requests.post(url, data=json_data)
    data = r.json()
    print(data)


def put_record():
    data = {
        'id': 1,
        'name': 'Ramesh',
        'address': 'Hyderabad',
        'email': 'ramesh@gmail.com',
        'age': 34,
    }
    json_data = json.dumps(data)
    r = requests.put(url, data=json_data)
    data = r.json()
    print(data)


def delete_record():
    data = {'id': 1}
    json_data = json.dumps(data)
    r = requests.delete(url, data=json_data)
    data = r.json()
    print(data)


# Only one call is left uncommented at a time while testing manually.
post_record()
# get_record()
# put_record()
# delete_record()
```

> Post is not correct — this URL is spelled wrong, that's why it's unable to find the URL. ... small mistakes — no problem — in the project-level `urls.py` we shouldn't give an application name; and in `test.py`, I hadn't included the trailing slash after the endpoint (`emp/`). These are small mistakes, no problem.

The workflow demonstrated: run the dev server in one terminal (`python manage.py runserver`); in a second terminal, inside the `restapp3` folder, run `python test.py` with exactly one of the four function calls uncommented; check the admin panel (refreshed) to confirm the record was actually inserted/updated/deleted; then comment that call out and uncomment the next one to test the next operation. Along the way, the video hits and fixes, live: (1) a missing trailing slash on the `emp/` endpoint in `test.py`'s URL string, (2) the project-level URL prefix mismatch described above, and (3) a reminder to re-run `makemigrations`/`migrate` after model-related changes.

> **[Gap-filled] — why this isn't "real" automated testing.** A script like this that just prints results to the console for a human to eyeball is useful for manual, exploratory testing during development, but it's not the same as Django's `TestCase`-based automated tests (used elsewhere in this course), which run in an isolated test database, can assert on exact expected values, and can be re-run automatically (e.g. in CI) without a human reading output. A more thorough version of this same script would use Python's `assert` statements, or migrate to `rest_framework.test.APITestCase` (DRF's own test-client base class), rather than only printing `r.json()` for manual inspection.

## 14. Where this is heading **[From video]**

> This much code — 10 to 15 lines per operation — is not something you'll write in real time, and they won't allow this much code either, especially in interviews. This is only for understanding purposes: how the data actually gets transformed from one form to another. In the next session, we'll go for reducing it — because DRF's built-in generic API view classes, concrete view classes, view sets, model view sets are there — using those, easily we can reduce this to 3 or 4 lines maximum.

The instructor is explicit that this lecture's verbosity is **deliberate**, for teaching purposes — to make every conversion step (JSON → stream → Python data → validated serializer → model instance, and back) visible, rather than hidden inside a built-in class. The stated plan for the rest of this unit: DRF's `APIView` (REST API Session 7), then generic API views and mixins, concrete view classes, and finally `ViewSet`/`ModelViewSet` (later lectures in this unit) will progressively collapse this ~15-line-per-method pattern down to just a few lines per operation.

## 15. Worked example: the full request/response cycle, end to end **[Example]**

To make the abstract "Python ↔ JSON" round trip concrete, here's what actually crosses the wire for a single POST, using the `EmployeeView` above.

**Request** (from `post_record()` in `test.py`):

```http
POST /emp/ HTTP/1.1
Host: 127.0.0.1:8000
Content-Type: application/json

{"name": "Raj", "address": "Hyderabad", "email": "raj@gmail.com", "age": 34}
```

Step by step, inside `EmployeeView.post()`:

1. `request.body` — the raw bytes above, still as JSON text.
2. `io.BytesIO(request.body)` — wraps those bytes in a stream object, because `JSONParser().parse()` expects a file-like stream, not a raw string.
3. `JSONParser().parse(stream)` — **deserialization step 1**: JSON text → a plain Python `dict`: `{'name': 'Raj', 'address': 'Hyderabad', 'email': 'raj@gmail.com', 'age': 34}`.
4. `EmployeeSerializer(data=py_data)` + `.is_valid()` — validates each field's type/format (e.g. `email` must look like an email); on success, `.validated_data` holds the cleaned dict.
5. `.save()` → `.create(validated_data)` — **deserialization step 2**: the validated dict → an actual saved `Employee` row in the database, with a new auto-generated `id`.
6. `result = {'msg': 'Data inserted into database'}` — this is the object about to be **serialized** for the reply (not the employee record itself, in this lecture's code).
7. `JSONRenderer().render(result)` — **serialization**: the Python dict → JSON bytes.
8. `HttpResponse(json_data, content_type='application/json')` — wraps those bytes as an actual HTTP response.

**Response** (what `post_record()` then prints via `r.json()`):

```json
{"msg": "Data inserted into database"}
```

And for a GET of all records once a couple of rows exist (via `get_record()` with no `id`) — this is the **serialization-only** path (no deserialization step at all, since nothing is being written):

```json
[
    {"name": "Raj", "address": "Hyderabad", "email": "raj@gmail.com", "age": 34},
    {"name": "Ramesh", "address": "Hyderabad", "email": "ramesh@gmail.com", "age": 34}
]
```

(Note, per the pitfall in Section 5: no `id` appears in this list, because the serializer never declared one — a small but real design flaw carried over from `restapp2`.)

---

## Wrap-up

- **From video:** a recap of serialization/deserialization and REST API Session 5's function-based CRUD; building a fresh app (`restapp3`) specifically to compare styles; reusing the `Employee` model, admin registration, and the hand-written `EmployeeSerializer` (`Serializer`, not yet `ModelSerializer`) unchanged; a full class-based `EmployeeView(View)` with `get`/`post`/`put`/`delete` methods, each parsing the request body via `io.BytesIO` + `JSONParser`, and rendering responses via `JSONRenderer` + `HttpResponse`; the `@method_decorator(csrf_exempt, name='dispatch')` pattern required for CSRF-exempting a class-based view; wiring URLs with `.as_view()`; a real, live-fixed routing bug (project-level `urls.py` wrongly prefixed with the app name) and a missing-trailing-slash bug in the test script; testing manually with a `requests`+`json` script rather than `TestCase`; and the instructor's own framing of this code as intentionally verbose, to be drastically shortened once DRF's generic views, concrete views, and ViewSets are introduced starting next session.
- **Gap-filled:** the serialization/deserialization "translation" analogy; the distinction between this lecture's plain Django `View` and DRF's `APIView` (coming in REST API Session 7); why `method_decorator(..., name='dispatch')` is needed instead of a plain `@csrf_exempt`; the fragility of reading a GET request's body for routing instead of using the URL path or query string; the missing-`id`-in-the-serializer pitfall; the missing `try`/`except` around every `.objects.get()` call; the missing HTTP status codes on every response; why a print-and-eyeball test script isn't the same as automated `TestCase`/`APITestCase` tests; and reconstructing the `serializers.py` code (confirmed unchanged by the video, but not re-dictated line by line).
- **Researched:** the Django docs' "Decorating class-based views" pattern for `method_decorator`; DRF's `status` module and its conventional status codes (`HTTP_201_CREATED`, `HTTP_400_BAD_REQUEST`, `HTTP_404_NOT_FOUND`); `get_object_or_404` as the idiomatic fix for the missing-instance case; DRF's `APITestCase` as the more rigorous alternative to a manual script.

Double-check against the transcript's own topic list: recap of serialization/deserialization ✓, recap of function-based CRUD ✓, new app setup ✓, model/admin reuse ✓, serializer reuse (with the forward-reference to `ModelSerializer`) ✓, the full class-based `get`/`post`/`put`/`delete` implementation ✓, the `method_decorator`/`csrf_exempt`/`dispatch` requirement ✓, URL wiring and `.as_view()` ✓, both live bugs (URL prefix, missing trailing slash) and the migrations reminder ✓, the manual `requests`-based test script for all four operations ✓, and the closing forward-reference to generic/concrete views and ViewSets ✓ — nothing from the transcript appears to have been left out. One scope note: this transcript does **not** actually use an `@api_view` decorator or DRF's `status` module anywhere — those are covered here only as gap-filled/researched additions (Sections 11 and the note in Section 10), not as something the video itself demonstrated.
