# Lecture 13 — GET vs. POST, CSRF tokens, and Django static files

Source: `transcripts/Django13.txt`
Covers: the difference between GET and POST forms, why POST forms need a CSRF token, reading submitted form data in a view, and the full setup for serving static files (CSS/JS/images) in Django.

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official Django docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. GET vs. POST **[From video]**

| | GET | POST |
|---|---|---|
| Where the data goes | Appended to the URL as a query string | Inside the request body — not in the URL |
| Visible to the user | Yes — in the address bar, browser history, and server logs | No |
| Size limit | A few thousand characters (browser/server dependent) | No practical limit |
| Use for | Non-sensitive lookups/filters (e.g. a search box) | Anything sensitive, or anything that changes data (logins, forms that write to a database) |

```html
<!-- home.html -->
<form action="add/" method="post">
    {% csrf_token %}
    Number 1: <input type="text" name="num1"><br>
    Number 2: <input type="text" name="num2"><br>
    <input type="submit" value="Add">
</form>
```

```python
# myapp/views.py
def add(request):
    a = int(request.POST.get('num1'))
    b = int(request.POST.get('num2'))
    result = a + b
    return render(request, 'result.html', {'result': result})
```

- `request.GET.get('num1')` / `request.POST.get('num1')` reads a submitted field by its `name` attribute — matching whichever method the form's `method` attribute declared.
- Everything read this way arrives as a **string**, even if the user typed digits — exactly the same lesson as JavaScript's `prompt()` back in Lecture 9. `a + b` without converting would concatenate `"10"` and `"20"` into `"1020"` instead of adding to `30` — `int(...)` fixes it.

> **[Gap-filled] — why `.get()`, not `request.POST['num1']`.** `request.POST` behaves like a dictionary. Indexing it directly (`request.POST['num1']`) raises an error if that field is missing for any reason. `.get('num1')` instead returns `None` if it's missing, which your code can then check for — the standard, safer way to read form data.

## 2. CSRF protection — required for every POST form **[From video]**

**CSRF** stands for **Cross-Site Request Forgery** — an attack where a malicious site tricks a logged-in user's browser into submitting a request to your site without their knowledge. Django blocks this by requiring a secret, per-session token inside every POST form.

```html
<form action="add/" method="post">
    {% csrf_token %}
    ...
</form>
```

Without this tag inside the `<form>`, submitting triggers a `403 Forbidden` — *"CSRF verification failed."* It's only required for `method="post"` forms; `GET` forms don't need it, since GET is meant to only read data, never change it.

> **[Gap-filled] — the trailing-slash trap.** The video also runs into an `APPEND_SLASH` redirect error: Django, by default, auto-redirects a URL missing its trailing slash (e.g. `/myapp/add` → `/myapp/add/`). For a GET request that's invisible and harmless. For a POST, the redirect drops the submitted data entirely, since a redirect is a fresh GET request. The fix: make sure your form's `action` — and the URL pattern it targets — already has the correct trailing slash, so Django never needs to redirect it.

## 3. Django static files **[From video]**

**Static files** are CSS, JavaScript, images, and anything else that doesn't change per-request. Django serves these through a dedicated system, separate from templates.

### Setup, step by step

1. **Confirm `django.contrib.staticfiles` is in `INSTALLED_APPS`** (in `settings.py`). PyCharm's project wizard adds it automatically; if you created the project another way, add it yourself.
2. **Create a `static` folder** at the project root (next to `manage.py`), with subfolders for organization — commonly `css/`, `javascript/`, `images/`. The names are convention, not a Django requirement.
3. **Point Django at that folder**, in `settings.py`:
   ```python
   import os

   STATIC_DIR = os.path.join(BASE_DIR, 'static')
   STATICFILES_DIRS = [STATIC_DIR]
   ```
4. **Load the static-file template tag** at the top of any template that uses one: `{% load static %}`
5. **Reference a file**:
   ```html
   {% load static %}
   <img src="{% static 'images/flag.jpg' %}" alt="Flag">
   ```

> **Common pitfall:** Forgetting `{% load static %}` produces *"Invalid block tag 'static'"* — exactly the error the video hits. It must be loaded in **every** template file that uses `{% static ... %}`, even ones that `{% extends %}` another template that already loaded it.

> **[Researched] — `STATIC_URL` vs. `STATICFILES_DIRS`.** The video moves between these without separating them clearly. Per the [Django docs](https://docs.djangoproject.com/en/stable/howto/static-files/): `STATIC_URL` (already set to `'/static/'` by default in a new project) is the URL *prefix* browsers use to request a static file — it's what `{% static %}` prepends to the path you give it. `STATICFILES_DIRS` is the list of folders Django actually searches *on disk*, during development, to find that file. You need both: one defines the public URL shape, the other defines where the real files live.

> **[Researched] — this setup is development-only.** Django's built-in static file serving (used automatically when you run `python manage.py runserver`) is explicitly documented as unsuitable for production. For a deployed site, you run `python manage.py collectstatic` to gather every app's static files into one folder, then serve that folder with a real web server (nginx, WhiteNoise, a CDN, etc.) instead of Django itself. This is a "later" concern, not something to set up yet — but good to know the local setup here isn't the final deployment story.

---

## Wrap-up

- **From video:** the GET/POST comparison and worked "add two numbers" example, CSRF token requirement, `request.GET`/`request.POST.get()`, and the full static-files setup (folder structure, `settings.py` config, `{% load static %}`, `{% static %}`).
- **Gap-filled:** why `.get()` beats direct dict indexing, and the trailing-slash/`APPEND_SLASH` gotcha with POST forms.
- **Researched:** the precise difference between `STATIC_URL` and `STATICFILES_DIRS`, and that this static-file setup is a development convenience, not the production story (`collectstatic` + a real web server).

Double-check: the `STATIC_URL`/`STATICFILES_DIRS` distinction and the production-vs-development note are both worth a quick read of the linked Django docs before you deploy anything for real.
