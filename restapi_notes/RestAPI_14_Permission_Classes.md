# REST API Session 14 — Permission Classes: From IsAuthenticated to DjangoModelPermissions

Source: `transcripts/restapi/REST API-14.txt`
Covers: picking up right after REST API Session 13's basic authentication, this lecture is a tour of every built-in DRF **permission class** — `AllowAny`, `IsAuthenticated`, `IsAdminUser`, `IsAuthenticatedOrReadOnly`, `DjangoModelPermissions`, `DjangoModelPermissionsOrAnonReadOnly`, and `DjangoObjectPermissions` — what each one actually allows or blocks, plus a live build-along introducing **session authentication** and a **custom permission class**, and a short teaser for token authentication (REST API Session 15).

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official DRF/Django docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. Authentication vs. permissions — recap **[From video]**

REST API Session 13 covered **basic authentication** — proving *who* a request is coming from by sending a username and password with every request. This lecture's opening recap draws a sharper line between two ideas that are easy to blur together:

> Although any user can access your API [with the wrong setup]... e-admin means only admin and staff data-support holder can access API.

- **Authentication** answers "**who are you?**" — it identifies the user making the request (or decides the request is anonymous).
- **Permissions** answer "**what are you allowed to do, now that we know who you are (or that we don't)?**" — read, write, both, or nothing.

> **[Gap-filled] — why these are always two separate steps in DRF, not one.** A request can be authenticated (DRF knows exactly which user it is) and *still* be denied — e.g. a logged-in regular user trying to delete something only admins can delete. Conversely, a request can be completely anonymous and still be allowed — e.g. a public blog's `GET` endpoint. Every DRF view therefore runs two independent checks, in order: first authentication (attach a `request.user`, or leave it as `AnonymousUser`), then permissions (given that user, is *this* action on *this* view allowed). Mixing these up is a common beginner mistake — "the user can log in but still can't do X" is almost always a permissions problem, not an authentication one.

The two are imported from separate DRF modules, matching this split:

```python
# Authentication classes — "who is this?"
from rest_framework.authentication import BasicAuthentication, SessionAuthentication

# Permission classes — "what can they do?"
from rest_framework.permissions import (
    AllowAny,
    IsAuthenticated,
    IsAdminUser,
)
```

## 2. Configuring authentication & permissions globally, in `settings.py` **[From video]**

