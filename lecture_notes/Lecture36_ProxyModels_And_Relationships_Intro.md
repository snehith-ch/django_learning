# Lecture 36 — Proxy model inheritance, and starting model relationships

Source: `transcripts/Django36.txt`
Covers: the third and final type of model inheritance — proxy models — closing out the model-inheritance arc from Lectures 35–36, then the start of a new topic: model (database) relationships, covering one-to-one and many-to-one in detail. This lecture ends mid-confusion about `on_delete=PROTECT` appearing not to work — resolved at the start of the next lecture.

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official Django docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. Proxy model inheritance **[From video]**

Recall the three types of model inheritance named back in Lecture 35: abstract base class, multi-table, and proxy. This lecture covers the third:

> The main use of a proxy model is to override the existing behavior of a model. It is a type of model inheritance without creating a new table in the database — it always queries the original model, just with overridden methods/behavior.

```python
# models.py
class University(models.Model):
    uname = models.CharField(max_length=20)
    ulocation = models.CharField(max_length=20)

class College(University):
    class Meta:
        proxy = True
```

- `College(University)` — inherits from `University`, same syntax as any model subclass.
- **`class Meta: proxy = True`** is what makes this a *proxy* model specifically: unlike multi-table inheritance (Lecture 35), where the child gets its own separate table, a proxy model **shares the exact same table** as its parent. No new table is ever created for `College` — `makemigrations` confirms this, generating no new `CREATE TABLE` for it at all.
- Because both classes point at the literal same underlying table, **adding or editing a record through either model reflects in both** — the video demonstrates this directly in the admin panel: adding a "university" record makes it immediately visible when browsing "colleges," and vice versa, since there's only ever one table behind both names.

```python
# admin.py
@admin.register(University)
class UniversityAdmin(admin.ModelAdmin):
    list_display = ('id', 'uname', 'ulocation')

@admin.register(College)
class CollegeAdmin(admin.ModelAdmin):
    list_display = ('id', 'uname', 'ulocation')
```

