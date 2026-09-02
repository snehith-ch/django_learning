# Lecture 38 — QuerySet values/values_list, set operations, and field lookups

Source: `transcripts/Django38.txt`
Covers: two alternative return formats for a queryset (`.values()` / `.values_list()`), combining two querysets with set-style operators (`.union()`, `.intersection()`, `.difference()`), and the start of Django's field lookup system — the double-underscore syntax behind precise filtering (`__exact`, `__contains`, `__gt`, date-part lookups, and more).

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official Django docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. `.values()` and `.values_list()` **[From video]**

By default, a queryset returns a list of full **model instances** (each with `.name`, `.address`, etc. as attributes — the format every earlier lecture has used). Two methods return the same underlying rows in different shapes instead:

```python
emp_data = Employee.objects.values()
# each row becomes a dict: {'id': 1, 'name': 'Mohan', 'address': 'Hyderabad', ...}

emp_data = Employee.objects.values_list()
# each row becomes a tuple: (1, 'Mohan', 'Hyderabad', ...)
```

> `.values()` returns a queryset that yields **dictionaries** rather than model instances. `.values_list()` is similar, except that instead of dictionaries it returns **tuples**.

> **[Gap-filled] — when either is actually useful.** Full model instances are the right default for most view code, since they give you `.method()` calls (like Lecture 37's `dance_by()`) and IDE autocomplete on field names. `.values()`/`.values_list()` earn their keep when you specifically want a lighter-weight, serializable shape — e.g. passing data to `json.dumps()`, building a CSV export, or feeding a charting library that expects plain dicts/tuples rather than Django objects. They're also slightly cheaper to construct, since Django skips building full model instances for each row.

> **[Researched] — `values_list(flat=True)`, a very common variant not shown in the video.** Per the [Django docs](https://docs.djangoproject.com/en/stable/ref/models/querysets/#values-list), when a `values_list()` call is limited to a single field, passing `flat=True` returns a flat list of just that field's values instead of a list of one-tuples — e.g. `Employee.objects.values_list('name', flat=True)` gives `['Mohan', 'Rajesh', ...]` directly, rather than `[('Mohan',), ('Rajesh',), ...]`. This is one of the most common real-world uses of `values_list()` — pulling out a plain list of IDs or names for use elsewhere in a view.

## 2. Combining querysets: union, intersection, difference **[From video]**

To demonstrate these, the video introduces a second model, `Manager`, alongside the existing `Employee`, both sharing some overlapping names to make the results visible:

```python
class Manager(models.Model):
    mname = models.CharField(max_length=20)
    address = models.CharField(max_length=20)
    salary = models.IntegerField()
    dob = models.DateField()
```

```python
qs1 = Employee.objects.values_list('id', 'name')
qs2 = Manager.objects.values_list('id', 'mname')

emp_data = qs2.union(qs1)          # combine, de-duplicated
emp_data = qs2.intersection(qs1)   # only rows common to both
emp_data = qs1.difference(qs2)     # only rows in qs1, not also in qs2
```

| Method | What it returns |
|---|---|
| `.union(other_qs)` | Combines two (or more) querysets — **`.union()` keeps only distinct/unique rows by default**, dropping duplicates that appear in both. |
| `.union(other_qs, all=True)` | Same as above, but **keeps duplicate rows** instead of collapsing them. |
| `.intersection(other_qs)` | Only the rows present in **both** querysets. |
| `.difference(other_qs)` | Only the rows present in the first queryset **but not** in the second. |

The video's concrete demonstration: two `Employee` records (`Sai`, `Rajesh`) share a name with two `Manager` records. `.union()` (default) shows each shared name **once**; `.union(all=True)` shows it **twice** (once from each table); `.intersection()` shows **only** the two shared names; `.difference()` (queryset one, relative to queryset two) shows only the `Employee` names that *don't* also appear among the managers.

> **[Gap-filled] — why `.values_list()` is required first.** All three operators (`union`/`intersection`/`difference`) need both querysets to have a matching, compatible shape to compare row-by-row — which is exactly what `.values_list('id', 'name')` provides (a plain tuple of the same two columns from each model), even though the two models (`Employee`, `Manager`) don't otherwise share a schema. Trying to combine two querysets of full model instances from *different* models wouldn't have a sensible row-by-row comparison to make.

> **Industry best practice:** These set-style operators are worth reaching for specifically when you have two genuinely different tables that need to be reported on together (e.g. "everyone in this organization" = employees union managers) — for combining or filtering records *within* a single model, `.filter()`/`.exclude()`/`Q` objects (a more advanced filtering tool not covered in this lecture) are usually the more direct tool.

## 3. Field lookups — the double-underscore syntax **[From video]**

Beyond simple equality (`.filter(name='Sai')`), Django's **field lookups** let a filter specify *how* to compare a field, using a double underscore (`__`) between the field name and the lookup type:

```python
Student.objects.filter(name__exact='Sai')
```

The video works through the following lookups, one at a time, against a fresh `Student` model (`name`, `sid`, `address`, `marks`, `pass_date`, `admit_date`):

| Lookup | Meaning | Example |
|---|---|---|
| `__exact` | Exact match, **case-sensitive** | `name__exact='Sai'` matches `'Sai'` but not `'sai'` |
| `__iexact` | Exact match, **case-insensitive** (the `i` prefix means "ignore case") | `name__iexact='sai'` matches `'Sai'` too |
| `__contains` | Field contains this substring, case-sensitive | `name__contains='a'` |
| `__icontains` | Same, case-insensitive | `name__icontains='a'` |
| `__in` | Field's value is one of a given list | `id__in=[2, 4]` |
| `__gt` | Greater than | `marks__gt=75` |
| `__lt` | Less than | `marks__lt=75` |
| `__gte` | Greater than **or equal to** | `marks__gte=67` |
| `__lte` | Less than or equal to | `marks__lte=67` |
| `__range` | Between two values, inclusive | `marks__range=(80, 90)` |

```python
Student.objects.filter(marks__range=(80, 90))
```

### Date-part lookups

Because `pass_date`/`admit_date` are `DateField`/`DateTimeField`, Django additionally supports filtering by a specific *part* of a date:

```python
Student.objects.filter(pass_date__year__gt=2021)
Student.objects.filter(admit_date__month=6)     # June
Student.objects.filter(admit_date__week=25)
Student.objects.filter(pass_date__quarter=2)    # Apr–Jun
```

> **[Gap-filled] — the video's own struggle with `__week` and `__quarter`, condensed.** A significant stretch of the transcript is the presenter manually trying to guess which ISO week number a given date falls into by counting "four weeks per month" — a real, live struggle that doesn't land on a clean, reliable answer, and isn't reproduced here verbatim for that reason (see the researched note below for what `__week` actually means). `__quarter`, by contrast, is demonstrated cleanly and works as expected: quarter 1 is January–March, quarter 2 is April–June, quarter 3 is July–September, quarter 4 is October–December, and filtering `pass_date__quarter=2` correctly returns only records whose `pass_date` falls in April/May/June.

> **[Researched] — what `__week` actually measures, precisely.** Per the [Django docs](https://docs.djangoproject.com/en/stable/ref/models/querysets/#week), `__week` returns the **ISO-8601 week number** (1–53) — a standardized calendar system where week 1 is the week containing the year's first Thursday, and weeks always run Monday–Sunday. This is *not* the same as "which four-week block of the month" the video was trying to count by hand — ISO week numbers don't reset at each month boundary, and a given month doesn't cleanly map to exactly four ISO weeks (months don't divide evenly into 7-day weeks). If you need to filter by an ISO week number in a real project, computing it with Python's own `datetime.date.isocalendar()` first (to know what number to actually filter for) is far more reliable than estimating it by hand.

---

## Wrap-up

- **From video:** `.values()` (dicts) vs. `.values_list()` (tuples) as alternatives to full model instances; combining two querysets with `.union()` (distinct by default, `all=True` to keep duplicates), `.intersection()` (common rows only), and `.difference()` (rows in the first queryset but not the second); the double-underscore field-lookup syntax — `__exact`/`__iexact`, `__contains`/`__icontains`, `__in`, `__gt`/`__lt`/`__gte`/`__lte`, `__range`, and date-part lookups (`__year`, `__month`, `__week`, `__quarter`).
- **Gap-filled:** when `.values()`/`.values_list()` are actually worth reaching for; why `.values_list()` specifically (not full model instances) is needed before combining querysets from different models; a condensed, honest note about the video's own unresolved struggle with manually computing week numbers.
- **Researched:** `values_list(flat=True)` as a very common real-world variant; the precise ISO-8601 definition behind `__week`, and why counting "four weeks per month" by hand doesn't match it.

Double-check: if you ever need `__week` filtering in a real project, don't estimate the week number by counting months — compute it properly with `datetime.date.isocalendar().week` (or read it off an ISO week calendar) and filter for that exact number instead.
