# Lecture 12 — Rebuilding project/app/view/URL basics, then render() and context

Source: `transcripts/Django12.txt`
Covers: a recap of creating a Django project/app, defining views and project- vs. app-level URLs, then the genuinely new material — why not to embed HTML in `views.py`, the `render()` function, and passing a context dictionary to a template.

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official Django docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. Recap: project, app, views, URLs **[From video — already covered in earlier lecture notes]**

The video re-covers ground already in these notes: creating a project (`django-admin startproject`), creating an app (`python manage.py startapp`), adding it to `INSTALLED_APPS`, writing a view with `HttpResponse`, connecting it in `urls.py`, and the project-level-vs-app-level `urls.py` + `include()` pattern (see the notes for Lectures 3–7). One extra wrinkle worth noting:

> **Common pitfall:** don't name a view function the same as a module you've imported — e.g. `import datetime` followed by `def datetime(request): ...`. Your function definition silently overwrites the imported module name, so any later use of `datetime.datetime.now()` inside that file fails with a confusing `AttributeError` (exactly what happens in the video). Pick a view name that doesn't collide with anything you've imported.

## 2. From `HttpResponse` to `render()` **[From video]**

Writing actual HTML inside a Python string (`HttpResponse("<h1>...</h1>")`) works, but mixes two languages in one file and gets unreadable fast. The fix: write the HTML in its own template file, and have the view hand data to it with `render()`.

```python
# myapp/views.py
from django.shortcuts import render

def home(request):
    return render(request, 'home.html', {'name': 'Mohan'})
```

- `render()` takes three things: the `request` object, the template's filename (found automatically inside your `templates` folder), and an optional **context** — a dictionary of values to make available inside that template.
- Inside `home.html`, that dictionary key becomes a template variable: `Hello {{ name }}` renders as `Hello Mohan`. (Full Django Template Language syntax is covered in Lecture 14/15's notes.)
- `context` can be passed positionally or as `context={'name': 'Mohan'}` — both work; the keyword form is just more explicit.

## 3. Templates folder location **[From video]**

The `templates` folder lives at the **project** level (not inside a specific app) — this is what lets multiple apps share the same templates. When a project is created via PyCharm's wizard, this folder (and its entry in `settings.py`'s `TEMPLATES` setting) is created automatically; with another editor/CLI setup, you'd need to create the folder and add its path to `TEMPLATES[0]['DIRS']` in `settings.py` yourself.

---

## Wrap-up

- **From video:** the full recap of project/app/view/URL creation, the reasoning against embedding HTML in `views.py`, and the introduction of `render()` with a context dictionary.
- **Gap-filled:** the module-name-collision pitfall (naming a view the same as an imported module).
- **Researched:** none needed this lecture.

Double-check: this lecture is mostly a clean recap plus one new mechanism (`render()`/context) — nothing here needed correction against the docs.
