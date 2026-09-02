# Lecture 20 — Update, delete, the messages framework, and ModelForm

Source: `transcripts/Django20.txt`
Covers: rounding out CRUD with update and delete, Django's messages framework for user feedback, and `ModelForm` — the shortcut that removes the field duplication between `models.py` and `forms.py`.

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official Django docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. The messages framework **[From video]**

A view that saves data and re-renders the same page has no built-in way to say "that worked" — the [messages framework](https://docs.djangoproject.com/en/stable/ref/contrib/messages/) is Django's answer: store a short message tied to the current request, then display it once, on the very next page render.

```python
from django.contrib import messages

def student_view(request):
    if request.method == 'POST':
        fm = StudentForm(request.POST)
        if fm.is_valid():
            # ...save as before...
            messages.success(request, "Record inserted successfully")
        else:
            messages.info(request, "Please enter valid data")
    else:
        fm = StudentForm()
    return render(request, 'student_form.html', {'form': fm})
```

```html
<!-- inside student_form.html -->
{% if messages %}
    {% for message in messages %}
        {{ message }}
    {% endfor %}
{% endif %}
```

`messages.success(request, "...")` and `messages.info(request, "...")` queue text against the current request; the template must explicitly loop over the built-in `messages` variable to display it — nothing shows up automatically just because a message was added.

> **[Gap-filled] — a debugging story worth knowing.** The video spends real time chasing a bug here: messages weren't appearing because the display loop simply hadn't been added to the template yet (they were reaching the admin panel's own message area, not the custom form page). If your messages seem to "disappear," check the obvious thing first — is `{% if messages %}{% for message in messages %}...{% endfor %}{% endif %}` actually present in the template you're rendering back to?

## 2. Updating a record **[From video]**

```python
s = Student.objects.get(pk=1)
s.name = nm
s.email = ml
s.address = ad
s.password = pw
s.save()
```

Same `.save()` call as an insert — the difference is entirely in *which instance* you're saving. Fetch an existing row by `pk`, reassign its attributes, then save.

> **[Researched] — how Django decides INSERT vs. UPDATE.** Per the [Django docs](https://docs.djangoproject.com/en/stable/ref/models/instances/#how-django-knows-to-update-vs-insert), `.save()` issues an `UPDATE` if the model instance's primary key already has a matching row in the database, and an `INSERT` otherwise. That's the entire mechanism — the video shows the "get, modify, save" pattern working without stating explicitly why the same method call does two different SQL operations depending on context.

## 3. Deleting a record **[From video]**

```python
s = Student.objects.get(pk=4)
s.delete()
```

`.delete()` on a fetched instance removes that row. As with update, you first retrieve the exact record by its primary key.

> **Industry best practice:** The video wires insert/update/delete through the same single view and button for demonstration speed, with two of the three code paths commented out at any given time. In real code, give each operation (or at least update/delete) its own view and URL — one view silently doing three different destructive things depending on which lines happen to be commented out is exactly the kind of thing that causes production incidents.

## 4. ModelForm — no more duplicated fields **[From video]**

A `ModelForm` generates its fields directly from a model, so they're defined in exactly one place.

```python
# forms.py
from django import forms
from db_app.models import Employee

class EmpForm(forms.ModelForm):
    class Meta:
        model = Employee
        fields = '__all__'   # or: fields = ['name', 'address']
                              # or: exclude = ['address']
```

The nested `Meta` class is how you configure a `ModelForm`: `model` says which model to base the form on, and `fields`/`exclude` control which of that model's fields actually appear. Compare to Lecture 18/19's `StudentForm`, which repeated every field name in both `models.py` and `forms.py` — a `ModelForm` only states the field list once, on the model.

## 5. Customizing a ModelForm's rendering **[From video]**

```python
class EmpForm(forms.ModelForm):
    class Meta:
        model = Employee
        fields = '__all__'
        labels = {'name': 'Enter Name', 'password': 'Enter Password'}
        error_messages = {
            'name': {'required': 'Name is mandatory'},
            'password': {'required': 'Password must be entered'},
        }
        widgets = {'password': forms.PasswordInput}
```

These go inside the same `Meta` class: `labels` overrides a field's displayed label text, `error_messages` overrides Django's default validation-error wording per field, and `widgets` swaps a field's HTML input type (e.g. masking a password) — the ModelForm equivalents of the plain-form field arguments from Lecture 18.

> **Industry best practice:** Reach for `ModelForm` whenever a form exists to create/edit a specific model's records — which is most forms in a typical Django app. Save a plain `forms.Form` for cases with no matching model at all (a contact form, a one-off search box).

---

## Wrap-up

- **From video:** the messages framework (`messages.success`/`messages.info` + the template display loop), the update pattern (`get` → reassign → `save`), the delete pattern (`get` → `delete`), and `ModelForm` with `Meta.model`/`fields`/`exclude`/`labels`/`error_messages`/`widgets`.
- **Gap-filled:** the "messages seem to vanish" debugging lesson from the video's own mistake.
- **Researched:** the actual mechanism behind `.save()` choosing INSERT vs. UPDATE (based on whether the primary key already exists in the database).

Double-check: the INSERT-vs-UPDATE mechanism is worth confirming against the docs link above if you're ever unsure why a `.save()` call did what it did.
