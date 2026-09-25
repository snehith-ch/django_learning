# REST API Session 7 — DRF Validations (Field-level, Object-level & Validators) and the ModelSerializer

Source: `transcripts/restapi/REST API-7.txt`

Covers: how to validate incoming data in Django REST Framework using the three built-in mechanisms — **field-level validation**, **object-level validation**, and **validators** — all demonstrated on the same `EmployeeSerializer` from the previous lecture's project, followed by an introduction to **`ModelSerializer`** (DRF's equivalent of Django's `ModelForm`) and its `read_only_fields` option.

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official DRF/Django docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## 0. A note on this lecture's actual content **[Gap-filled]**

The working title carried over for this lecture number was "CRUD with class-based views (APIView)" — but after reading this transcript in full, that isn't what it covers. The opening lines of the transcript make the actual sequence clear:

> So in the last session, we will discuss about current [CRUD] operation using... APAV [APIView], current API using class-based review [view]. We will discuss. So here we have only one class. And in this class, I have included all the methods like post-method, foot [put] method, patch method, all the methods I included here. This is class-based review, current APAVU [APIView]. Now today we look into another... Application. So that is Django, REST Framework, validations.

In other words: **the CRUD-with-`APIView` rewrite (one class holding `get`, `post`, `put`/`patch`, `delete` methods instead of separate `@api_view` functions) was the *previous* lecture's topic**, referenced here only as "last session." This lecture (the one actually transcribed in `REST API-7.txt`) picks up immediately afterward and is about something different: **DRF's data-validation system**, demonstrated on the very same `EmployeeSerializer`/`Employee` model used in the last couple of lectures, followed by **`ModelSerializer`**. These notes describe what this transcript actually teaches — validation and `ModelSerializer` — while keeping the lecture number fixed and briefly recapping the class-based `APIView` context that opens the session, since the next lecture's notes may refer back to it.

---

## 1. Recap: last session's class-based `APIView` for full CRUD **[From video]**

The video opens by summarizing what the previous lecture built: a single class-based view, subclassing DRF's `APIView`, that bundled **all** the CRUD operations for the `Employee` resource into one class — a `get` method (read), a `post` method (create), a `put`/`patch` method (update), and a `delete` method (remove) — rather than the separate `@api_view`-decorated functions used even earlier in the course. That single class is the file this lecture keeps editing (`serializer.py`, referred to throughout as "serializer dot py") to add validation logic on top of.

> **[Gap-filled] — why this matters for reading the rest of this lecture.** Because the previous lecture already had full CRUD working end-to-end (insert, update, delete all demonstrated and confirmed against the database), this lecture doesn't rebuild any of that. It reuses the *exact same* `EmployeeSerializer` and `Employee` model, and everything below is purely about tightening up **what data is allowed in**, not about the view layer at all.

---

## 2. What is validation, and DRF's three approaches **[From video]**

**Validation**, in plain terms, is the process of checking whether the data a user (or API client) submitted is actually correct/acceptable before it's allowed into the database. If the submitted data passes the checks, it's allowed through; if it fails, the request is rejected with an error message explaining what's wrong. This is the exact same concept as Django's own form validation (`clean_<field>()`, `clean()`, custom validators on a `forms.Field`) — the video explicitly calls this out — just applied to DRF's **serializers** instead of Django **forms**, since a DRF API doesn't use HTML forms at all.

DRF gives you **three different ways** to perform validation on a serializer:

| # | Technique | Scope | Method/mechanism |
|---|-----------|-------|-------------------|
| 1 | **Field-level validation** | One specific field only | `validate_<field_name>(self, value)` |
| 2 | **Object-level validation** | Multiple fields (or all fields) at once, together | `validate(self, data)` |
| 3 | **Validators** | A specific field, via a standalone reusable function | a plain function, attached with `validators=[...]` on the field |

All three live inside the same serializer file. The video works through them one at a time, on the same `EmployeeSerializer`, before moving on to `ModelSerializer`.

---

## 3. Field-level validation **[From video]**

**Field-level validation** checks one specific field only. The syntax is a method named `validate_<field_name>`, taking `self` and `value` (the value the client submitted for that one field):

```python
def validate_<field_name>(self, value):
    ...
```

