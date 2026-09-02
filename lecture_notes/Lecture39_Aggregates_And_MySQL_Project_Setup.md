# Lecture 39 — QuerySet aggregate functions, and starting a MySQL CRUD project

Source: `transcripts/Django39.txt`
Covers: closing out the QuerySet API arc with aggregate functions (average, sum, min, max, count), then a hard pivot into project-building mode — setting up a new Django project from the command line, connecting it to a real MySQL database instead of the default SQLite, and building the first model/form/template for a CRUD (Create, Read, Update, Delete) project that continues into the next lecture.

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official Django docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. QuerySet aggregate functions **[From video]**

> Aggregate functions: average, max, min, sum, count.

```python
from django.db.models import Avg, Sum, Min, Max, Count

def v1(request):
    student_data = Student.objects.all()
    result = student_data.aggregate(
        average=Avg('marks'),
        total=Sum('marks'),
        minimum=Min('marks'),
        maximum=Max('marks'),
        total_count=Count('marks'),
    )
    return render(request, 'emp2.html', {'student_data': student_data, **result})
```

- `Avg`, `Sum`, `Min`, `Max`, `Count` are imported from **`django.db.models`** — they're not queryset methods themselves, but classes you pass *into* `.aggregate()`.
- **`.aggregate(...)`** is called on a queryset and returns a single **dictionary** (not another queryset) — one aggregate calculation per keyword argument supplied.
- The keyword name on the left (`average=`, `total=`, etc.) becomes the dictionary key in the result — e.g. `result['average']` holds the computed average.

```html
<!-- emp2.html -->
<h1>Average marks: {{ average }}</h1>
<h1>Total marks: {{ total }}</h1>
<h1>Minimum marks: {{ minimum }}</h1>
<h1>Maximum marks: {{ maximum }}</h1>
<h1>Total students: {{ total_count }}</h1>
```

> **[Researched] — the default key name when no alias is given.** The video's own experiment (leaving out `average=`/`total=` and calling `.aggregate(Avg('marks'), Sum('marks'), ...)` directly) confirms this live: without an explicit keyword, Django falls back to naming each result key `<field>__<function>` in lowercase — e.g. `marks__avg`, `marks__sum`, `marks__min`, `marks__max`, `marks__count`. Per the [Django docs](https://docs.djangoproject.com/en/stable/topics/db/aggregation/#generating-aggregates-for-each-item-in-a-queryset), giving an explicit keyword (as the video does first) is the clearer, more common style in real code — it avoids the somewhat awkward auto-generated `field__function` key names.

> **[Example]** A standalone illustration of aggregating over a *filtered* subset, rather than an entire table — a common real pattern the video doesn't demonstrate directly:
> ```python
> from django.db.models import Avg
>
> hyderabad_avg = Student.objects.filter(address='Hyderabad').aggregate(avg_marks=Avg('marks'))
> # {'avg_marks': 82.5} — the average, computed only over Hyderabad students
> ```
> Because `.filter()` (Lecture 37) and `.aggregate()` both work on querysets, they chain naturally — filter down to the subset you care about, then aggregate over just that subset.

This closes out the QuerySet API arc spanning Lectures 37–39: `.all()`/`.filter()`/`.exclude()`/`.order_by()`/`.reverse()`, `.values()`/`.values_list()`, `.union()`/`.intersection()`/`.difference()`, the double-underscore field lookups, and now aggregate functions.

---

## 2. Starting a new project: CRUD with MySQL **[From video]**

The video shifts from concept lectures to project-building, starting with a full **CRUD (Create, Read, Update, Delete)** project — deliberately using **MySQL** instead of the SQLite default every earlier lecture has relied on, to show what actually changes when a project connects to a real production-style database.

```bash
cd Desktop
django-admin startproject mysql_crud_project
cd mysql_crud_project
python manage.py startapp mysql_crud_app
```

> **[Gap-filled] — why the command line, not the PyCharm project wizard.** Every earlier example in this course used PyCharm's "New Project" wizard, which auto-generates a `templates/` folder and pre-configures a few conveniences. Building this project via `django-admin`/`manage.py` directly instead means those conveniences (notably a `templates/` folder) are **not** created automatically — the video runs into exactly this gap partway through (Section 5 below) and has to create the folder by hand. This is a genuinely useful thing to see once: the wizard isn't doing anything magic, it's just automating steps you can always do yourself from the command line.

## 3. Switching the database from SQLite to MySQL **[From video]**

Every project since Lecture 3 has used Django's default, zero-configuration SQLite backend. Connecting to MySQL instead requires: MySQL Server itself installed and running, a database created inside it, and `settings.py` pointed at that database's connection details.

```sql
-- created in MySQL Workbench
CREATE DATABASE django_project_4pm;
```

```python
# settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'django_project_4pm',
        'USER': 'root',
        'PASSWORD': 'your_mysql_password',
        'HOST': 'localhost',   # or '127.0.0.1'
        'PORT': '3306',
    }
}
```

