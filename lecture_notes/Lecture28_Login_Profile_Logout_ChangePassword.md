# Lecture 28 — Login, profile, logout, and changing a password two ways

Source: `transcripts/Django28.txt`
Covers: rounding out the authentication flow started last lecture — a login view built on `AuthenticationForm`, a profile page gated behind `is_authenticated`, a logout view, and two different built-in forms for changing a password (with the old password, and without it).

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official Django docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. The login view **[From video]**

```python
# authentication_app/views.py
from django.contrib.auth.forms import AuthenticationForm
from django.contrib.auth import authenticate, login
from django.http import HttpResponseRedirect
from django.contrib import messages

def user_login(request):
    if request.method == 'POST':
        fm = AuthenticationForm(request=request, data=request.POST)
        if fm.is_valid():
            uname = fm.cleaned_data['username']
            pwd = fm.cleaned_data['password']
            user = authenticate(username=uname, password=pwd)
            if user is not None:
                login(request, user)
                return HttpResponseRedirect('profile/')
            else:
                messages.info(request, 'Invalid username or password')
        else:
            messages.info(request, 'Invalid username or password')
    else:
        fm = AuthenticationForm()
    return render(request, 'login.html', {'form': fm})
```

Walking through each new piece:

- **`AuthenticationForm(request=request, data=request.POST)`** — unlike `SignUpForm` from last lecture, `AuthenticationForm` takes the current `request` as well as the posted data; it needs the request internally to check things like whether the account is active.
- **`fm.cleaned_data['username']` / `['password']`** — `cleaned_data` is the dictionary of a form's *validated* field values (first introduced with `.save()`-based forms back in Lecture 19) — reading the username/password back out of the form after `is_valid()` confirms they're present and well-formed.
- **`authenticate(username=..., password=...)`** — a function from `django.contrib.auth` that checks the given username/password pair against the database and returns the matching `User` object if correct, or `None` if not. This is the actual "are these credentials real" check — `AuthenticationForm.is_valid()` only confirms the *fields* are filled in correctly, not that the account exists.
- **`login(request, user)`** — another `django.contrib.auth` function; this is what actually establishes the logged-in session for this request (creating the session data and session-ID cookie described back in Lecture 23), given a `User` object already confirmed valid by `authenticate()`.
- **`HttpResponseRedirect('profile/')`** — on success, sends the browser to the profile page.

> **[Gap-filled] — why `authenticate()` and `login()` are two separate steps.** It would be possible to imagine one function doing both, but Django deliberately splits "verify these are correct credentials" (`authenticate()`) from "actually log this already-verified user in" (`login()`). This split matters because it lets you `authenticate()` a user for reasons other than an interactive login (e.g. checking credentials in a script or API), without every such check side-effecting a live session — logging in is an explicit, separate action.

