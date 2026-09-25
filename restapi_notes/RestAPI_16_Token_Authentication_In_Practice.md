# REST API Session 16 — Token Authentication in Practice: Generating Tokens & Testing the API with HTTPie

Source: `transcripts/restapi/REST API-16.txt`
Covers: a hands-on continuation of REST API Session 15 (which introduced token authentication conceptually). This lecture is almost entirely a live demo: the instructor generates DRF auth tokens for users **four different ways** (admin panel, a management command, an exposed API endpoint, and a `post_save` signal), then uses a command-line HTTP client (HTTPie) to prove the token actually protects and grants access to a REST API — doing full CRUD (list, create, update, delete) against a token-protected endpoint by attaching an `Authorization: Token <key>` header to every request.

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official DRF/Django docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## A note on this transcript **[Gap-filled]**

This transcript is unusually garbled even by this course's standards — a lot of live copy-pasting into a terminal, several failed attempts, and a mid-demo detour into a Command Prompt vs. PowerShell problem. The underlying DRF mechanics are standard, well-documented patterns (the token-generation options here are close to word-for-word what the [official DRF token authentication docs](https://www.django-rest-framework.org/api-guide/authentication/#tokenauthentication) describe), so where the audio is unrecoverable, the code below reconstructs the *correct* DRF pattern rather than transcribing nonsense. Specific values that were clearly demonstrated on screen (model field names, the four token-generation methods, the HTTP verbs tested, the Windows terminal issue) are kept as described.

---

## 1. Recap — what REST API Session 15 covered **[From video]**

The instructor opens by summarizing the previous session: token-based authentication was introduced conceptually, and token generation was demonstrated in outline. Today's session is described as finishing that demonstration properly and, new for this lecture, actually **testing** a protected endpoint with a generated token.

> "In the last session I showed how to generate tokens — admin panel, the terminal, and by exposing API endpoints. And also using signals we can generate tokens. Today I'll show all of these, one by one, and then test the API using a token."

If you haven't read REST API Session 15's notes: **token authentication** is a way for an API client (a mobile app, a JavaScript frontend, a script, `curl`/HTTPie, etc.) to prove *who it is* on every request, without logging in with a session cookie the way a browser normally does. Instead of a username/password on every call, the client sends a single opaque string — the **token** — in a request header, and Django REST Framework (DRF) looks that token up to find out which user it belongs to.

## 2. The demo project's shape **[From video]**

The instructor works inside a small demo Django project (opened in PyCharm) built specifically to demonstrate this feature, with a single app containing:

- **A `Manager` model** — a plain model with fields for `name`, `address`, `email`, and `age` (the video refers to it, through transcription noise, as "manager model"/"monitor model" — it's the same model both times).
- **`ManagerAdmin`**, registering the model in `django.contrib.admin` so its records (id, name, address, email, age) are visible/editable from `/admin/`.
- **`ManagerSerializer`**, a standard DRF serializer for the `Manager` model — "included serializer, this is the common process every day," as the instructor puts it, i.e. this step is now routine after the earlier serializer lectures (44–48).
- **`ManagerViewSet`**, a DRF `ViewSet` with `queryset = Manager.objects.all()` and `serializer_class = ManagerSerializer` — the `ViewSet` pattern from REST API Session 12.
- **A `DefaultRouter`** in `urls.py`, registering the `ManagerViewSet` — again, the router pattern from REST API Session 12, generating the list/create/retrieve/update/delete URLs automatically instead of writing them by hand.
- Two imports already sitting in `views.py`, ready but *commented out / unused* until testing time: `from rest_framework.authentication import TokenAuthentication` and `from rest_framework.permissions import IsAuthenticated`. The instructor is explicit that these are **not needed yet** while only *generating* tokens — they only matter once you want to *protect* an endpoint and *test* that protection, which is the second half of this lecture.

```python
# serializers.py
from rest_framework import serializers
from .models import Manager

class ManagerSerializer(serializers.ModelSerializer):
    class Meta:
        model = Manager
        fields = '__all__'   # id, name, address, email, age
```

```python
# views.py — the "generation" phase: no auth/permission classes active yet
from rest_framework import viewsets
from .models import Manager
from .serializers import ManagerSerializer

class ManagerViewSet(viewsets.ModelViewSet):
    queryset = Manager.objects.all()
    serializer_class = ManagerSerializer
```

```python
# app urls.py
from rest_framework.routers import DefaultRouter
from .views import ManagerViewSet

router = DefaultRouter()
router.register('manager', ManagerViewSet, basename='manager')

urlpatterns = router.urls
```

## 3. The one required settings change: `rest_framework.authtoken` **[From video, expanded]**

The instructor stresses this is the step people most often forget, calling it out directly:

> "Most important thing — especially when you're working with token authentication — in `settings.py` you must include `rest_framework.authtoken` in `INSTALLED_APPS`. Only then can you generate tokens."

```python
# settings.py
INSTALLED_APPS = [
    # ... django's default apps ...
    'django.contrib.admin',
    'django.contrib.auth',
    # ...
    'rest_framework',
    'rest_framework.authtoken',   # <-- REQUIRED for any of the 4 token methods below
    'myapp',
]
```

> **[Gap-filled] — why this line is required, and a step the video never says out loud.** `rest_framework.authtoken` is a small, self-contained Django app that DRF ships with. Adding it to `INSTALLED_APPS` gives you three things at once: (1) a `Token` **model** (one row per user, holding a random 40-character hex key), (2) that model **registered in the admin site** — which is exactly the "Tokens" section the video clicks into — and (3) a management command, `drf_create_token`. Because it's a normal Django app with its own model, Django needs to create a database table for it — **you must run `python manage.py migrate` after adding it to `INSTALLED_APPS`** (and after any `makemigrations`, if applicable) before any of the four generation methods below will work. The video skips over this step entirely, presumably because it was already done earlier off-screen, but it's a genuinely common stumbling block: forgetting the migrate step here produces a "no such table: authtoken_token" error the first time you try to generate a token.

## 4. Four ways to generate a token **[From video]**

The instructor demonstrates all four, "one by one, quickly," reiterating that the first two were already shown in REST API Session 15 and are being repeated for completeness before the two newer ones (endpoint + signal) get the main focus.

### 4.1 Via the Django admin panel

1. Run the server (`python manage.py runserver`) and visit `/admin/`.
2. Log in with an admin/superuser account.
3. Because `rest_framework.authtoken` is installed, a **Tokens** section appears in the admin index alongside your app's own models.
4. Click **Tokens → Add Token**, pick a user from the dropdown, and click **Save**. A random token is generated and immediately associated with that user.

```
Admin → Tokens → Add token
    User: [ dropdown of existing users ]
    Save  →  a 40-character hex key is generated for that user
```

The video shows several users (Kiran, Shanvi, Mohan) that already have tokens from REST API Session 15, confirming a user can only ever have **one** token via this model (it's a one-to-one relationship between `User` and `Token` by default).

### 4.2 Via the `manage.py` management command

```bash
python manage.py drf_create_token <username>
# e.g.
python manage.py drf_create_token mohan
```

The instructor explains the important behavioral detail here:

> "If the user already has a token, that token will just be returned in the response. But if the user doesn't have a token, a new one will automatically be generated."

So this command is **safe to re-run** — it's not going to silently create a second, conflicting token for a user who already has one; it just reports back the existing one. The video demonstrates this against `mohan`, who already had a token (`b9…60`), and the command echoes that same token back rather than replacing it.

> **[Researched] — the regenerate flag.** The transcript doesn't mention it, but per the [DRF docs](https://www.django-rest-framework.org/api-guide/authentication/#generating-tokens), passing `-r` (or `--reset`) forces a *new* token to be generated even if one already exists, invalidating the old one: `python manage.py drf_create_token -r mohan`. This is useful if a token has leaked and needs to be rotated.

### 4.3 By exposing an API endpoint (`obtain_auth_token`)

This is the method the video spends the most time on, and it's also where most of the transcript's confusion happens (copy-paste mistakes, a Command Prompt problem — see §6). The steps demonstrated:

**Step 1 — install HTTPie**, a command-line HTTP client (the transcript garbles this repeatedly as "HTTP Pi" — it's [HTTPie](https://httpie.io/)):

```bash
pip install httpie
```

> "HTTPie is a command-line HTTP client, made to be as human-friendly as possible — it lets you send arbitrary HTTP requests using simple, natural syntax, and displays beautifully colorized output."

> **[Gap-filled] — what HTTPie is, for anyone who hasn't used it or `curl` before.** Up to this point in the course, the API was only ever tested through DRF's own **browsable API** (a webpage DRF auto-generates that lets you click buttons to GET/POST/PUT/DELETE) — see REST API Session 4/50. HTTPie is the command-line equivalent: instead of clicking through a web page, you type one line in a terminal and it fires the HTTP request for you, which is faster for repeated testing and is what real API clients (scripts, other services) effectively do under the hood. `curl` is the older, more famous tool that does the same job with less friendly syntax; HTTPie was built specifically to be easier to read and write. Either works for testing an API like this one — HTTPie is just what this course uses.

**Step 2 — expose a URL that hands out tokens.** DRF ships a ready-made view for exactly this, `obtain_auth_token`, which the instructor wires up in the *project-level* `urls.py`:

```python
# project urls.py
from django.contrib import admin
from django.urls import path, include
from rest_framework.authtoken import views as authtoken_views

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('myapp.urls')),      # the ManagerViewSet router
    path('get-token/', authtoken_views.obtain_auth_token),
]
```

`obtain_auth_token` is a built-in DRF view: **POST** it a `username` and `password`, and if they're valid, it responds with `{"token": "<the user's token>"}` — generating a new token for that user the first time, or returning their existing one on subsequent calls (same "safe to re-call" behavior as the management command in §4.2).

**Step 3 — call it with HTTPie:**

```bash
http POST http://127.0.0.1:8000/get-token/ username=durga password=mohan@123
```

```json
// 200 OK
{
    "token": "a639f4e1c2b8d7a05e3f9c1b6a8d4e2f7c9b1a30"
}
```

The instructor first creates a plain (non-superuser) user, "durga," through the admin panel with no token yet, then runs the command above to prove a fresh token gets generated for a brand-new user purely by hitting this endpoint with valid credentials — no admin panel, no terminal management command needed.

> **[Gap-filled] — why the video briefly "comments out" some code here.** Partway through, the instructor mentions commenting out the signal-based code from §4.4 before testing the endpoint method, explaining they don't want two generation mechanisms firing at once while demonstrating this one in isolation. This isn't a conflict in principle (a user could get a token from the signal at creation time, and separately have it looked up again by the endpoint) — it's just the instructor keeping each demo clean and easy to follow, one mechanism at a time.

> **[Researched] — the endpoint's URL name and a security note.** DRF's own docs use the path name `api-token-auth/` in their examples rather than `get-token/` — either name works, it's just a URL you choose. More importantly: **anyone can call this endpoint with a guessed username and any password**, and it will happily tell them "invalid credentials" or return a token — so this view effectively becomes a login/password-checking surface exposed over your API. In production, it's standard practice to rate-limit this endpoint (see the throttling lecture later in this course) so it can't be used to brute-force passwords, and to always serve it over HTTPS so tokens and passwords aren't sent in plain text.

### 4.4 Via a `post_save` signal (automatic, at user-creation time)

The last method: generate a token *automatically*, the instant a new `User` is created — no manual step at all.

```python
# models.py (or a dedicated signals.py)
from django.conf import settings
from django.db.models.signals import post_save
from django.dispatch import receiver
from rest_framework.authtoken.models import Token

@receiver(post_save, sender=settings.AUTH_USER_MODEL)
def create_auth_token(sender, instance=None, created=False, **kwargs):
    if created:
        Token.objects.create(user=instance)
```

Line by line:

- `@receiver(post_save, sender=settings.AUTH_USER_MODEL)` — registers this function to run automatically every time a `save()` succeeds on the user model (`django.contrib.auth.models.User` by default, referenced here as `settings.AUTH_USER_MODEL` so it also works with a custom user model).
- `post_save` — a Django **signal**: an event that fires *after* a model instance has been saved to the database. `signal` here is the actual DRF/Django term the transcript garbles as "signaling."
- `def create_auth_token(sender, instance=None, created=False, **kwargs):` — the function Django calls whenever that signal fires. `instance` is the `User` object that was just saved; `created` is `True` only on the very first save (i.e. when the user is being created, not updated).
- `if created:` — only generate a token on creation, not on every subsequent edit to that user.
- `Token.objects.create(user=instance)` — creates a new row in the `Token` table, linked to this exact user.

> "As soon as a user is created, the signal is sent, and immediately, for that user, a token is automatically generated."

The instructor demonstrates this by creating a new user, "prasad," through the admin panel and showing that — with no extra click, no management command, no endpoint call — "prasad" already has a token the moment the save completes, alongside the other users (durga, kiran, shanvi, mohan) shown earlier.

> **[Researched] — this is DRF's own recommended pattern.** This exact signal (function name, decorator, and all) is the [pattern given directly in DRF's official docs](https://www.django-rest-framework.org/api-guide/authentication/#generating-tokens) for auto-issuing a token per user. It's commonly placed in an `AppConfig.ready()` method or a dedicated `signals.py` imported from there, specifically so the signal is guaranteed to be registered once when the app starts — putting it directly at module level in `models.py`, as this demo does for simplicity, also works but is slightly more fragile in larger projects (it depends on Django importing that module early enough).

> **[Example] — a standalone illustration beyond the video.** Say `Token.objects.create(user=instance)` runs for a new user "asha":
>
> Input: `User.objects.create_user(username="asha", password="Secret#123")`
>
> Output (new row in the `authtoken_token` table):
> ```
> key                                        | user_id
> --------------------------------------------|--------
> 7f0e3c9a1b5d8e2f4a6c0b9d3e7f1a5c8b2d4e6f    | (asha's user id)
> ```
> No code anywhere else has to call anything — the signal handles it the instant `asha` is saved.

## 5. Testing the API: activating `TokenAuthentication` + `IsAuthenticated` **[From video]**

With tokens generated, the second half of the lecture is proving a token actually works. First, the view has to be told to *require* one — this is where the two imports from §2 finally get used:

```python
# views.py — now protected
from rest_framework import viewsets
from rest_framework.authentication import TokenAuthentication
from rest_framework.permissions import IsAuthenticated
from .models import Manager
from .serializers import ManagerSerializer

class ManagerViewSet(viewsets.ModelViewSet):
    queryset = Manager.objects.all()
    serializer_class = ManagerSerializer
    authentication_classes = [TokenAuthentication]
    permission_classes = [IsAuthenticated]
```

> "Authentication classes I'm using TokenAuthentication, permission classes I'm using IsAuthenticated — only authenticated users, meaning only people who have a token, can access your API."

- `authentication_classes = [TokenAuthentication]` — tells DRF *how* to figure out who's making the request: look for an `Authorization: Token <key>` header, and if it matches a row in the `Token` table, treat the request as coming from that token's user.
- `permission_classes = [IsAuthenticated]` — tells DRF *what to require* once identity is known: reject the request unless a user was actually identified. This is the same `IsAuthenticated` permission class from REST API Session 14 — token authentication only figures out *who's asking*; it's still the permission class's job to decide whether that's *allowed*.

> **[Gap-filled] — why both lines are needed, not just one.** `authentication_classes` alone doesn't block anyone — it only *attempts* to identify the caller; an unauthenticated (anonymous) request would still be allowed through if no permission class objects. `permission_classes = [IsAuthenticated]` alone, without `TokenAuthentication`, would have no way to *recognize* a token in the first place (DRF falls back to its default authentication classes from `settings.py`, typically session + basic auth, which don't understand the `Authorization: Token ...` header format). You need the pairing: one to parse the credential, one to enforce that a credential was found.

The instructor also removes the `get-token/` URL used only for demo purposes in §4.3 at this point, noting it's "not required at the moment" since tokens are already generated for the users being tested with — trimming the project back down before moving into the CRUD tests below.

## 6. Testing full CRUD against the token-protected endpoint **[From video]**

With the server running, every request now needs an `Authorization: Token <key>` header attached, using HTTPie's syntax of `"Header-Name: value"` as an extra argument.

### Read (GET — list records)

```bash
http http://127.0.0.1:8000/manager/ "Authorization: Token be7d3c9a1f5e8b2d4c6a0f9b3e7d1c5a8f2b4d6e"
```

```json
// 200 OK
[
    {
        "id": 1,
        "name": "Kiran",
        "address": "Hyderabad",
        "email": "kiran@gmail.com",
        "age": 28
    },
    {
        "id": 3,
        "name": "Roger",
        "address": "Chennai",
        "email": "roger@gmail.com",
        "age": 31
    }
]
```

The video confirms this succeeds and returns exactly the two `Manager` records already sitting in the admin panel (Kiran and Roger) — proof the token was accepted and the request was authenticated correctly.

### Create (POST — insert a record)

```bash
http POST http://127.0.0.1:8000/manager/ \
    name=Kumar address=Techoid email=kumar@gmail.com age=44 \
    "Authorization: Token <prasad's token>"
```

```json
// 201 Created
{
    "id": 4,
    "name": "Kumar",
    "address": "Techoid",
    "email": "kumar@gmail.com",
    "age": 44
}
```

The video confirms the new "Kumar" record appears in the admin panel immediately after this call.

### Update (PUT — replace a record)

```bash
http PUT http://127.0.0.1:8000/manager/1/ \
    name=Manoj address=Hyderabad email=manoj@gmail.com age=33 \
    "Authorization: Token <prasad's token>"
```

```json
// 200 OK
{
    "id": 1,
    "name": "Manoj",
    "address": "Hyderabad",
    "email": "manoj@gmail.com",
    "age": 33
}
```

Record `id=1` (previously "Kiran, Hyderabad") is overwritten with the new values — the instructor confirms this by refreshing the admin panel and seeing the updated row.

### Delete (DELETE — remove a record)

```bash
http DELETE http://127.0.0.1:8000/manager/2/ "Authorization: Token <prasad's token>"
```

```
204 No Content
```

Record `id=2` no longer appears in the admin panel or in a subsequent GET request after this call.

Throughout all four, the same "prasad" token is reused — the instructor notes that *any* valid user's token works, since the only requirement is that the caller can identify themselves; nothing here restricts which user's token is allowed to hit this particular endpoint (there's no ownership check, just "are you logged in at all").

> **[Researched] — what happens with a missing or invalid token.** Per DRF's docs, if `Authorization` is missing entirely, `IsAuthenticated` rejects the request with:
> ```json
> // 401 Unauthorized
> { "detail": "Authentication credentials were not provided." }
> ```
> If a token is present but doesn't match any row in the `Token` table (e.g. mistyped, or an old/rotated token):
> ```json
> // 401 Unauthorized
> { "detail": "Invalid token." }
> ```
> Both are useful error shapes to test for deliberately — a common early-DRF mistake is testing only the "happy path" (valid token) and never checking that the endpoint actually *rejects* bad or missing credentials.

## 7. The Command Prompt vs. PowerShell problem **[From video]**

A significant chunk of the transcript is the instructor repeatedly failing to get a request through, retyping and re-pasting the same command, before diagnosing the actual cause:

> "Command Prompt will not allow you to execute a particular command properly, for security reasons — it will not accept it. That's why I shifted to PowerShell. In PowerShell, these requests with token-based authentication will go through properly."

> **[Gap-filled] — what's actually going on here.** This is a real, well-known Windows quirk, not a DRF or HTTPie bug: the classic Command Prompt (`cmd.exe`) handles quoting of arguments like `"Authorization: Token ..."` differently from PowerShell, and can mangle or drop the quotes/spaces needed to pass a header value correctly, especially with a colon and spaces inside quotes. PowerShell's argument parsing handles this case correctly. The practical takeaway for anyone following along on Windows: **run HTTPie (or `curl`) commands from PowerShell, not the legacy Command Prompt**, particularly whenever a command includes a header value with spaces or a colon in it. On macOS/Linux, the default terminal shell doesn't have this problem.

> **Industry best practice:** be deliberate and careful copy-pasting tokens between the admin panel and the terminal — the video hits this exact problem more than once ("be careful in the copy-paste process... sometimes you may miss a space"). A trailing space, a missing character, or an extra newline silently turns a valid token into an invalid one, producing a confusing `"Invalid token."` error that looks like a code bug but is actually a clipboard mistake. When scripting this for real (rather than demoing by hand), read the token from an environment variable or a config/secrets file instead of copy-pasting it each time.

## 8. Security notes on tokens **[Gap-filled / Researched]**

The video doesn't dwell on this, but it's worth calling out explicitly since this lecture is entirely about handling live credentials:

- **A DRF auth token does not expire by default.** Unlike a session, which Django can expire after a configurable time, `rest_framework.authtoken`'s `Token` model is valid forever until it is explicitly deleted or regenerated (the `-r`/`--reset` flag from §4.2, or deleting the row in admin). This is a deliberate simplicity trade-off in DRF's built-in implementation — a leaked token stays valid indefinitely unless someone notices and rotates it.
- **A token is a bearer credential** — whoever holds the string can act as that user, no password needed again. It should be treated with the same care as a password: never logged, never committed to source control, never put in a URL query string in production (URLs often get logged by proxies/servers, unlike headers).
- **Always use HTTPS in production** when sending tokens — over plain HTTP, the token is visible to anyone who can see the network traffic.
- **For expiring/rotating credentials**, many real-world DRF projects reach for a third-party package like [`djangorestframework-simplejwt`](https://django-rest-framework-simplejwt.readthedocs.io/) (JWT-based auth with short-lived access tokens and longer-lived refresh tokens) instead of, or alongside, the built-in `TokenAuthentication` shown here. DRF's own docs note the built-in token scheme is intentionally minimal and suggest looking at third-party packages for more advanced needs (per-device tokens, expiry, etc.).

## 9. What's next **[From video]**

The instructor closes by previewing the rest of the REST Framework unit:

> "If you want to do custom authentication, we can also generate that — this will be discussed in tomorrow's session. Only two or three sessions are pending: custom authentication, pagination, and throttling. After that, mostly on Saturday, I'll do a Django project explanation session — a closing session — and I'll share those projects on Google Drive."

This matches the brief's lecture list: REST API Session 17 (custom authentication), REST API Sessions 18–20 (filtering/pagination), REST API Session 21 (throttling), consistent with what's coming next in this course.

---

## Wrap-up

- **From video:** the demo project's `Manager` model/serializer/`ViewSet`/router setup; the required `rest_framework.authtoken` entry in `INSTALLED_APPS`; all four ways to generate a token (admin panel, `drf_create_token` management command, the `obtain_auth_token` API endpoint tested with HTTPie, and a `post_save` signal that auto-generates a token at user-creation time); activating `TokenAuthentication` + `IsAuthenticated` on a view; testing full CRUD (GET/POST/PUT/DELETE) against the protected endpoint with `Authorization: Token <key>` headers; the Command Prompt vs. PowerShell header-quoting issue on Windows; the course's remaining roadmap (custom authentication, pagination, throttling, then a closing Django-project session).
- **Gap-filled:** why `migrate` is required after adding `rest_framework.authtoken`; why both `authentication_classes` and `permission_classes` are needed together, not just one; why the instructor comments out the signal while demoing the endpoint method; what's actually happening in the Command Prompt vs. PowerShell issue; a caution about careful token copy-pasting; general token-as-secret handling advice.
- **Researched:** the `-r`/`--reset` flag for `drf_create_token`; DRF's official signal snippet for auto-generating tokens; the exact 401 error bodies for missing vs. invalid tokens; the security caveat that DRF's built-in tokens never expire by default and the mention of `djangorestframework-simplejwt` as a fuller alternative; a rate-limiting note on the token-obtaining endpoint.
- **Example:** a standalone illustration of the signal firing for a new user "asha," beyond the video's own users (durga, kiran, shanvi, mohan, prasad).

Double-check against the completeness pass: recap of REST API Session 15 ✓, demo project structure (model/serializer/viewset/router) ✓, required settings change ✓, all four token-generation methods ✓, activating auth+permission classes on the view ✓, full CRUD testing with HTTPie ✓, the Windows terminal gotcha ✓, token security notes ✓, next-session preview ✓ — nothing from the transcript was left uncovered.
