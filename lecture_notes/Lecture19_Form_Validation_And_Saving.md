# Lecture 19 — Validating and saving form data

Source: `transcripts/Django19.txt`
Covers: `is_valid()`/`cleaned_data`, cross-field validation with `clean()`, single-field validation with `clean_<fieldname>()`, built-in and custom validators, and the first full example of writing form data into the database.

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official Django docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. `is_valid()` and `cleaned_data` **[From video]**

```python
def student(request):
    if request.method == 'POST':
        fm = StudentForm(request.POST)
        if fm.is_valid():
            nm = fm.cleaned_data['name']
            mk = fm.cleaned_data['marks']
            print("Student name is", nm)
        else:
            print("Form data is not valid")
    else:
        fm = StudentForm()
    return render(request, 'myform.html', {'form': fm})
```

- `StudentForm(request.POST)` — binds the submitted data to a form instance (an unbound, empty form is created with no arguments, as in the `else` branch).
- `fm.is_valid()` — runs every field's validation and returns `True`/`False`.
- `fm.cleaned_data` — a dictionary, populated *only if the form is valid*, keyed by each field's name, holding the validated, type-converted value (e.g. `marks` comes back as an actual `int`, not text).

## 2. Cross-field validation with `clean()` **[From video]**

To validate two fields *together* — like checking a password and a re-typed confirmation match — override `clean()` on the form:

```python
class StudentForm(forms.Form):
    name = forms.CharField()
    password = forms.CharField(widget=forms.PasswordInput)
    rpassword = forms.CharField(widget=forms.PasswordInput)

    def clean(self):
        super().clean()
        pwd = self.cleaned_data.get('password')
        rpwd = self.cleaned_data.get('rpassword')
        if pwd != rpwd:
            raise forms.ValidationError("Password mismatch")
```

`super().clean()` runs Django's normal per-field validation first; only after that do you compare the two already-cleaned values and raise `forms.ValidationError` if they don't match. This error then causes `is_valid()` to return `False` for the whole form.

## 3. Single-field validation with `clean_<fieldname>()` **[From video]**

```python
def clean_name(self):
    name = self.cleaned_data['name']
    if len(name) < 6:
        raise forms.ValidationError("Enter name with 6 or more characters")
    return name
```

Naming the method `clean_<fieldname>` (matching a real field name exactly) tells Django to call it automatically while validating that one field. Unlike `clean()`, this method **must return the value** — that return value becomes the field's final cleaned value.

## 4. Built-in and custom validators **[From video]**

```python
from django.core import validators

def starts_with_s(value):
    if value[0] != 'S':
        raise forms.ValidationError("Name should start with S only")

class StudentForm(forms.Form):
    name = forms.CharField(validators=[validators.MaxLengthValidator(6), starts_with_s])
```

The `validators` argument takes a *list* of callables. Django ships many ready-made ones in `django.core.validators` (like `MaxLengthValidator`); you can also pass your own plain function, which just needs to accept the value and raise `forms.ValidationError` when it's invalid.

## 5. Actually saving to the database **[From video]**

Combining everything so far — a model, a form with matching fields, and a view that turns valid `cleaned_data` into a saved row:

```python
# views.py
from db_app.models import Student
from db_app.forms import StudentForm

def student_view(request):
    if request.method == 'POST':
        fm = StudentForm(request.POST)
        if fm.is_valid():
            nm = fm.cleaned_data['name']
            ml = fm.cleaned_data['email']
            ad = fm.cleaned_data['address']
            pw = fm.cleaned_data['password']
            s = Student(name=nm, email=ml, address=ad, password=pw)
            s.save()
    else:
        fm = StudentForm()
    return render(request, 'student_form.html', {'form': fm})
```

`Student(...)` builds a new, unsaved model instance from the cleaned form data; `.save()` is what actually issues the `INSERT` and writes the row.

> **[Gap-filled]** This "copy each cleaned field into a new model instance by hand" pattern works, but notice it re-lists every field a second time — first in `forms.py`, then again here in the view. `ModelForm`, covered in Lecture 20, removes this duplication entirely.

---

## Wrap-up

- **From video:** `is_valid()`/`cleaned_data`, cross-field validation via `clean()`, single-field validation via `clean_<fieldname>()`, built-in validators (`django.core.validators`) and custom validator functions, and a full working create-flow: new app, model, form, and a view that saves to the database.
- **Gap-filled:** flagging that this manual field-copying pattern is exactly what `ModelForm` (next lecture) exists to eliminate.
- **Researched:** none needed this lecture.

Double-check: nothing flagged as incorrect — this is a thorough, accurate walkthrough of Django's form validation and save flow.
