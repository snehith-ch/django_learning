# Lecture 27 — Built-in auth form classes, and building a sign-up form

Source: `transcripts/Django27.txt`
Covers: a tour of the form/model/view classes Django's authentication system ships out of the box, then the first real authentication code of the course — an `authentication_app` with a working sign-up (registration) form, first using a built-in form directly, then extending it with a custom form to collect more fields.

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official Django docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. A tour of the built-in auth classes **[From video]**

Continuing Lecture 26's file-system tour, the video opens `django/contrib/auth/forms.py`, `models.py`, and `views.py` directly in an editor to show what's actually available to use, out of the box:

| File | Notable built-in classes |
|---|---|
| `forms.py` | `UserCreationForm`, `UserChangeForm`, `AuthenticationForm`, `PasswordResetForm`, `SetPasswordForm`, `PasswordChangeForm`, `AdminPasswordChangeForm` |
| `models.py` | `AbstractUser`, `User`, `Permission`, `Group`, `UserManager` |
| `views.py` | `LoginView`, `LogoutView`, `PasswordResetView`, `PasswordResetConfirmView`, and related password-flow views |

The point being made: these are the exact same form classes that power the admin panel's own login and "add user" screens (visible back in Lecture 26's admin walkthrough) — nothing about them is admin-specific, so they can be imported and reused in ordinary application code.

> **[Gap-filled] — reading this as a toolbox, not a checklist to memorize.** None of these class names need to be memorized up front; what matters is knowing the toolbox exists so that, when a form's requirements match one of these ("I need to let someone create an account," "I need to check a username/password pair," "I need to let someone change their password"), the first move is checking whether `django.contrib.auth` already solves it — rather than writing that logic from scratch. This lecture demonstrates exactly that workflow with `UserCreationForm`.

## 2. Setting up the authentication app **[From video]**

```bash
python manage.py startapp authentication_app
```

```python
# settings.py
INSTALLED_APPS = [
    # ...
    'authentication_app',
]
```

Same registration pattern used for every app throughout the course.

## 3. Using a built-in form directly: `UserCreationForm` **[From video]**

The simplest possible sign-up view — no custom form of your own at all, just the built-in form rendered directly:

```python
# authentication_app/views.py
from django.contrib.auth.forms import UserCreationForm
from django.shortcuts import render

def sign_up(request):
    fm = UserCreationForm()
    return render(request, 'sign_up.html', {'form': fm})
```

```html
<!-- sign_up.html -->
<form method="post">
    {% csrf_token %}
    {{ form.as_p }}
    <input type="submit" value="Submit">
</form>
```

Visiting this view renders a working registration form with exactly three fields: **username**, **password**, and **password confirmation** — nothing else, because that's all `UserCreationForm` defines by default. The look and validation rules (password strength requirements, "passwords didn't match," etc.) are visibly identical to what the admin panel's own "add user" screen showed back in Lecture 26 — because it's the literal same form class.

> **[Researched] — `UserCreationForm`'s exact default field set.** Per the [Django docs](https://docs.djangoproject.com/en/stable/topics/auth/default/#django.contrib.auth.forms.UserCreationForm), the form's `Meta.fields` is `("username",)` plus the two password fields it defines itself (`password1`, `password2`) — so "username + password + confirm password" is not a video simplification, it's the actual documented default. Anything beyond that (first name, last name, email, …) requires either a custom form (next section) or manually adding fields to a subclass.

## 4. Extending it: a custom `SignUpForm` for extra fields **[From video]**

To collect more than username/password, the video creates a proper `forms.py` for the app and defines a form that **inherits from** `UserCreationForm`, adding fields via `Meta`:

```python
# authentication_app/forms.py
from django.contrib.auth.models import User
from django.contrib.auth.forms import UserCreationForm

class SignUpForm(UserCreationForm):
    class Meta:
        model = User
        fields = ['username', 'first_name', 'last_name', 'email', 'password1', 'password2']
```

