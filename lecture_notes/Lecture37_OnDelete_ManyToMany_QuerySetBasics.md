# Lecture 37 — Resolving on_delete, many-to-many relationships, and the QuerySet API

Source: `transcripts/Django37.txt`
Covers: clearing up last lecture's `on_delete=PROTECT` confusion with a corrected demonstration, the third and final relationship type — many-to-many — and the start of a new major topic: the Django QuerySet API, Django's toolkit for filtering and shaping data without writing raw SQL.

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official Django docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. Resolving the `on_delete` confusion **[From video]**

Picking up exactly where Lecture 36 left off, the video re-tests `on_delete` correctly this time:

```python
user = models.ForeignKey(User, on_delete=models.PROTECT)
```

Instead of deleting a `Book`/`Post` row (last lecture's mistaken test), the video now attempts to delete the **`User`** that a `Book` row points to, through the admin panel. This time it behaves exactly as expected:

> Deleting the selected user would require deleting the following protected related objects... [the operation is blocked].

> **`on_delete=PROTECT` means: the user (or other referenced row) which is associated with a book/post *cannot be deleted* while that association exists.** Switching back to `on_delete=CASCADE` and repeating the same deletion succeeds immediately — deleting the `User` this time also deletes the linked `Book`/`Post` automatically, with no error.

| `on_delete` value | What happens when the referenced row (e.g. `User`) is deleted |
|---|---|
| `models.CASCADE` | The row holding the foreign key (e.g. `Book`) is deleted too, automatically |
| `models.PROTECT` | The deletion is **blocked** — Django raises an error instead, as long as any row still references it |

> **[Gap-filled] — the one-sentence fix for last lecture's confusion.** `on_delete` is entirely about **what happens on the "one" side when it's deleted**, never about restricting deletion of the "many" side. A `Book`/`Post` row can always be deleted on its own, regardless of `on_delete` — that setting only ever activates when someone tries to delete the `User` (or other parent) it points to. Testing it by deleting the child row, as Lecture 36 did, can never demonstrate `PROTECT` doing anything, because that's not the operation it protects against.

> **[Researched] — the other `on_delete` options Django provides.** Per the [Django docs](https://docs.djangoproject.com/en/stable/ref/models/fields/#django.db.models.ForeignKey.on_delete), beyond `CASCADE` and `PROTECT` (both covered in the video), Django also offers: `SET_NULL` (sets the foreign key to `NULL` instead of deleting — requires the field to allow `null=True`), `SET_DEFAULT` (sets it to a configured default value), `SET()` (a callable/value you supply), `DO_NOTHING` (takes no action at the Django level — dangerous, since it can leave the database in an inconsistent state unless the database itself enforces something), and `RESTRICT` (similar to `PROTECT`, but allows the deletion to proceed if it's also cascading from elsewhere in the same operation — a more recent, more nuanced addition). `on_delete` has no default value — it must always be specified explicitly on every `ForeignKey`/`OneToOneField`.

## 2. Many-to-many relationships **[From video]**

> In a many-to-many relationship, each record of the first table can be related to many records of the second table, and each record of the second table can be related to many records of the first table.

Defined with **`ManyToManyField`**:

```python
# model_relationship_app/models.py
class Dance(models.Model):
    user = models.ManyToManyField(User)
    dance_name = models.CharField(max_length=20)
    dance_duration = models.IntegerField()

    def dance_by(self):
        return ", ".join(str(u) for u in self.user.all())
```

- `models.ManyToManyField(User)` — no `on_delete` argument here at all (unlike `OneToOneField`/`ForeignKey`), because a many-to-many relationship isn't implemented as a column on either table — Django creates a separate, hidden "join table" behind the scenes purely to record which users are linked to which dances.
- **`dance_by(self)`** is a custom model method (not a field) — it's added specifically to give a human-readable summary of *all* the users linked to this one `Dance`, since the admin's default list display can't usefully show a many-to-many field's full contents as a single column otherwise.
- `self.user.all()` — because `user` is many-to-many, accessing it returns a full queryset of every linked `User`, not a single object (contrast with `OneToOneField`/`ForeignKey`, where accessing the field returns one related object directly).
- `", ".join(str(u) for u in ...)` — builds one comma-separated string out of every linked user's text representation (`__str__`, from Lecture 24). This is a standard Python idiom, not Django-specific: `str.join()` concatenates an iterable of strings using the string it's called on as the separator.

```python
# admin.py
class DanceAdmin(admin.ModelAdmin):
    list_display = ('dance_name', 'dance_duration', 'dance_by')

admin.site.register(Dance, DanceAdmin)
```

Adding a "dance" record through the admin now shows a **multi-select** widget for `user` (letting several users be checked at once), and the admin's list view shows every linked user's name, comma-separated, in the `dance_by` column — confirmed live: the same user can be linked to multiple dances, and the same dance can list multiple users, satisfying the many-to-many definition in both directions.

> **[Gap-filled] — the relationship summary, side by side.** | Relationship | Field | One side can have... | Real-world example |
> |---|---|---|---|
> | One-to-one | `OneToOneField` | exactly one match | A person and their passport |
> | Many-to-one | `ForeignKey` | many rows pointing at one | Many orders, one customer |
> | Many-to-many | `ManyToManyField` | many-to-many both ways | Students and the courses they're enrolled in |

---

## 3. Introducing the QuerySet API **[From video]**

> The QuerySet API lets you interact with Django's database interaction layer — filtering, slicing, and constructing queries — without writing raw SQL or hitting the database directly for every operation.

> A QuerySet can be constructed, filtered, and sliced, and generally passed around without actually hitting the database, until it's evaluated.

## 4. Core QuerySet properties and methods **[From video]**

```bash
python manage.py startapp queryset_api_app
```

```python
# queryset_api_app/models.py
class Employee(models.Model):
    eid = models.IntegerField(unique=True, null=False)
    name = models.CharField(max_length=20)
    address = models.CharField(max_length=20)
    age = models.IntegerField()
    dob = models.DateField()
```

| Method/property | What it does | Syntax |
|---|---|---|
| `.all()` | Returns every record, as a queryset | `Employee.objects.all()` |
| `.filter(**kwargs)` | Returns a new queryset containing only records matching the given lookup(s) | `Employee.objects.filter(address='Hyderabad')` |
| `.exclude(**kwargs)` | The inverse of `.filter()` — everything that does **not** match | `Employee.objects.exclude(address='Hyderabad')` |
| `.order_by(*fields)` | Sorts the queryset | `Employee.objects.order_by('name')` |
| `.query` | A property (not a method) returning the actual SQL Django would run for this queryset | `Employee.objects.all().query` |

```python
# queryset_api_app/views.py
def v1(request):
    emp_data = Employee.objects.all()
    print(type(emp_data))       # QuerySet
    print(emp_data.query)       # the raw SQL
    return render(request, 'emp2.html', {'emp_data': emp_data})
```

Printing `emp_data.query` to the terminal shows the literal `SELECT ... FROM queryset_api_app_employee` statement Django generated — a direct, concrete way to see exactly what SQL a given queryset translates to.

### Ordering, in detail

```python
Employee.objects.order_by('name')     # ascending (A→Z)
Employee.objects.order_by('-name')    # descending (Z→A) — leading "-"
Employee.objects.order_by('?')        # random order, every time
```

`.reverse()` flips the order of an *already-ordered* queryset — and, per the video's own emphasis, **only works when the queryset already has an `.order_by()` applied**. Calling `.reverse()` on an unordered queryset has no defined effect to flip.

```python
qs = Employee.objects.order_by('eid')[1:4].reverse()
```

> **[Gap-filled] — Python slicing applied to a QuerySet.** `[1:4]` here is ordinary Python list-slicing syntax (`start:stop`, stop exclusive) — applied to a Django QuerySet, it translates into a `LIMIT`/`OFFSET` at the SQL level rather than fetching everything and slicing in Python. `[1:4]` takes the 2nd, 3rd, and 4th records (index 1 up to, but not including, index 4); `.reverse()` then flips just that already-sliced, already-ordered set of three.

> **[Researched] — QuerySets are lazy, and why that matters here.** Per the [Django docs](https://docs.djangoproject.com/en/stable/topics/db/queries/#querysets-are-lazy), none of `.all()`, `.filter()`, `.exclude()`, `.order_by()`, or slicing actually run a database query the moment they're called — they build up a description of the query, which is only executed once the queryset is *evaluated* (iterated over, converted to a list, printed, etc.). This is why the video can chain `.order_by(...)[1:4].reverse()` in one expression without three separate round-trips to the database — it's all resolved into a single SQL query only when the template actually loops over it.

---

## Wrap-up

- **From video:** the corrected `on_delete` demonstration (`PROTECT` blocks deleting the *referenced* row, not the row holding the foreign key); many-to-many relationships (`ManyToManyField`, the hidden join table, no `on_delete`, a custom `dance_by()` method using `str.join()` to summarize linked users); the QuerySet API's core methods — `.all()`, `.filter()`, `.exclude()`, `.order_by()` (ascending/descending/random), `.reverse()` (requires prior ordering), and the `.query` property for inspecting generated SQL.
- **Gap-filled:** a one-sentence fix for the `on_delete` misunderstanding; the other `on_delete` options beyond what the video shows; a side-by-side table of all three relationship types; Python slicing applied to a QuerySet.
- **Researched:** the full set of `on_delete` options (`SET_NULL`, `SET_DEFAULT`, `SET()`, `DO_NOTHING`, `RESTRICT`); QuerySet laziness and why it matters for chained operations.

Double-check: `on_delete` has **no default** — Django requires it explicitly on every `ForeignKey`/`OneToOneField`, which is worth remembering the next time a model definition throws a `TypeError` for a missing argument on one of these fields.
