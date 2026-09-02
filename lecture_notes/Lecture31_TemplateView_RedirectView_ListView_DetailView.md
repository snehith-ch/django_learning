# Lecture 31 — TemplateView, RedirectView, and starting the generic display views

Source: `transcripts/Django31 (1).txt`
Covers: the two remaining base class-based views (`TemplateView`, `RedirectView`), rounding out the base-view family previewed in Lecture 29–30, then the start of Django's **generic** class-based views — `ListView` and `DetailView` — the two "display" views.

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official Django docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. `TemplateView` — rendering a template without hand-writing `get()` **[From video]**

Recall from Lecture 30: `TemplateView` is one of the three base class-based views (alongside `View` and `RedirectView`), and it's essentially what you'd get by writing `TemplateView`'s job by hand with a plain `View` subclass — set a template name, and it handles calling `render()` for you.

```python
# class_based_view_app/views.py
from django.views.generic.base import TemplateView

class MyTemplateView(TemplateView):
    template_name = 'home.html'

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['name'] = 'Mohan'
        context['mail'] = 'mohan@gmail.com'
        return context
```

- `template_name` is a class attribute — no `get()` method needs to be written at all for the simple case of "just render this template."
- `get_context_data(self, **kwargs)` is the hook for supplying context (the dictionary of values available inside the template). `**kwargs` collects any keyword arguments — the double-star syntax means "gather these into a dictionary."
- `super().get_context_data(**kwargs)` calls the same method on the parent class (`TemplateView`, which itself inherits this from a further base class) first, getting back whatever context Django's own machinery already builds, **then** the code adds its own keys (`name`, `mail`) on top before returning it.

```python
# urls.py
path('temp/', views.MyTemplateView.as_view()),
```

Visiting `/temp/` renders `home.html` with `{{ name }}` and `{{ mail }}` available, exactly as if a plain `render(request, 'home.html', {'name': ..., 'mail': ...})` had been called from an ordinary function-based view.

> **[Gap-filled] — why `super().get_context_data(**kwargs)` matters, not just `context = {}`.** It's tempting to just build a fresh dictionary from scratch. The reason to call `super()` first is that `TemplateView`'s own base classes may already populate context with useful built-in values (e.g. the view instance itself, under `view`) — skipping the `super()` call silently discards whatever the built-in machinery would have added. This is the same "call the parent's version first, then extend it" pattern as `SignUpForm`'s `super().clean()` back in Lecture 26, and `PasswordChangeForm`'s underlying `form_valid()` chain from Lecture 28 — a recurring idiom throughout Django's class-based machinery.

## 2. Passing extra context straight from the URL configuration **[From video]**

```python
# urls.py
path('temp/', views.MyTemplateView.as_view(extra_context={'age': 38})),
```

`extra_context` is a built-in attribute every `TemplateView` (and its relatives) supports — anything in this dictionary is automatically merged into the template's context, no `get_context_data()` override required for this simpler case. Visiting `/temp/` now also shows `{{ age }}` as `38`, supplied entirely from the URL configuration rather than from inside the view class.

