# Lecture 17 — Retrieving one record, `__str__`, and customizing the admin

Source: `transcripts/Django17.txt`
Covers: fetching a single record with `.get(pk=...)`, the "not iterable" template gotcha that follows from it, the `__str__` method for readable admin entries, and the `ModelAdmin` class for table-style admin listings.

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official Django docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. Getting a single record **[From video]**

```python
emp = Employee.objects.get(pk=1)
```

`pk` (primary key) is a fixed, required keyword name here — not a name you choose — and `.get()` returns a single model instance, not a QuerySet.

> **Common pitfall:** A template written for `.all()` — with `{% for e in emp %}` — breaks on a single object returned by `.get()`, with *"Employee object is not iterable."* A single model instance isn't a list, so it can't be looped. The fix is a separate template (or a separate block) that accesses fields directly — `{{ emp.ename }}` — with no `{% for %}` at all.

## 2. Readable admin entries with `__str__` **[From video]**

By default, the admin panel lists every record of a model as `Employee object (1)`, `Employee object (2)`, … — not useful for telling records apart. Defining `__str__` on the model fixes this:

```python
class Employee(models.Model):
    enumber = models.IntegerField()
    ename = models.CharField(max_length=20)
    esalary = models.IntegerField()
    eaddress = models.CharField(max_length=20)

    def __str__(self):
        return str(self.enumber)
```

`__str__` is a standard Python method (not Django-specific) that's called whenever an object needs to be shown as text. Django's admin uses it to label each record. It must **return a string** — since `enumber` is an `IntegerField`, it has to be wrapped in `str(...)` first, or Django raises a `TypeError`.

## 3. Table-style listings with `ModelAdmin` **[From video]**

`__str__` only shows one field per record. To show every field, in columns, you register a `ModelAdmin` class instead of the bare model:

```python
# myapp/admin.py
from django.contrib import admin
from .models import Employee

class EmployeeAdmin(admin.ModelAdmin):
    list_display = ('enumber', 'ename', 'esalary', 'eaddress')

admin.site.register(Employee, EmployeeAdmin)

# -- or, equivalently, using the decorator style --
# @admin.register(Employee)
# class EmployeeAdmin(admin.ModelAdmin):
#     list_display = ('enumber', 'ename', 'esalary', 'eaddress')
```

`list_display` controls both which fields show as columns and their left-to-right order. Both registration styles do the same thing — `admin.site.register(Model, ModelAdmin)` is more explicit, `@admin.register(Model)` is a shorter decorator form.

> **[Gap-filled] — a likely interview question.** The video flags this directly: *"What is the use of ModelAdmin? What happens if it's missing?"* Answer: without a `ModelAdmin`, the admin panel still works, but every record shows as an opaque `Employee object (N)` unless you've defined `__str__` — and even then, you only see one field, not a full table row per record. `ModelAdmin` is what makes the admin list view look like an actual table.

---

## Wrap-up

- **From video:** `.get(pk=...)` for a single record, the iteration error it causes if the template still expects a QuerySet, `__str__` for readable admin labels, and `ModelAdmin`/`list_display` with both registration styles.
- **Gap-filled:** framing the ModelAdmin explanation as a direct answer to a likely interview question, since the video poses it that way itself.
- **Researched:** none needed this lecture.

Double-check: nothing flagged as incorrect this lecture — it's a clean, accurate walkthrough of single-record retrieval and admin customization.
