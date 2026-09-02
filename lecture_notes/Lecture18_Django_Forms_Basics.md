# Lecture 18 — Django Forms: building and rendering

Source: `transcripts/Django18.txt`
Covers: why Django forms exist alongside plain HTML forms, creating a `forms.Form` class, the four ways to render one, form field arguments, reordering fields, and the GET/POST + `novalidate` interplay.

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official Django docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. Why a Django form instead of a plain HTML form **[From video]**

- Far less code — a form field is one line of Python instead of a full `<input>` tag with attributes.
- Built-in validation (Lecture 19's topic), instead of relying only on HTML5's `required`/`type` checks.
- A form's fields can mirror a model's fields closely, which becomes fully automatic with `ModelForm` (Lecture 20).

## 2. Creating a form **[From video]**

```python
# myapp/forms.py
from django import forms

class StudentForm(forms.Form):
    name = forms.CharField()
    marks = forms.IntegerField()
```

`forms.py` is a filename convention, not a Django requirement — any name works, but this is what every real project uses. `StudentForm` must inherit from `forms.Form`. Each class attribute (`name`, `marks`) becomes one HTML input field, typed by the Django field class used (`CharField` → text input, `IntegerField` → number-validated text input, and so on).

## 3. Rendering it in a view and template **[From video]**

```python
# views.py
from myapp.forms import StudentForm

def student(request):
    fm = StudentForm()
    return render(request, 'myform.html', {'form': fm})
```

```html
<!-- myform.html -->
<form method="get">
    <table>
        {{ form.as_table }}
    </table>
    <input type="submit" value="Submit">
</form>
```

> **Common pitfall:** A Django form only ever renders its *fields* — never a `<form>` tag and never a submit button. Forgetting to add those manually is a common first mistake; without them the fields display, but there's nothing to actually submit the page.

## 4. Rendering styles **[From video]**

| Syntax | Wraps each field in |
|---|---|
| `{{ form.as_table }}` | Table rows (`<tr>`) — needs a surrounding `<table>` |
| `{{ form.as_ul }}` | List items (`<li>`) — needs a surrounding `<ul>` |
| `{{ form.as_p }}` | Paragraphs (`<p>`) |
| `{{ form.name }}, {{ form.marks }}` | Nothing — full manual control over layout, one field at a time |

## 5. Field arguments **[From video]**

```python
class StudentForm(forms.Form):
    name = forms.CharField(initial='Mohan', required=True)
    address = forms.CharField(disabled=True)
    message = forms.CharField(widget=forms.Textarea, help_text="Max 100 characters")
    password = forms.CharField(widget=forms.PasswordInput)
```

- `initial` — a default value pre-filled in the field (still editable, unlike a placeholder).
- `required=False` — allows the field to be submitted empty (all fields are required by default).
- `disabled=True` — renders the field but the user cannot edit its value at all.
- `widget=forms.Textarea` — renders as a multi-line `<textarea>` instead of a single-line input.
- `widget=forms.PasswordInput` — masks typed characters, same as HTML's `type="password"`.
- `help_text` — extra guidance text shown next to the field.
- `label_suffix` — the character after a field's label (default is `:`).

## 6. Reordering fields **[From video]**

```python
fm = StudentForm()
fm.order_fields(field_order=['address', 'marks', 'name'])
```

`order_fields()` is called on a form *instance* in the view (after creating it, before rendering) — not in `forms.py` — and only affects display order, not anything about the field definitions themselves.

## 7. GET forms, and turning off HTML5 validation **[From video]**

A Django form works with either `method="get"` or `method="post"` on the surrounding `<form>` — the same GET/POST rules from Lecture 13 apply (GET puts values in the URL; POST needs `{% csrf_token %}`).

By default, the browser's own HTML5 validation (from each field's `required` attribute) blocks submission before Django ever sees the request. To test Django's own validation messages (Lecture 19) instead of the browser's, add `novalidate` to the `<form>` tag:

```html
<form method="post" novalidate>
```

---

## Wrap-up

- **From video:** the case for Django forms, creating a `forms.Form`, all four rendering styles, the full set of field arguments demonstrated (`initial`, `required`, `disabled`, `widget`, `help_text`, `label_suffix`), `order_fields()`, and the `novalidate` attribute.
- **Gap-filled:** none needed beyond what's already noted as a common pitfall (missing `<form>` tag/submit button).
- **Researched:** none needed this lecture.

Double-check: nothing flagged as incorrect this lecture — it's a thorough, accurate tour of form construction and rendering options.
