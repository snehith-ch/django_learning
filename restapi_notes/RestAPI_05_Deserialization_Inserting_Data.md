# REST API Session 5 — Deserialization: creating, updating & deleting records (POST/PUT/DELETE) with `JSONParser`

Source: `transcripts/restapi/REST API-5.txt`
Covers: continuing straight on from REST API Session 4 (the browsable-API/testing session), this lecture completes the **deserialization** side of the API — taking incoming JSON, turning it back into Python data, and using it to **insert (POST)**, **update (PUT)**, and **delete (DELETE)** records, then re-demonstrates **reading (GET)** in the same view for a full CRUD picture. Everything is built as one hand-written, function-based view that branches on `request.method`, and tested with a separate `test.py` script rather than the browsable API or a tool like Postman. The video is explicit that this verbose, manual style is for *understanding the mechanics only* — real projects should use DRF's generic API views instead, which is where the course is headed next.

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from the official [DRF docs](https://www.django-rest-framework.org/) or [Django docs](https://docs.djangoproject.com/)
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. Serialization vs. deserialization — where this lecture picks up **[From video]**

The video opens by recapping the split the course has been building toward:

> In the last session we were discussing about how to deserialize the data... that means how to insert the data, which is passing JSON data to Python type.

- **Serialization** (REST API Sessions 2–3): a Python object (a model instance, or a queryset) → converted into JSON so it can be sent to a client as an HTTP response. This is the *read* direction.
- **Deserialization** (this lecture): the reverse — JSON arriving in a request body → converted into Python data → used to create, update, or delete a database record. This is the *write* direction.

The instructor notes that the *insert* half of deserialization was technically already demonstrated in the previous session, and today's goal is to round it out with **update** and **delete**, plus a repeat pass over **read**, so that all four CRUD operations exist side by side in one view.

> **[Gap-filled] — "CRUD" as a term.** CRUD is a standard acronym for the four basic operations any data-backed application needs: **C**reate, **R**ead, **U**pdate, **D**elete. In plain Django these map to `Model.objects.create()`/`.save()`, `Model.objects.get()`/`.filter()`, `instance.save()` on a modified instance, and `instance.delete()`. In a REST API they conventionally map to HTTP verbs: **POST** = create, **GET** = read, **PUT**/**PATCH** = update, **DELETE** = delete. This lecture builds exactly that mapping, by hand, into a single view function.

---

## 2. The project so far: the `Manager` model and admin registration (recap) **[From video]**

The video briefly re-orients to the project set up in REST API Session 2 before adding new code:

- A model (the video calls it the **`Manager`** model) with fields: `name`, `address`, `mail` (email), `age`.
- Registered into `admin.py` via a `ModelAdmin` so the admin's list view shows `id`, `name`, `address`, `mail`, `age` as columns.

```python
# admin.py
from django.contrib import admin
from .models import Manager

@admin.register(Manager)
class ManagerAdmin(admin.ModelAdmin):
    list_display = ('id', 'name', 'address', 'email', 'age')  # columns shown in the admin list page
```

> **[Gap-filled]** The video doesn't re-dictate the `models.py` code in this session (it was written in an earlier one), only refers back to it. Based on the fields named, the model is a plain `models.Model` subclass — something like `name = CharField()`, `address = CharField()`, `email = EmailField()`, `age = IntegerField()`. It's included here only so the serializer code below makes sense; treat the exact field types as a reasonable reconstruction, not a verbatim quote.

The video also mentions, as a forward reference, that values can eventually be entered **at runtime through DRF's browsable API** — "in that view there will be provided in text box that text box values we can enter" — meaning DRF's auto-generated HTML forms let you type field values directly in the browser and submit a POST/PUT without any external script. That browsable-API workflow isn't built or demonstrated in *this* lecture (this lecture uses `test.py` instead); it's called out as something to expect soon.

---

## 3. From raw bytes to a Python dict: `JSONParser` **[From video]**

This is the mechanical heart of deserialization, and the video is careful to walk through every conversion step.

> The body should come from `test.py`... it should be stored into JSON format... on the JSON data we are converted by using JSON parser, because this is binary format... this binary format is converted into Python data by using JSON parser.

Breaking that down into what's actually happening on the server when a request arrives:

1. **`request.body`** — Django gives you the raw HTTP request body as a **bytes** object (binary data), not text and not a dict. Whatever the client sent (a JSON string) arrives here as raw bytes.
2. **Wrap it in a stream** — the video refers to "the Ivo method... available in the Ivo module" — a garbled way of saying the **`io`** module, specifically **`io.BytesIO`**, which wraps raw bytes into a file-like "stream" object (something with a `.read()` method).
3. **`JSONParser().parse(stream)`** — DRF's `JSONParser` class (from `rest_framework.parsers`) reads that stream, decodes the JSON text inside it, and returns a plain **Python `dict`** (or list of dicts). This is the actual deserialization step: JSON text → Python data structure.

```python
import json
from io import BytesIO
from django.http import HttpResponse
from rest_framework.parsers import JSONParser
from rest_framework.renderers import JSONRenderer
from .models import Manager
from .serializers import ManagerSerializer

def create_manager(request):
    if request.method == 'POST':
        stream = BytesIO(request.body)       # wrap the raw bytes so JSONParser can .read() them
        py_data = JSONParser().parse(stream)  # JSON text -> Python dict, e.g. {"name": "Sai", ...}
        # ... continued in section 4
```

> **[Researched] — the official DRF tutorial does this slightly differently.** DRF's own ["Working with function based views" tutorial](https://www.django-rest-framework.org/tutorial/1-serialization/) passes the `request` object straight into `JSONParser().parse(request)`, without manually wrapping it in `io.BytesIO`. That works because Django's `HttpRequest` already behaves enough like a file (it exposes a `.read()` over the underlying WSGI input) for `JSONParser` to consume it directly. Wrapping `request.body` in `io.BytesIO` yourself, as this video does, is a slightly more manual but equally valid alternative — and it's actually *necessary* if you've already accessed `request.body` elsewhere in the same view first (accessing `.body` reads the underlying stream once; wrapping the resulting bytes in a fresh `BytesIO` gives `JSONParser` something re-readable). Either style works; you'll see both in real DRF codebases.

---

## 4. Inserting a record: the serializer's `create()` method **[From video]**

Once `py_data` is a Python dict, it's handed to the serializer, and the standard three-step "deserialize and save" pattern happens:

```python
serializer = ManagerSerializer(data=py_data)   # unbound serializer, given incoming data to validate
if serializer.is_valid():                      # runs field validation (required fields, types, etc.)
    serializer.save()                          # valid -> persist to the database
    return HttpResponse("Data inserted", status=201)
return HttpResponse(serializer.errors, status=400)
```

- **`ManagerSerializer(data=py_data)`** — passing data in via the `data=` keyword (instead of `instance=`) tells the serializer "this is incoming data to validate and save," not "here's an existing object to represent as JSON."
- **`serializer.is_valid()`** — checks the incoming data against each field's rules (e.g. is `age` actually a number, is `email` a valid email format, are required fields present). Returns `True`/`False`.
- **`serializer.save()`** — only call this after `is_valid()` returns `True`. Internally, `save()` looks at whether the serializer was built with an `instance=` (existing object) or not, and calls either `.update()` or `.create()` accordingly. Since no `instance` was given here, `.create()` runs.

The `create()` method itself is what actually writes to the database:

```python
# serializers.py
from rest_framework import serializers
from .models import Manager

class ManagerSerializer(serializers.ModelSerializer):
    class Meta:
        model = Manager
        fields = ['id', 'name', 'address', 'email', 'age']

    def create(self, validated_data):
        return Manager.objects.create(**validated_data)
```

> **[Gap-filled] — this exact `create()` code isn't re-dictated in this transcript.** The video says the `create()` method "for inserting the data" was already written in the previous session and just reused here — it isn't retyped on screen in *this* recording. The version above is the standard, idiomatic implementation for exactly this situation (reconstructed from DRF conventions, not invented from nowhere): it unpacks the validated dict as keyword arguments into the model's `.objects.create()`.

> **[Researched] — you often don't need to write `create()`/`update()` at all.** Per the [DRF `ModelSerializer` docs](https://www.django-rest-framework.org/api-guide/serializers/#modelserializer), a `ModelSerializer` (unlike a plain `Serializer`) **already provides default `.create()` and `.update()` implementations** for free, based on the `Meta.model` you declared — you only need to override them when you want custom behavior beyond "just save these fields" (e.g. hashing a password field, creating a related object at the same time, sending a welcome email on creation). This video writes them out explicitly for teaching purposes — so the flow is visible step by step — but in a real `ModelSerializer`-based project, plain create/update logic like this needs no override at all.

### 4a. Why `serializer.errors` matters **[Gap-filled]**

The transcript doesn't dwell on the failure path, but it's a core part of the same pattern the assignment for this lecture calls out explicitly. If `serializer.is_valid()` returns `False` (say, `age` was sent as `"forty-five"` instead of a number, or a required field was left out), the serializer does **not** raise an exception — it just doesn't save. The specific problems are collected in **`serializer.errors`**, a dictionary keyed by field name:

```python
if not serializer.is_valid():
    print(serializer.errors)
    # {'age': ['A valid integer is required.']}
```

**Best practice:** always branch on `is_valid()` and return `serializer.errors` (with an HTTP `400 Bad Request` status) when it fails, rather than silently ignoring bad input or letting an unhandled exception surface as a 500 error. This is the "otherwise there are some [errors]" the video gestures at but doesn't spell out.

### Example — the insert pipeline in isolation **[Example]**

Sample input → output for `create_manager`, stripped down to just the concept:

```
Input (JSON sent in the POST body):
{"name": "Sai", "address": "Hyderabad", "email": "sai@example.com", "age": 27}

Pipeline:
request.body (bytes)
  -> BytesIO(...)                       (readable stream)
  -> JSONParser().parse(...)            (Python dict: py_data)
  -> ManagerSerializer(data=py_data)    (unbound, holds raw data)
  -> serializer.is_valid()              -> True
  -> serializer.save()                  -> calls .create(validated_data)
                                         -> Manager.objects.create(**validated_data)

Output: a new row in the `manager` database table with id=<next pk>, name="Sai", ...
```

A realistic use case: a mobile app's "Add employee" screen collects name/address/email/age in a form, serializes that form to JSON, and `POST`s it to this endpoint — this exact pipeline is what turns that JSON into a permanent database row.

---

## 5. Testing the insert with `test.py` **[From video]**

Rather than a browser or Postman, this course tests the API with a small standalone Python script, run from a second terminal window while `manage.py runserver` runs in the first:

```python
# test.py
import json
import requests

url = "http://127.0.0.1:8000/create-manager/"   # the project's URL for this view

def post_record():
    data = {"name": "Sai", "address": "Hyderabad", "email": "sai@example.com", "age": 27}
    json_data = json.dumps(data)          # Python dict -> JSON string (this is *serialization* on the client side)
    response = requests.post(url, json_data)
    print(response.text)

post_record()
```

> **[Gap-filled]** The transcript never names the `requests` library explicitly, but describes exactly its usage pattern (building a dict, `json.dumps()`-ing it, sending it to the running server, and printing/inspecting what comes back) across every operation in this lecture. `requests` is the standard third-party Python HTTP client library and is the natural fit here; it's used consistently in this reconstruction for all four operations (POST/PUT/DELETE/GET) below.

Demonstrated flow: start `manage.py runserver` in one terminal window, run `python test.py` in a second window, then check the Django admin panel (refresh the `Manager` table) to confirm a new row appeared. The video does exactly this and confirms "it's inserted a new record successfully."

---

## 6. Updating a record: the serializer's `update()` method **[From video]**

This is the first fully new code written in the session, and the transcript gives it almost verbatim:

> How to take instance — it's my first parameter. Then which value I want to update... instance.name = validated_data.get('name')... same for address, mail, age... then instance.save()... and I want to return this instance also.

```python
# serializers.py (continued)
class ManagerSerializer(serializers.ModelSerializer):
    class Meta:
        model = Manager
        fields = ['id', 'name', 'address', 'email', 'age']

    def create(self, validated_data):
        return Manager.objects.create(**validated_data)

    def update(self, instance, validated_data):
        instance.name = validated_data.get('name')        # take each field from the incoming data...
        instance.address = validated_data.get('address')  # ...and overwrite it on the existing object
        instance.email = validated_data.get('email')
        instance.age = validated_data.get('age')
        instance.save()          # persist the changed fields back to the database
        return instance          # DRF expects update() to return the (now-updated) instance
```

- **`instance`** — the *existing* database object being modified (a `Manager` row already in the table, fetched by primary key before the serializer is even constructed — see below).
- **`validated_data`** — the new field values from the incoming request, already checked by `is_valid()`.
- **`.get('name')`** — dictionary `.get()`, used instead of `validated_data['name']` so a missing key returns `None` rather than raising a `KeyError`.
- Every field must be reassigned by hand, then a single `instance.save()` writes all the changes to the database in one go, and the method must `return instance` — DRF's `serializer.save()` uses that return value as `serializer.instance` afterward.

> **[Gap-filled] — a subtlety the video glosses over.** Using `.get('name')` with no default means that if a field is *missing* from the incoming JSON, `instance.name` gets overwritten with `None` — silently wiping that field, not "leaving it unchanged." That's fine for a full `PUT` (which is supposed to replace *all* fields), but it's exactly the wrong behavior for a *partial* update — see the `PATCH`/`partial=True` note in section 8.

### Wiring `update()` into the view — fetching the right record first **[From video]**

Unlike POST, an update needs to know **which** row to modify. The video is explicit that this is the most important new piece: "which ID you want to update — without it, update cannot happen."

```python
from .models import Manager

def create_manager(request):
    if request.method == 'POST':
        ...  # as in section 4

    elif request.method == 'PUT':
        stream = BytesIO(request.body)
        py_data = JSONParser().parse(stream)
        id = py_data.get('id')                      # the request body must include which record to update
        manager = Manager.objects.get(id=id)         # fetch the existing row first
        serializer = ManagerSerializer(manager, data=py_data)  # instance + data => update(), not create()
        if serializer.is_valid():
            serializer.save()
            return HttpResponse("Data updated", status=200)
        return HttpResponse(serializer.errors, status=400)
```

The key line is `ManagerSerializer(manager, data=py_data)`: passing **both** an existing `instance` (`manager`, the first positional argument) **and** `data=` is what makes `serializer.save()` route to `.update(instance, validated_data)` instead of `.create(validated_data)`. This is the mechanism the video calls "same code — copy and paste — just the method changed from post to put," but functionally the branch does something quite different under the hood because an `instance` is now supplied.

> **[Researched]** This create-vs-update routing is documented directly in DRF's [`Serializer.save()` source/docs](https://www.django-rest-framework.org/api-guide/serializers/#saving-instances): *"If the serializer was instantiated with an existing `instance`, calling `.save()` will update that instance... If it was instantiated without an existing instance, calling `.save()` will create a new instance."* That single `if instance is not None` check is the entire mechanism — no magic beyond that.

---

## 7. Testing PUT — and a live debugging pitfall from the video **[From video]**

```python
# test.py (continued)
def update_data():
    data = {"id": 1, "name": "Ganesh", "address": "Ameerpet", "email": "ganesh@example.com", "age": 49}
    json_data = json.dumps(data)
    response = requests.put(url, json_data)   # must be .put(), not .post()
    print(response.text)
```

The video hits a genuine bug while demonstrating this, worth keeping in the notes because it's a realistic mistake:

> This is not an insertion, sir — something went wrong... Now you can see... we forgot to use the PUT method here — this is the mistake I'm doing in `test.py`... previously we used POST, but here we still had `requests.post` instead of `requests.put`... it inserted into the database instead of updating.

He'd left `requests.post(...)` in `update_data()` after copy-pasting the function, so the client silently hit the `if request.method == 'POST':` branch on the server instead of the `elif request.method == 'PUT':` branch — no error was raised anywhere, it just quietly did the wrong operation (inserted a new row) instead of the intended one (updating an existing row).

> **Industry best practice / pitfall:** when a client script and a server view both branch on HTTP method, a mismatch between the two doesn't usually raise an exception — it just silently executes the *other* branch (or no branch, returning nothing) as if that's what you asked for. There's no automatic safety net catching "you meant PUT but sent POST." Always double check that the HTTP verb your client actually sends (`requests.post` / `.put` / `.delete` / `.get`) matches the verb your view is checking for. This is exactly the kind of bug that DRF's generic views and viewsets (covered in upcoming lectures) help eliminate — they map each verb to a dedicated method (`.post()`, `.put()`, `.delete()`) instead of one big function with hand-written `if`/`elif` branches, so a mismatched verb gets DRF's own `405 Method Not Allowed` instead of quietly running the wrong branch.

Once corrected, the demonstrated result: ID 1's record (originally something like "Durga / durga@email.com / 45") is overwritten to "Ganesh / ganesh@gmail.com / 49," confirmed by restarting the server, running `test.py`, and refreshing the admin panel.

---

## 8. Partial updates: `PATCH` and `partial=True` (forward reference) **[From video]**

The video flags, without building it out yet, that DRF also supports **partial updates**:

> We have to use the PATCH method also... I can include that in the next communication, but here — if you want to do a partial update, you can pass `partial=True`.

```python
serializer = ManagerSerializer(manager, data=py_data, partial=True)
```

> **[Researched] — filling in what `partial=True` actually does.** By default, a serializer used for update assumes you're sending *every* field (a full `PUT`) — any field missing from the incoming data is treated as `None`/blank during validation (and, as noted in section 6, will overwrite that field with `None` if your `update()` uses plain `.get('field')`). Passing **`partial=True`** relaxes this: `is_valid()` no longer requires every field to be present, so you can send just `{"id": 1, "age": 50}` and only touch that one field. Per the [DRF docs](https://www.django-rest-framework.org/api-guide/serializers/#partial-updates), this is exactly what the `PATCH` HTTP verb is conventionally for (as opposed to `PUT`, which conventionally replaces the whole resource) — REST convention: `PUT` = full replace, `PATCH` = partial modify. Note that if you use `partial=True`, your `update()` method also needs to be written more carefully — using `validated_data.get('name', instance.name)` (fall back to the *current* value if the field wasn't sent) rather than `validated_data.get('name')` (which defaults to `None` and would wipe the field) — otherwise a partial update would still blank out any field you didn't include.

---

## 9. Deleting a record — no serializer needed **[From video]**

The video makes an important structural point here:

> For deletion purpose, reading purpose — [the serializer class] is not required at all... directly `manager.delete()` — no need to take it through serializer.py at all.

```python
elif request.method == 'DELETE':
    stream = BytesIO(request.body)
    py_data = JSONParser().parse(stream)
    id = py_data.get('id')
    manager = Manager.objects.get(id=id)
    manager.delete()                          # plain Django ORM delete — no serializer involved
    result = {"message": "data deleted"}
    json_data = json.dumps(result)
    return HttpResponse(json_data, content_type='application/json')
```

Deleting a row doesn't involve turning data *into* a model instance's fields (that's what serializers are for) — you already have the exact row (fetched by `id`), and `.delete()` is a plain Django ORM method on any model instance, nothing DRF-specific. The response is just a small hand-built JSON confirmation message, built with plain `json.dumps()` rather than a serializer, since there's no model data left to represent after deletion.

> **[Gap-filled]** This is a good moment to state the rule the video is building toward across the whole lecture: **a `ModelSerializer`'s `create()`/`update()` methods matter only for the write operations that go through `serializer.save()`** — i.e., POST and PUT/PATCH. `GET` (read) only ever needs the serializer to go *forward* (instance → JSON), never `.save()`. `DELETE` doesn't need the serializer at all, because deleting is a one-line ORM call on an object you already have, with nothing to validate or convert.

---

## 10. Testing DELETE with `test.py` **[From video]**

```python
def delete_data():
    data = {"id": 2}
    json_data = json.dumps(data)
    response = requests.delete(url, json_data)
    print(response.text)
```

Demonstrated: starting from 3 existing records, calling `delete_data()` with `"id": 2` removes that row; refreshing the admin panel afterward shows only 2 records remain, confirming the delete worked.

---

## 11. Reading again: GET for one record or all records, in the same view **[From video]**

To round out CRUD in the same function, the video adds back a `GET` branch (conceptually a repeat of REST API Session 3's serialization work, now reusing the same `JSONParser`-based body-reading pattern for consistency):

```python
elif request.method == 'GET':
    stream = BytesIO(request.body)
    py_data = JSONParser().parse(stream)
    id = py_data.get('id', None)               # default to None if no id was sent

    if id is not None:
        manager = Manager.objects.get(id=id)              # one specific record
        serializer = ManagerSerializer(manager)            # single instance -> no many=True
    else:
        managers = Manager.objects.all()                   # every record
        serializer = ManagerSerializer(managers, many=True)  # a queryset -> many=True required

    json_data = JSONRenderer().render(serializer.data)  # Python data -> JSON bytes
    return HttpResponse(json_data, content_type='application/json')
```

- **`serializer.data`** — once a serializer is built from an *instance* (not from `data=`), `.data` gives you the serialized (JSON-ready) representation as an ordered Python dict.
- **`JSONRenderer().render(...)`** — DRF's `JSONRenderer` (from `rest_framework.renderers`) converts that Python dict into actual JSON-formatted bytes, ready to go in an HTTP response body — the mirror image of `JSONParser`, which goes the other direction.
- **`many=True`** — required whenever you're serializing a queryset/list of multiple objects rather than a single instance; it tells the serializer to loop over each item and produce a JSON array instead of a single JSON object.

> **[Researched] — a simpler, more idiomatic alternative for this exact GET branch.** Django itself ships `django.http.JsonResponse`, which can take a Python dict/list directly and handle the JSON-encoding for you: `return JsonResponse(serializer.data, safe=False)` (the `safe=False` flag is required when passing a list instead of a dict, e.g. for the `many=True` case). This does the same job as `JSONRenderer().render(...)` + `HttpResponse(..., content_type='application/json')` with less code, and is exactly what the [official DRF function-based-views tutorial](https://www.django-rest-framework.org/tutorial/1-serialization/#working-with-request-and-response-objects) uses. The video's `JSONRenderer` + `HttpResponse` approach is not wrong — it's just more manual, and useful to know both, since you'll see either style in real DRF code.

### Testing GET with `test.py` **[From video]**

```python
def get_data(id=None):
    data = {"id": id}
    json_data = json.dumps(data)
    response = requests.get(url, json_data)   # note: sending a body on a GET request
    print(response.text)

get_data(1)   # -> just the record with id=1
get_data()    # -> id defaults to None -> every record
```

Demonstrated both ways: calling with `id=1` returns just that one record's JSON; calling with no `id` returns the full list (2 records remained in the database at that point in the demo, after the earlier delete).

> **Pitfall / industry best practice:** sending a request **body** on a `GET` request (as `test.py` does here, and as the server-side code above expects, by calling `JSONParser().parse()` even inside the `GET` branch) is unusual and goes against REST convention. `GET` requests are conventionally expected to be **safe and idempotent** and to carry all of their parameters in the **URL** — either as path segments (e.g. `/managers/1/`) or query parameters (e.g. `/managers/?id=1`) — not in a request body. Some HTTP clients, servers, and proxies don't reliably support a GET body at all. This video's approach works here only because both the custom `test.py` script and the custom view were written by the same person to agree with each other; a real API consumer (a browser, a mobile app, another team's service) would expect `id` in the URL, not the body. This is another thing DRF's generic views (`RetrieveAPIView`, etc., covered soon) handle correctly by default, by reading the id from the URL.

---

## 12. Why this is "for understanding only" — and where the course goes next **[From video]**

The video closes with a candid, important framing of everything just built:

> In our upcoming session, for inserting data, getting the data — this many lines of code, no need to write... I'm not encouraging you to write this type of code in real time also... Django REST Framework has a lot of flexible ways — generic API views are there, API views are there — using that we can easily fulfill the requirement. This is for idea purpose and knowledge purpose only... don't write this type of code in real time.

Key points made explicitly:

- **The rule recapped:** for POST and PUT, the serializer class needs `create()`/`update()` methods; for GET and DELETE, it doesn't need any extra methods at all.
- **All four operations share one URL/view function** in this lecture (`create_manager`, or whatever it's routed as) — the *only* thing distinguishing insert/update/delete/read is the `if`/`elif` branch on `request.method`. That's a deliberately manual, teaching-oriented design, not how a production DRF app is normally structured.
- **This entire hand-written approach exists purely so the underlying mechanics are visible** — how a request's raw bytes actually become a saved database row, step by step — before those steps get hidden behind DRF's built-in generic views and `APIView` classes, which do the same work with far less code.
- **Two things are explicitly promised for upcoming sessions:** the same CRUD operations rebuilt using **class-based views** (so the function-based vs. class-based contrast is visible directly), and testing moving away from a standalone `test.py` script toward DRF's own **browsable API** — the auto-generated in-browser interface that shows GET/PUT/PATCH/POST/DELETE options automatically for any endpoint, without a separate script.

> **[Gap-filled]** Don't take "don't write this type of code in real time" as meaning this lecture's content was wasted effort — quite the opposite. Understanding exactly what `JSONParser`, `is_valid()`, `.save()`, and `serializer.errors` are each individually responsible for (built by hand, once) is what makes DRF's generic views legible later — those classes are, under the hood, doing almost exactly this sequence of steps automatically. Skipping straight to generic views without having built this manually first tends to leave "how does `CreateAPIView` actually save my data?" as a black box.

---

## Wrap-up

- **From video:** the full deserialization pipeline (`request.body` → `io.BytesIO` → `JSONParser().parse()` → Python dict → serializer); inserting via `ManagerSerializer(data=...)` + `is_valid()` + `.save()` (routing to `create()`); updating via `ManagerSerializer(instance, data=...)` (routing to `update()`, with the full `update()` method dictated field-by-field); a live debugging moment where a leftover `requests.post` (instead of `.put`) in `test.py` silently inserted instead of updating; a brief forward-reference to `PATCH`/`partial=True`; deleting via a plain `manager.delete()` with no serializer involved; re-demonstrating GET for a single record and for all records (`many=True`) inside the same view; testing every operation via a standalone `test.py` script using (implicitly) the `requests` library; and an explicit statement that this manual, function-based, single-view style is for teaching the mechanics only, not a real-world pattern — generic API views, class-based CRUD, and the browsable API are flagged as upcoming replacements.
- **Gap-filled:** the CRUD acronym; reconstructed `models.py`/`create()` code not re-dictated in this transcript; the `serializer.errors`/400-response failure path (named in this lecture's assignment brief but not dwelt on in the transcript); the `.get('field')`-defaults-to-`None` trap that partial updates need to avoid; the "which methods does a serializer actually need" rule stated explicitly as a takeaway; framing of why hand-writing this once is pedagogically useful before moving to generic views.
- **Researched:** the official DRF tutorial's slightly different (but equivalent) `JSONParser().parse(request)` pattern; `ModelSerializer`'s built-in default `create()`/`update()` implementations (meaning this lecture's explicit versions are usually optional in real projects); the exact `save()` → `create()`/`update()` routing rule from DRF's docs; what `partial=True` and `PATCH` are for; `JsonResponse` as a simpler alternative to `JSONRenderer().render()` + `HttpResponse`; why sending a body on a GET request (as demonstrated) goes against REST convention.

Double-check against the completeness pass: recap of serialization vs. deserialization ✓; `Manager` model/admin recap ✓; browsable-API forward reference ✓; `io.BytesIO`/`JSONParser` mechanics ✓; POST/`create()` ✓; `serializer.errors` ✓; testing POST via `test.py` ✓; PUT/`update()` with full field-by-field code ✓; the POST-vs-PUT debugging pitfall ✓; `PATCH`/`partial=True` forward reference ✓; DELETE with no serializer ✓; testing DELETE ✓; GET for one record and all records (`many=True`) reusing the same view ✓; testing GET, including the body-on-GET oddity ✓; the "don't write this in real code" closing framing and the generic-views/class-based-views/browsable-API forward references ✓. Nothing from the transcript was left uncovered.
