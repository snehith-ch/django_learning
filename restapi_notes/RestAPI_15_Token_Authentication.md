# REST API Session 15 — Token Authentication

Source: `transcripts/restapi/REST API-15.txt`
Covers: DRF's built-in **`TokenAuthentication`** scheme — installing `rest_framework.authtoken`, migrating to create its token table, and the four different ways DRF lets you generate a token for a user (admin interface, a `manage.py` command, an exposed API endpoint, and signals). The lecture builds a small demo app (a `Manager` model, serializer, and `ViewSet`) from scratch to have something to attach tokens to, then walks through generating tokens via the admin interface and the `manage.py` command live. The third method (exposing an endpoint, using a tool called HTTPie to call it) got blocked by an installation error on the instructor's machine and is picked up properly in the next lecture — these notes fill in that piece from DRF's official docs, clearly labeled as researched rather than transcribed.

Legend:
- **[From video]** — explained or demonstrated directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped, rushed, or garbled something
- **[Researched]** — pulled from the official [DRF docs](https://www.django-rest-framework.org/) or [Django docs](https://docs.djangoproject.com/)
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. Recap and where this fits **[From video]**

The instructor opens by recapping: the previous sessions covered **basic authentication** and **session authentication** (REST API Session 13), and mentioned that **custom authentication** would be covered later (this turns out to be REST API Session 17 — this lecture doesn't build a custom scheme, so treat that as a forward pointer, not something covered here). Today's topic is **token authentication** — a third built-in DRF authentication scheme, alongside basic and session authentication.

> **[Gap-filled] — how the three schemes relate.** DRF ships several authentication classes out of the box, and they aren't mutually exclusive — a project can enable more than one at once (DRF tries each in turn until one succeeds, or falls through to "unauthenticated"). Basic authentication (REST API Session 13) sends a username/password pair, base64-encoded, on *every single request*. Session authentication (also touched on in REST API Session 13) relies on Django's own login/cookie/session machinery — it's what powers the browsable API's "Log in" link. Token authentication, this lecture's topic, is a middle ground: you authenticate **once** to obtain a token, then send that one token (not your password) on every subsequent request.

## 2. Setting up a fresh project **[From video]**

Because the instructor was on a new machine, the lecture starts by creating a brand-new Django project from scratch in PyCharm (project + one app, e.g. `myapp`), then installing Django REST Framework into it with pip:

```bash
# Upgrade pip first (not required, but the video does it to avoid warnings)
python -m pip install --upgrade pip

# Install Django REST Framework into the project's environment
pip install djangorestframework
```

As with every DRF project (see REST API Session 2), `'rest_framework'` then has to be added to `INSTALLED_APPS` in `settings.py` — without this, none of DRF's features (serializers, views, the browsable API) are available at all.

```python
# settings.py
INSTALLED_APPS = [
    # ...Django's own default apps...
    'rest_framework',   # required for any DRF project
    'myapp',
]
```

> **[Gap-filled]** None of this setup is specific to token authentication — it's the same "turn a plain Django project into a DRF project" step from REST API Session 2, repeated here only because the instructor switched machines. If your project already has `rest_framework` installed and listed, you can skip straight to Section 3.

## 3. What token authentication is **[From video]**

> Token authentication is a simple, token-based HTTP authentication scheme. It's appropriate for client-server setups such as native desktop and mobile clients.

Breaking that down for a beginner:

- A **token** here is just a long, random, opaque string (DRF generates a 40-character hex string by default) that stands in for "prove who you are" — instead of sending a username and password with every request, the client sends this one string.
- **"Client-server setups such as native desktop and mobile clients"** means: token authentication is the natural fit when the thing calling your API isn't a web browser rendering HTML pages, but a separate program — a mobile app, a desktop app, another backend service, a single-page JavaScript app — that just wants to talk to your API directly over HTTP.
- The token is obtained **once** (by logging in, or by an admin creating it), then reused for every request after that, in an `Authorization` header, until it's revoked or regenerated.

> **[Researched] — why not just use session authentication for everything?** Per the [DRF authentication docs](https://www.django-rest-framework.org/api-guide/authentication/), session authentication relies on Django's session framework, which is backed by cookies — and cookies are a browser concept. A mobile app or a script calling your API with `requests`/`curl` doesn't have a browser's cookie jar, and even if it did, session auth needs a CSRF token for unsafe methods (POST/PUT/DELETE) which adds friction for a non-browser client. Token authentication sidesteps both problems: no cookies, no CSRF token, just one header sent explicitly on every call. That's exactly why it's called out for "native desktop and mobile clients" — those clients can easily store and attach a header, but don't have (or don't want) a browser's session/cookie machinery.

## 4. Configuring token authentication in settings.py **[From video]**

Getting `TokenAuthentication` working needs **two separate pieces**, and the video is explicit that both are compulsory:

1. **Add `'rest_framework.authtoken'` to `INSTALLED_APPS`** — this is a small app that ships *inside* `djangorestframework` itself, containing just one model (`Token`) and its admin registration. Without it, Django has no database table to store tokens in, and the admin panel won't show a "Tokens" section at all.
2. **Add `TokenAuthentication` to the authentication classes** DRF uses — either globally (in a `REST_FRAMEWORK` settings dict) or per-view.

```python
# settings.py
INSTALLED_APPS = [
    # ...
    'rest_framework',
    'rest_framework.authtoken',   # <- required specifically for token authentication
    'myapp',
]

# Tell DRF, project-wide, to accept tokens as proof of identity
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.TokenAuthentication',
    ],
}
```

Then, because `rest_framework.authtoken` defines a new model (`Token`), Django needs to create a table for it in the database — the same **migrate** step used any time a new model shows up:

```bash
python manage.py migrate
```

> **[Gap-filled] — a common pitfall this step guards against.** If you add `TokenAuthentication` to your settings but forget to add `'rest_framework.authtoken'` to `INSTALLED_APPS` (or forget to run `migrate` afterward), you'll get errors the moment DRF tries to look up a token — there's no `authtoken_token` table for it to query. Both halves of Section 4 are required together; skipping either one breaks the feature.

<div class="code-note-md">

**Note on `DEFAULT_AUTHENTICATION_CLASSES` vs. permissions:** setting `TokenAuthentication` here only controls *how* DRF figures out who's making a request (by reading the `Authorization` header). It does **not**, by itself, block anyone — that's the job of **permission classes** (REST API Session 14, e.g. `IsAuthenticated`). Authentication answers "who is this?"; permissions answer "are they allowed to do this?". You typically need both configured for token auth to actually restrict access to anything.

</div>

## 5. What DRF gives you once a request is authenticated by token **[From video, corrected]**

The transcript reads out (garbled) a summary that's actually a direct paraphrase of the DRF docs:

> If successfully authenticated, `TokenAuthentication` provides `request.user` as a Django `User` instance, and `request.auth` as a `rest_framework.authtoken.models.Token` instance. Unauthenticated responses that fail authentication will result in an HTTP 401 Unauthorized response.

- **`request.user`** — inside any view, once a request carries a valid token, `request.user` is set to the actual Django `User` object that token belongs to (so you can do things like `request.user.username`, or filter a queryset by `request.user`, exactly as with any other authenticated request).
- **`request.auth`** — is set to the `Token` object itself (not just the string) — useful if you ever need to inspect the token record, e.g. its `created` timestamp.
- **A request with no token, or an invalid/unknown token,** gets rejected with an HTTP status code in the 401/403 family, instead of `request.user` being set.

> **[Gap-filled] — the transcript garbled the status code as "404."** 404 means *Not Found* (wrong URL/resource) — that's not what happens here; the URL and view both exist, the request is just not authenticated. The correct code, per the [DRF docs](https://www.django-rest-framework.org/api-guide/authentication/#tokenauthentication), is **401 Unauthorized** (with a `WWW-Authenticate: Token` response header), or **403 Forbidden** if that header is suppressed. This is worth getting right: 401 tells a client "you need to (re-)authenticate," which is very different advice from 404's "this doesn't exist."

## 6. Building a demo model to attach tokens to **[From video]**

Before generating any tokens, the video builds a small, self-contained DRF-backed app — a `Manager` model, with a matching serializer and viewset — purely so there's a real API and a real user to test token authentication against later.

### The model

```python
# models.py
from django.db import models

class Manager(models.Model):
    name = models.CharField(max_length=20)
    address = models.CharField(max_length=20)
    email = models.EmailField(max_length=20)   # video uses EmailField (CharField also mentioned as an option)
    age = models.IntegerField()
```

> **[Gap-filled] — naming clash to be aware of.** Calling this model `Manager` is a coincidence of the demo's business domain (it represents a company manager — name/address/email/age), **not** Django's own `Manager` concept (the `objects` attribute every model gets, e.g. `Manager.objects.all()`, is itself an instance of `django.db.models.Manager`). It works fine as a model name — Python and Django don't confuse the two — but it's a slightly confusing choice for teaching material; in your own projects, prefer a name that doesn't double as a core Django term (e.g. `Employee`, `StaffMember`).

### Registering it in the admin

```python
# admin.py
from django.contrib import admin
from .models import Manager

class ManagerAdmin(admin.ModelAdmin):
    list_display = ('id', 'name', 'address', 'email', 'age')  # controls the columns shown in the admin list view

admin.site.register(Manager, ManagerAdmin)
```

This is the same `ModelAdmin` + `list_display` pattern from Lecture 24 — nothing token-authentication-specific here, it just makes the demo data easy to enter and see.

### Migrations, a superuser, and two sample records

```bash
python manage.py makemigrations
python manage.py migrate
python manage.py createsuperuser   # video creates a user "mohan" here
```

After logging into `/admin/` with that superuser, the video adds two `Manager` records by hand through the admin form (one named "Siz"/similar, one named "Ram") — just sample data, not something with any special meaning.

> **[Gap-filled]** The video's demo passwords (e.g. the superuser's password being the same as the username) are fine for a disposable local demo project, but never do this in anything real — see the best-practices note in Section 13.

## 7. The serializer **[From video]**

```python
# serializers.py
from rest_framework import serializers
from myapp.models import Manager

class ManagerSerializer(serializers.ModelSerializer):
    class Meta:
        model = Manager
        fields = ['id', 'name', 'address', 'email']   # video reuses the same field list from list_display
```

This is the `ModelSerializer` pattern from REST API Session 3/48 — its main advantage, as the video notes, is that you don't have to redeclare every field by hand; you point it at the model and list which fields to expose.

> **[Gap-filled]** The video's `fields` list (copied from the admin's `list_display`) leaves out `age`. That's likely just following the admin list along without re-checking it, rather than a deliberate choice — in your own code, make sure the serializer's `fields` list actually matches what you intend clients to see, rather than mirroring an unrelated list by habit. (Using `fields = '__all__'` is the simplest way to expose every model field without maintaining a separate list.)

## 8. The ViewSet and router-based URLs **[From video]**

```python
# views.py
from myapp.models import Manager
from myapp.serializers import ManagerSerializer
from rest_framework import viewsets

class ManagerModelViewSet(viewsets.ModelViewSet):
    queryset = Manager.objects.all()
    serializer_class = ManagerSerializer
    # authentication_classes / permission_classes deliberately left out for now —
    # the video is still in the process of generating tokens, not yet enforcing them.
```

```python
# urls.py (project-level, per the video's choice)
from django.contrib import admin
from django.urls import path, include
from myapp import views
from rest_framework.routers import DefaultRouter

router = DefaultRouter()
router.register('managerview', views.ManagerModelViewSet, basename='manager')

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include(router.urls)),
]
```

This is the `ViewSet` + `DefaultRouter` combination from REST API Session 12 — one `ModelViewSet` class automatically gets list/retrieve/create/update/delete behavior, and `router.register(...)` wires up all the corresponding URLs (`/managerview/`, `/managerview/<pk>/`) in one step, instead of writing each `path()` by hand.

> **[Gap-filled]** Notice `authentication_classes`/`permission_classes` are *not* set on this viewset yet — the video is being deliberate about sequencing: generate tokens first, wire up enforcement second (that pairing is picked up in REST API Session 16). Right now, with only the project-wide `DEFAULT_AUTHENTICATION_CLASSES` set and no permission classes anywhere, this endpoint is still open to anyone — token auth alone doesn't lock a view down (see the note at the end of Section 4).

## 9. Four ways to generate a token **[From video]**

The instructor lists four different ways to create a `Token` for a user, then demonstrates the first two live:

1. **Using the admin interface** — add a `Token` row by hand, picking a user.
2. **Using a `manage.py` command in the terminal** — a management command shipped with `rest_framework.authtoken`.
3. **By exposing an API endpoint** — a login-style endpoint that a client calls with a username/password and gets a token back in the response.
4. **Using signals** — automatically create a token the moment a new `User` is created.

> **[Researched]** All four of these are documented, standard approaches in the [DRF docs' "Generating Tokens" section](https://www.django-rest-framework.org/api-guide/authentication/#generating-tokens) — the video's list matches it, just in a different order (the docs lead with signals and the endpoint; the video demos admin and the management command first because they're the simplest to show quickly).

## 10. Option 1 — generating a token via the admin interface **[From video]**

Once `'rest_framework.authtoken'` is in `INSTALLED_APPS` and migrations have run, the Django admin (`/admin/`) automatically gains a new section: **Tokens** (registered by the `authtoken` app itself — no extra code needed). The video demonstrates:

1. Run the dev server (`python manage.py runserver`) and visit `/admin/`.
2. Click **Tokens → Add token**.
3. Pick a user from the dropdown (the video picks the superuser it created earlier, "mohan").
4. Click **Save** — a 40-character token string is generated automatically and now belongs to that user.

> **[Gap-filled]** You never type the token string yourself — DRF generates it (a random 40-character hex key) the moment you save the form; the admin form only asks you *which user* the token belongs to.

## 11. Option 2 — generating a token via `manage.py` **[From video]**

`rest_framework.authtoken` also ships a custom management command, usable straight from the terminal:

```bash
python manage.py drf_create_token <username>

# video's example — creating a token for a second user, "shanvi"
python manage.py drf_create_token shanvi
```

This prints the generated token straight to the terminal, and it also now shows up as a new row in the admin's Tokens list.

> **[From video] — an important behavior detail.** The instructor deliberately re-runs the command for a user who *already* has a token ("mohan," from the admin-interface demo in Section 10) to show what happens: **it does not create a second token.** Instead it prints the same existing token back. The rule, stated directly in the video: "if the user has [a] token, it will return that token; if the user doesn't have any token, then it will create a new token."

> **[Researched] — regenerating a compromised token.** Per the DRF docs, if you need to force a *new* token for a user who already has one (e.g. their old token leaked), `drf_create_token` accepts a `-r` (`--reset`) flag to regenerate it: `python manage.py drf_create_token -r <username>`. This deletes the old token and issues a fresh one — necessary because, as shown above, running the command without `-r` just hands back the existing token unchanged.

> **[Gap-filled] — why a user can only have one token by default.** DRF's built-in `Token` model links to `User` with a `OneToOneField` — meaning, by design, each user has **at most one** token at a time (not one-per-device, one-per-app-install, etc.). That's why "create a token" behaves like "get-or-create" rather than "always make a new one." If your app needs multiple tokens per user (e.g. a separate token per logged-in device, so one device can be logged out without affecting others), you'd need a custom token model — DRF's default `TokenAuthentication` doesn't support that out of the box.

## 12. Option 3 — generating a token by exposing an API endpoint **[From video attempt, completed via research]**

The video's plan for this option: use a command-line HTTP client called **HTTPie** to call an endpoint and get a token back in the response, instead of using the admin or the terminal command.

> HTTPie is a command-line HTTP client to make CLI (command-line) interaction with web services as human-friendly as possible. It provides a simple `http` command that allows sending arbitrary HTTP requests using simple, natural syntax, and displays beautifully colorized output.

The video tries to install it with `pip install httpie`, but the install fails on the instructor's machine — a wheel-build error for a dependency (`multidict`), which the instructor attributes to a Windows 11 compatibility issue and defers to "we'll solve it and continue in tomorrow's session" (i.e., this is picked up in REST API Session 16, not completed here).

> **[Researched] — filling in what this option actually looks like, since the demo didn't finish.** DRF ships a ready-made view for exactly this purpose: `rest_framework.authtoken.views.obtain_auth_token`. Wiring it up needs one URL entry:
>
> ```python
> # urls.py
> from rest_framework.authtoken.views import obtain_auth_token
>
> urlpatterns += [
>     path('api-token-auth/', obtain_auth_token),
> ]
> ```
>
> A client then `POST`s a username and password to that URL, and gets a token back as JSON — no login page, no cookies, just one request/response:
>
> ```bash
> # what HTTPie was going to be used for — the same call also works with curl:
> curl -X POST http://127.0.0.1:8000/api-token-auth/ \
>      -d "username=shanvi&password=mohan@123"
> ```
>
> ```json
> {"token": "9944b09199c62bcf9418ad846dd0e4bbdfc6ee4"}
> ```
>
> This is the "login-style" flow real client apps actually use in practice: a mobile app's login screen collects a username/password once, calls this endpoint, and stores the returned token locally (never the password) for every future request. It's also possible to subclass `ObtainAuthToken` to return extra fields (like the user's ID or email) alongside the token — useful so the client doesn't need a second request just to know who it logged in as.

## 13. Option 4 — generating a token via signals **[Researched — named but not demoed in the video]**

The instructor names this option ("using signals... we can able to expose API endpoints clearly here also we will discuss") but doesn't demonstrate it in this lecture.

> **[Researched]** The DRF docs' recommended pattern is a `post_save` signal on the `User` model, so that **every newly created user automatically gets a token the moment they sign up** — no separate step required at all:
>
> ```python
> # signals.py (or directly in an apps.py AppConfig.ready())
> from django.conf import settings
> from django.db.models.signals import post_save
> from django.dispatch import receiver
> from rest_framework.authtoken.models import Token
>
> @receiver(post_save, sender=settings.AUTH_USER_MODEL)
> def create_auth_token(sender, instance=None, created=False, **kwargs):
>     # `created` is True only on the very first save (i.e. a brand-new user)
>     if created:
>         Token.objects.create(user=instance)
> ```
>
> This is the option best suited to a real sign-up flow: instead of an admin manually creating tokens (Section 10) or someone running a terminal command (Section 11) for each new user, the token simply exists the instant the account does.

## 14. Actually using the token: the `Authorization: Token <key>` header **[Researched/Gap-filled — the goal stated in the video, completed from DRF docs]**

The video is explicit that generating a token is only half the job — "even once we get the tokens, we have to test the API by using the tokens... whether the API is working or not" — but runs out of time before demonstrating an authenticated call (this is what REST API Session 16, "Token authentication in practice," actually walks through). Here's how it works, straight from the DRF docs, so the concept isn't left half-finished:

Once a client has a token (from any of the four methods above), every subsequent request includes it in the HTTP `Authorization` header, using the keyword `Token`:

```http
GET /managerview/ HTTP/1.1
Host: 127.0.0.1:8000
Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4
```

```bash
# the same request with curl
curl http://127.0.0.1:8000/managerview/ \
     -H "Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4"
```

> **[Gap-filled] — contrasting this with REST API Session 13's basic authentication header.** Basic authentication's header looks similar in shape but very different in content: `Authorization: Basic <base64(username:password)>` — literally the account's password, re-encoded (not encrypted — base64 is trivially reversible) and resent on *every single request*. Token authentication's header, `Authorization: Token <key>`, never contains the password at all after the initial login — only an opaque key that can be revoked/regenerated independently of the account's actual password. That difference is the main practical reason token authentication is preferred for anything beyond a quick demo: a leaked token can be deleted and reissued without forcing the user to change their password.

> **[Researched] — the keyword is case-sensitive and configurable.** Per the docs, the header keyword defaults to the literal word `Token`; a project can change it (e.g. to `Bearer`, matching common convention for other token schemes like OAuth2/JWT) by subclassing `TokenAuthentication` and setting its `keyword` attribute.

## 15. Best practices and pitfalls **[Gap-filled/Researched]**

> **Industry best practice — token authentication requires HTTPS in production.** Because the token is sent as plain text in a header on every request (no encryption of its own), anyone intercepting an unencrypted HTTP connection can read it and impersonate that user until the token is revoked. Django/DRF projects handling real users should always be served over HTTPS; token auth is not "safe by default" over plain HTTP.

> **Industry best practice — treat a leaked token like a leaked password.** Because DRF's token doesn't expire on its own (no built-in time limit) and stays valid until explicitly deleted/regenerated, a compromised token is a standing risk until someone notices and resets it (`drf_create_token -r <username>`, or deleting the row in the admin). Some teams add their own expiry logic on top (checking the token's `created` timestamp in custom middleware/authentication) since DRF doesn't do this itself.

> **[Researched] — how this compares to JWT (JSON Web Tokens), a very common alternative.** DRF's built-in token is intentionally simple: an opaque random string, looked up in the database on every request to find the matching user (one query per request). **JWT**, a different and very popular token format (not built into DRF itself — it needs a third-party package like `djangorestframework-simplejwt`), instead encodes the user's identity *inside* the token itself (cryptographically signed), so the server can verify it without a database lookup, and commonly supports built-in expiry/refresh. DRF's simple `TokenAuthentication` is a perfectly good, easy starting point (and is what this course teaches); JWT is worth knowing about as the more scalable option many production APIs reach for once they need token expiry or want to avoid a database hit on every request.

> **Common pitfall — enabling `TokenAuthentication` without any permission class.** As flagged in Section 4 and Section 8, adding `TokenAuthentication` to `DEFAULT_AUTHENTICATION_CLASSES` only tells DRF *how* to identify a user from a token if one is present — it doesn't require one. Without a permission class like `IsAuthenticated` (REST API Session 14) on the view, anonymous requests (no `Authorization` header at all) are still allowed through; `request.user` just ends up being Django's `AnonymousUser` instead of a real user.

## 16. Worked example — the full cycle, start to finish **[Example]**

Putting Sections 10–14 together into one concrete walk-through, beyond what the video itself completed:

1. **Setup (once):** `'rest_framework.authtoken'` in `INSTALLED_APPS`, `TokenAuthentication` in `DEFAULT_AUTHENTICATION_CLASSES`, `python manage.py migrate`.
2. **A user signs up** (username `priya`, some password) — a `User` row is created.
3. **A token is generated for `priya`** — any of the four ways from Sections 10–13; say, via the terminal:
   ```bash
   python manage.py drf_create_token priya
   # Generated token f4a8c9e21b3d...  (example output)
   ```
4. **`priya`'s app stores that token** locally (e.g. in the mobile app's secure storage) — never her password.
5. **Every future request** from `priya`'s app includes the token:
   ```bash
   curl http://127.0.0.1:8000/managerview/ \
        -H "Authorization: Token f4a8c9e21b3d..."
   ```
   ```json
   [
     {"id": 1, "name": "Siz", "address": "Hyderabad", "email": "siz@gmail.com"},
     {"id": 2, "name": "Ram", "address": "Chennai", "email": "ram@gmail.com"}
   ]
   ```
6. **A request with no header, or a wrong/old token,** gets rejected:
   ```bash
   curl http://127.0.0.1:8000/managerview/
   ```
   ```json
   {"detail": "Authentication credentials were not provided."}
   ```
   — an HTTP `401 Unauthorized` (or `403 Forbidden`, depending on the `WWW-Authenticate` header setup), matching Section 5's corrected explanation, not the transcript's garbled "404."

---

## Wrap-up

- **From video:** what token authentication is and when it's appropriate (native/mobile/desktop clients); the two-part settings change (`rest_framework.authtoken` in `INSTALLED_APPS` + `TokenAuthentication` in the authentication classes) plus `migrate`; `request.user`/`request.auth` once authenticated; building a demo `Manager` model, admin registration, `ManagerSerializer`, and a `ManagerModelViewSet` wired up with `DefaultRouter`; the four ways to generate a token (admin, `manage.py drf_create_token`, an exposed endpoint, signals); a full live demo of the admin-interface and `manage.py` methods, including the "already has a token → returns the same one" behavior; an incomplete attempt at the endpoint method, blocked by an HTTPie install failure on Windows 11 and deferred to the next session.
- **Gap-filled:** how the three authentication schemes (basic/session/token) relate; the `Manager`-model-vs-`Manager`-class naming clash; the missing `age` field in the serializer's `fields` list; why `TokenAuthentication` alone doesn't restrict access without a permission class; why a user only gets one token by default (`OneToOneField`); contrasting the `Authorization: Token` header with basic auth's header from REST API Session 13; best-practice notes on HTTPS and treating leaked tokens like leaked passwords.
- **Researched:** the correct 401 Unauthorized status code (transcript said "404"); the full `obtain_auth_token` endpoint wiring and request/response shape (completing the demo the video didn't finish); the `-r`/`--reset` flag for regenerating a token; the signals-based auto-token-on-signup pattern; the `keyword` customization on `TokenAuthentication`; a brief comparison to JWT as a common alternative.

Double-check against the completeness checklist: recap/context ✅, project/DRF setup ✅, the definition and purpose of token authentication ✅, both required settings changes plus `migrate` ✅, `request.user`/`request.auth`/401 behavior ✅, the demo model/admin/serializer/viewset/router ✅, all four token-generation methods (two demoed, two researched) ✅, the `Authorization: Token` header usage ✅, best practices/pitfalls ✅, a standalone worked example with sample input→output ✅. Nothing from the transcript was left out; the one genuinely incomplete part of the video itself (the HTTPie/endpoint demo) is clearly marked as picked up in REST API Session 16 rather than invented here.