- **`ENGINE`** is the setting that actually determines which database backend Django talks to — swapping `django.db.backends.sqlite3` (the default) for `django.db.backends.mysql` is the single change that redirects every future `makemigrations`/`migrate` at this new database instead.
- **`NAME`** here is the database name created in MySQL Workbench beforehand — Django does not create the database itself, only the tables inside it once it's pointed at an existing one.
- **`USER`**/**`PASSWORD`** are real MySQL server credentials, not a Django-specific concept — the same username/password used to log into MySQL Workbench.
- **`PORT`** — MySQL's default port is `3306` (contrast with, e.g., PostgreSQL's default of `5432`).

> **[Researched] — the missing piece the video doesn't mention: the MySQL Python driver.** Per the [Django docs](https://docs.djangoproject.com/en/stable/ref/databases/#mysql-notes), connecting Django to MySQL also requires a Python MySQL driver package to be installed in the project's environment — commonly `mysqlclient` (`pip install mysqlclient`), which is what actually lets Python talk to a MySQL server at all; Django's `ENGINE` setting alone doesn't include this. The video's own MySQL connection presumably already had this installed from an earlier, off-camera step, since it isn't shown — worth knowing as a prerequisite if reproducing this setup from scratch, since `mysqlclient` (or an alternative like `PyMySQL`) not being installed is one of the most common first errors when connecting Django to MySQL for the first time.

## 4. Defining the model, with an explicit table name **[From video]**

```python
# mysql_crud_app/models.py
from django.db import models

class User(models.Model):
    uname = models.CharField(max_length=20)
    uemail = models.EmailField(max_length=30)
    upassword = models.CharField(max_length=20)

    class Meta:
        db_table = 'users'
```

**`class Meta: db_table = 'users'`** overrides Django's usual `<app_name>_<modelname>` table-naming convention (Lecture 23) with an explicit name — here, the table is created as simply `users` in MySQL, rather than the default `mysql_crud_app_user`. Confirmed by inspecting MySQL Workbench directly after `makemigrations`/`migrate`: the table appears named exactly `users`, with columns `id`, `uname`, `uemail`, `upassword`.

```bash
python manage.py makemigrations
python manage.py migrate
```

Running these against the MySQL-configured project creates the real table inside the `django_project_4pm` MySQL database — the exact same two commands used throughout this course, now targeting a different backend entirely because of the `ENGINE` change in Section 3. No other code differs.

## 5. A `ModelForm`, and the missing templates folder **[From video]**

```python
# mysql_crud_app/forms.py
from django import forms
from mysql_crud_app.models import User

class UserForm(forms.ModelForm):
    class Meta:
        model = User
        fields = '__all__'
```

Same `ModelForm` pattern from Lecture 27. Rendering it, though, immediately surfaces the gap flagged in Section 2:

```python
# mysql_crud_app/views.py
from django.shortcuts import render
from mysql_crud_app.forms import UserForm

def insert_view(request):
    form = UserForm()
    return render(request, 'index.html', {'form': form})
```

Visiting this view raises `TemplateDoesNotExist` — because, unlike every PyCharm-wizard-created project so far, **no `templates/` folder exists yet**. The fix:

1. Create a `templates/` folder manually at the project root (next to `manage.py`).
2. Add it to `settings.py`'s `TEMPLATES` setting:
   ```python
   TEMPLATES = [
       {
           # ...
           'DIRS': ['templates'],
           # ...
       },
   ]
   ```
3. Create `templates/index.html` and render `{{ form.as_table }}` inside it.

```html
<!-- templates/index.html -->
<table>{{ form.as_table }}</table>
```

With this in place, the form renders — showing `uname`/`uemail`/`upassword` fields, but (as the video notes, setting up the next lecture) with no visible `<form>` tag, submit button, or styling yet — a bare, unstyled `ModelForm` rendering, exactly the starting point Lecture 40 picks up and finishes with Bootstrap styling and full insert/show/edit/update/delete views.

---

## Wrap-up

- **From video:** the QuerySet aggregate functions (`Avg`, `Sum`, `Min`, `Max`, `Count` from `django.db.models`, used inside `.aggregate()`, returning a plain dictionary); starting a MySQL-backed CRUD project via the command line instead of the PyCharm wizard; switching `settings.py`'s `DATABASES` from SQLite to MySQL (`ENGINE`, `NAME`, `USER`, `PASSWORD`, `HOST`, `PORT`); a `User` model with an explicit `Meta.db_table` name; confirming the real MySQL table via MySQL Workbench; a `ModelForm` for it; discovering and fixing the missing `templates/` folder that PyCharm's wizard normally creates automatically.
- **Gap-filled:** why building via the command line specifically surfaces the missing-templates-folder problem, and what that reveals about what the PyCharm wizard was quietly doing for every earlier project.
- **Researched:** the default `field__function` key naming when `.aggregate()` is called without explicit keyword aliases; the MySQL Python driver (`mysqlclient` or similar) required alongside the `ENGINE` setting, which the video's own walkthrough doesn't show installing.

Double-check: if reproducing this MySQL setup and it fails to connect at all (rather than just misbehaving), the missing Python MySQL driver from Section 3's researched note is the first thing worth checking — it's a separate `pip install` step the video's walkthrough skips over.
