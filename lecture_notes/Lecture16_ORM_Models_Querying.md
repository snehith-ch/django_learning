# Lecture 16 — The Django ORM, models in practice, and querying records

Source: `transcripts/Django16.txt`
Covers: the ORM concept, a real model class, migration commands in full, the DB Browser for SQLite tool, and the first end-to-end example of pulling records out of the database and displaying them in a template.

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official Django docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. What "ORM" means **[From video]**

**ORM** stands for **Object-Relational Mapper**. It's the layer that lets your application interact with a database (SQLite, MySQL, Oracle, …) by writing Python — classes, objects, method calls — instead of raw SQL. Django's ORM reads your model classes and automatically generates the matching database schema (tables and columns).

```python
# myapp/models.py
from django.db import models

class Employee(models.Model):
    enumber = models.IntegerField()
    ename = models.CharField(max_length=20)
    esalary = models.IntegerField()
    eaddress = models.CharField(max_length=20)
```

- The table name defaults to `<app_name>_<modelclassname>` (lowercased) — here, `myapp_employee`.
- Each field becomes a column, typed and validated according to the field class (`IntegerField`, `CharField(max_length=...)`, etc.).

> **[Gap-filled]** Since no field above is marked as the primary key, Django automatically adds one: an `id` column, integer, auto-incrementing, unique per row. You never declare this yourself unless you want a *different* field to be the primary key.

## 2. Migration commands, in full **[From video]**

| Command | What it does |
|---|---|
| `makemigrations` | Compares your models to the last known state and writes a new migration file describing the difference |
| `migrate` | Actually runs the pending migration(s) — creates/alters the real tables |
| `sqlmigrate <app> <migration_number>` | Prints the raw SQL a migration *would* run, without running it |
| `showmigrations` | Lists every migration and whether it's been applied |

`makemigrations` + `migrate` are the two you run constantly, every time a model changes. `sqlmigrate` and `showmigrations` are inspection tools you reach for occasionally.

## 3. Inspecting the database directly **[From video]**

Besides the admin panel, **DB Browser for SQLite** ([sqlitebrowser.org](https://sqlitebrowser.org/)) is a free desktop tool that opens your project's `db.sqlite3` file directly and lets you browse every table (including Django's own internal ones) in a spreadsheet-like view — handy for double-checking that a migration or a save actually did what you expected, without going through the admin login.

## 4. Querying & displaying records **[From video]**

```python
# views.py
from myapp.models import Employee

def display_emp(request):
    emp = Employee.objects.all()
    return render(request, 'employee.html', {'emp': emp})
```

```html
<!-- employee.html -->
{% if emp %}
<table border="1">
    <tr><th>Number</th><th>Name</th><th>Salary</th><th>Address</th></tr>
    {% for e in emp %}
    <tr><td>{{ e.enumber }}</td><td>{{ e.ename }}</td><td>{{ e.esalary }}</td><td>{{ e.eaddress }}</td></tr>
    {% endfor %}
</table>
{% else %}
<h1>No data</h1>
{% endif %}
```

`Employee.objects.all()` returns a **QuerySet** — Django's representation of "every row in this table," which behaves like a list of model instances. `{% for e in emp %}` iterates it exactly like a plain Python list; `e.enumber` etc. use DTL's dot-lookup rules (Lecture 14/15) to read each field off the model instance.

> **[Researched] — QuerySets are lazy.** Per the [Django docs](https://docs.djangoproject.com/en/stable/topics/db/queries/#querysets-are-lazy), `Employee.objects.all()` does not hit the database the moment it's called — it builds a query description and only actually runs SQL when the QuerySet is evaluated (iterated over, as in the template's `{% for %}`, or converted to a list). This is why you can keep refining a QuerySet (filtering, ordering) across several lines of Python without triggering multiple database round-trips.

---

## Wrap-up

- **From video:** the ORM concept, a real `Employee` model with typed fields, all four migration commands, the DB Browser for SQLite tool, and the full flow of querying with `.objects.all()` and rendering a QuerySet in a template.
- **Gap-filled:** the auto-created `id` primary key when no field is explicitly marked as one.
- **Researched:** QuerySet laziness — queries don't run until evaluated.

Double-check: QuerySet laziness is a subtle but important Django-specific behavior worth confirming for yourself in the docs, since it affects how you reason about performance once queries get more complex (filters, joins) later on.