Every earlier lecture set `authentication_classes` and `permission_classes` directly as attributes on a view. The video shows there's also a **project-wide default**, configured once in `settings.py`, so you don't have to repeat the same two lines on every single view:

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
}
```

- `REST_FRAMEWORK` — a single dictionary, the standard place DRF looks for **all** of its project-level settings (not just these two — pagination, throttling, and filtering, all covered in later lectures, are configured the same way).
- `DEFAULT_AUTHENTICATION_CLASSES` — a list of authentication classes (as **import-path strings**, not the classes themselves) applied to every view that doesn't explicitly override it.
- `DEFAULT_PERMISSION_CLASSES` — same idea, for permissions.

> This is entirely optional — the video is explicit that "not compulsory, just make changes" if you want it. Setting `authentication_classes`/`permission_classes` directly on a view (or viewset) always **overrides** whatever `settings.py` says for that one view.

> **[Researched] — what DRF uses if you configure neither.** Per the [DRF settings docs](https://www.django-rest-framework.org/api-guide/settings/), if `DEFAULT_PERMISSION_CLASSES` is never set at all, DRF's built-in default is `['rest_framework.permissions.AllowAny']` — i.e., **wide open** by default. This is a genuinely important pitfall: a fresh DRF project with no permission configuration anywhere lets *anyone* read and write every model exposed through it. Setting a sane project-wide default (commonly `IsAuthenticated`) in `settings.py`, and only loosening it per-view where a public endpoint is actually intended, is the safer default habit.

## 3. The built-in permission classes, one by one **[From video, precision gap-filled]**

The video lists all seven of DRF's built-in permission classes back to back before demonstrating a few of them live. Table first, details after:

| Permission class | Anonymous (not logged in) | Authenticated, no special model permissions | Authenticated, with model permissions |
|---|---|---|---|
| `AllowAny` | full access | full access | full access |
| `IsAuthenticated` | denied entirely | full access | full access |
| `IsAdminUser` | denied entirely | denied unless `is_staff=True` | denied unless `is_staff=True` |
| `IsAuthenticatedOrReadOnly` | read-only (`GET`/`HEAD`/`OPTIONS`) | full access | full access |
| `DjangoModelPermissions` | denied entirely | denied (even reads) | full access, method-by-method |
| `DjangoModelPermissionsOrAnonReadOnly` | read-only | denied for writes without permission | full access, method-by-method |
| `DjangoObjectPermissions` | denied entirely | denied without per-*object* permission | access checked per row, not just per model |

```python
from rest_framework.permissions import (
    AllowAny,
    IsAuthenticated,
    IsAdminUser,
    IsAuthenticatedOrReadOnly,
    DjangoModelPermissions,
    DjangoModelPermissionsOrAnonReadOnly,
    DjangoObjectPermissions,
)
```

### `AllowAny` **[From video]**

> Aloe any means it's a permission class[es], only — any user can use an API.

The simplest possible permission: nobody is checked at all, authenticated or not. Useful for genuinely public endpoints (e.g. a public product catalog, a health-check endpoint).

### `IsAuthenticated` **[From video]**

> Is authenticated. Is authenticated user can access.

Blocks every anonymous request outright; any logged-in user (regardless of role) gets full access. This was the class referenced back in REST API Session 13's basic-authentication demo.

### `IsAdminUser` **[From video + gap-fill]**

> E is admin means admin users only can able to use this API... admin user only can access.

> **[Gap-filled] — the detail the video glosses over: "admin" here means `is_staff`, not `is_superuser`.** `IsAdminUser`'s actual check (from the DRF source) is `bool(request.user and request.user.is_staff)`. `is_staff` (Lecture 26) is the "can this account log into `/admin/`" flag — it's a weaker condition than `is_superuser` ("has every permission, unconditionally"). A staff user who is *not* a superuser still passes `IsAdminUser`. This distinction matters in practice: if you actually mean "only my superusers," `IsAdminUser` alone is not that check — you'd need a custom permission class (covered in §10) testing `request.user.is_superuser` instead.

### `IsAuthenticatedOrReadOnly` **[From video]**

> Is authenticated or read only means authenticated users can access API and authenticated users can only read the API but cannot write the API... [correction, stated later, more clearly] authenticated users can read and write also... non-authenticated persons can read [but] cannot write.

(The video briefly states this backwards the first time, then corrects itself in the live demo — the corrected version, matching the demo's actual behavior, is what's captured in the table above.) Anonymous requests are allowed **only** for safe, read-only HTTP methods (`GET`, `HEAD`, `OPTIONS`); any authenticated user gets full read **and** write access. This is the shape most public content APIs actually want: "anyone can browse, only logged-in users can contribute."

### `DjangoModelPermissions` **[From video + researched]**

> Django model permission means user should be authenticated... and must have enough permission for read and write user... on the model.

This one is stricter than it first sounds, and the video's demo makes the strictness visible: an authenticated user with **zero** model permissions assigned still gets denied, even for a plain `GET`.

> **[Researched] — the exact rule, from DRF's source.** `DjangoModelPermissions` maps each HTTP method to a required Django model permission, using the model behind the view's `queryset`:
>
> | HTTP method | Required permission |
> |---|---|
> | `GET`, `HEAD`, `OPTIONS` | *(none listed — but see below)* |
> | `POST` | `<app_label>.add_<model_name>` |
> | `PUT`, `PATCH` | `<app_label>.change_<model_name>` |
> | `DELETE` | `<app_label>.delete_<model_name>` |
>
> Even though `GET` has no permission listed in that table, `DjangoModelPermissions` has a separate flag, `authenticated_users_only = True` by default, which denies **every** method — reads included — to anyone who isn't authenticated at all. So in practice: anonymous users are blocked completely; an authenticated user with no model permissions can't even read; an authenticated user needs the matching `add_`/`change_`/`delete_` permission (granted via Django's built-in auth permission system — Lecture 42's researched note on `has_perm()`/named permissions) for each action they want to perform.

### `DjangoModelPermissionsOrAnonReadOnly` **[From video]**

> D-jango model permission and non-read only... if user is authenticated, he should have the permission for read and write. If user is not authenticated, but user can able to read... user can able to read only.

Exactly `DjangoModelPermissions`, with one flag flipped: `authenticated_users_only = False`. The effect: anonymous users are now allowed the safe/read methods (since those require no permission in the table above and are no longer blocked outright), but any write still requires an authenticated user with the matching model permission.

### `DjangoObjectPermissions` **[From video + researched]**

> D-jango object permission means user must [be] authenticated and user should have enough permissions on particular objects only — means particular records only.

Same idea as `DjangoModelPermissions`, extended one level deeper: instead of "can this user add/change/delete *any* row of this model," it can check "can this user add/change/delete *this specific row*." The video only names it in the list and doesn't build a working demo of it.

> **[Researched] — why the video doesn't demo this one live.** Per the [DRF permissions docs](https://www.django-rest-framework.org/api-guide/permissions/#djangoobjectpermissions), plain Django (and DRF) has **no built-in storage for per-object permissions** — Django's own permission system only tracks permissions per *model*, not per individual row. To actually use `DjangoObjectPermissions` for anything, the project needs an object-permission-checking backend added to `AUTHENTICATION_BACKENDS`, most commonly the third-party package **`django-guardian`**. Without such a backend configured, `DjangoObjectPermissions` behaves the same as `DjangoModelPermissions` (object-level checks just always pass/fail the same as the model-level ones) — which is why it isn't practical to demo without extra setup the video doesn't do in this lecture.

## 4. HTTP status codes when access is denied: 401 vs. 403 **[From video + researched]**

> Authenticated response that are denied permission will result in an HTTP... 403 forbidden response — means if the response is denied actually from the server, you will see HTTP 403 forbidden response only.

The video's live demo consistently shows the message **"Authentication credentials were not provided"** when an anonymous request is rejected.

> **[Researched] — the precise rule DRF actually follows.** Per the [DRF authentication docs](https://www.django-rest-framework.org/api-guide/authentication/#custom-authentication), DRF returns one of two different status codes for a denied request, and which one depends on the *authentication scheme(s)* configured, not the permission class:
> - **`401 Unauthorized`** — returned when the request has no valid credentials **and** at least one configured authentication class supports a `WWW-Authenticate` challenge header (e.g. `BasicAuthentication`, `TokenAuthentication`). This is the standard HTTP signal for "log in and try again."
> - **`403 Forbidden`** — returned either when the request *is* properly authenticated but still isn't allowed (wrong permissions), **or** when it isn't authenticated at all but none of the configured authentication classes support that challenge header — which is exactly `SessionAuthentication`'s situation (used throughout this lecture's demo), since a session cookie has no equivalent "please log in" HTTP challenge to issue.
>
> This is why the video's demo — built on `SessionAuthentication` — shows `403 Forbidden` even for a plain "you're not logged in at all" case, where REST API Session 13's `BasicAuthentication` demo would more typically show `401 Unauthorized` for the same underlying problem. Same underlying cause (no valid credentials), different status code, because the authentication *scheme* differs.

## 5. Session authentication, introduced **[From video]**

> The authentication scheme uses Django's default session backend for authentication — automatically, whenever we send a request to the server, for every request... if successfully authenticated, session authentication provides the following credentials, like `request.user`.

`SessionAuthentication` is DRF's authentication class built on top of Django's own **session framework** — the same cookie-based "remember that this browser is logged in" mechanism used by `django.contrib.auth`'s `login()`/`logout()` (Lecture 28) and by the browsable API's own "Log in" link. Once a browser has a valid session cookie (set at login time), every subsequent request automatically carries it, and DRF reads `request.user` straight off that session — no separate header or token to attach by hand.

```python
from rest_framework.authentication import SessionAuthentication
```

> **[Gap-filled] — when session authentication actually makes sense.** `SessionAuthentication` is a natural fit when the API is consumed by the **same site's own browser-based front end** (e.g. a Django template, or a JS app served from the same domain) — the session cookie is already there from the normal login flow, so there's nothing extra to set up. It's a poor fit for a truly separate client — a mobile app, a different domain, a server-to-server integration — because there's no browser session cookie to piggyback on there; those cases are what token authentication (REST API Session 15, teased at the end of this lecture) and other stateless schemes are for. `SessionAuthentication` also requires standard CSRF protection for unsafe methods, same as any other session-based Django form submission.

## 6. Building the demo app **[From video]**

The video creates a new app to demonstrate everything above, reusing the same `Manager` model/serializer pattern from earlier CRUD lectures (48–54) rather than re-typing it:

```bash
python manage.py startapp restapp10
```

```python
# settings.py
INSTALLED_APPS = [
    # ...
    'restapp10',
]
```

```python
# restapp10/models.py
class Manager(models.Model):
    name = models.CharField(max_length=50)
    email = models.EmailField()
    age = models.IntegerField()
