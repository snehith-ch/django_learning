# REST API Session 17 — Custom authentication (and a first look at filtering)

Source: `transcripts/restapi/REST API-17.txt`
Covers: writing your **own** DRF authentication class by subclassing `BaseAuthentication` and implementing `authenticate(self, request)`, wired into a `ManagerViewSet` and tested end-to-end with a query-string username scheme. The transcript then pivots, in its second half, into the very start of DRF **filtering** (overriding `get_queryset()` to return only the logged-in user's own records) — the instructor explicitly says this filtering topic will be "finished" in the next session, so treat that part of these notes as a preview, not the full picture (REST API Session 18 covers filtering in depth).

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official DRF/Django docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## Completeness checklist (topics this transcript touches)

1. Recap: last session = token authentication; today = custom authentication
2. What "custom authentication" means and why you'd write one
3. The `authenticate(self, request)` contract: return a `(user, auth)` tuple / return `None` / raise `AuthenticationFailed`
4. Recap of the app already in progress: `Manager` model, admin registration, `ManagerSerializer`
5. Using **DB Browser for SQLite** to inspect the built-in `auth_user` table
6. Creating `authentication.py` and the required imports (`BaseAuthentication`, `User`, `AuthenticationFailed`)
7. Writing the `CustomAuthentication` class step by step
8. Wiring it into `ManagerViewSet` via `authentication_classes` / `permission_classes`
9. URL config recap (`DefaultRouter`, `router.register(...)`, `basename`)
10. Testing: no credentials → "Authentication credentials were not provided"
11. The "log in via the browsable API" false start — why session login didn't satisfy the custom scheme
12. Testing: `?username=<existing user>` → success; `?username=<unknown user>` → `AuthenticationFailed`
13. Security caveat: query-string credentials vs. header-based credentials
14. `authenticate_header()` and 401 vs. 403 status codes (not shown in video — researched)
15. A first look at filtering: default queryset behavior of generic views
16. Overriding `get_queryset()` to filter by `self.request.user`
17. New demo app (`myapp1`), `Student` model with a `user` foreign key, migrations after a field rename
18. Admin panel: creating `Student` records tied to different users; the `is_staff` checkbox gotcha for non-superuser admin logins
19. Testing the filtered list per logged-in user
20. Teaser for what's next (filter backends, search filters, pagination, throttling)

---

## 1. Recap — the authentication classes so far **[From video]**

> In the last session, we discussed token authentication — how to generate tokens, and how to use a token to access your API.

By this point in the course, three built-in DRF authentication schemes have been covered:

- **Basic authentication** — username/password sent on every request, base64-encoded in an `Authorization` header.
- **Session authentication** — relies on Django's own login session/cookie (mainly useful for the browsable API and same-site JavaScript clients).
- **Token authentication** — a client logs in once, receives a permanent token string, and sends `Authorization: Token <token>` on every later request.

Today's topic — **custom authentication** — is presented as the natural fourth option: instead of using one of DRF's ready-made schemes, you write your **own** authentication class with your own rules for what counts as "authenticated."

## 2. What custom authentication is, and why you'd write one **[From video]**

> Custom authentication means we can prepare our own authentication system, and we can use that authentication to access the API.

A plain-language way to put it: DRF's three built-in schemes cover the common cases, but real projects sometimes need something else — a signed API key issued to partner companies, a one-time link token, a header format required by an existing mobile app, or (as in this lecture's toy example) a simplified scheme built purely to demonstrate the mechanism. **[Gap-filled]** Whenever none of Basic/Session/Token fit your exact requirement, DRF lets you plug in any class that knows how to (a) look at an incoming `request` and (b) decide who's making it — that's all "authentication" means at the framework level. DRF doesn't care *how* your class decides; it only cares that your class follows a specific contract, described next.

## 3. The `authenticate(self, request)` contract **[From video]**

