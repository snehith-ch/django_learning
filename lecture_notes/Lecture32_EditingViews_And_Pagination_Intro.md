# Lecture 32 — Editing views (Form/Create/Update/Delete), and introducing pagination

Source: `transcripts/Django32.txt`
Covers: the four generic "editing" class-based views that round out Lecture 30–31's taxonomy — `FormView`, `CreateView`, `UpdateView`, `DeleteView` — then a topic switch to pagination: splitting a large set of database records across multiple pages, built first with a plain function-based view.

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official Django docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. `FormView` — display and validate a plain form **[From video]**

> `FormView` refers to a view to display and verify a Django form.

Unlike `CreateView` (next), `FormView` isn't tied to saving a model — it's the generic-CBV equivalent of the plain `forms.Form`-handling pattern from Lectures 25–26, just expressed as a class.

```python
# generic_views/forms.py
from django import forms

class StudentForm(forms.Form):
    name = forms.CharField()
    mail = forms.EmailField()
    message = forms.CharField(widget=forms.Textarea)
```

```python
# generic_views/views.py
from django.views.generic.edit import FormView
from .forms import StudentForm

class StudentFormView(FormView):
    template_name = 'student.html'
    form_class = StudentForm
    success_url = 'success/'

    def form_valid(self, form):
        print(form.cleaned_data['name'])
        print(form.cleaned_data['mail'])
        print(form.cleaned_data['message'])
        return super().form_valid(form)
```

| Attribute/method | Meaning |
|---|---|
| `template_name` | The template used to display the (unbound or invalid) form. |
| `form_class` | Which form class to instantiate — the class-based equivalent of `fm = StudentForm(request.POST)` in a function-based view. |
| `success_url` | Where to redirect once the form is successfully validated and processed. |
| `initial` | (Not used in this example, but part of the same attribute family) Initial values for the form's fields. |
| `form_valid(self, form)` | Called automatically once the posted form has passed validation — this is where you'd act on the data (here, just printing it to the terminal). |

`return super().form_valid(form)` at the end is the same "call the parent, then extend" idiom from Lecture 31's `get_context_data()` — the base `FormView.form_valid()` already knows how to redirect to `success_url`; overriding the method to add a `print()` and still calling `super()` keeps that built-in redirect behavior intact.

```python
# generic_views/urls.py
path('student/', views.StudentFormView.as_view()),
path('success/', views.SuccessTemplateView.as_view(), name='success'),
```

```python
# a plain TemplateView (Lecture 31) reused for the success page
class SuccessTemplateView(TemplateView):
    template_name = 'success.html'
```

```html
<!-- student.html -->
<form method="post" novalidate>
    {% csrf_token %}
    {{ form.as_p }}
    <input type="submit" value="Submit">
</form>
```

```html
<!-- success.html -->
<h2>Form submitted successfully</h2>
```

Submitting valid data prints the three cleaned values to the terminal (mirroring `print()`-based debugging from earlier lectures) and redirects to `success.html`; submitting with blank fields shows Django's standard "This field is required" messages, exactly as any `forms.Form` would.

## 2. `CreateView` — inserting a new record **[From video]**

> `CreateView` refers to a view to create an instance of a table in the database — meaning we're going to create a record, insert a record into a database table.

```python
# generic_views/views.py
from django.views.generic.edit import CreateView
from .models import Student

class StudentCreateView(CreateView):
    model = Student
    fields = ['name', 'mail', 'message']
    success_url = 'success/'
```

- `model` — which model this view creates a new row in (same role `model` plays in `ListView`/`DetailView`).
- `fields` — which of that model's fields should appear on the generated form — the same idea as `ModelForm`'s `Meta.fields` (Lecture 27), because `CreateView` is, under the hood, building and validating a `ModelForm` for you automatically.
- Same **naming convention** as `ListView`/`DetailView`: the default template is `<model>_form.html` — here, `student_form.html`.

