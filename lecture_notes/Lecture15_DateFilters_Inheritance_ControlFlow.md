# Lecture 15 — Date/time filters, template inheritance, and DTL control flow

Source: `transcripts/Django15.txt`
Covers: Django's named date/time format filters, template inheritance (`{% extends %}`/`{% block %}`), and DTL's `if`/`elif`/`else` and `for` tags.

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official Django docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. Date & time filters **[From video]**

Django provides named, localizable presets for formatting a `datetime` value directly in a template:

| Format name | Example output |
|---|---|
| `DATE_FORMAT` | May 16, 2022 |
| `DATETIME_FORMAT` | May 16, 2022, 3:51 p.m. |
| `SHORT_DATE_FORMAT` | 05/16/2022 |
| `SHORT_DATETIME_FORMAT` | 05/16/2022 3:51 p.m. |
| `TIME_FORMAT` | 3:51 p.m. |

```html
{{ dt|date:"DATE_FORMAT" }}
{{ dt|date:"SHORT_DATETIME_FORMAT" }}
```

> **[Researched] — custom format strings are more commonly used.** These named presets are convenient but fixed. In practice, most Django projects pass the `date` filter a custom format string built from Django's own format characters instead, for exact control: `{{ dt|date:"d M Y" }}` → `16 May 2022`, or `{{ dt|date:"H:i" }}` → 24-hour `15:51`. Common characters: `d` (day, 2-digit), `M` (month name, short), `Y` (4-digit year), `H:i` (24-hour hour:minute). Full character reference: [Django — date filter](https://docs.djangoproject.com/en/stable/ref/templates/builtins/#date).

## 2. Template inheritance **[From video]**

Same idea as class inheritance in Python: a child template reuses everything from a parent template, and only overrides the specific pieces it needs to.

```html
<!-- base.html -->
<body style="background-color: orange;">
    <title>{% block title %}Default Title{% endblock %}</title>
    <h1>Welcome to Durga's site</h1>
    {% block content %}
    {% endblock %}
</body>
```

```html
<!-- emp.html (child) -->
{% extends 'base.html' %}

{% block title %}Hello{% endblock %}

{% block content %}
    <h2>Employee ID: {{ EID }}</h2>
{% endblock %}
```

`base.html` defines named "slots" with `{% block name %} ... {% endblock %}`. `emp.html` declares `{% extends 'base.html' %}` and then re-defines the same-named blocks — Django slots that content into the matching spot in `base.html`, while everything else in `base.html` (the orange background, the `<h1>`) comes through unchanged.

> **[Gap-filled]** `{% extends %}` must be the very *first* line in a child template — Django requires this; anything (other than a comment) before it is an error. Any block the child template *doesn't* override just falls back to the parent's default content for that block — which is why giving a block sensible default content in `base.html` (like `Default Title` above) is worth doing.

> **Industry best practice:** Nearly every real Django project has one `base.html` with blocks for things like `title`, `content`, and often `extra_css`/`extra_js`, and every other page template extends it. This is one of the most-used DTL features in practice — it's how a whole site shares one consistent layout without duplicating markup.

## 3. Conditionals **[From video]**

```html
{% if name and age %}
    <h1>{{ name }}, age {{ age }}</h1>
{% elif name %}
    <h1>Hello {{ name }}</h1>
{% else %}
    <h1>Name is not present</h1>
{% endif %}
```

- Supports `and`, `or`, and `not` (there's no `!` operator in DTL) — e.g. `{% if not name %}`.
- Every `{% if %}` must be closed with `{% endif %}` — unlike Python, indentation means nothing here.

> **[Gap-filled]** DTL's `if` isn't limited to checking whether a variable exists — it also supports comparison operators: `==`, `!=`, `<`, `>`, `<=`, `>=`, exactly like the video's `{% if num == 1 %}` example. Easy to miss since most of the video's earlier examples only check truthiness.

## 4. For loops **[From video]**

```html
<ul>
{% for name in names %}
    <li>{{ name }}</li>
{% endfor %}
</ul>
```

`names` is a list passed in the context; `name` is the loop variable, taking each item's value in turn. Unlike Python's `for`/`else`, DTL's `{% for %}` only needs `{% endfor %}` — there's no matching "else" requirement.

> **[Researched] — `forloop.counter`.** The video's loop examples never need to know the current position, but numbering list items is a very common real need. Inside any `{% for %}`, Django automatically provides a `forloop` variable: `forloop.counter` (1-indexed position), `forloop.counter0` (0-indexed), `forloop.first`/`forloop.last` (booleans). Example: `<li>{{ forloop.counter }}. {{ name }}</li>` numbers each item. Full reference: [Django — for tag](https://docs.djangoproject.com/en/stable/ref/templates/builtins/#for).

---

## Wrap-up

- **From video:** the five named date/time format filters, template inheritance with `extends`/`block`/`endblock`, `if`/`elif`/`else` conditionals (including `and`/`or`/`not` and equality checks), and `for` loops over a list.
- **Gap-filled:** the hard rule that `{% extends %}` must be the first line of a child template, and that DTL's `if` supports full comparison operators, not just truthiness checks.
- **Researched:** custom date format strings as the more common real-world alternative to the named presets, and `forloop.counter`/`forloop.first`/`forloop.last` for numbering or special-casing loop items.

Double-check: `forloop.counter` and the custom date-format characters are both things you'll likely reach for constantly in real templates — worth trying both hands-on even though the video doesn't demonstrate them.
