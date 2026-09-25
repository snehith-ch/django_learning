# REST API Session 4 — Deserialization: Inserting Data via POST, and Testing It with test.py

Source: `transcripts/restapi/REST API-4.txt`
Covers: this session picks up right after REST API Session 3's serialization work (reading data out of the
database and sending it back as JSON) and flips the direction — **deserialization**: taking JSON data
sent *in* to the API and turning it into a saved database row. To demonstrate it, a brand-new app
(`restapp2`) and model (`Manager`) are built from scratch, along with a hand-written serializer, a
function-based "create" view, a URL, and a `test.py` script that POSTs data at the API and prints back
the JSON response — including a live debugging session when the demo hits real errors. The lecture
closes by naming what's still ahead (PUT, PATCH, DELETE, and browsable API testing) without covering
them yet.

> **A note on this lecture's working title.** This lecture was provisionally slated as "Testing the API
> & the DRF browsable API," but the actual transcript content is almost entirely about
> **deserialization / inserting data via POST**, tested with a `test.py` script — the DRF browsable API
> (a web-page-based way to test endpoints without writing a script) is only mentioned once, in passing,
> as something to be covered in a *future* session. The title above has been adjusted to match what
> this transcript actually teaches; treat any earlier reference to "REST API Session 4 covers the browsable API"
> as inaccurate — that material comes later.

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official Django/DRF docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. Recap: what serialization did (REST API Session 3) **[From video]**

> So in the last session of REST API, we were discussing about how to create API and how to test that
> API... So we did only serialization concept in the last session.

The instructor recaps the previous lecture before starting new material:

- A `test.py` file was written that talks to a running API over a URL like
  `http://127.0.0.1:8000/` + a resource path (transcribed as "AMP" — almost certainly a garbled
  "emp," short for **Employee**, since the recap goes on to say the view "is going to be written all
  the employee details from database"). That URL routed to a view that read every row from an
  `Employee`-style model.
- The view took the **model instances** (query results straight from the database — Django calls this
  a "complex type" because it's a Python object, not plain text/numbers) and handed them to a
  **serializer**, which converted them into a JSON-shaped Python structure and finally rendered that as
  raw JSON text in an `HttpResponse`.
- That whole reading path — database rows → Python data → JSON — is what the video calls
  **serialization**.

> **[Gap-filled]** The exact URL name and view name from REST API Session 3 are garbled beyond confident
> recovery in this transcript ("AMP underscore all underscore details"). Rather than guess a specific
> name, just know: it was a GET-style "list all records" endpoint on a first app (`restapp1`), and this
> lecture builds a parallel, second app (`restapp2`) to demonstrate the opposite direction (writing
> data), so the two examples don't depend on each other's exact naming.

## 2. Serialization vs. deserialization, side by side **[From video]**

The instructor draws this out explicitly as two mirrored processes:

| | Serialization (REST API Session 3) | Deserialization (this lecture) |
|---|---|---|
| **Direction** | out of the database, to the client | from the client, into the database |
| **Starts as** | complex type (a model instance / `QuerySet`) | JSON data |
| **Step 1** | complex type → Python data type | JSON → Python data type |
| **Step 2** | Python data type → JSON | Python data type → complex type (saved to DB) |
| **Ends as** | JSON sent to the browser/client | a row saved in the database |

> Serialization means what actually we are taking complex type, means object type... and this complex
> type is converted into Python types. Python types converted into JSON format... Now deserialization —
> we are sending the data JSON format only... this JSON data convert into Python data type. Finally
> Python data type should be convert into complex type back again.

<dfn>Serialization</dfn> is the process of turning a database object into a format (JSON) that any
front-end or other program can understand. <dfn>Deserialization</dfn> is the reverse: taking data sent
by a client (JSON) and turning it back into something Django's ORM can save to the database — but only
**after validating it**, which is the part deserialization adds that a plain "trust the input" approach
wouldn't have.

> **[Researched]** This matches the official DRF description almost word for word: "Serializers in REST
> framework are responsible for converting objects... into data types understandable by front end
> frameworks... Serializers also provide deserialization, allowing parsed data to be converted back into
> complex types, after first validating the incoming data." (See the
> [DRF Serializers documentation](https://www.django-rest-framework.org/api-guide/serializers/).) The
> validation step is the key detail: deserialization isn't just "JSON in, database row out" — it's
> "JSON in, checked against the serializer's field rules, *then* database row out." If validation fails,
> nothing gets saved.

## 3. Python's `io` module and streams **[From video / Researched]**

Before the JSON data sent by a client can be handed to a serializer, it has to pass through Python's
built-in **`io` module**.

> So I.O. module means input and output here... the I.O. module provides Python main facilities for
> dealing with various types of input and output... A stream is basically a sequence of data... a
> `BytesIO` is used for binary data only.

- <dfn>io module</dfn> — Python's standard library module for reading and writing data, whether that
  data is coming from a file on disk, a network request, or (as here) an incoming HTTP request body.
- <dfn>Stream</dfn> — a sequence of data that flows through a program piece by piece, the way water
  flows through a pipe, rather than existing all at once as a single value. An HTTP request's raw body
  is treated as a stream of bytes.
- <dfn>Byte stream</dfn> — a stream made of raw bytes (not yet decoded into text or any particular
  format). `request.body` in Django gives you the raw bytes a client sent.
- <dfn>`io.BytesIO`</dfn> — a class that wraps a plain bytes object (like `request.body`) so it *behaves
  like a file* — something you can read from using file-style methods — without actually writing
  anything to disk. It's an in-memory, file-like object.

The reason this step exists: DRF's `JSONParser` (next section) is written to read from a **file-like
object**, not a raw bytes value directly. `io.BytesIO(request.body)` is the adapter that lets the raw
bytes of the request act like a file so the parser can read them.

> **[Researched]** Per the [Python `io` docs](https://docs.python.org/3/library/io.html), `io.BytesIO`
> is literally described as "a stream implementation using an in-memory bytes buffer... useful in cases
> where mimicking a file stored on disk is desired." It's commonly used exactly this way — to feed
> in-memory data to any API written for file objects, without touching the filesystem.

> **[Example]** A minimal standalone demonstration of what `BytesIO` does, with no Django involved:
> ```python
> import io
>
> raw_bytes = b'{"name": "Test"}'   # pretend this came from request.body
> stream = io.BytesIO(raw_bytes)    # wrap it so it behaves like an open file
>
> print(stream.read())              # -> b'{"name": "Test"}'
> ```
> `stream.read()` works exactly like reading from a file opened in binary mode (`open(path, 'rb')`),
> except the "file" only ever existed in memory.

## 4. `JSONParser`: turning JSON bytes into a Python dict **[From video]**

> JSON parser — it parses the incoming request JSON content into Python content type, that is
> dictionary format... data parsing is the process of taking data in one format and transforming it to
> another format.

`rest_framework.parsers.JSONParser` is DRF's built-in tool for the "JSON → Python" half of
deserialization:

```python
from rest_framework.parsers import JSONParser

py_data = JSONParser().parse(stream)   # stream = io.BytesIO(request.body)
```

- `JSONParser().parse(stream)` reads the byte stream, decodes it as JSON text, and returns a plain
  Python **dictionary** (or list of dictionaries) — the same shape you'd get from `json.loads()`, but
  wired to accept a stream instead of requiring you to call `.read()` and decode manually first.
- <dfn>Data parsing</dfn>, generally: taking data in one format (here, JSON text) and transforming it
  into another format your program can actually work with (here, a Python `dict`).

> **[Researched]** DRF ships several parser classes beyond `JSONParser` — `FormParser`,
> `MultiPartParser` (for file uploads), etc. — selectable per-view via a `parser_classes` attribute on
> class-based views. This lecture uses `JSONParser` directly and manually because it's building a plain
> Django function-based view "by hand," not yet DRF's `APIView` (that arrives in a later lecture); a
> real `APIView`/generic view picks the right parser automatically based on the request's
> `Content-Type` header, so you'd rarely call `JSONParser().parse()` yourself in practice. See the
> [DRF Parsers documentation](https://www.django-rest-framework.org/api-guide/parsers/).

## 5. Setting up a second app: `restapp2` and the `Manager` model **[From video]**

To keep the new "write" demo separate from the "read" demo built in REST API Session 3, the instructor creates
a **second Django app** from scratch rather than reusing `restapp1`.

```bash
# From the project root
python manage.py startapp restapp2
```

Then registers it in the project's `settings.py`:

```python
INSTALLED_APPS = [
    # ...
    'restapp1',
    'restapp2',   # newly added
]
```

A new model is written in `restapp2/models.py`, copied and renamed from `restapp1`'s model to save
time:

```python
# restapp2/models.py
from django.db import models

class Manager(models.Model):
    name = models.CharField(max_length=20)
    address = models.CharField(max_length=100)   # exact length not stated in the video; see note below
    mail = models.CharField(max_length=100)       # see note below on EmailField
    age = models.IntegerField()
```

> **[Gap-filled]** The transcript only confirms `name`'s limit explicitly — `max_length=20` — because a
> later part of the demo deliberately triggers a "field too long" error against that exact limit. The
> `address` and `mail` fields are also `CharField`s per the video, but no specific `max_length` is
> stated for them; the values above are reasonable placeholders, not transcribed facts. In your own
> code, always pick explicit, deliberate lengths rather than leaving them to guesswork.

Registered in the admin so records can be viewed/managed through Django's built-in admin site:

```python
# restapp2/admin.py
from django.contrib import admin
from restapp2.models import Manager

admin.site.register(Manager)
```

Then the usual two-step migration process (Lecture 8-era pattern, still the same here):

```bash
python manage.py makemigrations
python manage.py migrate
```

After running the server and logging into `/admin/` with an existing superuser account, the `Manager`
table is confirmed to exist but is **empty** — setting up the "before" state the rest of the lecture
fills in.

> **[Researched] — why `CharField` for an email address is a weaker choice.** Django ships a dedicated
> [`EmailField`](https://docs.djangoproject.com/en/stable/ref/models/fields/#emailfield) (a `CharField`
> subclass that adds email-format validation) specifically for this situation. Using a plain
> `CharField` for `mail`, as this demo does, accepts *any* text — `"not an email"` would pass model-level
> validation just as happily as a real address. For a field that's supposed to hold an email address,
> `models.EmailField(max_length=254)` (and the matching `serializers.EmailField()` on the DRF side,
> covered next) is the better default.

## 6. Writing `ManagerSerializer` by hand — not `ModelSerializer` **[From video]**

> In my previous application I use `ModelSerializer`... but just for practice we can use same serializer
> class only... how many fields are there in the models? Name, address, mail, age. Same fields we have
> to include into the serializer.py also.

`restapp2/serializers.py`:

```python
from rest_framework import serializers
from restapp2.models import Manager

class ManagerSerializer(serializers.Serializer):
    # Every field the model has must be re-declared here by hand,
    # because plain serializers.Serializer does NOT read the model automatically.
    name = serializers.CharField(max_length=20)
    address = serializers.CharField()
    mail = serializers.CharField()
    age = serializers.IntegerField()

    def create(self, validated_data):
        # Called automatically by .save() once the incoming data has passed validation.
        # validated_data is a plain dict, e.g. {'name': 'Mohan', 'address': 'Hyd', ...}
        return Manager.objects.create(**validated_data)
```

Line-by-line:

- `class ManagerSerializer(serializers.Serializer):` — subclasses DRF's base `Serializer` class
  (**not** `ModelSerializer`), meaning nothing is inferred from the `Manager` model automatically. Every
  field the model has must be listed again here, by hand, matching name-for-name.
- `name = serializers.CharField(max_length=20)` — declares a field named `name`, expected to be text, no
  longer than 20 characters. This mirrors — but does not read from — the model's own
  `max_length=20`. If they ever drift out of sync (e.g. the model changes to `max_length=30` but the
  serializer still says `20`), the serializer's limit wins for API requests, which is a real footgun.
- `age = serializers.IntegerField()` — expects a whole number; a string like `"abc"` for age would fail
  validation.
- `def create(self, validated_data):` — a method DRF's base `Serializer` expects you to implement
  yourself whenever you call `.save()` on new data (as opposed to updating an existing instance, which
  needs an `update()` method too — not shown yet in this lecture; a plain `Serializer` requires both
  methods for full CRUD, and this demo only wires up `create`). `self` is the serializer instance;
  `validated_data` is the dictionary of *already-validated* field values DRF hands you automatically
  once `is_valid()` has passed.
- `Manager.objects.create(**validated_data)` — the `**` unpacks the dictionary into keyword arguments,
  so `{'name': 'Mohan', 'address': 'Hyd', 'mail': 'mohan@gmail.com', 'age': 38}` becomes
  `Manager.objects.create(name='Mohan', address='Hyd', mail='mohan@gmail.com', age=38)`. This is the
  exact same `**kwargs` unpacking pattern used throughout Python, applied here to hand a dict's contents
  to a function as named arguments instead of passing the dict itself.

> **[Gap-filled]** `**validated_data` only works cleanly because the serializer's field names
> (`name`, `address`, `mail`, `age`) match the model's field names exactly. If a serializer field were
> named differently from its model column, you couldn't just splat the dict straight into
> `.objects.create()` — you'd need to translate the keys first.

> **[Researched] — why `ModelSerializer` is the better everyday choice.** Per the
> [DRF `ModelSerializer` docs](https://www.django-rest-framework.org/api-guide/serializers/#modelserializer),
> a `ModelSerializer` automatically generates the field list *and* a working `create()`/`update()` pair
> by introspecting the model — meaning the equivalent of everything written in this section collapses
> to:
> ```python
> class ManagerSerializer(serializers.ModelSerializer):
>     class Meta:
>         model = Manager
>         fields = ['id', 'name', 'address', 'mail', 'age']
> ```
> The instructor is explicit that the hand-written version here is "just for practice" — in a real
> project, reaching for `ModelSerializer` first (and dropping to a plain `Serializer` only when a field
> genuinely doesn't map onto a model, like a computed or write-only value) avoids exactly the
> field-name-drift risk noted above.

## 7. The `create_manager` view: JSON in, saved row out **[From video]**

The imports at the top of `restapp2/views.py`:

```python
import io
from rest_framework.parsers import JSONParser
from rest_framework.renderers import JSONRenderer
from restapp2.serializers import ManagerSerializer
from django.http import HttpResponse
from django.views.decorators.csrf import csrf_exempt
```

The view itself:

```python
@csrf_exempt
def create_manager(request):
    if request.method == 'POST':
        stream = io.BytesIO(request.body)          # wrap the raw request body as a file-like object
        py_data = JSONParser().parse(stream)        # JSON bytes -> Python dict

        serializer = ManagerSerializer(data=py_data)  # hand the dict to the serializer for validation

        if serializer.is_valid():
            serializer.save()                        # -> calls ManagerSerializer.create() internally
            result = {'message': 'Data Inserted'}
        else:
            result = serializer.errors                # a dict describing what failed validation

        json_data = JSONRenderer().render(result)     # Python dict -> JSON bytes
        return HttpResponse(json_data, content_type='application/json')
```

Step by step, why each line exists:

1. **`if request.method == 'POST':`** — this view only handles data-creation requests; a GET request
   to the same URL would fall through without a response, since no `else` branch is defined (worth
   fixing in a real project — see the pitfall note below).
2. **`io.BytesIO(request.body)`** — `request.body` on a Django `HttpRequest` gives the raw bytes of
   whatever the client sent in the request body; wrapping it in `BytesIO` makes it readable by
   `JSONParser` (Section 3–4 above).
3. **`JSONParser().parse(stream)`** — decodes those bytes as JSON and returns a Python `dict`, e.g.
   `{'name': 'Mohan', 'address': 'Hyd', 'mail': 'mohan@gmail.com', 'age': 38}`.
4. **`ManagerSerializer(data=py_data)`** — passing `data=` (as opposed to `instance=`) tells the
   serializer "validate this incoming dict," which is the deserialization mode.
5. **`serializer.is_valid()`** — checks every declared field's rules (max lengths, correct types, etc.)
   against `py_data`. Returns `True`/`False` and, on failure, populates `serializer.errors`.
6. **`serializer.save()`** — only called when validation passed. Internally, `.save()` calls the
   `create()` method written in Section 6, which is what actually writes the row to the database via
   `Manager.objects.create(**validated_data)`.
7. **`JSONRenderer().render(result)`** — the reverse of `JSONParser`: takes a Python dict (`result`) and
   turns it into JSON-formatted bytes, ready to send back to the client.
8. **`HttpResponse(json_data, content_type='application/json')`** — wraps the JSON bytes in an actual
   HTTP response. `content_type='application/json'` matters because it tells the receiving client (a
   browser, `requests`, or any other program) *how* to interpret the bytes — as JSON, not as plain text
   or HTML.

> **[Gap-filled] — a real bug worth naming.** Neither branch of this view sets an explicit HTTP
> **status code**. Django's `HttpResponse` defaults to `200 OK` when none is given — so even the
> `else` branch, which returns validation *errors*, still reports success (`200`) at the HTTP level. A
> client checking `response.status_code` (rather than reading the JSON body) would wrongly believe the
> insert worked. A corrected version would pass `status=201` (Created) on success and
> `status=400` (Bad Request) on validation failure — e.g.
> `HttpResponse(json_data, content_type='application/json', status=400)`. This is exactly the kind of
> repetitive bookkeeping DRF's own `Response` class (used once the project moves to `APIView`, covered
> in a later lecture) handles for you automatically.

## 8. Why `@csrf_exempt` is needed here **[From video / Gap-filled]**

> In the Django point of view, whenever we are going to post the data in an HTML template, every POST
> method we use CSRF token... but in this context we have to use CSRF exemption in the view section.

- <dfn>CSRF</dfn> (Cross-Site Request Forgery) protection is a Django security feature: any POST request
  coming from an HTML `<form>` must include a special hidden **CSRF token** value, proving the request
  really originated from your own site's page and not a malicious third-party site tricking a logged-in
  user's browser into submitting a form on their behalf.
- Django enforces this automatically for POST requests by default. Since `test.py` isn't an HTML form —
  it's a plain Python script using the `requests` library — it has no CSRF token to send, and the
  request would be rejected (HTTP 403 Forbidden) without an exemption.
- `@csrf_exempt`, from `django.views.decorators.csrf`, is a **decorator** — a function that wraps another
  function to change its behavior — applied directly above the view function. It tells Django "skip
  CSRF checking for requests hitting this specific view."

```python
@csrf_exempt
def create_manager(request):
    ...
```

> **[Researched] — the more scalable alternative.** Per the
> [Django CSRF docs](https://docs.djangoproject.com/en/stable/ref/csrf/), sprinkling `@csrf_exempt`
> across every API view removes a real security protection, not just an inconvenience — it should be
> used narrowly and deliberately, not as a default habit. DRF's own class-based views (`APIView` and
> up, covered starting in a later lecture) take a different approach entirely: they exempt CSRF
> automatically for *session-authenticated* requests only under specific conditions, and expect
> API clients to authenticate via **token authentication** (also covered later in this course) instead
> of session cookies — which sidesteps the CSRF problem altogether, since CSRF specifically targets
> cookie-based (session) authentication. `@csrf_exempt` on a hand-rolled function-based view, as done in
> this lecture, is a reasonable teaching shortcut but not the pattern a production DRF API should lean
> on long-term.

## 9. Wiring up the URL **[From video]**

The instructor copies `restapp1`'s `urls.py` as a starting point, then edits the path/name to match the
new view:

```python
# restapp2/urls.py
from django.urls import path
from restapp2 import views

urlpatterns = [
    path('create_manager/', views.create_manager, name='create_manager'),
]
```

And includes it at the project level, alongside `restapp1`'s URLs:

```python
# <project_name>/urls.py
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('restapp1.urls')),
    path('', include('restapp2.urls')),   # newly added
]
```

The demo hits a real error here mid-lecture — a "no reverse match" style failure — traced back to the
URL `name` not being wired consistently between `urls.py` and the client script, plus initially testing
against the wrong app's URL prefix. Both are fixed by making sure the name used in `test.py`'s URL
string matches the `path()` actually registered under `restapp2`.

> **[Gap-filled]** This is a common, unglamorous class of bug when building APIs by hand: the URL string
> a client hard-codes (`http://127.0.0.1:8000/create_manager/`) has to match, character for character,
> whatever path was actually registered — there's no automatic connection between the two. Any typo, a
> missing/extra trailing slash, or including the wrong app's URL prefix produces a 404 instead of
> reaching the view at all. This is one motivation for Django's named-URL system
> (`{% url 'name' %}` in templates) — but a plain external script like `test.py` has no access to that,
> so the URL has to be typed out literally and kept in sync by hand.

## 10. Testing inserts with `test.py` **[From video]**

`restapp2/test.py` (a plain script, run manually — not a Django/pytest test suite, despite the
filename):

```python
import requests
import json

url = 'http://127.0.0.1:8000/create_manager/'

data = {
    'name': 'Mohan',
    'address': 'Hyd',
    'mail': 'mohan@gmail.com',
    'age': 38,
}

json_data = json.dumps(data)                       # Python dict -> JSON text
r = requests.post(url=url, data=json_data)          # send it as the POST body

print(r.json())                                     # decode the response body as JSON and print it
```

Line-by-line:

- `import requests` — a popular third-party Python library for making HTTP requests (GET, POST, etc.)
  from a script, without needing a browser.
- `import json` — Python's built-in module for converting between Python objects and JSON text.
- `json.dumps(data)` — <dfn>`dumps`</dfn> ("dump string") serializes a Python dict into a JSON-formatted
  **string**. (Contrast with `json.loads`, which parses a JSON string back into a Python object — not
  used here, but the natural counterpart.)
- `requests.post(url=url, data=json_data)` — sends an HTTP POST request to `url`, with `json_data` as
  the request body. This is exactly the "client" side of the deserialization flow: this line is what
  produces the `request.body` bytes the view (Section 7) reads with `io.BytesIO`.
- `r.json()` — a convenience method on the `requests` response object that parses the server's response
  body as JSON and returns it as a Python object (here, the `{'message': 'Data Inserted'}` dict the view
  rendered).

Running it (in a separate terminal from the running dev server, `cd`'d into the correct app directory):

```bash
python manage.py runserver          # terminal 1 — leave this running
```
```bash
cd restapp2
python test.py                       # terminal 2
```

> **[Gap-filled] — how this compares to `curl`.** The same request could be sent without Python at all,
> using `curl` from a terminal:
> ```bash
> curl -X POST http://127.0.0.1:8000/create_manager/ \
>   -H "Content-Type: application/json" \
>   -d '{"name": "Mohan", "address": "Hyd", "mail": "mohan@gmail.com", "age": 38}'
> ```
> `test.py` does the same job as this one `curl` command, just from inside a reusable Python script —
> useful once you want to chain several requests together, print/inspect the response programmatically,
> or reuse the same data across a few test cases.

## 11. Validation errors in the JSON response **[From video]**

To demonstrate what happens when the incoming data *doesn't* satisfy the serializer's rules, the
instructor deliberately sends a `name` value longer than the model/serializer's 20-character limit.
Because `serializer.is_valid()` returns `False` in that case, the view's `else` branch runs instead:

```json
{
  "name": [
    "Ensure this field has no more than 20 characters."
  ]
}
```

> Whenever any errors message will come, in the sense whenever you do mistake in the test.py... in the
> model case, max length of name is 20 characters only. If I try to give a name that's too long, this
> name is too long, action — according to model creation, your database field length is too length —
> that's what in this case we will get notified, JSON errors only.

This confirms `serializer.errors` — the dict DRF automatically builds when `is_valid()` fails — is
itself JSON-renderable (via `JSONRenderer`) just like any other Python dict, and no database write
happens at all when validation fails; the admin panel is checked afterward to confirm no bad row was
inserted.

> **[Example] — sample input → output for the whole deserialization path, both outcomes:**
>
> **Valid input:**
> ```json
> {"name": "Mohan", "address": "Hyd", "mail": "mohan@gmail.com", "age": 38}
> ```
> **Response:**
> ```json
> {"message": "Data Inserted"}
> ```
> A new `Manager` row appears in the database (visible in `/admin/`) with those exact values.
>
> **Invalid input** (name over 20 characters):
> ```json
> {"name": "ThisNameIsWayTooLongForTheField", "address": "Hyd", "mail": "mohan@gmail.com", "age": 38}
> ```
> **Response:**
> ```json
> {"name": ["Ensure this field has no more than 20 characters."]}
> ```
> No row is created; the database is left unchanged.

## 12. End-to-end flow, recapped **[From video / Gap-filled]**

The instructor walks through the whole chain once more at the end. Laid out as a single ordered list,
combining the video's recap with the pieces already detailed above:

1. `test.py` builds a Python dict, converts it to a JSON string with `json.dumps()`, and POSTs it to
   `http://127.0.0.1:8000/create_manager/` using `requests.post()`.
2. Django's URL router matches `create_manager/` against `restapp2/urls.py` (included from the
   project-level `urls.py`) and calls the `create_manager` view.
3. `@csrf_exempt` lets the POST through without a CSRF token.
4. `io.BytesIO(request.body)` wraps the raw JSON bytes from the request as a file-like stream.
5. `JSONParser().parse(stream)` turns that stream into a Python dict (`py_data`).
6. `ManagerSerializer(data=py_data)` wraps that dict for validation.
7. `serializer.is_valid()` checks every field's rules.
   - **If valid:** `serializer.save()` calls `ManagerSerializer.create()`, which calls
     `Manager.objects.create(**validated_data)`, writing a new row to the database (the "complex type").
     `result` becomes a success message dict.
   - **If invalid:** no database write happens; `result` becomes `serializer.errors`.
8. `JSONRenderer().render(result)` converts `result` back into JSON bytes.
9. `HttpResponse(json_data, content_type='application/json')` sends those bytes back to `test.py`.
10. `r.json()` in `test.py` decodes the response and prints it.

> So this is the way to handle the situation, how to store the data into database using API. This API
> we are testing into test.py — clearly, even you can test the API separately by using test.py, or else
> we can go for browsable API testing also — there, in the upcoming session, will discuss more methods
> like POST method, even DELETE method, PUT method, PATCH method — partial update, full update — all
> methods. After discussing, then I'll introduce browsable API testing... means, automatically, no need
> to write any test.py file — browser only, we can test the API.

## 13. What's next, per the video **[From video]**

The lecture closes by naming, but not covering, several upcoming topics:

- **PUT** — full update of an existing record.
- **PATCH** — partial update of an existing record (only some fields).
- **DELETE** — removing a record.
- The distinction between **partial update** and **full update** in general.
- **Browsable API testing** — DRF's built-in, auto-generated HTML interface for hitting an API directly
  from a browser, with forms for each method, instead of writing a `test.py` script by hand for every
  request.

> **[Researched] — what the browsable API actually is, since this lecture only names it.** Per the
> [DRF Browsable API documentation](https://www.django-rest-framework.org/topics/browsable-api/), when a
> browser (as opposed to a script like `test.py`) requests a DRF endpoint, DRF detects that and renders
> a full HTML page instead of raw JSON — showing the JSON response nicely formatted, plus interactive
> forms for POST/PUT/PATCH/DELETE requests, generated automatically from the view's serializer. It's
> enabled by default on any DRF `APIView`/generic view/viewset without extra configuration. Since this
> lecture's `create_manager` view is a **plain Django function-based view** (manually wired with
> `JSONParser`/`JSONRenderer`, not DRF's `APIView`), it does **not** get the browsable API for free —
> that benefit only shows up once the project moves to DRF's own class-based views, which later lectures
> in this unit build toward.

---

## Wrap-up

- **From video:** the serialization/deserialization recap and diagram; Python's `io` module, streams,
  and `io.BytesIO`; DRF's `JSONParser` and `JSONRenderer`; building a second app (`restapp2`) and
  `Manager` model from scratch; writing `ManagerSerializer` by hand (`serializers.Serializer`, not
  `ModelSerializer`) with an explicit `create()` method; the full `create_manager` function-based view
  (`@csrf_exempt`, `io.BytesIO`, `JSONParser().parse()`, `serializer.is_valid()`/`.save()`,
  `JSONRenderer().render()`, `HttpResponse`); wiring the URL and debugging a name-mismatch error live;
  testing inserts with a `test.py` script using the `requests` and `json` libraries; a validation-error
  demo (`name` field over its 20-character limit) showing `serializer.errors` rendered as JSON; and a
  closing preview of PUT/PATCH/DELETE and the (not-yet-covered) browsable API.
- **Gap-filled:** the likely meaning of the garbled "AMP" (Employee) reference to REST API Session 3's
  recap; the risk of a plain `Serializer`'s field declarations drifting out of sync with the model;
  the missing HTTP status codes on both response branches of `create_manager` (a real bug worth
  flagging); a `curl` equivalent of the `test.py` request; and the reasoning behind naming this
  lecture "deserialization" rather than keeping the original "testing & browsable API" working title,
  since the browsable API isn't actually demonstrated here.
- **Researched:** the official DRF description of serialization/deserialization; `io.BytesIO` from the
  Python docs; DRF's `ModelSerializer` as the more maintainable everyday alternative to a hand-written
  `Serializer`; `EmailField` vs. plain `CharField` for an email column; Django's CSRF protection and why
  DRF's class-based views typically avoid needing `@csrf_exempt` at all once token authentication is in
  use; and what the DRF browsable API actually is, ahead of the lecture that covers it directly.

Double-check against the completeness pass: recap of serialization ✓, the serialization/deserialization
diagram ✓, the `io` module/streams/`BytesIO` ✓, `JSONParser` ✓, creating the second app and `Manager`
model ✓, `ManagerSerializer` and its `create()` method ✓, the full view logic including `@csrf_exempt`
✓, URL setup and the live debugging of a broken URL ✓, `test.py` and the `requests`/`json` libraries ✓,
the validation-error demo ✓, the end-to-end flow recap ✓, and the closing preview of PUT/PATCH/DELETE
and the browsable API ✓. Nothing from the transcript was left out. The one thing genuinely **not**
covered in this lecture, despite the original working title, is the DRF browsable API itself — that's
explicitly deferred to a future session.