```html
<!-- student_form.html -->
<form method="post">
    {% csrf_token %}
    {{ form.as_p }}
    <input type="submit" value="Submit">
</form>
```

Filling in the form and submitting inserts a new `Student` row and redirects to `success_url` — with **zero hand-written save logic**: no `Student(name=..., mail=..., message=...)`, no `.save()` call anywhere in the view. `CreateView` handles the entire "build a `ModelForm`, validate it, call `.save()`" cycle from Lecture 27 internally.

> **[Gap-filled] — a real debugging detour worth knowing about, condensed.** The video hits a messy stretch here trying to add a `message` field to the existing `Student` model from Lecture 31, running into a migration error (`it is impossible to add a non-nullable field 'message' to student without specifying a default`) — because existing rows in the table have no value for a newly-added required column. Rather than resolve it by supplying a default, the video sidesteps the problem by creating a **brand-new** model (`StudentOne`) with all three fields from the start, and continues the rest of the lecture on that new model instead. The durable lesson, independent of the specific detour: **adding a new non-nullable field to a model that already has rows requires Django to know what value existing rows should get** — you're prompted to either supply a one-time default value (Django's own suggested fix) or make the field nullable (`null=True`) so existing rows can simply have no value there. Creating a parallel model to dodge the question, as the video does, avoids the migration but isn't the normal fix in a real project — see the researched note below for the actual recommended approach.

