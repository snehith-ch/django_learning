# Lecture 40 — Finishing the MySQL CRUD project

Source: `transcripts/Django40.txt`
Covers: picking up the bare, unstyled MySQL-backed project from Lecture 39 and turning it into a complete, Bootstrap-styled CRUD (Create, Read, Update, Delete) application — insert, list, edit, update, and delete, all working against the real MySQL table.

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official Django docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. Styling the form with Bootstrap **[From video]**

Lecture 39 ended with a bare `{{ form.as_table }}` — no `<form>` tag, no submit button, no styling. This lecture wraps it properly:

```html
<!-- templates/index.html -->
<head>
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/css/bootstrap.min.css">
</head>
<body>
<div class="container">
    <form method="post" action="/">
        {% csrf_token %}
        <div class="form-group">
            <label><b>User Name</b></label>
            {{ form.uname }}
        </div>
        <div class="form-group">
            <label><b>User Email</b></label>
            {{ form.uemail }}
        </div>
        <div class="form-group">
            <label><b>User Password</b></label>
            {{ form.upassword }}
        </div>
        <input type="submit" name="insert" value="Insert" class="btn btn-primary">
    </form>
</div>
</body>
```

- The Bootstrap `<link>` is loaded via CDN (Lecture 18) — no local files needed.
- `container` and `form-group` are Bootstrap's own layout classes (Lecture 18); `btn btn-primary` styles the submit button.
- `{{ form.uname }}` etc. render **one field at a time** (Lecture 25's "no wrapper" rendering style) — necessary here because each field needs its own `<div class="form-group">` wrapper for Bootstrap's spacing to apply correctly, which a single `{{ form.as_table }}` call can't provide per-field.

To make each individual input actually pick up Bootstrap's `form-control` styling (rather than rendering as a plain, unstyled `<input>`), the form fields themselves need a matching **widget attribute**:

```python
# forms.py
class UserForm(forms.ModelForm):
    uname = forms.CharField(widget=forms.TextInput(attrs={'class': 'form-control'}))
    uemail = forms.CharField(widget=forms.TextInput(attrs={'class': 'form-control'}))
    upassword = forms.CharField(widget=forms.TextInput(attrs={'class': 'form-control'}))

    class Meta:
        model = User
        fields = '__all__'
```

`widget=forms.TextInput(attrs={'class': 'form-control'})` is the same field-customization mechanism from Lecture 25 (`widget=forms.PasswordInput`, etc.) — here used specifically to inject Bootstrap's CSS class onto the rendered `<input>` tag, since a `ModelForm`'s auto-generated fields don't carry any CSS class by default.

## 2. The insert view **[From video]**

```python
# views.py
from django.http import HttpResponse
from django.shortcuts import render
from mysql_crud_app.models import User
from mysql_crud_app.forms import UserForm

def insert(request):
    if request.method == 'POST':
        form = UserForm(request.POST)
        if form.is_valid():
            try:
                form.save()
                return HttpResponse("Data inserted into database successfully.")
            except:
                pass
    else:
        form = UserForm()
    return render(request, 'index.html', {'form': form})
```

Same GET-shows-empty-form/POST-validates-and-saves shape established since Lecture 26, with a `try`/`except` wrapped around `form.save()`. The video hits a real, live debugging moment here worth knowing about:

> **[Gap-filled] — a genuine debugging story, and the actual lesson in it.** After wiring this up, submitting the form silently failed to show the expected "data inserted successfully" message — with no visible error at all. The eventual fix, found by re-reading the code carefully line by line, was a typo: a stray or misplaced reference among `form.uname`/`form.uemail`/`form.upassword` inside the template that didn't match the form's actual field names. The durable lesson, independent of the exact typo: **when a Django view produces no error but also doesn't do what's expected, check the template's field references against the form/model's actual field names character-for-character** — a mismatched name here doesn't raise an exception, it just silently fails to render or bind correctly, which is a harder class of bug to spot than a traceback.

Confirmed working by checking MySQL Workbench directly (`SELECT * FROM users;`) after a few submissions — new rows appear with the submitted `uname`/`uemail`/`upassword` values.

## 3. The show view — listing every record **[From video]**

```python
def show(request):
    users = User.objects.all()
    return render(request, 'show.html', {'users': users})
```

```html
<!-- templates/show.html -->
<div class="container">
    <table class="table table-bordered text-center shadow">
        <tr>
            <th>User Name</th>
            <th>User Email</th>
            <th>User Password</th>
            <th></th>
            <th></th>
        </tr>
        {% for user in users %}
        <tr>
            <td>{{ user.id }}</td>
            <td>{{ user.uname }}</td>
            <td>{{ user.uemail }}</td>
            <td>{{ user.upassword }}</td>
            <td><a href="/delete/{{ user.id }}/" class="btn btn-danger">Delete</a></td>
            <td><a href="/edit/{{ user.id }}/" class="btn btn-success">Edit</a></td>
        </tr>
        {% endfor %}
    </table>
</div>
```

Same `.objects.all()` + `{% for %}` loop pattern from Lecture 23, now with Bootstrap table classes (`table table-bordered text-center shadow`) and, in the last two columns, **plain links** (not buttons wired to a form) to the delete and edit actions for each row — `{{ user.id }}` interpolated directly into the `href`, the same dynamic-URL-building idea from Lecture 21.

> **Industry best practice / [Gap-filled]:** Note that "Delete" here is a plain `<a href>` link, which means it's a **GET** request — clicking it immediately deletes the record with no confirmation step. Compare this to Lecture 32's generic `DeleteView`, which deliberately requires a **POST** (via a confirm-then-submit page) specifically so a browser prefetch, a crawler, or an accidental click can never trigger a real deletion. This project's plain-link delete is simpler to build but is exactly the kind of shortcut a real production app should avoid — the video's own project prioritizes getting CRUD working end-to-end over following that safer pattern.

## 4. The delete view **[From video]**

```python
def delete(request, id):
    User.objects.filter(id=id).delete()
    return redirect('show')
```

`id` arrives as a URL path converter (Lecture 21's `<int:id>` mechanism), used to filter down to exactly one row and delete it, then redirect back to the `show` view so the updated list is immediately visible.

## 5. The edit and update views **[From video]**

Editing is split into two views: one that loads an existing record's data *into* a form for editing, and a second that actually saves the submitted changes back.

```python
def edit(request, id):
    user = User.objects.get(id=id)
    return render(request, 'edit.html', {'user': user})
```

```html
<!-- templates/edit.html -->
<div class="container">
    <form method="post" action="/update/{{ user.id }}/">
        {% csrf_token %}
        <div class="form-group">
            <label>User Name</label>
            <input type="text" name="uname" class="form-control" value="{{ user.uname }}">
        </div>
        <div class="form-group">
            <label>User Email</label>
            <input type="text" name="uemail" class="form-control" value="{{ user.uemail }}">
        </div>
        <div class="form-group">
            <label>User Password</label>
            <input type="text" name="upassword" class="form-control" value="{{ user.upassword }}">
        </div>
        <input type="submit" value="Update" class="btn btn-success">
    </form>
</div>
```

```python
def update(request, id):
    user = User.objects.get(id=id)
    form = UserForm(request.POST, instance=user)
    if form.is_valid():
        form.save()
    return redirect('show')
```

- `edit.html` is **hand-written HTML**, not a rendered `ModelForm` — each `<input>`'s `value="{{ user.uname }}"` manually pre-fills the field with the existing record's current value, and the form's `action` points at `/update/<id>/`, carrying the record's ID forward to the view that actually performs the save.
- `update(request, id)` uses the exact same **`instance=`** mechanism from Lecture 29's `EditUserProfileForm` — fetching the existing row by `id`, then binding the posted data to that specific instance so `form.save()` updates the existing row rather than creating a new one.

> **[Gap-filled] — why `edit.html` isn't just `{{ form }}` again.** Since `edit.html` needs each field manually pre-filled with the *current* record's value (via `value="{{ user.uname }}"`), and the Bootstrap styling needs to be hand-applied per input anyway, writing the fields out directly is simpler here than constructing a bound `UserForm(instance=user)` and looping its fields — either approach works, but the video takes the more explicit, hand-written route for this particular template.

## 6. Wiring up all five views **[From video]**

```python
# urls.py
from django.urls import path
from mysql_crud_app import views

urlpatterns = [
    path('', views.insert),
    path('show/', views.show),
    path('delete/<int:id>/', views.delete),
    path('edit/<int:id>/', views.edit),
    path('update/<int:id>/', views.update),
]
```

Every one of the delete/edit/update URLs uses the `<int:id>` path converter from Lecture 21 to carry a specific record's ID from a link/form action straight into the matching view's `id` parameter — the same dynamic-URL mechanism used consistently since it was first introduced, now applied across an entire small CRUD app rather than a single toy example.

---

## Wrap-up

- **From video:** Bootstrap-styling a `ModelForm` field-by-field (`{{ form.fieldname }}` plus a matching `widget=forms.TextInput(attrs={'class': 'form-control'})` on each field); the insert view (`form.save()` inside a `try`/`except`), including a real, honestly-described debugging story about a silent field-name mismatch; the show view (`.objects.all()` in a Bootstrap-styled table, with plain-link delete/edit actions per row); the delete view (`.filter(id=id).delete()`); the edit/update pair (a hand-written pre-filled form, then `UserForm(request.POST, instance=user)` to save changes to the existing row); wiring all five views together with `<int:id>` path converters.
- **Gap-filled:** the actual lesson behind the video's silent-failure debugging moment (mismatched field names fail silently, not with a traceback); flagging the plain-link delete as a real shortcut worth avoiding in production, contrasted with Lecture 32's safer `DeleteView` pattern; why `edit.html` is hand-written rather than a rendered `ModelForm`.

Double-check: this project's "click a link to delete" pattern is convenient for a learning project but is the wrong choice for anything real — see Lecture 32's `DeleteView` (confirm-then-POST) for the safer, GET-never-deletes pattern to use instead in production code.