> **[Researched] — `extra_context` vs. overriding `get_context_data()`.** Per the [Django docs](https://docs.djangoproject.com/en/stable/ref/class-based-views/mixins-simple/#django.views.generic.base.ContextMixin.extra_context), `extra_context` is meant for static, unchanging values known at URL-configuration time (like the `age` above). `get_context_data()` is for anything that needs to be *computed* per-request (a database query, something derived from `self.kwargs`, etc.) — `extra_context` alone can't do that, since it's just a fixed dictionary set once when the URL pattern is defined.

## 3. `RedirectView` — sending a request straight to another URL **[From video]**

```python
# class_based_view_app/views.py
from django.views.generic.base import RedirectView

class FlipkartView(RedirectView):
    url = 'https://flipkart.com'
```

```python
# urls.py
path('flip/', views.FlipkartView.as_view()),
```

Visiting `/flip/` redirects the browser straight to `https://flipkart.com` — no template, no context, just a redirect. `url` is the class attribute holding the destination.

```python
# alternative: resolve the destination inside the URL config instead of the view
class FlipkartView(RedirectView):
    pass  # url intentionally left unset here

# urls.py
path('flip/', views.FlipkartView.as_view(url='https://flipkart.com')),
```

Exactly parallel to `extra_context` above and `template_name` from Lecture 30's multi-template technique: the destination can either live as a class attribute on the view, or be supplied per-URL via `as_view(url=...)` — same "configure through `as_view()`" mechanism, just applied to a different attribute.

> **[Gap-filled] — a realistic use for `RedirectView`.** The video's e-commerce-site example is a genuine, common pattern: an "affiliate link" or "visit our partner" button that always sends visitors to one fixed external destination. It's also commonly used *internally* — e.g. redirecting an old, retired URL (`/old-page/`) to its replacement (`/new-page/`) after a site restructure, without needing a full view with a template.

## 4. A fresh app for the generic (display + editing) views **[From video]**

To avoid confusion with the existing `class_based_view_app`, the video starts a new app specifically for exploring Django's **generic** class-based views:

```bash
python manage.py startapp generic_views
```

```python
# settings.py
INSTALLED_APPS = [
    # ...
    'generic_views',
]
```

Recall the taxonomy from Lecture 30: generic views split into **display views** (`ListView`, `DetailView`) and **editing views** (`FormView`, `CreateView`, `UpdateView`, `DeleteView`). This lecture covers the display views; editing views follow in Lecture 32.

```python
# generic_views/models.py
from django.db import models

class Student(models.Model):
    name = models.CharField(max_length=20)
    mail = models.CharField(max_length=20)
    address = models.CharField(max_length=20)
```

```python
# generic_views/admin.py
from django.contrib import admin
from .models import Student

@admin.register(Student)
class StudentAdmin(admin.ModelAdmin):
    list_display = ('name', 'mail', 'address')
```

Same `startapp` → register → model → `makemigrations`/`migrate` → admin registration cycle used throughout the course, with a handful of `Student` records entered through the admin panel for the examples below.

## 5. `ListView` — displaying every record **[From video]**

```python
# generic_views/views.py
from django.views.generic import ListView
from .models import Student

class StudentListView(ListView):
    model = Student
```

That's the entire view — no template, no queryset code, nothing else. Wiring it up:

```python
# generic_views/urls.py
from generic_views import views

urlpatterns = [
    path('', views.StudentListView.as_view()),
]
```

Visiting the URL immediately produces a `TemplateDoesNotExist` error naming a very specific file: **`student_list.html`**. This is the crux of the lesson:

> **`ListView` expects a template following the naming convention `<app_lowercase_model_name>_list.html`** — here, because the model is `Student`, the required (default) template name is `student_list.html`. Nothing about this name is arbitrary or made up by the developer; it's a fixed convention `ListView` looks for automatically, so the template just needs to exist with the right name for it to be found with zero extra configuration.

```html
<!-- templates/student_list.html -->
{% for stu in student_list %}
    <p>{{ stu.name }} — {{ stu.mail }} — {{ stu.address }}</p>
{% endfor %}
```

The default context variable — the name used to loop over the records inside the template — is also convention-based: **`<model_name_lowercase>_list`**, here `student_list`. (The video's own first attempt trips on exactly this: naming the template `student.html` instead of `student_list.html` produces the same `TemplateDoesNotExist` error, since the convention isn't optional.)

### Customizing `ListView`

```python
class StudentListView(ListView):
    model = Student
    template_name = 'student_list.html'   # override the default template name
    ordering = 'name'                      # ORDER BY name
    context_object_name = 'students'       # override the default context variable name

    def get_queryset(self):
        return Student.objects.filter(name='Durga')
```

| Attribute/method | What it changes |
|---|---|
| `template_name` | Overrides the default `<model>_list.html` convention with an explicit name. |
| `ordering` | Sorts the queryset — `'name'` ascending, `'-name'` descending (same `-` convention as elsewhere in the ORM). |
| `context_object_name` | Overrides the default `<model>_list` context variable name (e.g. to `students`) — the template's `{% for %}` loop must then use the matching new name. |
| `get_queryset(self)` | Overrides *which* records the view fetches at all — here, filtering to only students named `'Durga'` instead of every record. |

> **[Gap-filled] — why `get_queryset()` matters beyond this toy filter.** `model = Student` alone always means "give me every `Student` row" (`Student.objects.all()`, implicitly). Overriding `get_queryset()` is how a `ListView` shows anything *other* than the full unfiltered table — a "my orders" page filtered to the logged-in user, an "active only" list excluding archived rows, or (as later lectures on pagination build on directly) records ordered and windowed in a specific way. This is the exact same underlying idea as the manual `Employee.objects.filter(...)` calls from earlier lectures — `ListView` just gives it a designated method to override instead of writing it inline in a function-based view.

## 6. `DetailView` — displaying one record **[From video]**

```python
# generic_views/views.py
from django.views.generic import DetailView

class StudentDetailView(DetailView):
    model = Student
```

```python
# generic_views/urls.py
path('detail/<int:pk>/', views.StudentDetailView.as_view()),
```

Same convention-based defaults as `ListView`, adapted for a single record:

- **Default template name:** `<app_lowercase_model_name>_detail.html` — here, `student_detail.html`.
- **Default context variable:** the lowercase model name itself — here, `student`.
- **How the specific record is chosen:** the URL pattern must supply a value named **`pk`** (primary key) — `DetailView` automatically looks for a URL keyword argument literally called `pk` and uses it to fetch that one row via `Student.objects.get(pk=...)` internally, with no code required in the view class to do that lookup by hand.

```html
<!-- templates/student_detail.html -->
<h2>{{ student.name }}</h2>
<h3>{{ student.mail }}</h3>
<p>{{ student.address }}</p>
```

Visiting `/detail/1/` shows the first student's record; `/detail/2/` shows the second; and so on — `<int:pk>` in the URL pattern (the same path-converter mechanism from Lecture 21) supplies whatever numeric ID was requested, and `DetailView` handles the rest.

> **[Researched] — `pk_url_kwarg`, in case the URL parameter can't be named `pk`.** Per the [Django docs](https://docs.djangoproject.com/en/stable/ref/class-based-views/generic-display/#django.views.generic.detail.DetailView), if a URL pattern needs to use a different parameter name than `pk` (say, `<int:student_id>`), setting `pk_url_kwarg = 'student_id'` on the `DetailView` subclass tells it to look for that name instead. The video doesn't need this (it uses `pk` directly), but it's the documented way to keep `DetailView`'s automatic lookup working with a differently-named URL parameter.

---

## Wrap-up

- **From video:** `TemplateView` (`template_name`, `get_context_data()` with `super()`, `extra_context` via `as_view()`); `RedirectView` (`url` attribute, settable via `as_view()` too); starting a fresh `generic_views` app; `ListView` (`model`, the `<model>_list.html`/`<model>_list` naming conventions, `template_name`, `ordering`, `context_object_name`, `get_queryset()` for filtering); `DetailView` (`model`, the `<model>_detail.html`/`<model>` naming conventions, the required `pk` URL keyword argument).
- **Gap-filled:** why `super().get_context_data()` matters instead of building context from scratch; a realistic non-toy use case for `RedirectView`; why `get_queryset()` is the general mechanism for any filtered/customized `ListView`, not just this one example.
- **Researched:** `extra_context` vs. overriding `get_context_data()` — which one actually fits static vs. per-request data; `pk_url_kwarg` for when a URL's primary-key parameter can't be named `pk`.

Double-check: the naming conventions (`<model>_list.html`/`<model>_list`, `<model>_detail.html`/`<model>`) are the single most load-bearing fact in this lecture — nearly every "why isn't my view finding a template" error with `ListView`/`DetailView` traces back to a template name that doesn't match this convention (unless `template_name` is explicitly set to override it).