The `Employee` model (and its serializer) has an `age` field. The video adds a check so that an age over 100 is rejected — clearly not a real age, but nothing was stopping it from being accepted before this check existed.

```python
# serializer.py — EmployeeSerializer
from rest_framework import serializers

class EmployeeSerializer(serializers.Serializer):
    name = serializers.CharField(max_length=50)
    email = serializers.EmailField()
    address = serializers.CharField(max_length=100)
    age = serializers.IntegerField()

    # ---- Field-level validation ----
    def validate_age(self, value):
        # `value` is whatever the client submitted for "age" specifically.
        if value > 100:
            # Rejecting the value: DRF's own exception type for a failed validation.
            raise serializers.ValidationError("Age should not exceed 100.")
        # A field-level validator MUST return the (possibly cleaned) value —
        # otherwise the field silently becomes None.
        return value
```

**How DRF calls this automatically:** because the method is named `validate_age`, DRF's serializer machinery calls it by itself — for the `age` field specifically — as part of `.is_valid()`. You never call `validate_age()` yourself; naming it this way is what wires it up.

**Demonstrated behavior (via a `test.py` script posting to the running API):**

```python
# test.py — POST a new employee record
import requests

url = "http://127.0.0.1:8000/employee/"
data = {"name": "Ganesh", "email": "ganesh@gmail.com", "address": "Hyderabad", "age": 102}
response = requests.post(url, json=data)
print(response.json())
```

```json
// Response — rejected, because age (102) is over 100
{"age": ["Age should not exceed 100."]}
```

Changing `age` to `25` and re-running the script inserts the record successfully — confirmed both by the script's printed response and by refreshing the Django admin panel to see the new row.

> **[Example] — field-level validation in isolation, on a different field.** The same pattern works for any single field. To reject an obviously fake email domain on a `Feedback` serializer:
> ```python
> def validate_email(self, value):
>     if value.endswith("@example.com"):
>         raise serializers.ValidationError("Please use a real email address.")
>     return value
> ```
> Sample input → output: `{"email": "test@example.com"}` → `{"email": ["Please use a real email address."]}`. `{"email": "test@gmail.com"}` → passes through unchanged.

---

## 4. A tangent: why the next inserted ID wasn't 1 or 2 **[Gap-filled]**

While re-testing inserts, the video notices the newly-inserted record doesn't get the ID you might expect (it isn't `1`, and `2` is visibly "missing") and briefly explains this is a side effect of all the insert/update/delete testing done in the *previous* lecture — some records were created and then deleted while building and testing the CRUD `APIView`.

> **[Researched] — auto-increment primary keys are never reused.** By default, Django model primary keys are an auto-incrementing integer field (`id`, an implicit `AutoField`). Per the underlying database engines' documented behavior (SQLite, MySQL, and PostgreSQL all behave this way), once an ID has been assigned — even if that row is later deleted — the counter does **not** roll back or reuse that number. The next insert always gets `(highest ID ever issued) + 1`, regardless of how many rows currently exist. This is normal, expected behavior, not a bug: relying on IDs being sequential/gap-free is a common beginner mistake, since any delete during testing (or in production) permanently leaves a gap.

---

## 5. Object-level validation **[From video]**

**Object-level validation** validates **multiple fields together, or all fields at once** — useful when one field's validity depends on another field's value (a check that no single field, in isolation, could express). The syntax is a method literally named `validate`, taking `self` and `data` (a dictionary of *all* the submitted, already-individually-validated field values):

```python
def validate(self, data):
    ...
```

The video's example: if the employee's `name` is `"mohan"` (case-insensitively), then their `address` is *required* to be `"hyderabad"` (also case-insensitively) — otherwise, reject the request.

```python
# serializer.py — EmployeeSerializer (continued)
    # ---- Object-level validation ----
    def validate(self, data):
        name = data.get('name')
        address = data.get('address')

        # .lower() normalizes both sides so "Mohan"/"MOHAN"/"mohan" all match,
        # and "Hyderabad"/"HYDERABAD"/"hyderabad" all match too.
        if name.lower() == 'mohan' and address.lower() != 'hyderabad':
            raise serializers.ValidationError("Address should be Hyderabad only.")

        # Object-level validation must also return the (validated) data dict.
        return data
```