```

```python
# restapp10/admin.py
from django.contrib import admin
from restapp10.models import Manager

admin.site.register(Manager)
```

```python
# restapp10/serializers.py
from rest_framework import serializers
from restapp10.models import Manager

class ManagerSerializer(serializers.ModelSerializer):
    class Meta:
        model = Manager
        fields = '__all__'
```

```bash
python manage.py makemigrations
python manage.py migrate
```

Two `Manager` records are then added through `/admin/` (the video's own example: `"Manager" / "manager@mail.com" / 25` and a second, similar record) — test data to authenticate and permission-check against in the demos below.

## 7. Wiring authentication & permissions on the view **[From video]**

```python
# restapp10/views.py
from rest_framework import viewsets
from restapp10.models import Manager
from restapp10.serializers import ManagerSerializer
from rest_framework.authentication import SessionAuthentication
from rest_framework.permissions import IsAuthenticatedOrReadOnly  # swapped out below

class ManagerModelViewSet(viewsets.ModelViewSet):
    queryset = Manager.objects.all()
    serializer_class = ManagerSerializer
    authentication_classes = [SessionAuthentication]
    permission_classes = [IsAuthenticatedOrReadOnly]
```

- `authentication_classes` — how DRF should figure out *who* is making this request (here: check for a valid Django session).
- `permission_classes` — given that user (or `AnonymousUser`), what they're allowed to do.
- Reusing `viewsets.ModelViewSet` (REST API Session 12) means all of `list`/`retrieve`/`create`/`update`/`partial_update`/`destroy` are covered by these same two settings at once — no per-action repetition needed.

```python
# urls.py
from rest_framework.routers import DefaultRouter
from restapp10.views import ManagerModelViewSet