> **[Researched] — the correct way to add a required field to an existing model.** Per the [Django docs on migrations](https://docs.djangoproject.com/en/stable/topics/migrations/#adding-fields-to-models), the standard options when adding a non-nullable field to a model with existing rows are: (1) give the field a `default=` value in the model definition itself, so both new and existing rows get a sensible value, or (2) set `null=True` so existing rows can hold `NULL` for that column. Django's interactive migration prompt (seen in the video) is offering exactly option 1 as a one-time value for the migration — accepting it is usually simpler and safer than creating a duplicate model, which leaves two near-identical tables to maintain going forward.

## 3. `UpdateView` — editing an existing record **[From video]**

> `UpdateView` refers to a view to update a particular instance of a table from the database.

```python
class StudentUpdateView(UpdateView):
    model = StudentOne
    template_name = 'student_form.html'   # reuses the same form template as CreateView
    fields = ['name', 'mail', 'message']
    success_url = 'success/'
```

```python
# urls.py
path('update/<int:pk>/', views.StudentUpdateView.as_view()),
```

Structurally almost identical to `CreateView` — same `model`, `fields`, `success_url`, and (here) even the same template, since a "create" form and an "edit" form usually look identical. The difference is entirely in the URL: **`UpdateView` requires a `pk` in the URL**, exactly like `DetailView` from Lecture 31, so it knows *which* existing row to load and pre-fill the form with, rather than building a blank one. Visiting `/update/1/` shows the first student's data already filled into the form; submitting changes updates that same row rather than creating a new one — the class-based counterpart of `instance=request.user` from Lecture 29's `EditUserProfileForm`.

## 4. `DeleteView` — removing a record, with a confirmation step **[From video]**

> `DeleteView` refers to a view to delete a particular instance of a table from the database.

```python
class StudentDeleteView(DeleteView):
    model = StudentOne
    fields = ['name', 'mail', 'message']
    success_url = 'success/'
```

```python
# urls.py
path('delete/<int:pk>/', views.StudentDeleteView.as_view()),
path('no_delete/', views.NoDeleteTemplateView.as_view(), name='no_delete'),
```

- Same `pk`-in-the-URL requirement as `UpdateView`, to identify which row is being targeted.
- **Default template name:** `<model>_confirm_delete.html` — here, `studentone_confirm_delete.html` — a deliberately different convention from `<model>_form.html`, because a delete view's first job isn't showing an editable form, it's asking the user to confirm the destructive action before it happens.

```html
<!-- studentone_confirm_delete.html -->
<p>Do you want to delete the record?</p>
<form method="post">
    {% csrf_token %}
    <input type="submit" value="Delete">
</form>
<a href="{% url 'no_delete' %}">Cancel</a>
```

- Submitting the form (a POST request) actually performs the deletion and redirects to `success_url`.
- The **Cancel** link is a plain `{% url %}` link (Lecture 21) to a separate `no_delete/` page — this isn't part of `DeleteView` itself, it's just an ordinary link the video adds so a user who changes their mind has somewhere to go instead of confirming.

```python
class NoDeleteTemplateView(TemplateView):
    template_name = 'no_delete.html'
```

```html
<!-- no_delete.html -->
<h3>You have not confirmed to delete the student record.</h3>
```

> **[Gap-filled] — why the confirmation step matters, and why it's the default.** `DeleteView`'s two-step behavior (GET shows a confirmation page, POST actually deletes) exists specifically so a plain link or accidental page visit can never trigger a deletion — only an explicit form submission can. This mirrors a general web-development principle already seen with GET vs. POST back in Lecture 19: GET requests are supposed to be safe/non-destructive, so anything that changes or removes data belongs behind a POST, never behind a link a search engine crawler or browser prefetch could follow by accident.

## 5. Class-based views, summarized **[From video]**

The video closes the CBV arc with a recap quiz, confirming the full taxonomy built up across Lectures 29–32:

- **Base class-based views** (base views): `View`, `TemplateView`, `RedirectView`.
- **Generic class-based views** (generic views) — two categories:
  - **Display views:** `ListView`, `DetailView`.
  - **Editing views:** `FormView`, `CreateView`, `UpdateView`, `DeleteView`.

> **Industry best practice:** The recurring theme across every editing view in this lecture — near-zero view code compared to the equivalent function-based version — is the real argument for reaching for generic CBVs whenever a view's job matches one of these five shapes exactly. The moment a view needs logic that doesn't fit the shape (multiple forms in one view, conditional branches based on business rules), that's the signal to drop back to a function-based view or a plain `View` subclass instead of fighting a generic CBV into an unnatural shape.

---

## 6. Introducing pagination **[From video]**

> Pagination is the process of displaying data across different pages. Pagination allows us to split the data into multiple pages.

Rendering every row of a large table onto one page (a hundred employee records, for instance) is impractical — an unbroken wall of content the visitor has to scroll through indefinitely. Pagination splits that same data across multiple pages, with a fixed number of records per page and next/previous navigation between them.

Django provides two built-in classes for this: **`Paginator`** and **`Page`**.

## 7. Pagination with a function-based view **[From video]**

```bash
python manage.py startapp pagination_app
```

```python
# pagination_app/models.py
from django.db import models

class Book(models.Model):
    title = models.CharField(max_length=20)
    author = models.CharField(max_length=20)
    description = models.TextField(max_length=300)
```

```python
# pagination_app/views.py
from django.core.paginator import Paginator
from django.shortcuts import render
from .models import Book

def page_view(request):
    all_pages = Book.objects.all().order_by('id')
    paginator = Paginator(all_pages, 3)   # 3 records per page
    page_number = request.GET.get('page')
    page_obj = paginator.get_page(page_number)
    return render(request, 'pages.html', {'page_obj': page_obj})
```

- `Book.objects.all().order_by('id')` — the full queryset, explicitly ordered (important: without a defined order, which records land on which page can be inconsistent between requests).
- `Paginator(all_pages, 3)` — wraps the queryset; the second argument (`3`) is how many records belong on each page.
- `request.GET.get('page')` — reads the requested page number from the URL's query string (e.g. `?page=2`) — a **GET** parameter, per the GET-vs-POST distinction from Lecture 19, since viewing a different page is a safe, non-destructive lookup.
- `paginator.get_page(page_number)` — returns a `Page` object holding just that page's slice of records, plus the navigation info the template needs.

```html
<!-- pages.html -->
<h1 style="color: red;">Book details</h1>
{% for page in page_obj %}
    <h2>{{ page.title }}</h2>
    <h2>{{ page.author }}</h2>
    <p>{{ page.description }}</p>
{% endfor %}

<div>
    <span>
        {% if page_obj.has_previous %}
            <a href="?page={{ page_obj.previous_page_number }}">Previous</a>
        {% endif %}
    </span>
    <span>{{ page_obj.number }}</span>
    <span>
        {% if page_obj.has_next %}
            <a href="?page={{ page_obj.next_page_number }}">Next</a>
        {% endif %}
    </span>
</div>
```

| `Page` attribute | Meaning |
|---|---|
| `has_previous` / `has_next` | Whether a previous/next page actually exists (so the corresponding link only shows when it's valid to click). |
| `previous_page_number` / `next_page_number` | The page number to link to. |
| `number` | The current page's own number, for display. |

Iterating `{% for page in page_obj %}` loops over just the current page's slice of `Book` records (despite the loop variable being named `page` here, each item is actually one `Book` instance — a naming choice worth noting since it can read as confusing).

> **[Gap-filled] — ordering the data before paginating it.** The video demonstrates this concretely: without `order_by`, records display in whatever order the database happens to return them, which can look arbitrary; adding `order_by('id')` (or `order_by('title')`, tried later in the same lecture) gives predictable, consistent paging — page 1 always shows the same records on every visit. Always order a queryset before paginating it, for the same reason a shuffled deck makes for confusing page numbers.

## 8. Handling a partial last page: `orphans` **[From video]**

With 10 total `Book` records and 3 per page, the last page ends up with just 1 leftover record — an "orphan." Django's `Paginator` has a dedicated parameter for folding a too-small leftover page back into the previous one:

```python
paginator = Paginator(all_pages, 3, orphans=1)
```

`orphans=1` tells the paginator: if the last page would otherwise have 1 or fewer records, merge them into the previous page instead of leaving a nearly-empty final page. With this setting, 10 records at 3-per-page become 3 pages of 3 and one page of **4** (the last page absorbing the 1 orphan), rather than 3 pages of 3 followed by an awkward 4th page holding a single record.

> **[Researched] — `orphans` is about avoiding a jarringly small last page, not a hard rule.** Per the [Django docs](https://docs.djangoproject.com/en/stable/ref/paginator/#django.core.paginator.Paginator), `orphans` sets the *minimum* number of items allowed on the last page before Django folds them into the second-to-last page instead — it defaults to `0` (no folding at all) unless explicitly set. It's a cosmetic/UX nicety, not something that changes how many total pages exist in the common case where the last page already has a reasonable number of records.

---

## Wrap-up

- **From video:** `FormView` (`template_name`, `form_class`, `success_url`, `form_valid()` with `super()`); `CreateView` (`model`, `fields`, the `<model>_form.html` convention, zero hand-written save logic); `UpdateView` (same shape as `CreateView` plus a required `pk` in the URL); `DeleteView` (the `<model>_confirm_delete.html` convention, POST-to-actually-delete, a cancel link to a separate no-delete page); a recap of the full base-view/generic-view taxonomy; pagination fundamentals (`Paginator`, `Page`, `get_page()`, `has_previous`/`has_next`/`previous_page_number`/`next_page_number`/`number`) built with a function-based view; the `orphans` parameter for merging a too-small last page.
- **Gap-filled:** condensing a messy live migration-error detour into its durable lesson (adding a required field to a populated table needs a default or nullability); why `DeleteView`'s confirm-then-POST shape exists, tied back to the GET/POST safety principle; why ordering a queryset before paginating it matters.
- **Researched:** the actually-recommended fix for adding a non-nullable field to an existing model (vs. the video's model-duplication workaround); the precise meaning and default of the `orphans` parameter.

Double-check: the video's own migration detour (creating `StudentOne` instead of fixing the `Student` model directly) is flagged here as a workaround, not a pattern to copy — the researched note above gives the actual recommended fix if you hit the same "cannot add a non-nullable field" error in your own project.