**Why `.lower()` on both sides:** the video is explicit about this — comparing strings in Python is case-sensitive by default, so `"Mohan" == "mohan"` is `False`. Without lower-casing both the submitted value and the value being compared against, a client typing `"MOHAN"` or `"Mohan"` (instead of exactly `"mohan"`) would incorrectly fail the check even though it's clearly the same name. Converting both sides to the same case first makes the comparison case-insensitive.

**Demonstrated behavior:**

```python
# test.py
data = {"name": "Mohan", "email": "mohan@gmail.com", "address": "HYD", "age": 25}
response = requests.post(url, json=data)
print(response.json())
```

```json
// Rejected — address isn't "Hyderabad"
{"non_field_errors": ["Address should be Hyderabad only."]}
```

Correcting `"address"` to `"Hyderabad"` and re-posting inserts the record successfully.

> **[Gap-filled] — where the error message shows up.** Notice the key in the JSON response is `non_field_errors`, not `address`. That's because an error raised inside `validate(self, data)` isn't tied to any one field — DRF has no way to know whether the problem was really about `name` or `address`, since the check spans both — so it's reported at the object level instead of under a specific field's key.

> **[Example] — object-level validation with a genuinely two-field rule.** A `DateRangeSerializer` where an end date must come after a start date — something no single field's own validator could check, since it needs to see *both* values at once:
> ```python
> def validate(self, data):
>     if data['end_date'] <= data['start_date']:
>         raise serializers.ValidationError("end_date must be after start_date.")
>     return data
> ```
> Input `{"start_date": "2026-01-10", "end_date": "2026-01-05"}` → rejected. Input `{"start_date": "2026-01-10", "end_date": "2026-01-20"}` → passes.

---

## 6. Validators: reusable, standalone validation functions **[From video]**

The third mechanism is **validators** — ordinary, standalone functions (not methods on the serializer class) that contain validation logic and can be **reused across multiple serializers/fields**, since they aren't tied to any one serializer class. A validator function takes the value being validated and raises `serializers.ValidationError` if it's invalid:

```python
# serializer.py — a standalone validator function
from rest_framework import serializers

def starts_with_a(value):
    # value[0] is the first character of the submitted string.
    if value[0].lower() != 'a':
        raise serializers.ValidationError("Name should start with letter A only.")
    # (Validators don't need to return anything — unlike validate_<field>,
    # their only job is to raise on failure; the original value is used as-is.)
```

It's then **attached to a specific field** by passing it in that field's `validators=[...]` list — which means the field has to be declared explicitly on the serializer (rather than relying on defaults), so there's somewhere to attach it:

```python
class EmployeeSerializer(serializers.Serializer):
    name = serializers.CharField(max_length=50, validators=[starts_with_a])
    email = serializers.EmailField()
    address = serializers.CharField(max_length=100)
    age = serializers.IntegerField()
```

**Demonstrated behavior:**

```python
# test.py
data = {"name": "Ganesh", "email": "ganesh@gmail.com", "address": "Hyderabad", "age": 25}
response = requests.post(url, json=data)
print(response.json())
```

```json
// Rejected — "Ganesh" doesn't start with "A"
{"name": ["Name should start with letter A only."]}
```

Changing `"name"` to `"Anish"` and re-posting inserts the record successfully.

> **[Gap-filled] — how a validator function differs from `validate_<field_name>`.** Both check one field, and both raise the same `serializers.ValidationError`. The real difference is **reuse**: `validate_age` only ever exists inside `EmployeeSerializer` — copy-pasting it into a `ManagerSerializer` means duplicating the code. A standalone validator function like `starts_with_a` is just a normal Python function, defined once, that can be imported and attached to a `name` field on *any* serializer via `validators=[starts_with_a]`. DRF ships several ready-made validators of exactly this shape too (see the Researched note below).

> **[Example] — a validator reused across two different serializers.** This is the scenario that makes "reusable" concrete:
> ```python
> def starts_with_a(value):
>     if value[0].lower() != 'a':
>         raise serializers.ValidationError("Must start with letter A.")
>
> class EmployeeSerializer(serializers.Serializer):
>     name = serializers.CharField(validators=[starts_with_a])
>
> class ManagerSerializer(serializers.Serializer):
>     mname = serializers.CharField(validators=[starts_with_a])
> ```
> The same function, defined once, now guards a field on two unrelated serializers — no copy-pasted logic.