router = DefaultRouter()
router.register('manager', ManagerModelViewSet, basename='manager')

urlpatterns = [
    # ...
    *router.urls,
]
```

`basename='manager'` — required here specifically because `ManagerModelViewSet.queryset` alone isn't enough for the router to auto-derive URL names in every case; the video calls out that the string passed to `basename` must match what's registered, or routing breaks.

## 8. Demo: `IsAuthenticatedOrReadOnly` in action **[From video]**

With `permission_classes = [IsAuthenticatedOrReadOnly]` and logged out of every session (admin included):

- Visiting the API root and the `manager` list endpoint **works** — a `GET` request, and anonymous `GET`s are explicitly allowed by this permission class.
- Attempting to add/update/delete a record from the browsable API is **not offered** — those are write operations, and an anonymous user only gets read access.

After logging in through `/admin/` as the existing `Manager` account created earlier:

- The same endpoint now offers full **create/update/delete** actions too, and using them succeeds — confirming an authenticated user gets unrestricted read+write with this permission class.

> **[Example] — the same behavior, as a standalone request/response walkthrough.** Same view, `permission_classes = [IsAuthenticatedOrReadOnly]`, no login:
> ```http
> GET /manager/
> → 200 OK
> [{"id": 1, "name": "Manager", "email": "manager@mail.com", "age": 25}]
>
> POST /manager/
> {"name": "New One", "email": "new@mail.com", "age": 30}
> → 403 Forbidden
> {"detail": "You do not have permission to perform this action."}
> ```
> Now the same `POST`, sent by a logged-in (session-authenticated) user:
> ```http
> POST /manager/
> {"name": "New One", "email": "new@mail.com", "age": 30}
> → 201 Created
> {"id": 3, "name": "New One", "email": "new@mail.com", "age": 30}
> ```

## 9. Demo: `DjangoModelPermissions` and per-user model permissions **[From video, reconstructed]**

The video switches `permission_classes` to `[DjangoModelPermissions]` and walks through several account states. Because the exact narration here is heavily garbled, the sequence below is reconstructed from DRF's actual `DjangoModelPermissions` behavior (§3) rather than transcribed word-for-word — the overall arc (anonymous → authenticated-but-unprivileged → explicitly granted permissions) does match what the video describes:

1. **Logged out entirely:** every request, including `GET`, is rejected with `"Authentication credentials were not provided."` — matches `authenticated_users_only=True` blocking anonymous access outright.
2. **A new, ordinary user is created** through `/admin/` (the video's own example: username `psi`, a password, **not** marked as staff) — a regular account with no special permissions, created via Django's normal "Add user" form.
3. **Logging in as that plain user still isn't enough on its own** — the demo shows continued denial until the account is explicitly given both **staff status** (to be manageable/visible the way the demo needs) and, separately, **model-level permissions** for the `Manager` model.
4. **A superuser (the video's `Mohan` account) then grants specific permissions** to the `psi` account, through the user's edit page in `/admin/` → **"User permissions"** section — individually checking `can add manager`, `can change manager`, `can delete manager`, `can view manager` (Django auto-creates these four permissions for every model, per the researched note in Lecture 42).
5. **After that grant, `psi` can perform exactly the actions their granted permissions cover** — e.g. granted `view` and `change` but not `delete` would mean the delete option simply isn't honored, matching the method → permission table in §3.

> **[Gap-filled] — the one precise, load-bearing detail this demo is illustrating.** The core lesson `DjangoModelPermissions` is teaching here: **being authenticated is not the same as being permitted.** A perfectly real, logged-in user (`psi`) is still denied every model action until a superuser deliberately grants permissions on that specific model, one action at a time, through Django's own auth permission system — the same `add_`/`change_`/`delete_`/`view_` permissions referenced in Lecture 42's note on `user.has_perm()`. This is the built-in, no-extra-package version of fine-grained access control DRF gives you, once you go beyond "logged in or not."

## 10. Demo: `DjangoModelPermissionsOrAnonReadOnly` **[From video]**

Switching `permission_classes` to `[DjangoModelPermissionsOrAnonReadOnly]` and logging out entirely:

> Even he don't have permission, but he can able to read the data — no problem.

- A fully anonymous `GET` request now **succeeds** (unlike plain `DjangoModelPermissions`, which blocked even reads for anonymous users) — matching the `authenticated_users_only = False` behavior from §3.
- Writing still requires both authentication **and** the matching granted model permission, exactly as with `DjangoModelPermissions`.

## 11. Custom permission classes **[From video]**

> Custom permissions... override base permission[s] and implement either... both of the following methods, like `.has_permission`, `.has_object_permission`... the method should return true if the request should be granted access and false otherwise.

Every built-in permission class above is itself just a subclass of `BasePermission` — nothing stops writing your own for rules DRF doesn't ship with.

```python
# restapp10/custom_permissions.py
from rest_framework.permissions import BasePermission

