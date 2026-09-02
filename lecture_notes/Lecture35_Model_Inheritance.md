# Lecture 35 — Model inheritance: abstract base classes and multi-table inheritance

Source: `transcripts/Django35 (1).txt`
Covers: applying Python's class-inheritance idea to Django models — sharing common fields across several models without redeclaring them — in its two demonstrated forms: abstract base class inheritance and multi-table inheritance. A third form, proxy model inheritance, is named but deferred to a following session not covered by this transcript.

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official Django docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. What model inheritance is, and why it exists **[From video]**

> If several model classes have common fields, it is not recommended to write those fields separately in every model class — it increases the length of the code and reduces usability.

Model inheritance works "almost the same as class inheritance" in plain Python — a base class defines fields once; other model classes inherit from it and automatically get those same fields, without redeclaring them. Django provides **three** types of model inheritance:

| Type | What it means |
|---|---|
| **Abstract base class** | The base class exists purely to hold shared fields — it never becomes a database table itself. |
| **Multi-table inheritance** | Both the base class and the child class become real, separate tables, linked together automatically. |
| **Proxy model inheritance** | *(Named by the video as the third type; not demonstrated in this transcript — see the gap-fill note in the wrap-up.)* |

> **[Gap-filled] — connecting this to Python class inheritance directly.** In ordinary Python, `class Dog(Animal):` means every `Dog` automatically has whatever `Animal` defines, without re-typing it. Django model inheritance is that exact same language feature — a model class is still just a Python class — applied specifically to models, with the added wrinkle that Django also has to decide what happens at the *database* level (does the base class get its own table or not?), which is precisely what distinguishes the three types above.

## 2. Abstract base class inheritance **[From video]**

```bash
python manage.py startapp model_inheritance_app
```

```python
# model_inheritance_app/models.py
from django.db import models

class CommonInfo(models.Model):
    name = models.CharField(max_length=20)
    age = models.IntegerField()
    date = models.DateField()

    class Meta:
        abstract = True
```

- `CommonInfo` holds the fields that will be shared: `name`, `age`, `date`.
- **`class Meta: abstract = True`** is what makes this an *abstract* base class — per the video, this means "just a blueprint or template," not a real, standalone model. When migrations run, **no `CommonInfo` table is ever created** — only the tables for models that inherit from it.

```python
class Trainer(CommonInfo):
    address = models.CharField(max_length=20)
    date = None   # this trainer doesn't need the inherited "date" field
```

```python
class Teacher(CommonInfo):
    date = models.DateTimeField()   # overrides the inherited DateField with a DateTimeField
```

- `Trainer(CommonInfo)` and `Teacher(CommonInfo)` both inherit `name` and `age` automatically — those two field declarations are written **once**, in `CommonInfo`, and never repeated.
- `Trainer` adds its own extra field (`address`) not shared with `Teacher`.
- **`date = None`** on `Trainer` explicitly *removes* an inherited field that model doesn't need — the same `field = None` exclusion technique already seen with `EditUserProfileForm.password = None` in Lecture 29, just applied to a model instead of a form.
- **`Teacher` overrides `date`** with a different field type (`DateTimeField` instead of the inherited `DateField`) — redeclaring a field name in the child class replaces the inherited version for that model only, without affecting `Trainer` or `CommonInfo`.

```python
# model_inheritance_app/admin.py
from django.contrib import admin
from .models import Trainer, Teacher

@admin.register(Trainer)
class TrainerAdmin(admin.ModelAdmin):
    list_display = ('id', 'name', 'age', 'address')   # no "date" — Trainer excluded it

@admin.register(Teacher)
class TeacherAdmin(admin.ModelAdmin):
    list_display = ('id', 'name', 'age', 'date')       # "date" here is a DateTimeField
```

Running `makemigrations`/`migrate` creates exactly **two** tables — `Trainer` and `Teacher` — confirmed directly in the video by counting migration output and cross-checking the admin panel: `CommonInfo` produces no table of its own at all, exactly as `abstract = True` promises.

> **[Gap-filled] — the practical payoff, stated plainly.** Without this, adding a `name`/`age` pair to ten different models would mean writing those two field declarations ten separate times, and changing `age`'s max digits (say) later would mean editing ten files instead of one. Abstract base classes turn "these models happen to share some fields" into a single, edited-once source of truth — the same DRY (Don't Repeat Yourself) motivation behind `ModelForm` removing field duplication back in Lecture 27, applied one layer down at the model level itself.

> **Industry best practice:** Reach for an abstract base class specifically when the *shared fields* don't need their own independent existence in the database — a common example beyond this lecture's is a `TimestampedModel` abstract base with `created_at`/`updated_at` fields, inherited by every "real" model in a project that needs creation/update tracking, without a meaningless standalone `TimestampedModel` table ever existing.

## 3. Multi-table inheritance **[From video]**