> You can create your own authentication class, and it should be inherited from `BaseAuthentication`. The method should return a tuple of `user` and `auth` if authentication succeeds. If authentication is not attempted, it should return `None`. If authentication is attempted but fails, raise an `AuthenticationFailed` exception.

Every custom authentication class must:

1. Subclass <dfn>`rest_framework.authentication.BaseAuthentication`</dfn> — the base class that defines the interface DRF expects.
2. Implement one method: **`authenticate(self, request)`**. DRF calls this automatically for every incoming request, for every authentication class listed in `authentication_classes`, until one of them succeeds (or all of them are exhausted).

`authenticate()` has exactly **three** possible outcomes, and DRF treats each one differently:

| Outcome | What you do | What DRF does next |
|---|---|---|
| **Success** | `return (user, auth)` — a two-item tuple: the authenticated user object, and an optional second value (often `None`) representing auth-scheme-specific data (e.g. the token used) | Sets `request.user` and `request.auth` to those values; the request is considered authenticated |
| **Not attempted** | `return None` (not a tuple — just the value `None`) | Treats this scheme as "didn't apply" and moves on to check any *other* authentication classes configured for the view; if none succeed, the request falls back to `AnonymousUser` |
| **Attempted but failed** | `raise AuthenticationFailed('some message')` | **Immediately** stops and returns an error response to the client — no other authentication class is checked, and no permission check runs; the failure message goes straight back to the caller |

**[Gap-filled] — why the "not attempted vs. failed" distinction matters.** This is easy to get backwards as a beginner, so it's worth restating plainly: `return None` means *"I have nothing to say about this request — maybe some other scheme can identify it, or maybe it should just be anonymous."* Raising `AuthenticationFailed` means *"this request DID try to use my scheme, and what it gave me was wrong — stop everything and tell the client immediately."* Get this wrong (e.g. always raising instead of returning `None` when no credentials are present at all) and you'll break any view that's supposed to allow anonymous access alongside your custom scheme, because a raised exception short-circuits everything else.

## 4. The demo app so far (recap) **[From video]**

The instructor is continuing an existing app (`myapp`, referred to as "my application") from the previous lectures, not starting fresh:

- A `Manager` model with fields `id`, `name`, `address`, `mail`, `age`.
- The model is registered in `admin.py` so it shows up in the Django admin.
- `makemigrations` / `migrate` have already been run, and the admin panel already has some `Manager` records in it.
- A `ManagerSerializer` (a `ModelSerializer`) already exists in `serializers.py`, exposing `id`, `name`, `address`, `mail`, `age`.

None of this is new material — it's the same `Manager` model and serializer used in the token-authentication lectures immediately before this one, now reused as the target for the new custom scheme.

## 5. A quick detour — looking at the `auth_user` table directly **[From video]**

Before writing the class, the instructor opens the SQLite database file for the project in **DB Browser for SQLite** and browses the built-in `auth_user` table directly, to show which users actually exist (their usernames will matter for testing later — the video names several, including "Mohan," "Durga," and "Prasad," though a couple of names are too garbled in the audio to transcribe with confidence).

> **[Gap-filled] — what DB Browser for SQLite is.** It's a free, standalone desktop GUI application (not part of Django or DRF) for opening a `.sqlite3` database file and browsing/editing its tables visually — useful for a quick sanity check ("which rows actually exist right now?") without writing a query or going through the Django admin. Django's default database backend, SQLite, stores the whole database as a single file (commonly `db.sqlite3`) sitting in the project folder, which is exactly the kind of file this tool opens. `auth_user` is the table Django's built-in `django.contrib.auth` app creates automatically for every project — one row per user, with columns like `username`, `password` (hashed, never plain text), `is_staff`, `is_superuser`, `is_active`, etc.

## 6. Building `authentication.py` **[From video]**

A new file is created inside the app: **`authentication.py`** (a plain Python file, not one Django generates for you — you add it yourself, the same way `serializers.py` was added by hand in earlier lectures).

### Required imports