- `SignUpForm(UserCreationForm)` — this is **inheritance**, the same pattern Lecture 20's `EmpForm(forms.ModelForm)` used, except the parent class here is a *pre-built* form (`UserCreationForm`) rather than the generic `forms.ModelForm`. `SignUpForm` starts with everything `UserCreationForm` already does (username/password fields, password validation, `save()` logic that hashes the password correctly) and layers extra fields on top.
- `model = User` — Django's built-in user model (the same `django.contrib.auth.models.User` behind the `auth_user` table from Lecture 26).
- `fields = [...]` — because `first_name`, `last_name`, and `email` are real columns on `User` (confirmed in Lecture 26's `auth_user` table inspection), simply listing them here is enough for Django to generate matching form fields — no manual `first_name = forms.CharField(...)` needed, the same convenience `ModelForm` provided back in Lecture 20.

```python
# authentication_app/views.py
from authentication_app.forms import SignUpForm
from django.contrib import messages
from django.shortcuts import render

def sign_up(request):
    if request.method == 'POST':
        fm = SignUpForm(request.POST)
        if fm.is_valid():
            messages.success(request, 'User registration successfully')
            fm.save()
    else:
        fm = SignUpForm()
    return render(request, 'sign_up.html', {'form': fm})
```

This is the exact same **GET-shows-empty-form / POST-validates-and-saves** shape every form view in this course has followed since Lecture 18–19 — the only thing that's changed is which form class is doing the work.

## 5. Cleaning up the rendered form: removing built-in labels, adding CSRF and a submit button **[From video]**

`{{ form.as_p }}` (used above for simplicity) renders Django's default labels and help text alongside every field — verbose, and not always wanted. The video builds the form manually instead, field by field, to control exactly what's shown:

```html
<!-- sign_up.html -->
<form method="post" novalidate>
    {% csrf_token %}
    {% for field in form %}
        {{ field.label_tag }}
        {{ field }}
        {{ field.errors|striptags }}
        <br><br>
    {% endfor %}
    <input type="submit" value="Submit">
</form>

{% if messages %}
    {% for msg in messages %}
        {{ msg }}
    {% endfor %}
{% endif %}
```

- `{% for field in form %}` — loops over each field in the form object individually (rather than dumping the whole form with `.as_p`), giving fine-grained control over how each one is rendered.
- `field.label_tag` — renders just that field's `<label>`.
- `field` — renders just that field's `<input>` (or equivalent widget).
- `field.errors|striptags` — `field.errors` is that field's list of validation error messages; the built-in `|striptags` template filter strips out the HTML list-item tags Django wraps them in by default, leaving plain error text (the video frames this as "removing the label tags," but what it's actually stripping is the surrounding markup around the *error messages*, not the labels themselves — the labels are rendered separately, on purpose, via `field.label_tag`).
- `novalidate` on the `<form>` tag disables the *browser's* built-in HTML5 validation popups, so Django's own server-side validation messages (rendered via `field.errors`) are what the user actually sees, instead of the browser intercepting submission first.
- The messages-display loop is the same pattern from Lecture 20's messages framework coverage.

> **[Gap-filled] — why loop over `form` instead of using `.as_p`/`.as_table`.** `{{ form.as_p }}` is the fastest way to get a form on a page (used in Lecture 18 for exactly that reason), but it renders every field the same way, with no per-field control. Looping `{% for field in form %}` is what you reach for the moment you need to change how errors are displayed, add per-field CSS classes, or (as here) omit certain default markup — it trades a little more template code for real control over the output.

## 6. What validation actually looks like **[From video]**

Submitting the form with mismatched or weak passwords surfaces `UserCreationForm`'s built-in validation messages directly: "This field is required," "The password fields didn't match," "This password is too similar to your username," "This password is too short," "This password is entirely numeric" — all supplied by Django itself, not written by hand.

> **[Researched] — this is Django's password validator framework, not something specific to this form.** Per the [Django docs](https://docs.djangoproject.com/en/stable/ref/settings/#auth-password-validators), those specific rules (`UserAttributeSimilarityValidator`, `MinimumLengthValidator`, `CommonPasswordValidator`, `NumericPasswordValidator`) are configured in `settings.py` under `AUTH_PASSWORD_VALIDATORS`, present by default in every new Django project. `UserCreationForm` runs whatever validators are configured there — so the same protection applies uniformly across sign-up, password-change, and password-reset flows without each one re-implementing password-strength rules.

## 7. Confirming the save actually worked **[From video]**

After a successful submission, opening `auth_user` in DB Browser for SQLite shows the new row — first/last name, username, and email exactly as entered, and the password stored in its usual hashed (unreadable) form, exactly as Lecture 26 predicted. `fm.save()` on a `UserCreationForm`-derived form handles the password hashing internally — nowhere in this view's code is a password ever handled or stored as plain text.

> **Industry best practice / [Gap-filled] — newly signed-up users are not staff by default.** A user created through this sign-up form has `is_staff = 0` (Lecture 26's flag) — they can use the site as an ordinary authenticated user, but cannot log into `/admin/`. This is exactly the right default: a public-facing registration form should never hand out admin access. Granting `is_staff`/`is_superuser` remains a deliberate action a superuser takes through the admin panel, never something a sign-up flow does automatically.

---

## Wrap-up

- **From video:** the built-in form/model/view classes in `django.contrib.auth` (`UserCreationForm`, `UserChangeForm`, `AuthenticationForm`, password-related forms; `User`, `Permission`, `Group`; login/logout/password views); creating `authentication_app`; using `UserCreationForm` directly (username/password/confirm-password only); extending it into a custom `SignUpForm` with extra `User` fields via `Meta.model`/`fields`; the GET/POST view pattern with `messages.success` + `fm.save()`; manually looping `{% for field in form %}` to control label/error rendering instead of `.as_p`; confirming the saved row (and its hashed password) in `auth_user`.
- **Gap-filled:** framing the built-in classes as "check here first" rather than a memorization list; clarifying exactly what `|striptags` strips (error-message markup, not labels); why looping fields beats `.as_p` once you need control; why new sign-ups correctly default to non-staff.
- **Researched:** `UserCreationForm`'s documented default field set; the `AUTH_PASSWORD_VALIDATORS` setting behind the password-strength messages seen during testing.

Double-check: if a future form's validation messages look unfamiliar, `AUTH_PASSWORD_VALIDATORS` in `settings.py` is the place to look — they're configurable, not hardcoded into the form classes themselves.