> In multi-table inheritance, each model has its own database table. Django creates a one-to-one field for the child model's relationship to the parent, automatically.

```python
class Bank(models.Model):
    bank_name = models.CharField(max_length=20)
    bank_address = models.CharField(max_length=20)

class BankManager(Bank):
    name = models.CharField(max_length=20)
    age = models.IntegerField()
```

The critical difference from Section 2: **`Bank` is *not* marked `abstract`** — no `class Meta: abstract = True` here at all. That single omission changes everything about how this inheritance behaves:

- `Bank` gets its **own real table** (`bank_name`, `bank_address`) — unlike `CommonInfo`, which produced no table.
- `BankManager` also gets its **own real table** (`name`, `age`) — but Django additionally, automatically, adds a hidden **one-to-one relationship** linking each `BankManager` row back to its corresponding `Bank` row.
- Because of that link, a `BankManager` instance can access `bank_name`/`bank_address` too (inherited through the relationship), even though those columns physically live in the separate `Bank` table.

Confirmed in the video's admin-panel walkthrough: registering `Bank` and `BankManager` and running migrations produces **two separate tables**, and adding a `BankManager` record (which shows `bank_name`/`bank_address` fields on its admin form, alongside its own `name`/`age`) automatically creates/links a matching row in the `Bank` table as well — visible by checking the `Bank` admin list afterward and seeing the same bank name/address appear there too.

> **[Gap-filled] — the mental model for "each model has its own table."** In the abstract case (Section 2), think of the base class as a stencil: it shapes the child tables but never becomes a table itself. In multi-table inheritance, think of it as two real, separate filing cabinets (`Bank` and `BankManager`) that happen to be wired together — every `BankManager` folder has an automatic cross-reference pointing at exactly one `Bank` folder, and you can look up either table's fields via that link. Abstract inheritance shares *field definitions*; multi-table inheritance shares (and links) *actual rows*.

> **[Researched] — the automatic link is a real `OneToOneField`, usable like any other.** Per the [Django docs on multi-table inheritance](https://docs.djangoproject.com/en/stable/topics/db/models/#multi-table-inheritance), the parent link Django creates behind the scenes is an ordinary `OneToOneField` (with a default related name based on the lowercased parent model name) — meaning it can be queried, traversed, and reasoned about the same way any explicitly-declared one-to-one relationship can (e.g. `some_bank.bankmanager` to go from a `Bank` instance to its linked `BankManager`, if one exists). This is worth knowing because it demystifies the "automatic" part — it isn't a special hidden mechanism, just Django adding a normal field you didn't have to type yourself.

## 4. Choosing between the two demonstrated types **[From video, synthesized]**

| | Abstract base class | Multi-table inheritance |
|---|---|---|
| Does the base class get its own table? | No | Yes |
| What's actually shared? | Field *definitions* (written once, copied into each child's table) | Real, linked *rows* across separate tables |
| Marked with | `class Meta: abstract = True` | Nothing extra — just normal model inheritance, no `abstract` |

> **[Gap-filled] — when to reach for which.** Use an **abstract base class** when the shared fields describe a *trait* several unrelated models happen to have (timestamps, an address, contact info) and the base concept itself is never a "real thing" you'd query on its own. Use **multi-table inheritance** when the base class *is* itself a meaningful, independently-useful table (here, `Bank` genuinely makes sense as its own concept — a bank exists whether or not it has a manager on file — while `CommonInfo` from Section 2 was never meant to represent anything queryable on its own).

---

## Wrap-up

- **From video:** the motivation for model inheritance (avoid redeclaring shared fields across models); abstract base class inheritance (`Meta.abstract = True`, no table for the base class, `field = None` to exclude an inherited field, redeclaring a field name to override its type); multi-table inheritance (no `abstract`, both base and child get real tables, an automatic one-to-one link connecting them, confirmed via the admin panel).
- **Gap-filled:** tying model inheritance directly back to ordinary Python class inheritance; the DRY-motivation parallel to `ModelForm`; a "stencil vs. two linked filing cabinets" mental model distinguishing the two types; explicit guidance on when to reach for each.
- **Researched:** confirmation that multi-table inheritance's automatic parent link is a normal, queryable `OneToOneField`, not a special hidden mechanism.

Double-check: **proxy model inheritance** — the third type the video names in its opening list — is not demonstrated anywhere in this transcript; the session explicitly ends with "proxy model inheritance we'll discuss tomorrow," and that follow-up session isn't among the transcripts covered here. Treat proxy model inheritance as an open topic, not something these notes (or the source video, at this point) have actually covered — per the [Django docs](https://docs.djangoproject.com/en/stable/topics/db/models/#proxy-models), for reference, it's the third form: a subclass that changes a model's Python-level behavior (custom methods, different default ordering, etc.) without creating any new table at all, sharing the parent's exact table one-for-one.
