# REST API Session 13 — Authentication & Permissions: Basic Authentication

Source: `transcripts/restapi/REST API-13.txt`
Covers: the conceptual difference between **authentication** ("who is this user?") and **permissions**
("what is this user allowed to do?") in Django REST Framework, DRF's built-in `BasicAuthentication`
class, the core permission classes (`AllowAny`, `IsAuthenticated`, `IsAdminUser`), and the two places
those classes can be configured — per-view (`authentication_classes` / `permission_classes` attributes)
or project-wide (`DEFAULT_AUTHENTICATION_CLASSES` / `DEFAULT_PERMISSION_CLASSES` in `settings.py`).

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official DRF/Django docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. Recap and today's topic **[From video]**

> So in the last session, we started with authentication and permissions. But we were used with basic
> authentication, but that I'll discuss now today... today topic is authentication and permissions in
> the [Django] rest[] framework.

The previous session (REST API Session 12, ViewSets) had already touched authentication in passing while
demonstrating `ModelViewSet`; this lecture is the dedicated, from-scratch explanation of **how DRF's
authentication and permission system actually works**, followed by a hands-on build using DRF's
**basic authentication** scheme.

## 2. Authentication vs. permissions — the core distinction **[From video]**

> Generally authentication is the process of checking the user credentials are correct or not — that
> is username and password... Authentication always runs at the very start of the view, before the
> permission checks occur, and before any other code is allowed to proceed. So authentication by itself
> won't allow or disallow any incoming request. It simply identifies the credentials that request was
> made with.

> Permissions determines whether a request should be granted or denied access... Permission checks are
> always run at the very starting of the view before any other code is allowed to proceed. Permission
> checks typically use the authentication information in the request — `request.user` [and]
> `request.auth` properties — [to] determine if the incoming request should be permitted.

Broken into plain language:

- <dfn>Authentication</dfn> answers **"who is making this request?"** It checks the credentials sent
  with the request (e.g. a username and password) and, if they're valid, attaches the matching user to
  the request as `request.user` (and any extra token/credential info as `request.auth`). Authentication
  on its own never blocks a request — an anonymous, unauthenticated request is still allowed to reach
  the permission-checking stage; it's just represented as `request.user` being Django's `AnonymousUser`.
- <dfn>Permissions</dfn> answers **"now that we know who this is (or that we don't), should this
  specific request be allowed to proceed?"** Permission checks run right after authentication, using
  whatever `request.user`/`request.auth` authentication just populated, and they are what actually
  grants or denies the request (returning an HTTP `403 Forbidden`, or `401 Unauthorized` if no one is
  authenticated at all, when a permission check fails).