> **Industry best practice / [Researched] — never reveal *which* field was wrong.** Per the [Django docs on `AuthenticationForm`](https://docs.djangoproject.com/en/stable/topics/auth/default/#django.contrib.auth.forms.AuthenticationForm), the deliberately generic "Invalid username or password" (rather than "that username doesn't exist" or "wrong password") is a real security consideration, not just simpler wording — a more specific message tells an attacker which half of a guessed credential pair was correct, making brute-force attacks measurably easier. The video's message (used identically for both the wrong-username and wrong-password cases) follows this convention correctly, whether or not it was chosen for that reason.

## 2. The profile view **[From video]**

```python
def profile(request):
    if request.user.is_authenticated:
        return render(request, 'profile.html', {'user': request.user})
    else:
        return HttpResponseRedirect('/login/')
```

- **`request.user`** — every request carries a `user` attribute, automatically set by Django's authentication middleware. If nobody is logged in, `request.user` is an `AnonymousUser` object (not `None`) rather than raising an error.
- **`request.user.is_authenticated`** — `True` for a real, logged-in user; `False` for `AnonymousUser`. This is the gate: only a genuinely logged-in visitor sees the profile page — anyone else is redirected straight back to `/login/`.

> **[Researched] — `AnonymousUser`, and why this check works without a `None`-check.** Per the [Django docs](https://docs.djangoproject.com/en/stable/ref/contrib/auth/#django.contrib.auth.models.AnonymousUser), `AnonymousUser.is_authenticated` is hardcoded to always return `False` (and a real `User`'s always returns `True`), specifically so template and view code can write `request.user.is_authenticated` unconditionally — no need to first check "is there even a user object" before checking whether they're logged in, because there's always *some* user object, real or anonymous.

## 3. The logout view **[From video]**

```python
from django.contrib.auth import logout

def user_logout(request):
    logout(request)
    return HttpResponseRedirect('/login/')
```

`logout(request)` ends the current session (internally, this is built on the same `flush()`-style session invalidation covered in Lecture 25) and redirects back to the login page — from which the now-anonymous visitor cannot reach `/profile/` again without logging back in, per the check in Section 2.

## 4. Wiring it all together **[From video]**

```python
# authentication_app/urls.py
from django.urls import path
from authentication_app import views

urlpatterns = [
    path('sign_up/', views.sign_up, name='sign_up'),
    path('login/', views.user_login, name='login'),
    path('profile/', views.profile, name='profile'),
    path('logout/', views.user_logout, name='logout'),
]
```

Every one of these URLs is **named** (Lecture 21) specifically so the templates can cross-link between them without hardcoding paths:

```html
<!-- sign_up.html, near the bottom -->
<a href="{% url 'login' %}">Click here to login</a>

<!-- login.html, near the bottom -->
<a href="{% url 'sign_up' %}">Click here to sign up</a>

<!-- profile.html -->
<a href="{% url 'logout' %}">Click here to logout</a>
```

> **[Gap-filled] — a debugging story worth keeping.** The video hits two real bugs building this cross-linking: first, a `NoReverseMatch`-style error from a template referencing a URL `name` (`'login'`) that hadn't actually been given that `name=` yet in `urls.py`; second, later, a logout `ValueError` traced to the logout *view* being registered under the wrong function name in `urls.py` (pointing at a view that didn't match what was actually defined). Both are the same category of mistake: a `{% url 'x' %}` reference and its matching `name='x'` in `urls.py` have to agree **exactly** — a typo or mismatch on either side breaks the link, and Django's error message names the missing/wrong URL name directly, which is the first place to look.

## 5. Changing a password *with* the old password: `PasswordChangeForm` **[From video]**

```python
from django.contrib.auth.forms import PasswordChangeForm
from django.contrib.auth import update_session_auth_hash

def user_password_change(request):
    if request.method == 'POST':
        fm = PasswordChangeForm(user=request.user, data=request.POST)
        if fm.is_valid():
            fm.save()
            update_session_auth_hash(request, fm.user)
            messages.success(request, 'Password change successfully')
            return HttpResponseRedirect('/profile/')
        else:
            messages.info(request, 'Please enter valid details')
    else:
        fm = PasswordChangeForm(user=request.user)
    return render(request, 'change_password.html', {'form': fm})
```

- `PasswordChangeForm(user=request.user, data=request.POST)` — this form is built *for* a specific already-known user (`request.user`), unlike `AuthenticationForm` which has to figure out *which* user from the submitted username. It renders three fields: old password, new password, new password confirmation.
- `fm.save()` validates the old password matches, then updates it to the new one.
- **`update_session_auth_hash(request, fm.user)`** — this is the critical, easy-to-miss step the video spends real time debugging (an `AttributeError: ... has no attribute 'check_password'` traced back to a missing import/indentation issue around this call). Without it, changing the password would immediately invalidate the user's *current* session, logging them out mid-request as an unintended side effect.

> **[Researched] — why `update_session_auth_hash` is required.** Per the [Django docs](https://docs.djangoproject.com/en/stable/topics/auth/default/#session-invalidation-on-password-change), Django deliberately ties a logged-in session to a hash of the user's password, specifically so that changing a password from *one* browser/device automatically invalidates sessions on *other* devices (a real security feature — if a password change happens because an account was compromised, old sessions shouldn't stay valid). The side effect is that the *current* request's own session would also be invalidated by that same mechanism unless `update_session_auth_hash(request, user)` explicitly updates the current session's hash to match the just-changed password, keeping the user logged in through their own password change.

## 6. Changing a password *without* the old password: `SetPasswordForm` **[From video]**

```python
from django.contrib.auth.forms import SetPasswordForm

def user_change_password_1(request):
    if request.method == 'POST':
        fm = SetPasswordForm(user=request.user, data=request.POST)
        if fm.is_valid():
            fm.save()
            update_session_auth_hash(request, fm.user)
            messages.success(request, 'Password change successfully')
            return HttpResponseRedirect('/profile/')
        else:
            messages.info(request, 'Please enter valid details')
    else:
        fm = SetPasswordForm(user=request.user)
    return render(request, 'change_password1.html', {'form': fm})
```

Structurally identical to `PasswordChangeForm` above — same `user=`/`data=` construction, same `update_session_auth_hash` requirement — except `SetPasswordForm` only asks for the **new password** and its confirmation, with no old-password field at all.

| Form | Fields shown | When to use |
|---|---|---|
| `PasswordChangeForm` | Old password, new password, confirm new password | The user is logged in and knows their current password — the normal "change my password" flow. |
| `SetPasswordForm` | New password, confirm new password | The user doesn't need to (or can't) supply the old password — e.g. as the final step of a password-reset flow after they've already proven identity another way (like a reset-link emailed to them). |

> **Industry best practice:** Because `SetPasswordForm` skips the old-password check, it should only ever be reachable from a flow that has *already* verified the user is who they claim to be through some other means (an emailed reset link, an admin action) — never exposed as a directly-reachable page for an already-authenticated user to bypass entering their current password, or it defeats the purpose of having a "current password" check at all elsewhere in the app.

Both password-change templates (`change_password.html`, `change_password1.html`) follow the same field-loop-plus-CSRF-plus-messages pattern established for `sign_up.html` in Lecture 27, with links back to `profile/` and `logout/`.

---

## Wrap-up

- **From video:** the login view (`AuthenticationForm`, `authenticate()`, `login()`, redirect on success); the profile view gated on `request.user.is_authenticated`; the logout view (`logout()`); named-URL cross-linking between sign-up/login/profile/logout; `PasswordChangeForm` (with old password) and `SetPasswordForm` (without it), both requiring `update_session_auth_hash()` after `.save()`.
- **Gap-filled:** why `authenticate()`/`login()` are deliberately separate calls; the `NoReverseMatch`/wrong-view-name debugging story as a general "names must match exactly" lesson.
- **Researched:** why a generic "invalid username or password" message is a real security practice, not just simpler code; the mechanics and purpose of `AnonymousUser.is_authenticated`; exactly why `update_session_auth_hash` exists and what breaks without it; when `SetPasswordForm` is (and isn't) appropriate to expose.

Double-check: `update_session_auth_hash()` is the single easiest step to forget in this lecture (the video itself loses real time to it) — any password-change view that logs the user out immediately after a successful change is almost certainly missing this call.