> **[Researched] — DRF's own built-in validators.** Per the [DRF documentation on validators](https://www.django-rest-framework.org/api-guide/validators/), DRF ships ready-made, importable validators for very common cases — e.g. `UniqueValidator` (rejects a value already present elsewhere in the table — commonly used to enforce a unique email), `RegexValidator` (from Django itself, matches a value against a regular expression), and `MaxValueValidator`/`MinValueValidator` for numeric ranges. Writing your own function like `starts_with_a` is exactly the pattern to reach for once a project's own custom business rule doesn't already exist as one of these.

---

## 7. Comparing the three validation techniques **[Gap-filled]**

| Technique | Method name | What it can see | Best for |
|---|---|---|---|
| Field-level validation | `validate_<field_name>(self, value)` | Just that one field's submitted value | A single field's own rule, specific to one serializer (e.g. "age can't exceed 100") |
| Object-level validation | `validate(self, data)` | Every field's submitted value at once (as a dict) | A rule that compares two or more fields against each other (e.g. "if name is X, address must be Y"; "end date after start date") |
| Validators | a plain function + `validators=[...]` on a field | Just the value passed to it | A single-field rule you want to **reuse** across multiple serializers/fields, or one of DRF's built-in validators |

All three can be combined on the same serializer — nothing stops a project from having a `validate_age`, a `validate`, and a `validators=[...]`-decorated field all in the same class, each enforcing a different rule.

> **Industry best practice.** Keep validation logic as close as possible to the narrowest mechanism that expresses it: a single-field rule belongs in `validate_<field>` (or a validator, if it'll be reused elsewhere); only reach for the broader `validate(self, data)` when a rule genuinely needs to compare multiple fields. Cramming every rule into one giant `validate()` method makes it harder to tell, later, which field a given error message is actually about — and (as seen above) errors raised from `validate()` show up as generic `non_field_errors` rather than pointing at a specific field, which is a worse experience for whoever's consuming the API.

---

## 8. `ModelSerializer` **[From video]**

Once a few different validation techniques have been shown, the video pivots to a much bigger simplification: **`ModelSerializer`**.

**The problem it solves:** so far, `EmployeeSerializer` has been a plain `serializers.Serializer` — meaning every field on the `Employee` model had to be **manually re-declared** on the serializer (`name = serializers.CharField(...)`, `email = serializers.EmailField()`, and so on), and a `create()` and `update()` method had to be **manually written** to actually save data. If the model has 4 fields, the serializer repeats all 4 fields again by hand — and if the model ever gains a 5th field, the serializer must be remembered and updated to match.

> **[Gap-filled] — this is the exact same problem `ModelForm` solves in plain Django.** The video draws this comparison directly: Django's own `forms.ModelForm` (covered earlier in this course) exists so a form doesn't need every model field re-typed by hand — you just point it at the model (`class Meta: model = Student`) and it generates the fields itself. `ModelSerializer` is DRF's version of exactly that idea, applied to serializers instead of forms.

A `ModelSerializer`:

1. **Automatically generates a set of serializer fields** based on the model's own fields — no manual re-declaration needed.
2. **Automatically generates validators** for those fields, based on the model's own field constraints (e.g. a model's `max_length`, `unique=True`, `null=False` are all picked up automatically).
3. **Provides a default implementation of `.create()` and `.update()`** — so those methods, previously written by hand, are no longer required at all.

---

## 9. Refactoring `EmployeeSerializer` into a `ModelSerializer` **[From video]**

Switching over is a small, dramatic-looking diff. Before:

```python
# serializer.py — the old, plain Serializer version (everything hand-written)
class EmployeeSerializer(serializers.Serializer):
    name = serializers.CharField(max_length=50)
    email = serializers.EmailField()
    address = serializers.CharField(max_length=100)
    age = serializers.IntegerField()

    def create(self, validated_data):
        return Employee.objects.create(**validated_data)

    def update(self, instance, validated_data):
        instance.name = validated_data.get('name', instance.name)
        instance.email = validated_data.get('email', instance.email)
        instance.address = validated_data.get('address', instance.address)
        instance.age = validated_data.get('age', instance.age)
        instance.save()
        return instance
```

