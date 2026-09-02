# Lecture 14 — Static CSS/JS in templates, and the Django Template Language

Source: `transcripts/Django14.txt`
Covers: using static CSS and JS files (not just images) in a template, and the introduction of the Django Template Language (DTL) — its purpose, syntax forms, variable rules, and filters.

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official Django docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. Static CSS and JS files, same mechanism as images **[From video]**

Lecture 13 covered loading a static *image*. CSS and JS files work through the exact same `{% load static %}` + `{% static '...' %}` mechanism:

```html
{% load static %}
<link rel="stylesheet" href="{% static 'css/sample.css' %}">
<script src="{% static 'javascript/sample.js' %}" type="text/javascript"></script>
```

`{% load static %}` must be present once at the top of the template (conventionally right after `<html>` or inside `<head>`) for any `{% static %}` reference lower in that file to work.

## 2. What the Django Template Language is **[From video]**

**DTL** (Django Template Language) is the small, deliberately-limited language embedded in your HTML files — it provides a bridge between HTML/CSS/JS and Python: it lets a template display Python values (like the `result` sent from a view via context) and apply light formatting, without turning the template into a Python file.

> **[Gap-filled]** Django also supports an alternative template engine, **Jinja2** (chosen at project-creation time), which uses similar `{{ }}`/`{% %}` syntax but with a somewhat different feature set. Unless you explicitly pick Jinja2, a new Django project defaults to DTL — which is what these notes (and this course) use throughout.

### The three syntax forms

| Syntax | Purpose |
|---|---|
| `{% tag %}` | Statements/logic — `if`, `for`, `extends`, `load`, etc. |
| `{{ variable }}` | Outputs a value from the context dictionary |
| `{# comment #}` | A single-line comment (never rendered) |

> **[Researched] — multi-line comments.** `{# ... #}` only works on one line. For a comment spanning multiple lines, Django provides a block tag instead: `{% comment %} ... {% endcomment %}`. The video only shows the single-line form.

## 3. Variable rules, and passing a dictionary as context **[From video]**

```python
# views.py
def emp_details(request):
    details = {
        'EID': 1234,
        'ename': 'Sai',
        'eaddress': 'Hyderabad',
        'designation': 'Manager',
        'salary': 45000,
        'age': 30,
    }
    return render(request, 'emp.html', details)
```

```html
<!-- emp.html -->
<h2 style="color: red;">Employee ID: {{ EID }}<br>
Name: {{ ename }}<br>
Address: {{ eaddress }}<br>
Designation: {{ designation }}<br>
Salary: {{ salary }}<br>
Age: {{ age }}</h2>
```

A dictionary's *keys* (`'EID'`, `'ename'`, …) become the variable names used inside `{{ }}` in the template. The key names and the template variable names must match exactly — but nothing requires the dictionary *value* to relate to the key's name at all; the key is purely the label the template uses to look the value up.

Variable names may combine letters, numbers, and underscores, but cannot start with an underscore and cannot contain spaces or punctuation.

> **[Researched] — the dot does more than the video shows.** The video's examples only ever look up a plain dictionary key. Per the [Django docs](https://docs.djangoproject.com/en/stable/ref/templates/language/#variables), the dot in a template variable is actually a single, uniform lookup operator that Django tries, in order, as: a dictionary key (`emp.name` → `emp['name']`), then an object attribute, then a list/tuple index, then — if none of those match — a method call with no arguments. This is why you'll commonly see things like `{{ employee.name }}` or `{{ students.0.name }}` in real templates, not just bare variable names.

The same employee data can also be laid out as an HTML table, using the exact same `{{ }}` variables inside `<td>` cells instead of plain text with `<br>` — the DTL syntax doesn't change based on what HTML structure it's inside.

## 4. Filters — transforming a value before it's displayed **[From video]**

```html
{{ ename|capfirst }}
{{ eaddress|upper }}
{{ designation|lower }}
```

A **filter**, applied with the pipe symbol `|`, transforms a variable's value at display time without changing the underlying data. `capfirst` capitalizes only the first letter, `upper`/`lower` change case entirely.

> **[Example] — chaining filters, and filters with arguments.** Filters can be chained (applied left to right), and some accept a `:argument`:
> ```html
> {{ ename|lower|capfirst }}         <!-- "SAI" -> "sai" -> "Sai" -->
> {{ description|truncatewords:5 }}  <!-- shows only the first 5 words, plus "…" -->
> ```

> **[Gap-filled]** The video shows only three filters (`capfirst`, `upper`, `lower`) as a taste of the concept. Django ships several dozen built-in filters — `length`, `default` (fallback if a variable is empty), `safe` (render HTML instead of escaping it), `date` (covered in Lecture 15's notes), and more. Full reference: [Django — built-in template filters](https://docs.djangoproject.com/en/stable/ref/templates/builtins/#built-in-filter-reference).

---

## Wrap-up

- **From video:** loading static CSS/JS the same way as images, what DTL is and why it exists, its three syntax forms, variable naming rules, the employee-details context example, and the `capfirst`/`upper`/`lower` filters.
- **Gap-filled:** Jinja2 as Django's alternative template engine, and that far more filters exist than the three shown.
- **Researched:** the multi-line `{% comment %}` tag, and DTL's dot-lookup operator (dict key → attribute → index → zero-arg method), which is far more general than the plain-dictionary-key examples in the video suggest.

Double-check: the dot-lookup explanation is the single most important "researched" addition here — it's the mechanism behind almost every real-world template variable you'll see (`object.attribute`, `list.0`), and the video never states it explicitly.