> **[Gap-filled] — why the order matters.** Authentication has to run before permissions because
> permissions almost always *need* to know who the user is before they can decide anything. A check
> like "only admins can delete this" (`IsAdminUser`) is meaningless until authentication has already
> figured out whether there's a logged-in admin behind the request at all. This is also why DRF
> processes them as two separate, sequential steps rather than one combined check — it keeps "identify
> the caller" and "decide if the caller may do this" as cleanly separate responsibilities, which is a
> reusable pattern outside DRF too (e.g. Django's own `django.contrib.auth` middleware identifies the
> user on every request; individual views then separately decide who's allowed in, as seen back in
> Lecture 28's `is_authenticated` checks).

## 3. DRF's authentication schemes **[From video / Researched]**

> There are different types of authentication — the basic authentication, token authentication, session
> authentication, remote user authentication, and custom authentication.

The video names five authentication approaches DRF supports (the transcript is heavily garbled here —
"connect indications" almost certainly means "token authentication," reconstructed from DRF's actual
documented list rather than quoted directly):

| Scheme | What it means |
|---|---|
| `BasicAuthentication` | Username & password sent (base64-encoded) on every request's `Authorization` header — this lecture's focus. |
| `TokenAuthentication` | A single opaque token string, issued once, sent on every request instead of a username/password — covered in REST API Sessions 15–16. |
| `SessionAuthentication` | Uses Django's own session/cookie mechanism (the same sessions from Lecture 23–24) — mainly useful when the API and a Django-rendered site share a browser session. |
| `RemoteUserAuthentication` | Delegates identity to a value already set by the web server (e.g. an upstream SSO/proxy setting `REQUEST.REMOTE_USER`) — a niche, deployment-specific scheme. |
| Custom authentication | DRF lets you write your own class implementing an `authenticate()` method for schemes it doesn't ship with — covered in REST API Session 17. |

> **[Researched]** Per the [DRF authentication docs](https://www.django-rest-framework.org/api-guide/authentication/),
> authentication in DRF always happens in two parts: an `authentication_classes` list is tried in order,
> and the *first* scheme that returns a successful `(user, auth)` pair wins; if none of them successfully
> authenticate the request, `request.user` is set to Django's `AnonymousUser` and `request.auth` to
> `None` — it does **not** immediately fail the request. Whether that unauthenticated request is then
> allowed to proceed is entirely up to the **permission** classes, which is exactly the "authentication
> doesn't allow/deny, it only identifies" point made in the transcript above.

## 4. DRF's permission classes **[From video / Researched]**

> Rest framework will provide a list of permission classes — `AllowAny` means anybody can [access].
> `IsAuthenticated` means only authenticated users can access. `IsAdminUser` means only administrative
> users can access the API. `IsAuthenticatedOrReadOnly` means authenticated persons can [write], others
> get read-only permission.

| Class | Meaning |
|---|---|
| `AllowAny` | No restriction at all — every request, authenticated or not, is allowed. |
| `IsAuthenticated` | Only requests from a **logged-in** user (any logged-in user — staff or not) are allowed; anonymous requests are rejected. |
| `IsAdminUser` | Only requests from a user whose `is_staff` flag is `True` are allowed — being merely logged in is not enough. |
| `IsAuthenticatedOrReadOnly` | Anyone can perform safe/read-only requests (`GET`, `HEAD`, `OPTIONS`); only authenticated users can perform write requests (`POST`, `PUT`, `PATCH`, `DELETE`). |

The video builds and tests the first three of these directly; `IsAuthenticatedOrReadOnly` is named but
not demonstrated in this session.

> **[Researched] — the rest of DRF's built-in permission classes.** Per the
> [DRF permissions docs](https://www.django-rest-framework.org/api-guide/permissions/), DRF also ships
> `DjangoModelPermissions` (ties access to a user's Django model-level `add`/`change`/`delete` permissions
> — the same permissions framework referenced in Lecture 42's MiniBlog notes), `DjangoModelPermissionsOrAnonReadOnly`,
> and `DjangoObjectPermissions` (per-object permissions, requires a third-party backend like
> `django-guardian`). These aren't covered in this lecture, but exist for projects that need finer-grained
> control than "logged in" vs. "staff."

> **[Gap-filled] — where do `request.user.is_staff` and `is_authenticated` come from?** Both are plain
> attributes on Django's own `User` model (from `django.contrib.auth`, first introduced in Lecture 26).
> `is_authenticated` is `True` for any real, logged-in `User` object and `False` for `AnonymousUser`.
> `is_staff` is a separate boolean a superuser can toggle per-account in the Django admin ("staff status")
> — it's what lets a user into `/admin/` and is exactly the flag `IsAdminUser` checks. Being `is_staff`
> is a lighter privilege than `is_superuser`: a staff user can be admitted to the admin site and, per
> this lecture, past `IsAdminUser`, without being a full superuser with every permission.

## 5. Setting up the demo app: model, admin, migrations **[From video]**

The video builds a fresh demo app to test authentication against, rather than reusing the earlier
`ViewSet` app from REST API Session 12.

```bash
python manage.py startapp restapp9
```

The new app is added to `INSTALLED_APPS` in `settings.py` (the standard step from every earlier
app-creation lecture, e.g. Lecture 7).

A single model, `Manager`, is defined — the transcript garbles the field list ("name address mail age...
Carfield, Carfield, Carfield, integer field") but the intent is clear enough to reconstruct:

```python
# restapp9/models.py
from django.db import models

class Manager(models.Model):
    name = models.CharField(max_length=100)     # manager's name
    address = models.CharField(max_length=200)  # postal/contact address
    email = models.CharField(max_length=100)    # email address
    age = models.IntegerField()                 # age, a whole number

    def __str__(self):
        return self.name
```

> **[Gap-filled]** The transcript never states exact `max_length` values or confirms `email` uses
> `CharField` vs. Django's dedicated `EmailField` (which adds automatic email-format validation) — the
> field values above are a reasonable, standard reconstruction, not a verbatim quote. In a real project,
> `models.EmailField()` would be the better choice for the email column since it validates the format
> for free.

The model is registered in the admin so records can be added/inspected through `/admin/`:

```python
# restapp9/admin.py
from django.contrib import admin
from restapp9.models import Manager

admin.site.register(Manager)
```

Then the usual two-step migration and a superuser account (reusing the `createsuperuser` command from
Lecture 8) to be able to log in later:

```bash
python manage.py makemigrations
python manage.py migrate
python manage.py createsuperuser
python manage.py runserver
```

A few Manager records are added through the Django admin so there's real data to query through the API.

## 6. Serializer and ViewSet **[From video / Gap-filled]**

The video refers to a serializer already existing for the app ("import Manager serializer... everything
is there") without dictating it on screen. Following the `ModelSerializer` pattern established in
earlier DRF lectures (e.g. REST API Session 2), it's reconstructed here rather than invented from nothing:

```python
# restapp9/serializers.py  [Gap-filled — pattern from earlier lectures, not dictated on screen]
from rest_framework import serializers
from .models import Manager

class ManagerSerializer(serializers.ModelSerializer):
    class Meta:
        model = Manager
        fields = '__all__'   # serialize every field on Manager
```

The view itself is a `ModelViewSet` (the all-in-one CRUD class covered in REST API Session 12) — first written
with **no** authentication or permission settings at all:

```python
# restapp9/views.py
from rest_framework import viewsets
from restapp9.models import Manager
from restapp9.serializers import ManagerSerializer

class ManagerModelViewSet(viewsets.ModelViewSet):
    queryset = Manager.objects.all()
    serializer_class = ManagerSerializer
    # no authentication_classes / permission_classes yet
```

## 7. Wiring up the router and URLs **[From video]**

> I'm importing... rest_app9 import views... rest_framework.routers import DefaultRouter and I need to
> create a router object here... register your view set with the router.

```python
# project-level urls.py
from django.urls import path, include
from restapp9 import views
from rest_framework.routers import DefaultRouter

router = DefaultRouter()
router.register('manager', views.ManagerModelViewSet, basename='manager')

urlpatterns = [
    path('', include(router.urls)),
]
```

`DefaultRouter` is a DRF class that, given a `ViewSet`, automatically generates the full set of standard
REST URLs for it (list, create, retrieve, update, delete) — the same automatic-URL benefit of
`ViewSet`s discussed conceptually in REST API Session 12. `basename='manager'` gives DRF a prefix to build
internal URL names from (`manager-list`, `manager-detail`, etc.) since there's no `queryset` name it
could otherwise infer cleanly.

## 8. First run: completely open API **[From video]**

> Right now we don't have any authentication and everything directly it will run... there is no
> permission is required. Anything you can able to post the data, read the data, all the data is
> available here.

With no `authentication_classes` or `permission_classes` set on the view (and none set globally yet),
the API is **wide open**: the browsable API loads, and any visitor — logged in or not — can list,
create, retrieve, update, and delete `Manager` records with no login prompt at all.

> **[Gap-filled] — why is it open by default, not locked by default?** This is DRF's own project-wide
> default, not something this code turned on deliberately. Per the
> [DRF settings docs](https://www.django-rest-framework.org/api-guide/settings/), if a project never
> configures `DEFAULT_PERMISSION_CLASSES`, DRF falls back to `AllowAny` for every view. This is a
> deliberately permissive default so a brand-new DRF project "just works" while you're experimenting —
> but it's also exactly the trap the next section's best-practice note is about: it's easy to forget to
> lock an API down before shipping it.

## 9. Adding authentication and permissions to the view **[From video]**

> To implement authentication system... `from rest_framework.authentication import BasicAuthentication`...
> `from rest_framework.permissions import IsAuthenticated, AllowAny, IsAdminUser`.

```python
from rest_framework.authentication import BasicAuthentication
from rest_framework.permissions import AllowAny, IsAuthenticated, IsAdminUser
```

These are added to the `ViewSet` as two new class attributes:

```python
class ManagerModelViewSet(viewsets.ModelViewSet):
    queryset = Manager.objects.all()
    serializer_class = ManagerSerializer
    authentication_classes = [BasicAuthentication]
    permission_classes = [AllowAny]   # changed across the lecture — see below
```

`authentication_classes` is a list of authentication schemes DRF should try, in order, to identify the
caller. `permission_classes` is a list of checks that must **all** pass for the request to be allowed
(DRF requires every listed permission class to grant access — it's an AND, not an OR).

The video then walks through the *same* view with three different `permission_classes` values, testing
each one live in the browsable API against two accounts: **Mohan** (a superuser, `is_staff=True`) and
**Durga** (a regular account with a valid username/password but `is_staff=False`, initially).

### 9a. `permission_classes = [AllowAny]` **[From video]**

> Allow any means anybody can access... you can post the data here also... let's post the data, that's
> also possible.

With `BasicAuthentication` configured but `AllowAny` as the permission, the behavior looks identical to
Section 8 — no login prompt, full CRUD access for anyone. This demonstrates that **having an
authentication scheme configured is not the same as requiring it** — `BasicAuthentication` here is
still available for anyone who *does* send credentials, but `AllowAny` means it's not *required*.

### 9b. `permission_classes = [IsAuthenticated]` **[From video]**

> Is authenticated means what — allow any, not now, this time I'm using is authenticated. Is
> authenticated users means... those who are having the username and password.

Switching to `IsAuthenticated` and restarting the server, an unauthenticated visitor now hits a browser
login prompt:

> Authentication is required... authorization requested by `http://127.0.0.1:8000`.

This native browser dialog (not DRF's own styled login page) is a direct side effect of
`BasicAuthentication` — see the researched note in Section 14 for why. Logging in as **Mohan**
(superuser) works immediately. The video then also logs in as **Durga** — a regular, *non-staff*
account — and that **also** succeeds:

> That user also having username and password, but he don't have any admin staff status account, [and]
> even though we can able to access the API... is authenticated, no matter the user is admin user or
> normal user — only authenticated is enough.

This is the key teaching point of this sub-section: `IsAuthenticated` only checks
"is there a valid, logged-in user at all?" — it does **not** distinguish between regular users and staff
users. Any account with correct credentials passes.

### 9c. `permission_classes = [IsAdminUser]` **[From video]**

> Admin user means those who are having staff status account — through that person only can access.

Switching again, this time to `IsAdminUser`, and testing the same two accounts:

- **Mohan** (superuser, `is_staff=True`) — succeeds, full CRUD access.
- **Durga** (`is_staff=False`) — logs in successfully (the *authentication* step passes — Durga's
  username/password are correct), but every request is then rejected:

  > You do not have permission to perform this action — because he is not a staff status account
  > holder.

This is the concrete illustration of Section 2's "authentication ≠ permission" point: Durga
*authenticates* fine (DRF knows exactly who Durga is), but *fails the permission check* because
`IsAdminUser` demands `is_staff=True`, which Durga's account doesn't have — yet.

The video then goes to the Django admin, logged in as Mohan (a superuser can manage other users'
accounts), opens Durga's user record, and ticks the **"Staff status"** checkbox to `True`, then saves.
After that change, logging back in as Durga against the same `IsAdminUser`-protected endpoint now
succeeds — full CRUD access, including update and delete.

> **[Gap-filled] — why does making Durga "staff" matter here, and not "superuser"?** `IsAdminUser`
> specifically checks `request.user.is_staff`, not `request.user.is_superuser`. The two are related but
> distinct Django `User` flags: `is_staff` only controls admin-site and (as shown here) `IsAdminUser`
> access; `is_superuser` additionally grants every individual permission automatically, bypassing
> Django's fine-grained permissions framework entirely. A superuser is always staff-equivalent for this
> check, but a plain staff user (like Durga, after the edit) is *not* automatically a superuser — Durga
> can pass `IsAdminUser` without having every privilege a superuser has.

## 10. Configuring authentication/permissions globally in `settings.py` **[From video]**

> Instead of implementing authentication in `views.py`... we can also implement the same authentication
> in `settings.py` also.

The video comments out `authentication_classes`/`permission_classes` entirely from the `ViewSet`, and
instead adds a `REST_FRAMEWORK` dictionary to the project's `settings.py`:

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.BasicAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
}
```

> Within a single code, `rest_framework.permissions.is_authenticated`... this is a dictionary. In
> dictionary, keys [are] `DEFAULT_AUTHENTICATION_CLASSES`, values [are]... `DEFAULT_PERMISSION_CLASSES`
> what keys, what permission... authenticated users can only access, not [`AllowAny`].

Restarting the server, the exact same authenticated-access-only behavior from Section 9b works again —
proving the setting applies globally to *every* view in the project that doesn't override it itself.

> **[Gap-filled] — why are these strings, not the classes themselves?** Unlike the view-level
> `authentication_classes = [BasicAuthentication]` (a list of actual imported Python classes),
> `settings.py` uses **string import paths**
> (`'rest_framework.authentication.BasicAuthentication'`) instead of importing and referencing the class
> object directly. This is a common Django settings convention (the same pattern used for
> `MIDDLEWARE` and `AUTH_PASSWORD_VALIDATORS` entries, if you look back at how those are written) — it
> avoids `settings.py` needing to import every app/library up front just to build this list, and lets
> Django resolve each string to the real class lazily, only when it's actually needed.

> **[Researched] — per-view settings still win.** Per the
> [DRF settings docs](https://www.django-rest-framework.org/api-guide/settings/), `DEFAULT_AUTHENTICATION_CLASSES`
> and `DEFAULT_PERMISSION_CLASSES` are exactly that — *defaults*. Any view that sets its own
> `authentication_classes`/`permission_classes` attributes overrides the global setting for that view
> only, which is why Section 9's per-view examples and this section's global example never conflicted —
> the video simply never had both defined at the same time.

## 11. What HTTP Basic Authentication actually is **[Researched]**

The transcript demonstrates *using* `BasicAuthentication` but never explains the HTTP mechanism behind
it, so this section fills that gap from
[DRF's authentication docs](https://www.django-rest-framework.org/api-guide/authentication/#basicauthentication)
and the underlying [HTTP Basic Auth standard (RFC 7617)](https://datatracker.ietf.org/doc/html/rfc7617).

`BasicAuthentication` expects the client to send an `Authorization` HTTP header on every single request,
shaped like this:

```http
GET /manager/ HTTP/1.1
Host: 127.0.0.1:8000
Authorization: Basic bW9oYW46bW9oYW5AMTIz
```

The value after `Basic ` is simply `username:password` run through **base64 encoding** (not
encryption — base64 is trivially reversible by anyone). For example, `mohan:mohan@123` base64-encodes
to `bW9oYW46bW9oYW5AMTIz`.

- If the header is missing or the credentials are wrong, DRF responds `401 Unauthorized` with a
  `WWW-Authenticate: Basic realm="api"` response header. That specific header is what triggers the
  **native browser login popup** seen in the video ("Authentication is required...") — it's the
  browser's built-in reaction to that header, not anything DRF drew itself.
- Because the credentials are only base64-encoded (readable by anyone who intercepts the request),
  **`BasicAuthentication` must only ever be used over HTTPS** in any real deployment — otherwise a
  username and password are sent in plain, reversible text on every request.
- It's also relatively expensive: the username/password (or their hash) is checked against the database
  on *every single request*, unlike a token that's just looked up once it's issued.

> **[Researched] — industry best practice.** The DRF docs themselves recommend `BasicAuthentication`
> mainly for **testing** and for simple server-to-server (not browser-facing) scenarios, always behind
> HTTPS. For a real-world, browser- or mobile-facing production API, `TokenAuthentication` (Lectures
> 57–58) or a session-based/OAuth scheme is the standard, safer choice — Basic Auth's "resend the raw
> password on every request" model is fine for a course demo, but a poor fit for anything user-facing
> in production.

## 12. Best practices and pitfalls **[Gap-filled / Researched]**

- **Don't ship an API with the default `AllowAny` behavior by accident.** As Section 8 showed, a fresh
  DRF project with no `DEFAULT_PERMISSION_CLASSES` configured is wide open to everyone by default. It's
  easy to build and test an API this way and forget to lock it down — always set an explicit
  `DEFAULT_PERMISSION_CLASSES` (or per-view `permission_classes`) before deploying anything real.
- **Prefer configuring authentication/permissions in `settings.py`** for anything that should apply
  project-wide, and reserve per-view `authentication_classes`/`permission_classes` for genuine
  exceptions (e.g. one public read-only endpoint in an otherwise locked-down API). Repeating the same
  two lines on every view class, as the video does in Sections 9a–9c for demonstration purposes, is
  fine for learning but is unnecessary duplication in a real project.
- **`permission_classes` is a list because it's an AND, not an OR.** `permission_classes = [IsAuthenticated, IsAdminUser]`
  would require *both* checks to pass — this wasn't shown in the video, but follows directly from how
  DRF evaluates the list (every class must return `True` from its `has_permission()` method).
- **Never confuse `is_staff` with "is a real admin of the business logic."** `IsAdminUser` is a coarse,
  all-or-nothing gate tied to Django's admin-site flag. For anything more nuanced ("editors can update
  but not delete," "only the post's own author can edit it") DRF's object-level permissions or a custom
  permission class (`has_object_permission()`) is the correct tool — not stacking more `is_staff`-style
  checks.

## 13. What's next **[From video]**

> Only basic authentication [today]... next session, working with session authentication and all these
> things — these are... object permissions — all will discuss again... first we have to look into basic
> authentication only.

The video closes by naming what's coming in upcoming sessions: **session authentication**, further
**permission/object-permission techniques**, and (per the wider course outline) **token authentication**
in REST API Sessions 15–16 and **custom authentication** in REST API Session 17.

---

## 14. Worked example — testing Basic Auth from the command line **[Example]**

Beyond the video's browser-based demo, here's the same `IsAuthenticated`-protected endpoint tested with
`curl`, to make the raw HTTP mechanics from Section 11 concrete:

```bash
# Without credentials — rejected
curl -i http://127.0.0.1:8000/manager/
# HTTP/1.1 401 Unauthorized
# WWW-Authenticate: Basic realm="api"
# {"detail":"Authentication credentials were not provided."}

# With credentials, using curl's -u flag (curl builds the Basic Auth header for you)
curl -i -u mohan:mohan@123 http://127.0.0.1:8000/manager/
# HTTP/1.1 200 OK
# [{"id":1,"name":"Anil","address":"Hyderabad","email":"anil@example.com","age":29}, ...]

# Same request, with the Authorization header built manually (what -u does under the hood)
curl -i -H "Authorization: Basic bW9oYW46bW9oYW5AMTIz" http://127.0.0.1:8000/manager/
```

**Sample input → output**, tying together Sections 9b and 9c for a single `GET /manager/1/` request:

| Caller | `authentication_classes` | `permission_classes` | Result |
|---|---|---|---|
| No credentials sent | `[BasicAuthentication]` | `[IsAuthenticated]` | `401 Unauthorized` |
| Durga (`is_staff=False`), valid credentials | `[BasicAuthentication]` | `[IsAuthenticated]` | `200 OK` — any logged-in user passes |
| Durga (`is_staff=False`), valid credentials | `[BasicAuthentication]` | `[IsAdminUser]` | `403 Forbidden` — not staff |
| Durga, after being made staff in `/admin/` | `[BasicAuthentication]` | `[IsAdminUser]` | `200 OK` |
| Mohan (superuser), valid credentials | `[BasicAuthentication]` | `[IsAdminUser]` | `200 OK` |

This table is a direct, condensed restatement of the video's live Mohan/Durga demo (Sections 9b–9c),
plus the `curl` commands as a realistic use case for calling a Basic-Auth-protected API from outside a
browser (e.g. from a script, a mobile app's backend call, or a CI job) where there's no browser to show
a native login popup.

---

## Wrap-up

- **From video:** the conceptual difference between authentication (identifies the caller) and
  permissions (grants/denies the request), DRF's five authentication schemes (basic, token, session,
  remote-user, custom) and four named permission classes (`AllowAny`, `IsAuthenticated`, `IsAdminUser`,
  `IsAuthenticatedOrReadOnly`); a full hands-on build (new `restapp9` app, `Manager` model, admin
  registration, migrations, a `ModelViewSet` wired through `DefaultRouter`); a live demo comparing
  `AllowAny` / `IsAuthenticated` / `IsAdminUser` against a superuser (Mohan) and a regular-then-staff
  account (Durga); configuring the same authentication/permissions either per-view or globally via
  `REST_FRAMEWORK` in `settings.py`; a preview of session authentication, token authentication, and
  object permissions as upcoming topics.
- **Gap-filled:** why authentication must run before permissions; the reconstructed `Manager` model
  fields and `ManagerSerializer` (never fully dictated on screen); why DRF is open (`AllowAny`) by
  default; why `is_staff` and `is_superuser` are related but distinct; why `settings.py` uses string
  import paths instead of class references; best practices around not shipping an unlocked API and
  preferring global settings over per-view repetition.
- **Researched:** DRF's full authentication-flow behavior (unauthenticated ≠ automatically rejected);
  the remaining built-in permission classes (`DjangoModelPermissions` and friends) not demonstrated in
  this lecture; the mechanics of HTTP Basic Authentication (base64, the `WWW-Authenticate` header, the
  HTTPS requirement) and why it's recommended mainly for testing/server-to-server use rather than
  production browser/mobile traffic.

Double-check against the completeness checklist: authentication vs. permissions distinction ✓; the five
authentication schemes named ✓; all four permission classes named (three demonstrated, one named only)
✓; app/model/admin/migration setup ✓; serializer and `ModelViewSet` ✓; `DefaultRouter`/URL wiring ✓;
the "wide open by default" state ✓; `AllowAny` / `IsAuthenticated` / `IsAdminUser` each demonstrated with
both the Mohan and Durga accounts ✓; making a user staff via the admin panel ✓; both the per-view and
global (`settings.py`) configuration styles ✓; the "next session" preview ✓. Nothing from the transcript
was left uncovered.