After:

```python
# serializer.py — the new ModelSerializer version
from rest_framework import serializers
from .models import Employee

class EmployeeSerializer(serializers.ModelSerializer):
    class Meta:
        model = Employee       # which model this serializer mirrors
        fields = '__all__'     # include every field the model has
```

That's the entire serializer now — no field declarations, no `create()`, no `update()`. `class Meta` is the same pattern as `ModelForm`'s `class Meta: model = ...; fields = ...` — a nested class that tells `ModelSerializer` which model to base itself on and which of that model's fields to include.

**Demonstrated behavior:** posting a new record (`{"name": "...", "email": "...", "address": "...", "age": ...}`) through `test.py` against this much shorter serializer still inserts successfully — confirmed in the admin panel — proving the auto-generated fields and default `create()` work exactly like the hand-written version did.

> **[Gap-filled] — `fields = '__all__'` vs. naming specific fields.** Just like `ModelForm`, `fields` doesn't have to be `'__all__'`. Passing a list instead — `fields = ['name', 'address', 'age']` — includes only those named fields in the serializer (e.g. deliberately leaving `email` out of what the API exposes), while `'__all__'` is the "include everything" shortcut used in this lecture's demo.

---

## 10. `read_only_fields` **[From video]**

Still inside `class Meta`, `ModelSerializer` supports a **`read_only_fields`** option: a list of field names that can be **read** (returned in responses) but **not written to** (any value submitted for them on a create/update is silently ignored).

```python
class EmployeeSerializer(serializers.ModelSerializer):
    class Meta:
        model = Employee
        fields = '__all__'
        read_only_fields = ['address', 'age']
```

**Demonstrated behavior:** an existing record (`id=5`, originally `name="Durga"`, `email="durga@gmail.com"`, `address="Hyderabad"`, `age=45`) is updated via `test.py`, attempting to change all four fields at once:

```python
# test.py — attempting to update record id=5
data = {"name": "Ramesh", "email": "ramesh@gmail.com", "address": "YJOK", "age": 34}
response = requests.put("http://127.0.0.1:8000/employee/5/", json=data)
print(response.json())
```

```json
// name and email DID update; address and age were silently ignored
{"id": 5, "name": "Ramesh", "email": "ramesh@gmail.com", "address": "Hyderabad", "age": 45}
```

`name` and `email` — not listed in `read_only_fields` — updated as submitted. `address` and `age` — listed in `read_only_fields` — kept their original values (`"Hyderabad"` and `45`) despite the request submitting different ones (`"YJOK"` and `34`). No error was raised; the read-only values were just quietly not applied.

> **Industry best practice.** `read_only_fields` (or an explicit `read_only=True` on an individual field declaration) is the standard way to expose data a client should be able to *see* but never directly *set* — the most common real example being a model's own primary key `id`, or a `created_at`/`updated_at` timestamp: useful in every response, but something the server — never the client — should control. Relying on "the frontend just won't send that field" is not a substitute for this, since any API client can send whatever JSON it wants regardless of what a particular frontend does.