```python
# myapp/authentication.py

# The base class every custom authentication class must inherit from
from rest_framework.authentication import BaseAuthentication

# Django's own built-in User model — the video reuses django.contrib.auth's
# User model rather than defining a separate "custom auth" table, so that the
# usernames it checks against are the same ones visible in auth_user / the
# Django admin's Users section
from django.contrib.auth.models import User

# The exception DRF expects you to raise when authentication is attempted
# but the credentials given are invalid
from rest_framework.exceptions import AuthenticationFailed
```

**[Gap-filled]** Two of these three imports come straight from `rest_framework`, because authentication classes and their exceptions are a DRF concept, not a plain-Django one — plain Django doesn't have "authentication classes" in this pluggable sense. The `User` import, on the other hand, comes from Django itself (`django.contrib.auth.models`), because DRF doesn't invent its own concept of "a user" — it reuses whatever user model your Django project is already using.

### The class itself

```python
class CustomAuthentication(BaseAuthentication):
    """
    A minimal custom authentication scheme: a request is considered
    authenticated if it supplies a ?username=<name> query parameter
    that matches an existing Django User.
    """

    def authenticate(self, request):
        # Pull the value of the "username" query-string parameter, e.g.
        # /manager/?username=Mohan  ->  username = "Mohan"
        # request.GET.get(...) is a dict-style lookup, but it returns None
        # instead of raising an error if the key isn't present at all.
        username = request.GET.get('username')

        # Case 1: no ?username=... was supplied at all -> nothing to check,
        # so we tell DRF "this scheme wasn't attempted" by returning None.
        if username is None:
            return None

        try:
            # Case 2 (happy path): try to find a matching User row.
            user = User.objects.get(username=username)
        except User.DoesNotExist:
            # Case 3: a username WAS supplied, but it doesn't match any
            # real user -> authentication was attempted and it failed,
            # so we raise, which stops everything immediately.
            raise AuthenticationFailed('User does not exist')

        # Success: return the (user, auth) tuple DRF expects. The second
        # value is for any extra per-request auth data; there isn't any
        # here, so it's just None.
        return (user, None)
```

This is a direct, cleaned-up transcription of what's typed on screen in the video — the logic (return `None` / raise `AuthenticationFailed` / return a tuple) matches the contract from section 3 exactly.

> **[Gap-filled] — this scheme has no password check.** Notice this class only checks that the *username* exists — it never verifies a password, a secret key, or anything else proving the requester actually is that user. That's a deliberate simplification for teaching the `authenticate()` mechanism in isolation; see the security callout in section 8 for why you would never ship this exact scheme to production.

## 7. Wiring the class into the view **[From video]**

Back in `views.py`, the `ManagerViewSet` (a DRF `ModelViewSet`, built in earlier lectures) is updated to actually use the new class:

```python
# myapp/views.py
from rest_framework import viewsets, permissions
from rest_framework.permissions import IsAuthenticated

from myapp.models import Manager
from myapp.serializers import ManagerSerializer
from myapp.authentication import CustomAuthentication   # the class from section 6

class ManagerViewSet(viewsets.ModelViewSet):
    queryset = Manager.objects.all()
    serializer_class = ManagerSerializer

    # Tell DRF: "use MY class to figure out who's making this request"
    authentication_classes = [CustomAuthentication]

    # Tell DRF: "only let the request through if someone WAS identified"
    permission_classes = [IsAuthenticated]
```

**[Gap-filled] — how these two settings cooperate.** `authentication_classes` and `permission_classes` do two different jobs that are easy to blur together as a beginner:

- **`authentication_classes`** answers *"who is this?"* — it runs first, tries each listed class's `authenticate()` in order, and sets `request.user` (to a real user, or to Django's built-in `AnonymousUser` if nothing succeeded and nothing raised).
- **`permission_classes`** answers *"is that identity allowed to do this?"* — it runs after authentication, and looks at whatever `request.user` ended up being. `IsAuthenticated` (covered in REST API Session 14's permission classes topic) simply checks `request.user.is_authenticated` and rejects the request with `403 Forbidden` if it's `False` — which is exactly what `AnonymousUser` reports.

So in this view: if `?username=` is missing entirely, `CustomAuthentication.authenticate()` returns `None`, `request.user` falls back to `AnonymousUser`, and `IsAuthenticated` then blocks the request. If `?username=` names a real user, `request.user` becomes that real `User` object, and `IsAuthenticated` lets it through. If `?username=` names a *nonexistent* user, `AuthenticationFailed` is raised straight out of the authentication step, before `permission_classes` is even consulted.

### URL config (unchanged / recap) **[From video]**

The instructor confirms the URL setup from previous lectures needs no changes:

```python
# myproject/urls.py
from django.contrib import admin
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from myapp.views import ManagerViewSet

router = DefaultRouter()
router.register('manager', ManagerViewSet, basename='manager')

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include(router.urls)),
    path('api-auth/', include('rest_framework.urls')),  # adds the browsable API's login/logout links
]
```

A `DefaultRouter` (introduced when ViewSets were covered) automatically generates the list/detail URL patterns for a registered ViewSet — `basename='manager'` gives those generated URL names a prefix (`manager-list`, `manager-detail`) since the ViewSet has no `queryset` attribute DRF can infer a name from... actually it does have one here, but `basename` is supplied explicitly either way as a matter of habit.

## 8. Testing it end-to-end **[From video]**

With the server running (`python manage.py runserver`), the instructor exercises the endpoint several different ways:

**No query parameter at all** — visiting `/manager/` with nothing extra:

```http
GET /manager/ HTTP/1.1
```
```json
{
  "detail": "Authentication credentials were not provided."
}
```
This matches the flow from section 7: `authenticate()` returns `None` (no `username` param) → `request.user` is anonymous → `IsAuthenticated` rejects it.

**A false start — logging in through the browsable API's own login link.** The instructor first tries clicking DRF's browsable-API "Log in" link (the one added by `rest_framework.urls`) and signing in there — and the request *still* comes back "Authentication credentials were not provided." He explicitly calls this out: *"this is not the way to provide the authentication scheme."*

> **[Gap-filled] — why the browsable-API login didn't help.** That login link authenticates a *browser session* (it's Django's own session/cookie login, the same mechanism behind `SessionAuthentication`). But `ManagerViewSet` only lists `CustomAuthentication` in `authentication_classes` — and `CustomAuthentication.authenticate()` only ever looks at `request.GET.get('username')`. It never looks at the session at all. So no matter how thoroughly you log in through the browser, this particular view has no code path that would notice it. This is a useful lesson about custom authentication in general: **a custom authentication class only recognizes exactly what you programmed it to look for** — nothing else "counts" unless your `authenticate()` method explicitly checks it.

**`?username=` naming an existing user** — e.g. `/manager/?username=Mohan`:

```http
GET /manager/?username=Mohan HTTP/1.1
```
```json
[
  {"id": 1, "name": "...", "address": "...", "mail": "...", "age": 30}
]
```
`authenticate()` finds a matching `User`, returns `(user, None)`, `IsAuthenticated` passes, and the manager list comes back normally. The video repeats this for a couple of other real usernames (e.g. "Durga"), confirming any username that exists in `auth_user` works.

**`?username=` naming a user that doesn't exist** — e.g. a typo'd or made-up name:

```http
GET /manager/?username=site HTTP/1.1
```
```json
{
  "detail": "User does not exist"
}
```
`User.objects.get(...)` raises `DoesNotExist`, the `except` block catches it and raises `AuthenticationFailed('User does not exist')`, and that exact message is what the client sees.

**An admin-panel side note — the `is_staff` gotcha.** While testing with one particular user ("Prasad"), logging into Django's `/admin/` with that account initially fails, and the instructor explains why: a `User` created without the "staff status" checkbox ticked in the admin form cannot log into `/admin/` at all, regardless of a correct username/password — he goes back into that user's admin record and enables **staff status** (and, separately, superuser status) before the login works.

> **[Researched] — `is_staff` vs. `is_superuser` vs. `is_active`.** Per the [Django auth docs](https://docs.djangoproject.com/en/stable/ref/contrib/admin/#django.contrib.admin.ModelAdmin), these are three independent boolean flags on every `User` row: **`is_active`** — must be `True` or the account can't log in anywhere at all (this is Django's "soft delete" flag for users). **`is_staff`** — must be `True` for the account to be allowed into the `/admin/` site at all, independent of what it's allowed to do once there. **`is_superuser`** — grants *every* permission automatically, bypassing Django's normal per-model permission checks entirely (this is the flag from Lecture 26's `auth_user` table walkthrough). A user can be staff without being a superuser (e.g. an employee who should only manage a couple of models in the admin), and — as this lecture shows — a user created directly, or via a form other than `createsuperuser`, defaults to `is_staff=False`, so they simply can't get into the admin until someone flips that flag.

## 9. Security note — query strings vs. headers **[Gap-filled / Researched]**

The demo scheme reads its credential from a URL query parameter (`request.GET.get('username')`). That's fine for a five-minute classroom demo of the `authenticate()` mechanism, but it would be a poor choice for production, for a few concrete reasons:

- **No secret is checked at all** — knowing (or guessing) any existing username is enough; there's no password, token, or signature involved, so anyone who can see the `auth_user` table (or just guess common usernames) can impersonate any user.
- **Query strings leak.** URLs get logged by web servers, proxies, and browser history, and get sent in the `Referer` header to any third-party resource the page loads — a genuinely sensitive value (an API key, a session token) showing up in a URL is a well-known way for it to end up somewhere it shouldn't.
- Real-world custom schemes almost always read their credential from a **request header** instead — e.g. `request.META.get('HTTP_X_API_KEY')` or (in modern DRF/Django) `request.headers.get('X-API-Key')` — mirroring how Basic and Token authentication both already work (`Authorization: <scheme> <credential>`), and pairing the identifier with an actual secret value that's checked against a hash, not just an existence check.

> **[Researched] — `authenticate_header()`, and why the demo returns 403 instead of 401.** Per the [DRF authentication docs](https://www.django-rest-framework.org/api-guide/authentication/#custom-authentication), a `BaseAuthentication` subclass can also override a second method, `authenticate_header(self, request)`, returning a string to use as the `WWW-Authenticate` header on a `401 Unauthorized` response. If a class doesn't implement it (as in this lecture's example), DRF cannot legally send `401` (the HTTP spec requires a `WWW-Authenticate` header alongside `401`), so it falls back to returning **`403 Forbidden`** instead whenever authentication fails or is missing — which is exactly the status the video's browsable-API screenshots would be showing, even though the on-screen JSON body just says "Authentication credentials were not provided."

## 10. Example — a header-based custom scheme **[Example]**

To contrast with the query-string version above, here's a small, self-contained example of the more realistic pattern (a shared secret sent in a custom header), demonstrating the *same* `authenticate()` contract with a safer credential source:

```python
# A more production-shaped variant: a fixed per-request header, checked
# against a value stored on the User model (e.g. a "api_key" field you'd
# add via a Profile model or similar — omitted here for brevity).

class ApiKeyAuthentication(BaseAuthentication):
    def authenticate(self, request):
        api_key = request.headers.get('X-Api-Key')  # e.g. "X-Api-Key: abc123"

        if not api_key:
            return None  # no header at all -> let other schemes/anonymous apply

        try:
            user = User.objects.get(profile__api_key=api_key)
        except User.DoesNotExist:
            raise AuthenticationFailed('Invalid API key')

        return (user, api_key)  # api_key is available afterwards as request.auth

    def authenticate_header(self, request):
        return 'X-Api-Key'  # lets DRF return a proper 401 instead of 403
```

**Sample request/response**, to make the effect concrete:

```http
GET /manager/ HTTP/1.1
X-Api-Key: abc123
```
→ `200 OK` with the manager list, if `abc123` matches some user's stored key.

```http
GET /manager/ HTTP/1.1
X-Api-Key: wrong-key
```
→ `401 Unauthorized`, body `{"detail": "Invalid API key"}`.

## 11. A first look at filtering **[From video]**

The transcript then shifts topic — the instructor explicitly frames this as a preview ("we'll finish this filter [topic] in tomorrow's session"), so treat it as an introduction only; REST API Session 18 covers filtering (including DRF's filter backend classes) in full.

> Filters means we can filter the data based on our requirement — generally the default behavior of a generic list view is to return the entire queryset of a model manager. Often you will want your API to restrict the items returned by the queryset. The simplest way to filter the queryset of any view that subclasses `GenericAPIView` is by overriding the `get_queryset()` method.

**[Gap-filled] — restating that plainly.** A "generic list view" like `ListAPIView` (covered in the generic-views lectures) normally just does `Model.objects.all()` under the hood and returns every row. **Filtering**, in the DRF sense introduced here, means making that view return a *subset* instead — e.g. only the rows belonging to whoever is currently logged in — by writing your own `get_queryset()` method that replaces the default "return everything" behavior with your own query.

### The new demo: a `myapp1` app with a `Student` model

A brand-new app is created purely to demonstrate this cleanly:

```bash
python manage.py startapp myapp1
```
...and registered in `INSTALLED_APPS` in `settings.py`, same as any new app.

```python
# myapp1/models.py
from django.db import models
from django.contrib.auth.models import User

class Student(models.Model):
    # A foreign key back to Django's own User model — this is what makes
    # "filter by whoever is logged in" possible later on. Each Student row
    # belongs to exactly one User.
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    name = models.CharField(max_length=100)
    s_id = models.IntegerField()          # renamed from an initial "id"-style field via a migration
    s_address = models.CharField(max_length=200)
    mail = models.EmailField()
    age = models.IntegerField()
```

**[Gap-filled] — reconstructing this model.** The transcript's audio badly garbles the field name used in `get_queryset()` (transcribed by the speech-to-text engine as the nonsense phrase "trying to buy," repeated many times). Piecing it together from context — the instructor repeatedly says things like *"filter... whoever is logged in... that person's records only"*, and separately shows creating each `Student` admin record by picking "the user" from a dropdown — the field in question is almost certainly a **`user` foreign key** on the `Student` model, and the filtering call is `Student.objects.filter(user=user)`. That reconstruction (not a direct quote) is what's used throughout this section.

After defining the model, the instructor runs `makemigrations`/`migrate`, registers `Student` in `admin.py`, and (mid-demo) renames a couple of fields (`id` → `s_id`, `address` → `s_address`), which requires running `makemigrations` again — Django's migration tool detects the rename and asks "did you rename `student.address` to `student.s_address`?", which the instructor confirms.

Several `Student` records are then added through the Django admin, each one assigned to a different existing `User` (Mohan, Durga, Prasad, and others) via that `user` foreign-key dropdown — this is the data the filtering demo will query against.

### Serializer and view

```python
# myapp1/serializers.py
from rest_framework import serializers
from myapp1.models import Student

class StudentSerializer(serializers.ModelSerializer):
    class Meta:
        model = Student
        fields = ['id', 'name', 's_id', 's_address', 'mail', 'age']
```

```python
# myapp1/views.py
from rest_framework import generics
from myapp1.models import Student
from myapp1.serializers import StudentSerializer

class StudentList(generics.ListAPIView):
    serializer_class = StudentSerializer

    def get_queryset(self):
        # self.request is available on any generic view -- it's the same
        # HttpRequest DRF wraps for every call. .user is whoever the
        # currently active authentication scheme identified (or
        # AnonymousUser if nobody did).
        user = self.request.user
        return Student.objects.filter(user=user)
```

The instructor first shows the view **without** the override (`queryset = Student.objects.all()`) — hitting `/student/` (or similar; the URL path itself isn't dictated in full) returns every student row, regardless of who's logged in. Then, after switching to the `get_queryset()` override shown above, the same request returns **only** the rows belonging to whichever user is currently logged in — logging in as "Mohan" shows only Mohan's students; logging out and logging in as "Prasad" instead shows only Prasad's.

> **[Example] — the effect, concretely.** Suppose `Student` has four rows: two owned by user `mohan`, one by `durga`, one by `prasad`. With the plain `Student.objects.all()` queryset, `/student/` always returns all four rows to anyone. With `get_queryset()` overridden as above, logging in as `mohan` and hitting `/student/` returns only mohan's two rows; the same URL, hit while logged in as `durga`, returns only durga's one row. Same endpoint, same code, different results — because the queryset is now computed per-request from `self.request.user` instead of being a fixed class attribute.

> **[Researched] — the difference between `queryset` and `get_queryset()`.** Per the [DRF generic views docs](https://www.django-rest-framework.org/api-guide/generic-views/#get_querysetself), setting `queryset = Student.objects.all()` as a class attribute is evaluated **once**, when the class is defined — it's a single, fixed queryset shared across every request. Overriding `get_queryset(self)` as a method instead means DRF calls it fresh **on every request**, so it can depend on things that change per-request — like `self.request.user`, URL parameters, or query-string filters. Any time a queryset needs to vary based on who's asking, `get_queryset()` is the correct tool; a plain `queryset =` attribute is only appropriate when the same data should be returned to everyone.

### What's coming next **[From video]**

The video closes by naming what's still ahead in the filtering topic — "Django filter backend," "search filters," and other filter backends — promising they'll be finished "tomorrow" (i.e., the next session, REST API Session 18), followed by pagination, and then throttling, with a goal of wrapping up the DRF unit's projects "by Saturday."

---

## Wrap-up

- **From video:** the full `authenticate(self, request)` contract (tuple / `None` / `AuthenticationFailed`) for a custom `BaseAuthentication` subclass; building `CustomAuthentication` step by step in a new `authentication.py`; wiring it into `ManagerViewSet` via `authentication_classes`/`permission_classes`; testing it via a `?username=` query parameter (missing → "credentials not provided," valid → success, invalid → "User does not exist"); the false start of trying to authenticate via the browsable API's session login; the `is_staff` admin-login gotcha; the opening of the filtering topic — default queryset behavior, overriding `get_queryset()`, and a `Student`/`user`-foreign-key demo filtering records to the logged-in user.
- **Gap-filled:** the "not attempted vs. failed" distinction and why it matters; how `authentication_classes` and `permission_classes` cooperate; why the browsable-API login didn't satisfy this particular custom scheme; the no-password-check caveat on the demo scheme; reconstructing the badly garbled `Student.user` foreign key and its role in the `get_queryset()` filter, from repeated context rather than a clean quote.
- **Researched:** `is_staff`/`is_superuser`/`is_active` as three independent flags (Django auth docs); `authenticate_header()` and why a scheme without it returns `403` instead of `401` on failure (DRF docs); query-string vs. header-based credentials as a security consideration; the `queryset` attribute vs. `get_queryset()` method distinction (DRF generic views docs).
- **Example:** a header-based `ApiKeyAuthentication` class contrasting with the video's query-string scheme, with sample request/response pairs; a worked before/after example of `get_queryset()` filtering by user.

Double-check against the checklist in section "Completeness checklist" above — all 20 items are covered. The one thing to flag for cross-referencing: this transcript's second half genuinely belongs partly to the filtering topic that REST API Session 18 covers in full — these notes include it because it's in *this* transcript, but expect REST API Session 18 to revisit and extend it (particularly the "filter backend"/search-filter material only named here, not yet demonstrated).