> **[Gap-filled] — why this is a genuinely different tool from the other two inheritance types.** Abstract base classes (Lecture 35) are about *not repeating field definitions* across genuinely separate tables. Multi-table inheritance (Lecture 35) is about *linking* genuinely separate tables. A proxy model isn't about the data at all — it's the *same* data, the *same* table — it exists purely to let you attach different Python-level behavior (different `Meta` options like default ordering, different custom methods, a different admin registration) to what is, at the database level, one and the same table. Per the [Django docs](https://docs.djangoproject.com/en/stable/topics/db/models/#proxy-models), a common real use is giving a model two different "views" with different default orderings or extra convenience methods, without maintaining two copies of the same data.

> **Industry best practice:** Reach for a proxy model specifically when you want to change *behavior* (methods, `Meta.ordering`, manager logic) for a subset of use cases without touching the schema at all — e.g. an `ActiveUser` proxy of `User` with a custom manager that only returns active accounts, sharing the exact same `auth_user` table. Don't reach for it when you actually need different *fields* — that's what multi-table (or abstract, if the base shouldn't be its own table) inheritance is for.

## 2. Introducing model relationships **[From video]**

With all three types of model inheritance now covered, the video moves to a related but distinct topic: **model relationships** — how one model's records connect to another's, mirroring standard database relationship theory.

> Model relationships are database relationships. There are three types: **one-to-one**, **many-to-one**, and **many-to-many**.

## 3. One-to-one relationships **[From video]**

> A one-to-one relationship is a type of relationship where both tables can have only one record on either side — like the relationship between a husband and wife: a husband has one wife, a wife has one husband.

Defined with **`OneToOneField`**:

```python
# model_relationship_app/models.py
from django.contrib.auth.models import User
from django.db import models

class Book(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE, primary_key=True)
    book_name = models.CharField(max_length=20)
    book_author = models.CharField(max_length=20)
    book_published_date = models.DateField()
```

- `models.OneToOneField(User, ...)` — links each `Book` to exactly one `User` (the built-in `auth_user` table from Lecture 26 onward).
- **`on_delete=`** is a **required** argument for any relationship field — it tells Django what should happen to *this* row if the row it points to (here, the linked `User`) is ever deleted. `models.CASCADE` here means: if that `User` is deleted, this `Book` row is automatically deleted too. (`on_delete` options are covered fully next lecture, once a live mix-up about them is resolved.)
- **`primary_key=True`** on the `OneToOneField` makes the *relationship itself* the table's primary key — reinforcing that this really is a strict one-to-one: a given `User` can be linked to at most one `Book` row, full stop. Trying to create a second `Book` for a `User` who already has one fails with a database uniqueness error ("book with this user already exists") — confirmed live in the video by attempting exactly that.

```python
# admin.py
class BookAdmin(admin.ModelAdmin):
    list_display = ('book_name', 'book_author', 'book_published_date', 'user')

admin.site.register(Book, BookAdmin)
```

## 4. Restricting the choices offered for a relationship field **[From video]**

```python
user = models.OneToOneField(
    User, on_delete=models.CASCADE, primary_key=True,
    limit_choices_to={'is_staff': True}
)
```

**`limit_choices_to`** filters which related objects appear as selectable options in the admin's dropdown for this field — here, restricting the "which user is this book for" dropdown to only users with `is_staff=True` (Lecture 26). The video confirms this directly: non-staff users, who previously appeared as selectable, disappear from the dropdown once this is added.

> **[Researched] — the actual purpose of `limit_choices_to`.** Per the [Django docs](https://docs.djangoproject.com/en/stable/ref/models/fields/#django.db.models.ForeignKey.limit_choices_to), this argument only constrains what the **admin form** (or a `ModelForm`'s default queryset for that field) offers as choices — it is *not* database-level validation. A `Book` could still, in principle, be created programmatically (outside the admin) pointing at a non-staff user; `limit_choices_to` only narrows the admin's dropdown, it doesn't enforce the rule at the database or model layer. Treat it as a UI convenience, not a data-integrity guarantee.

## 5. Many-to-one relationships **[From video]**

> A many-to-one relationship is where the first table can have one or many records related to the second table, but the second table is related to only one record in the first table — like the relationship between you and your parents: your parents can have any number of children, but you have only one (set of) parent(s).

Defined with **`ForeignKey`**:

```python
class Post(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    post_title = models.CharField(max_length=20)
    post_category = models.CharField(max_length=20)
    post_published_date = models.DateField()
```

The mechanical difference from `OneToOneField` is small — `ForeignKey` instead of `OneToOneField`, and (compare Section 3) no `primary_key=True` — but the meaning is very different: because there's no uniqueness constraint on `user` here, **one `User` can have many `Post` rows**, which the video confirms directly by creating multiple posts for the same user without error (unlike the `Book` example above, which rejected a second row for the same user).

> **[Gap-filled] — why `ForeignKey` is "the" standard relationship in practice.** `OneToOneField` and `ManyToManyField` (next lecture) are genuinely useful but comparatively rare in real schemas. `ForeignKey` — one row on the "many" side pointing at one row on the "one" side — is by far the most common relationship in real-world database design (an order belongs to one customer, a comment belongs to one post, an employee belongs to one department), which is why it's worth having solid before the less common variants.

## 6. The `on_delete=PROTECT` confusion **[From video]**

The video attempts to demonstrate `models.PROTECT` as an alternative to `models.CASCADE` — expecting it to *prevent* deleting a `Book`/`Post` record whose associated `User` still exists. Setting `on_delete=models.PROTECT` and then deleting `Book`/`Post` rows through the admin, the deletion **succeeds anyway** — the opposite of what was expected, with no error shown. The video ends this lecture unresolved, planning to investigate before the next session.

> **[Gap-filled] — what's actually going on here (resolved explicitly at the start of Lecture 37).** This isn't a Django bug — it's a live misunderstanding of what `on_delete` actually governs, cleared up plainly at the top of the next lecture: **`on_delete` controls what happens to the row holding the foreign key (here, `Book`/`Post`) when the row it *points to* (the `User`) is deleted — not the other way around.** Deleting a `Book` or `Post` row itself is never affected by `on_delete` at all; that deletion always just works, regardless of the setting. `on_delete=PROTECT` specifically means: attempting to delete a `User` who still has a `Book`/`Post` pointing at them will be *blocked*, raising a `ProtectedError`, rather than allowed to cascade. The video's test — deleting the `Book`/`Post` rows themselves — was simply testing the wrong side of the relationship. See the next lecture's notes for this confirmed working correctly once the right side is tested.

---

## Wrap-up

- **From video:** proxy model inheritance (`Meta.proxy = True`, no new table, shared data across both model names, useful for attaching different behavior to the same table); the start of model relationships — one-to-one (`OneToOneField`, `on_delete`, `primary_key=True` enforcing strict uniqueness, `limit_choices_to`) and many-to-one (`ForeignKey`, allowing multiple children per parent); an unresolved live confusion about `on_delete=PROTECT` appearing not to work.
- **Gap-filled:** what distinguishes a proxy model's purpose from the other two inheritance types; why `ForeignKey` is the most common relationship type in practice; a full, explicit resolution of the `PROTECT` confusion — deleting the child row is never blocked, only deleting the *parent* it points to is.
- **Researched:** the real, narrower scope of `limit_choices_to` — a UI/admin-form convenience, not a database-level constraint.

Double-check: if the `PROTECT` explanation above still feels unclear, hold the question until the next lecture's notes — the video itself re-demonstrates it correctly there, deleting the *User* (not the *Book*/*Post*) while one is still linked, and shows the block actually happening as intended.