> **[Researched] — the difference between `read_only_fields` and leaving a field out of `fields` entirely.** Per the [DRF documentation on `ModelSerializer`](https://www.django-rest-framework.org/api-guide/serializers/#specifying-read-only-fields), a field named in `read_only_fields` is still **included in every serialized response** (GET/POST/PUT output) — it's just protected from being written to. Leaving a field out of `fields` altogether (or listing it in the separate `exclude` option) means it never appears in responses at all. Use `read_only_fields` for "show it, but don't let clients change it"; leave a field out of `fields` for "never expose it via the API at all."

> **[Example] — a realistic use case for `read_only_fields`.** An `OrderSerializer` where a customer can update their shipping address but should never be able to change the price they were quoted or the order's status:
> ```python
> class OrderSerializer(serializers.ModelSerializer):
>     class Meta:
>         model = Order
>         fields = '__all__'
>         read_only_fields = ['price', 'status']
> ```
> A client PATCHing `{"shipping_address": "New address", "price": 0.01, "status": "shipped"}` would have their `shipping_address` change go through, while `price` and `status` are quietly left untouched — exactly the same shape as this lecture's `address`/`age` demo.

---

## 11. Custom validation still works on a `ModelSerializer` **[Researched]**

The transcript doesn't explicitly re-demonstrate `validate_age`/`validate()`/custom `validators=[...]` on top of the new `ModelSerializer` version — but it's worth being explicit that switching to `ModelSerializer` **only replaces the boilerplate** (field declarations, `create()`, `update()`); it does **not** remove the ability to add custom business-rule validation. Per the [DRF documentation](https://www.django-rest-framework.org/api-guide/serializers/#modelserializer), `validate_<field_name>(self, value)` and `validate(self, data)` work exactly the same way on a `ModelSerializer` as they did on a plain `Serializer` — you'd simply add them as methods on the `ModelSerializer` subclass, same as before. What `ModelSerializer` auto-generates for you are the validators implied by the **model's own field definitions** (e.g. a `CharField(max_length=50)` on the model automatically gets a matching max-length check on the serializer field) — it has no way to know about a business rule like "age can't exceed 100" unless the model itself enforces it (e.g. via a `validators=[MaxValueValidator(100)]` on the model field) or the serializer adds `validate_age` back in explicitly.

---

## 12. What's next **[From video]**

The lecture closes by previewing the next session's topic: reducing the amount of view-level code even further. The video points out that, even with `ModelSerializer` cutting down the *serializer* file, the *view* — the `APIView` class from the earlier recap, with its separate `get`/`post`/`put`/`delete` method bodies — still has a fair amount of repeated code in each method. The next lecture introduces DRF's **generic API views**, which come with built-in `get`/`post`/`put`/`patch`/`delete` behavior already implemented, cutting each view method down to just a couple of lines.

> **[Gap-filled] — connecting this forward to the rest of the unit.** This is the same direction the rest of this DRF unit's lecture list continues in: generic API views and mixins, then concrete generic view classes, then `ViewSet`s — each step trading a little explicitness for a lot less repeated boilerplate, the same trend visible here going from a hand-written `Serializer` to `ModelSerializer`.

---

## Wrap-up

- **From video:** a recap of the previous lecture's class-based `APIView` (one class, all CRUD methods); what validation means and DRF's three mechanisms — field-level (`validate_<field_name>`), object-level (`validate(self, data)`), and validators (standalone reusable functions attached via `validators=[...]`) — each demonstrated live against the running API with `test.py`; a tangent on auto-incrementing IDs not resetting after a delete; `ModelSerializer` (automatic fields, automatic validators, default `create()`/`update()`), refactoring `EmployeeSerializer` to use it; `read_only_fields` and its demonstrated behavior (some fields update, protected ones silently don't); a preview of generic API views as the next topic.
- **Gap-filled:** an explicit note that this lecture's actual content (validations + `ModelSerializer`) differs from the assigned working title (which describes the *previous* lecture's `APIView` CRUD rewrite); why `.lower()` is needed for case-insensitive comparisons in object-level validation; why an object-level validation error shows up as `non_field_errors`; the difference between `validate_<field>` and a standalone validator function; a comparison table of all three validation techniques; a best-practice note on keeping validation logic narrowly scoped; a realistic `read_only_fields` use case (protecting `price`/`status` on an order); a note connecting this lecture to the rest of the DRF unit's progression toward less boilerplate.
- **Researched:** why database auto-increment IDs are never reused after a delete; DRF's own built-in validators (`UniqueValidator`, etc.) as an alternative to writing custom ones; the precise difference between `read_only_fields` and omitting a field from `fields` entirely; confirmation that custom `validate_<field>`/`validate()` still work normally on a `ModelSerializer` (it only replaces the model-mirroring boilerplate, not custom rules).

Double-check against the completeness checklist: field-level validation ✓, object-level validation ✓, validators ✓, the auto-increment ID tangent ✓, `ModelSerializer` and its three benefits ✓, the `EmployeeSerializer` refactor ✓, `read_only_fields` ✓, the recap of the previous lecture's `APIView` ✓, the closing preview of generic API views ✓. Nothing from the transcript was left out.