class CustomPermission(BasePermission):
    def has_permission(self, request, view):
        if request.method == 'GET':
            return True
        return False
```

- Must inherit from **`BasePermission`** — DRF's permission system calls `has_permission()` (view-level) and/or `has_object_permission()` (object-level, only reached for detail actions like retrieve/update/delete on a specific row) on whatever class is listed in `permission_classes`, and expects a plain `True`/`False` back.
- This particular example allows **only `GET` requests** through — every other method (`POST`, `PUT`, `PATCH`, `DELETE`) is denied, regardless of who's making the request (no authentication check at all here — this custom class is entirely about the HTTP method).

```python
# restapp10/views.py
from restapp10.custom_permissions import CustomPermission

class ManagerModelViewSet(viewsets.ModelViewSet):
    queryset = Manager.objects.all()
    serializer_class = ManagerSerializer
    authentication_classes = [SessionAuthentication]
    permission_classes = [CustomPermission]
```

The video then demos the opposite rule live — flipping the condition to allow **only `POST`**:

```python
class CustomPermission(BasePermission):
    def has_permission(self, request, view):
        if request.method == 'POST':
            return True
        return False
```

With this version, `GET` requests are now rejected (`"Authentication credentials were not provided"` is shown, even though the real reason is the permission check failing, not authentication — a slightly misleading but common message from DRF's generic denial handling), while `POST` succeeds.

> **[Example] — a more realistic custom permission: "owner or read-only."** A very common real-world need this simple GET/POST-only example doesn't show: letting anyone read, but only the object's own creator edit or delete it (e.g. a blog post, a comment). Assuming the model has an `author` field pointing at a `User`:
> ```python
> from rest_framework.permissions import BasePermission, SAFE_METHODS
>
> class IsOwnerOrReadOnly(BasePermission):
>     def has_object_permission(self, request, view, obj):
>         # Reads are always allowed, for anyone.
>         if request.method in SAFE_METHODS:
>             return True
>         # Writes are only allowed if this object's author is the requester.
>         return obj.author == request.user
> ```
> This uses `has_object_permission` (not `has_permission`) specifically because the check depends on *which row* is being accessed (`obj.author`) — that method only runs once DRF already has a specific object in hand (e.g. inside `retrieve`/`update`/`destroy`), not for list/create actions.

> **[Researched] — DRF's own `SAFE_METHODS` constant.** Per the [DRF permissions source](https://www.django-rest-framework.org/api-guide/permissions/), `rest_framework.permissions.SAFE_METHODS` is simply the tuple `('GET', 'HEAD', 'OPTIONS')` — the same set of "read-only, never changes data" methods that `IsAuthenticatedOrReadOnly` and `DjangoModelPermissionsOrAnonReadOnly` build their read/write split around. Using this constant instead of hand-listing `'GET'` (as the video's own custom example does) is the standard, more maintainable way to write this check.

## 12. Looking ahead **[From video]**

The lecture closes with a short roadmap of what's still to come in this DRF unit:

> One more important authentication is there: token authentication... after generating a token, he will get some tokens; next time whenever he wants to access the API for doing operations, he should provide the token... Token is nothing but a ticket.

- **Token authentication** (REST API Session 15) — a stateless alternative to session authentication, better suited to client-server setups (mobile apps, separate front ends) where there's no shared browser session cookie. Requires adding `TokenAuthentication` to `DEFAULT_AUTHENTICATION_CLASSES` and `'rest_framework.authtoken'` to `INSTALLED_APPS` — explicitly deferred to "tomorrow's session" rather than covered here.
- **Custom authentication** — writing your own authentication class (parallel to the custom *permission* class in §11), mentioned as coming later.
- **Filtering** — including field-level filtering and ordering filters.
- **Pagination** — the DRF-specific version of the pagination concept already covered for plain Django.
- **Throttling** — rate-limiting how often a client can call the API.

None of these are explained in this lecture beyond being named — they're flagged here only so later lecture notes can cross-reference back to "mentioned in REST API Session 14" without re-deriving that context.

---

## Wrap-up

- **From video:** the authentication-vs-permissions distinction; configuring `DEFAULT_AUTHENTICATION_CLASSES`/`DEFAULT_PERMISSION_CLASSES` in `settings.py` versus per-view; all seven built-in permission classes (`AllowAny`, `IsAuthenticated`, `IsAdminUser`, `IsAuthenticatedOrReadOnly`, `DjangoModelPermissions`, `DjangoModelPermissionsOrAnonReadOnly`, `DjangoObjectPermissions`) as introduced and named; the 403 Forbidden response for denied requests; `SessionAuthentication` introduced and wired up; a full build-along (`restapp10`, `Manager` model/serializer/admin, a `ModelViewSet`, router registration) demoing `IsAuthenticatedOrReadOnly`, `DjangoModelPermissions` (including granting per-user model permissions through `/admin/`), and `DjangoModelPermissionsOrAnonReadOnly`; a custom `BasePermission` subclass restricting access by HTTP method; a closing teaser naming token authentication, custom authentication, filtering, pagination, and throttling as upcoming topics.
- **Gap-filled:** the authentication-vs-permission mental model spelled out explicitly; why `IsAdminUser` checks `is_staff` and not `is_superuser`; the reconstructed, precise sequence of the `DjangoModelPermissions` per-user-permission demo (the transcript's narration of this section was especially garbled); when session authentication is and isn't the right choice; a realistic "owner or read-only" custom permission example.
- **Researched:** the exact DRF default (`AllowAny`) when no permission classes are configured at all, and why that's a pitfall; `DjangoModelPermissions`'s precise method → permission mapping and its `authenticated_users_only` flag; why `DjangoObjectPermissions` needs a package like `django-guardian` to do anything beyond model-level checks; the real 401-vs-403 rule (depends on the authentication scheme, not just the permission outcome); DRF's `SAFE_METHODS` constant.

Double-check against the checklist: authentication-vs-permission framing ✓, settings.py global config ✓, all seven built-in permission classes defined and distinguished ✓, 401 vs. 403 ✓, session authentication introduced ✓, the full demo app build-along ✓, all three demoed permission classes' live behavior ✓, custom permission class (both the video's GET-only/POST-only version and an added realistic owner-or-read-only example) ✓, the closing roadmap (token auth, custom auth, filtering, pagination, throttling) ✓. Token authentication itself is only named here, not explained — that's REST API Session 15's job, not this lecture's.
