# REST API Session 21 — Throttling: Rate-Limiting API Requests with AnonRateThrottle, UserRateThrottle & ScopedRateThrottle

Source: `transcripts/restapi/REST API-21.txt`
Covers: the final lecture of the Django REST Framework unit — **throttling**, DRF's system for limiting *how often* a client may call an API, as distinct from authentication (who are you) and permissions (are you allowed at all). Walks through the built-in `AnonRateThrottle` and `UserRateThrottle` classes, the `DEFAULT_THROTTLE_CLASSES`/`DEFAULT_THROTTLE_RATES` settings, a live demo (new `throttle_app`, a `Student` model, session authentication + `IsAuthenticated`, then throttling layered on top) showing anonymous vs. authenticated users hitting their limits and getting throttled, and a custom throttle class built by subclassing `UserRateThrottle` with its own `scope` — the same mechanism behind DRF's `ScopedRateThrottle`.

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official DRF/Django docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. What throttling is, and how it differs from permissions **[From video]**

> Throttling concept is similar to permission, but in this throttling concept, so we can able to restrict the users — how many times user can able to send the request to the API? So how many number of times users can able to send the request to the API? We can able to restrict that. That is concept called throttling.

The instructor opens by drawing a direct comparison to **permissions** (REST API Sessions 13–14): both concepts sit at the "should this request even be allowed to reach the view?" checkpoint, but they answer *different* questions.

> Throttling concept in Django REST Framework. We are limiting — limiting to the user for requesting an API. Means if we said one of the user three times only — three times per hour, for three times per day. So in one day, user can able to send the request to the API only three times. If they are trying to more than three times, then it will not allow. So that is limiting request to the API. That concept is called throttling.

**[Gap-filled]** — the plain-language distinction, worth stating up front since the rest of the lecture builds on it:

| Concept | Question it answers | Covered in |
|---|---|---|
| **Authentication** | *Who* is making this request? (identify the user, or decide it's anonymous) | REST API Sessions 13, 15–17 |
| **Permissions** | *Is this user allowed to do this at all* (this action, on this resource)? A yes/no gate that doesn't change over time. | REST API Sessions 13–14 |
| **Throttling** (this lecture) | Is this user allowed to do this **right now**, given *how many times* they've already done it recently? A gate based on **rate/frequency**, that resets over time. | REST API Session 21 |

A concrete way to see the difference: a permission check (`IsAuthenticated`) never changes its answer for the same user and the same request shape — either they're logged in or they're not. A throttle's answer to the *exact same request* can flip from "allowed" to "blocked" purely because the clock ticked past some count, and then flip back to "allowed" again once enough time has passed. That's why the official docs (quoted in the video and below) call it a "temporary state" rather than a fixed rule.

## 2. The official DRF definition **[From video / Researched]**

The instructor pulls this straight from the [DRF throttling documentation](https://www.django-rest-framework.org/api-guide/throttling/):

> Throttling is similar to permissions, in that it determines if a request should be authorized. Throttles indicate a temporary state, and are used to control the rate of requests that clients can make to an API.

> As with permissions, multiple throttles may be used. Your API might have a restrictive throttle for unauthenticated requests, and a less restrictive throttle for authenticated requests.

**[From video, cleaned up]** — the instructor restates this in his own words: you can attach more than one throttle to the same view — e.g. a tighter limit for anonymous requests and a looser limit for authenticated requests — exactly what the live demo later builds (3 requests/day for anonymous users, 4 requests/hour for a logged-in user).

**[Researched]** — the docs give two further reasons you might combine multiple throttles, both mentioned in passing in the transcript but worth spelling out:

1. **Different constraints for different, resource-intensive parts of an API.** A cheap `GET /products/` list might allow far more requests per minute than an expensive `POST /reports/generate/` endpoint that triggers a heavy computation.
2. **Burst limits *and* sustained limits at the same time.** The docs' own example: allow a user a maximum of 60 requests per minute (a short "burst" ceiling), **and** a maximum of 1000 requests per day (a longer "sustained" ceiling) — both throttles run together, and the request is blocked the moment *either* limit is exceeded.

```python
# Researched — combining a burst throttle and a sustained throttle on one view,
# per the DRF docs' own example (not built in the video, but named directly in it)
class BurstRateThrottle(UserRateThrottle):
    scope = 'burst'

class SustainedRateThrottle(UserRateThrottle):
    scope = 'sustained'

class ExampleView(APIView):
    throttle_classes = [BurstRateThrottle, SustainedRateThrottle]
```
```python
# settings.py
'DEFAULT_THROTTLE_RATES': {
    'burst': '60/min',
    'sustained': '1000/day',
}
```

**[Researched]** — the docs also note throttles aren't *only* about request-count rate limiting, a point the instructor mentions but says he won't dig into ("we will not go into detailed things... not required for us"):

> ...throttles do not necessarily only refer to rate-limited requests. For example a storage service might also need to throttle against bandwidth, and a paid data service might want to throttle against a certain number of records being accessed.

This is a reminder that "throttle" in DRF is really a general-purpose *hook* for deciding "should this request through, right now" — the built-in classes just happen to implement that hook as a request-count-per-time-window check. A fully custom throttle class could, in principle, check bandwidth used or rows returned instead of request count — DRF doesn't restrict what the check is based on.

## 3. Built-in throttle classes **[From video / Researched]**

> If once user crossed the limit of the request to the API, then it will show something like this after your limit is crossed... For this purpose we use a non rate total class, we have to use a user rate total class, and scope or late total.

**[Gap-filled, de-garbling "a non rate total"/"user rate total"/"scope or late total"]** — the transcript mangles the class names throughout ("total"/"totaling" for **throttle**/**throttling**, "a non" for **Anon**). The three built-in classes named are:

| Class | Who it limits | Identified by |
|---|---|---|
| `AnonRateThrottle` | **Anon**ymous (not logged in) requests | The client's IP address |
| `UserRateThrottle` | Authenticated requests | The user's primary key (`user.pk`) |
| `ScopedRateThrottle` | Requests to specific, individually-named parts of the API | A `throttle_scope` attribute set on the view, combined with either the IP or the user ID |

**[Researched]** — per the [DRF source](https://www.django-rest-framework.org/api-guide/throttling/#anonratethrottle) and docs, all three inherit from a common base, `SimpleRateThrottle`, which itself implements `BaseThrottle`. `SimpleRateThrottle` does the actual work — it:
1. Builds a **cache key** identifying "this client" (IP for anonymous, user ID for authenticated, or scope+identity for scoped).
2. Looks up (in Django's cache framework) the timestamps of that client's recent requests under that key.
3. Drops timestamps older than the configured time window.
4. If the number of *remaining* (recent) timestamps is still `>=` the allowed count, the request is throttled; otherwise the new timestamp is recorded and the request proceeds.

This matters because it means **throttling depends on Django's cache backend actually working** — covered as a pitfall in Section 9 below.

## 4. Configuring throttling globally: `DEFAULT_THROTTLE_CLASSES` and `DEFAULT_THROTTLE_RATES` **[From video]**

> If you want to work with throttling concept here in globally, we have to do settings... default settings for entire your application level or global options, global throttle if you want to set... in `settings.py` we have to write `REST_FRAMEWORK` equals to curly braces, and inside, in single quotes, `DEFAULT_THROTTLE_CLASSES` colon, inside the square bracket... `rest_framework.throttling.AnonRateThrottle`, comma, `rest_framework.throttling.UserRateThrottle`... then comma, and `DEFAULT_THROTTLE_RATES`... how many times you want to send a request you have to decide that.

**[Gap-filled, reconstructed from the garbled dictation]** — the settings block the instructor is describing, cleaned up:

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/day',
        'user': '1000/day',
    },
}
```

> ...here I'm giving that anon [means] unauthenticated users, normal users, anonymous users purpose — 100 requests per day I'm giving for unauthenticated persons, and authenticated persons — for authenticated users, I'm giving 1000 requests per day.

Two things this global setup does, explained plainly:

- **`DEFAULT_THROTTLE_CLASSES`** is a list of throttle classes that apply, by default, to **every view** in the project (unless a view overrides them with its own `throttle_classes` attribute — the same override pattern as `authentication_classes`/`permission_classes` from REST API Sessions 13–14).
- **`DEFAULT_THROTTLE_RATES`** is a dictionary that supplies the actual numeric rate for each throttle's `scope`. `AnonRateThrottle`'s scope is the string `'anon'`; `UserRateThrottle`'s scope is `'user'` — so the dictionary keys `'anon'` and `'user'` above are not arbitrary, they must exactly match each class's `scope` attribute (more on custom scopes in Section 8).

> But make sure that this throttling rate — whatever I have given — throttling rate may include seconds, minutes, hours, days as the throttle break. You can also [do] per hour, per minute, per second, per day, like this. I'm using per day here, you can also take per second, per minute also, [as] throttle rates here.

**[Researched]** — the exact rate-string grammar, per the [DRF docs](https://www.django-rest-framework.org/api-guide/throttling/#setting-the-throttling-policy): a rate is written as `'<number of requests>/<period>'`, where the period is one of `s` / `sec` / `second`, `m` / `min` / `minute`, `h` / `hour`, or `d` / `day`. So `'100/day'`, `'5/min'`, and `'1000/h'` are all valid — the instructor's "you can take per second, per minute also" is accurate; he simply demos only day- and hour-based rates.

## 5. Building the demo: a fresh `throttle_app` **[From video]**

The instructor starts (after a couple of false starts creating and deleting a scratch app called `myapp2`, visible in the transcript as garbled attempts and a "no module named myapp2" error from referencing an app he'd already deleted) with a clean, purpose-named app:

```bash
python manage.py startapp throttle_app
```

```python
# settings.py
INSTALLED_APPS = [
    # ...
    'rest_framework',
    'throttle_app',
]
```

**[Gap-filled]** — as with every app in this course since Lecture 2, `startapp` only creates the folder and files; Django won't recognize the app's models/admin until it's added to `INSTALLED_APPS`.

### The `Student` model

> This model I'm creating, models... student model I'm creating... name, ID, address model...

**[Gap-filled] — reconstructing a thin description.** The transcript only gives "name, ID, address" in passing (it says the model was copied over from a similar model used in an earlier session's demo). Reconstructed in the same shape as the `Student`/`Manager`-style demo models used in prior lectures (e.g. REST API Session 12's `Manager` model):

```python
# throttle_app/models.py
from django.db import models

class Student(models.Model):
    name = models.CharField(max_length=100)
    address = models.CharField(max_length=255)

    def __str__(self):
        return self.name
```

### Admin, serializer, migrations

```python
# throttle_app/admin.py
from django.contrib import admin
from .models import Student

admin.site.register(Student)
```

```python
# throttle_app/serializers.py
from rest_framework import serializers
from .models import Student

class StudentSerializer(serializers.ModelSerializer):
    class Meta:
        model = Student
        fields = '__all__'
```

> Serializer I'm not going to use any other thing so — realizer's `ModelSerializer` I'm using, that's it totally.

```bash
python manage.py makemigrations
python manage.py migrate
python manage.py runserver
```

The instructor logs into `/admin/` as a superuser named **Mohan** and adds three `Student` records by hand through the admin UI (student IDs 101, 102, 103 with placeholder addresses) — purely to have data available once the API is wired up, the same seed-through-admin workflow used in earlier lectures.

## 6. The view: `StudentModelViewSet` with authentication and permissions (no throttle yet) **[From video]**

Before adding throttling, the instructor first wires up the exact same `authentication_classes`/`permission_classes` pattern from REST API Sessions 13–14, to have a working, protected endpoint to throttle in the next step:

```python
# throttle_app/views.py
from throttle_app.models import Student
from throttle_app.serializers import StudentSerializer
from rest_framework import viewsets
from rest_framework.permissions import IsAuthenticated
from rest_framework.authentication import SessionAuthentication

class StudentModelViewSet(viewsets.ModelViewSet):
    queryset = Student.objects.all()
    serializer_class = StudentSerializer
    authentication_classes = [SessionAuthentication]
    permission_classes = [IsAuthenticated]
```

> Class name is `StudentModelViewSet`... `viewsets.ModelViewSet`... queryset object equals to `Student.objects.all()`... serializer class equals to `StudentSerializer`... authentication classes equals to — this is `SessionAuthentication`.

**[Gap-filled]** — `SessionAuthentication` (from `rest_framework.authentication`) authenticates a request using Django's regular session cookie — the same mechanism a logged-in browser session already uses after logging in through Django's normal `/admin/`-style login form. Paired with `IsAuthenticated`, this means only a request coming from a browser that's currently logged in may reach any action on this ViewSet at all — a plain `curl` request or a logged-out browser tab is rejected outright, before throttling ever gets a chance to run.

### Router and the browsable API's login/logout URLs

```python
# project urls.py
from django.contrib import admin
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from throttle_app import views

router = DefaultRouter()
router.register('student', views.StudentModelViewSet, basename='student')

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include(router.urls)),
    path('api-auth/', include('rest_framework.urls', namespace='rest_framework')),
]
```

> Now include router... browsable API, that's what I'm going to use... `rest_framework.urls`... name equals to `rest_framework`... this is actually why I'm using this one, because of the browsable API — I want to run this one, so now, only authenticated person can able to access this.

**[Gap-filled]** — `path('api-auth/', include('rest_framework.urls', namespace='rest_framework'))` is a standard, documented addition (from the [DRF quickstart docs](https://www.django-rest-framework.org/tutorial/quickstart/#urls)) that adds ready-made **Login**/**Logout** links to the top of the browsable API's pages — without it, a logged-out visitor to the browsable API has no on-page way to log in and would have to go through `/admin/` separately. This is exactly what the instructor uses to demonstrate logging in and out as "Mohan" directly from the API page.

### First demo: authentication/permission behavior only

> Now observe here — only authenticated person can able to access this... now you can see he can send [the] request any time... is authenticated? Yes, I think authenticated users also can able to access... but I have authenticated — why is it showing me authenticated? Once I log out, you can see... now you can see, "authentication credentials are not provided."

With no throttle yet, the behavior matches REST API Session 13–14 exactly: while logged in as Mohan, `GET /student/` works and can be called **any number of times**, with no rate limit at all. Logging out and re-requesting the same URL returns DRF's standard permission-denial response:

```http
GET /student/ HTTP/1.1
```
```json
HTTP/1.1 401 Unauthorized
Content-Type: application/json

{
    "detail": "Authentication credentials were not provided."
}
```

The instructor explicitly frames this as the motivating gap throttling fills: *"if user is authenticated, that user can able to make request any number of times, no limit — but... in this throttling case, if user is authenticated, I want to limit that user to send the request only [a] few requests per minute, per day."* Being authenticated and permitted is not the same as being unlimited — that's the new restriction this lecture adds on top.

## 7. Adding throttling: `AnonRateThrottle` and `UserRateThrottle` on the view **[From video]**

> Now I'm going to import a total [throttling] concept here... from `rest_framework` dot throttling, import `AnonRateThrottle` comma `UserRateThrottle`... the anon total [throttle] class is used for non-authenticated users' [requests] — how many requests you can limit — that user rate total [throttle] means authenticated person's [requests] — how many requests you want to set — that is `UserRateThrottle` only.

```python
# throttle_app/views.py
from throttle_app.models import Student
from throttle_app.serializers import StudentSerializer
from rest_framework import viewsets
from rest_framework.permissions import IsAuthenticated
from rest_framework.authentication import SessionAuthentication
from rest_framework.throttling import AnonRateThrottle, UserRateThrottle

class StudentModelViewSet(viewsets.ModelViewSet):
    queryset = Student.objects.all()
    serializer_class = StudentSerializer
    authentication_classes = [SessionAuthentication]
    permission_classes = [IsAuthenticated]
    throttle_classes = [AnonRateThrottle, UserRateThrottle]
```

> Now, after permission classes, is authentication, now we have to include throttle classes also — now you can see, throttle classes equals to `AnonRateThrottle` comma `UserRateThrottle` — we have to use it. Once we use this, now you have to decide how many requests, how many requests you want to allow — [for] a non-authenticated user [and for a] user-authenticated [i.e. authenticated] person, how many requests you want to allow — that we have to do in settings.

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_RATES': {
        'anon': '3/day',
        'user': '4/hour',
    },
}
```

> Here I'm going to use — a non-user purpose, [a] non-user — how many requests, three requests today I'm taking, three requests only for [a] non-authenticated person... [and for an] authenticated person, I'm giving only four requests for our — four requests for our [per hour], I'm giving. This is what actually I'm giving — limit the rate.

**[Gap-filled]** — note that `throttle_classes` is set **on the view**, exactly the same override pattern as `authentication_classes`/`permission_classes`. This is a deliberate, tiny demo rate (3/day, 4/hour) rather than a realistic production number — chosen purely so the limit can be hit and observed within the lecture instead of waiting hours or days.

### Testing the anonymous limit

> Now you can see, [an] authenticated person can send the request. Now you can see [a] non-user can send the request — one time I send the request... next time I'm going to send the request, second request, third request — now [the] fourth request, whenever I'm hitting, now you can see the problem is — the request was throttled, expected available in 86382 seconds.

Logged **out** (so requests are anonymous, throttled by IP under the `'anon': '3/day'` rate), the first three `GET /student/` requests succeed normally. The fourth request within the same day is rejected:

```http
GET /student/ HTTP/1.1
```
```json
HTTP/1.1 429 Too Many Requests
Content-Type: application/json

{
    "detail": "Request was throttled. Expected available in 86382 seconds."
}
```

**[Gap-filled]** — the number of seconds shown (~86382, close to but slightly under 86400 = 24 hours) is the time remaining until the *oldest* of the client's three recorded requests ages out of the one-day window, at which point a new request is allowed again — not a fixed countdown from a page-load, but a live calculation based on when the earliest still-counted request happened.

### Testing the authenticated limit

> Authenticated user means — yes, again I'm logging in here... authenticated user can able to send the request how many times? Four per hour. I'm sending request... second request... third request... fourth request I'm taking here... next, fourth [again] request, now can see the request was throttled, expected available in 3575 seconds — one hour, only then you can able to make the request.

Logged back **in** as Mohan (now throttled by user ID under the `'user': '4/hour'` rate, and no longer subject to the anonymous throttle at all), the same pattern repeats at a different limit: four `GET`/`POST`/etc. requests within an hour succeed, and the request after that returns:

```json
HTTP/1.1 429 Too Many Requests
Content-Type: application/json

{
    "detail": "Request was throttled. Expected available in 3575 seconds."
}
```

**[Gap-filled]** — 3575 seconds is just under one hour (3600 seconds), for the same reason as above: it's the time left until the earliest of Mohan's four counted requests falls outside the rolling one-hour window.

> **[Example]** — the same 429 response shown as a realistic client-facing error, the way a front-end or mobile app calling this API would actually receive it:
```http
POST /student/ HTTP/1.1
Content-Type: application/json
Cookie: sessionid=abc123...

{"name": "New Student", "address": "Hyderabad"}
```
```json
HTTP/1.1 429 Too Many Requests
Content-Type: application/json

{
    "detail": "Request was throttled. Expected available in 42 seconds."
}
```

## 8. Custom throttle classes and `ScopedRateThrottle` **[From video / Researched]**

> One more important thing — you can able to prepare your own scope of throttle also... for that, we need to create one Python file. File name is `throttling.py`. In this `throttling.py`, I'm implementing: `from rest_framework.throttling import UserRateThrottle`. I'm going to change — my own class through `UserRateThrottle` — class, suppose, I'm giving my name Mohan — `MohanRateThrottle`, class which is inherited from `UserRateThrottle`, and `scope` equals to what I'm taking — scope equals to what actually I'm giving — my name is Mohan — "mohan_user".

```python
# throttle_app/throttling.py
from rest_framework.throttling import UserRateThrottle

class MohanRateThrottle(UserRateThrottle):
    scope = 'mohan_user'
```

```python
# throttle_app/views.py
from throttle_app.throttling import MohanRateThrottle
# from rest_framework.throttling import AnonRateThrottle, UserRateThrottle  # (commented out, per the video)

class StudentModelViewSet(viewsets.ModelViewSet):
    queryset = Student.objects.all()
    serializer_class = StudentSerializer
    authentication_classes = [SessionAuthentication]
    permission_classes = [IsAuthenticated]
    throttle_classes = [MohanRateThrottle]
```

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_RATES': {
        'mohan_user': '2/min',
    },
}
```

> Instead of user rate throttle, I'm going to use my own — like this, this throttle... in my settings, you can see I'm using here `Mohan` colon... okay, then three requests, or four requests — let me give two requests per minute, I'm taking — within one minute, only two requests he can able to make, [that's] Mohan, now, this time.

> Whenever I'm trying to log it in as, so far, [a] user, now let me request this — request, request, over, now third request — you can see request was throttled, expected available in 48 seconds. Just wait 48 seconds, after 48 seconds... after 48 seconds you will get [a] result... two more times you can send request.

**[Gap-filled] — what this custom class actually is, and why the countdown numbers vary.** The instructor's own demo shows the "wait X seconds" figure counting down across several retries (roughly 48 → 21 → 14 → 10 → 8 → 6 → 4 seconds in the recording, with one apparent jump back up, most likely because a couple of the retries were sent close enough together that the older timestamp aging out shifted the math) — this exact sequence is too inconsistent in the transcript to reproduce as a literal quote, so it's summarized here as the general behavior rather than an exact log: each time the client retries too soon, the 429 response reports a *smaller* remaining wait than the previous attempt, because time has passed since the oldest counted request; once enough time passes for a slot to free up, the next request succeeds again.

`MohanRateThrottle` is a plain subclass of `UserRateThrottle` that overrides only one thing: its `scope` attribute. This is a fully documented, supported way to give **one specific view (or a few views)** a *different* rate than the site-wide default `UserRateThrottle` rate — because DRF looks up the numeric rate using the class's `scope` string as the dictionary key into `DEFAULT_THROTTLE_RATES`, changing `scope` to `'mohan_user'` means this class reads the `'mohan_user'` entry instead of the shared `'user'` entry, letting it have its own independent limit without touching the global `UserRateThrottle` rate used everywhere else.

**[Researched] — how this maps onto DRF's actual `ScopedRateThrottle` class.** What the instructor built is the general *technique* — a custom-scoped subclass — that DRF's own built-in `ScopedRateThrottle` (mentioned by name in this lecture's assignment topic, though the transcript itself never says "scoped rate throttle" outright, only "scope of throttle") formalizes into a ready-made class. Per the [DRF docs](https://www.django-rest-framework.org/api-guide/throttling/#scopedratethrottle):

```python
# Researched — the actual built-in ScopedRateThrottle, used differently from a hand-rolled subclass
from rest_framework.throttling import ScopedRateThrottle
from rest_framework.views import APIView

class ContactListView(APIView):
    throttle_classes = [ScopedRateThrottle]
    throttle_scope = 'contacts'          # <- set per VIEW, not per throttle class
    # ...

class ContactDetailView(APIView):
    throttle_classes = [ScopedRateThrottle]
    throttle_scope = 'contacts'          # same scope -> shares one limit with the view above
    # ...

class UploadView(APIView):
    throttle_classes = [ScopedRateThrottle]
    throttle_scope = 'uploads'           # a different, independent limit
    # ...
```
```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_RATES': {
        'contacts': '1000/day',
        'uploads': '20/day',
    },
}
```

The key difference: `ScopedRateThrottle` is **one reusable class** whose scope comes from a `throttle_scope` attribute set on each *view* — so several unrelated views can share one throttle bucket (`'contacts'` above) just by giving them the same `throttle_scope` string, with no new Python class needed per bucket. The instructor's `MohanRateThrottle` approach instead creates a **new class per bucket** (baking the scope into the class itself) — functionally similar for a single view, but `ScopedRateThrottle` is the more direct tool when the goal is "give this named group of endpoints its own shared limit," which is the DRF-idiomatic way to do what this lecture's demo approximates by hand.

## 9. How throttling works under the hood, and settings not shown in the video **[Researched]**

The transcript never explains the *mechanism*, so this section fills that gap from the DRF source and docs.

- **Cache-backed counters.** Every built-in throttle class stores its per-client request timestamps in Django's **cache framework** (`django.core.cache`) — not the database. Each check reads the cached list of recent timestamps for that client's cache key, discards any older than the throttle's time window, and either allows the request (recording a new timestamp) or blocks it.
- **`429 Too Many Requests`.** A throttled request always returns HTTP status **429**, DRF's `Throttled` exception, with the body `{"detail": "Request was throttled. Expected available in <N> seconds."}` — exactly the message garbled in the transcript as "request was total expected available in X seconds."
- **`Retry-After` header.** DRF's throttled response also sets a standard `Retry-After: <N>` HTTP header (in seconds) alongside the JSON body, which well-behaved API clients are expected to read and respect before retrying — not shown in the video, but part of the documented response.
- **Cache key identity.** `AnonRateThrottle` keys its cache entries by the request's IP address (via `request.META['REMOTE_ADDR']`, adjusted for `X-Forwarded-For` if `NUM_PROXIES` is configured); `UserRateThrottle` keys by `request.user.pk` when authenticated, and falls back to IP-based identity for anonymous requests using the exact same key format as `AnonRateThrottle`.

## 10. Where to configure throttling: global vs. per-view **[Gap-filled]**

This follows the identical pattern already established for authentication and permissions in REST API Sessions 13–14:

| Setting | Applies to |
|---|---|
| `DEFAULT_THROTTLE_CLASSES` in `settings.py` | Every view in the project, unless overridden |
| `throttle_classes = [...]` on a view/ViewSet | Only that view — completely replaces (does not add to) the global default for that view |
| `DEFAULT_THROTTLE_RATES` in `settings.py` | The numeric rate looked up by **any** throttle class's `scope`, whether that class is applied globally or on one view |

A view with its own `throttle_classes` (as `StudentModelViewSet` ends up with, in Section 7 above) is **not** additionally throttled by whatever's in `DEFAULT_THROTTLE_CLASSES` — same override-not-merge behavior as `permission_classes`.

## 11. Throttling vs. authentication vs. permissions — one more pass **[Gap-filled]**

Since the whole lecture is framed as "similar to permissions, but not the same," a final side-by-side, now that all three mechanisms have been covered across this unit:

| | Authentication | Permissions | Throttling |
|---|---|---|---|
| Question | Who are you? | Are you allowed to do this? | Are you allowed to do this **right now**? |
| Changes over time for the same user? | No | No (until an admin changes their group/role) | **Yes** — resets as the time window passes |
| Denial status code | `401 Unauthorized` | `403 Forbidden` | `429 Too Many Requests` |
| Typical purpose | Identify the caller | Authorize specific actions | Protect server resources from overuse/abuse |

## 12. Industry best practices & pitfalls **[Researched]**

> **Pitfall — the default cache backend doesn't work correctly across multiple server processes.** By default, Django's cache framework uses `LocMemCache` — an in-memory cache local to a single Python process. If a production deployment runs multiple worker processes (which almost all real deployments do, e.g. multiple Gunicorn/uWSGI workers), each worker has its **own separate copy** of the throttle counts, so a client could actually get up to `(rate × number of workers)` requests through before every worker independently notices the limit. For throttling to enforce a rate accurately across a real multi-process deployment, configure a shared cache backend (Redis or Memcached) via Django's `CACHES` setting.

> **Pitfall — anonymous throttling by IP punishes shared networks.** Because `AnonRateThrottle` keys on IP address, every anonymous user behind the same corporate NAT, university network, or proxy shares one throttle bucket — a handful of unrelated people can exhaust the limit for everyone behind that IP. This is a known, accepted trade-off of IP-based throttling, not a bug; it's worth choosing anonymous rate limits generously enough to absorb this.

> **Pitfall — throttling is not a substitute for permissions.** A throttle only limits *how often*; it never blocks an action outright the way `IsAuthenticated` or a custom permission does. Don't rely on a low throttle rate as a stand-in for "this endpoint shouldn't be public" — use permission classes for that, and throttling only to bound the *frequency* of otherwise-permitted access.

> **Best practice — set both a burst rate and a sustained rate for anything resource-intensive.** As shown in Section 2, DRF explicitly supports attaching two throttle classes (e.g. a `'60/min'` burst class and a `'1000/day'` sustained class) to the same view — this catches both "one client hammering the endpoint in a tight loop" and "one client slowly draining the daily quota," which a single rate alone can't.

> **Best practice — return the `Retry-After` header's information to API consumers clearly, and document your rates.** Since throttling is often invisible until a client hits it, published API documentation should state the rate limits up front (as this course's own settings do) so client developers can build retry/backoff logic rather than discovering the limit by trial and error in production.

> **Pitfall — forgetting that `throttle_classes` on a view replaces, not adds to, `DEFAULT_THROTTLE_CLASSES`.** Exactly like the same pitfall with `permission_classes` (REST API Session 14) — setting `throttle_classes = [MohanRateThrottle]` on one view means that view is throttled *only* by `MohanRateThrottle`'s rate, even if the global settings also list `AnonRateThrottle`/`UserRateThrottle`. If both an anonymous and an authenticated limit are still wanted on that view, all the desired throttle classes must be listed there together.

## 13. Standalone extra example **[Example]**

A `ReviewViewSet` that limits anonymous browsing more loosely than posting-heavy actions, showing scoped throttling used the DRF-idiomatic way (Section 8's `ScopedRateThrottle`, rather than a hand-rolled subclass), applied to a different, realistic use case — a product-review API where reads should be cheap but writes should be tightly capped to deter spam:

```python
# reviews/views.py
from rest_framework import viewsets
from rest_framework.throttling import ScopedRateThrottle
from .models import Review
from .serializers import ReviewSerializer

class ReviewViewSet(viewsets.ModelViewSet):
    queryset = Review.objects.all()
    serializer_class = ReviewSerializer
    throttle_classes = [ScopedRateThrottle]
    throttle_scope = 'reviews'
```

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_RATES': {
        'reviews': '30/min',   # generous enough for normal browsing + occasional posting
    },
}
```

Sample input → output when the limit is exceeded (client sends its 31st request to any `reviews`-scoped endpoint within a minute):

```http
POST /reviews/ HTTP/1.1
Content-Type: application/json

{"product_id": 42, "rating": 5, "comment": "Great product!"}
```
```json
HTTP/1.1 429 Too Many Requests
Retry-After: 17
Content-Type: application/json

{
    "detail": "Request was throttled. Expected available in 17 seconds."
}
```

Because both the review-listing view and the review-posting view share `throttle_scope = 'reviews'`, a client that reads reviews heavily contributes to the *same* 30-per-minute bucket that limits how often it can post — a single shared quota across a related group of endpoints, which is exactly the scenario `ScopedRateThrottle` (as opposed to one throttle class per view) is designed for.

## 14. Closing note: end of the lecture, end of the unit **[From video]**

> This is a simple concept, so that's it for today, and we'll see [the] next concept tomorrow. Okay, we have some pending topic, I think so — that is JWT concept, that I'll finish it once...

The instructor closes by noting throttling is intentionally kept light ("we will not go into detailed things, it's not required for us, but I want to show you some throttling concept — how to limit the request to the API, that is my main motto"), and mentions one remaining, not-yet-covered topic (**JWT** — JSON Web Token authentication, garbled in the transcript as "JW to") as something to pick up in a future session.

**[Gap-filled]** — this transcript is REST API Session 21, the **last lecture in this 21-session Django REST Framework unit** (which began at REST API Session 1 — "Introduction to REST APIs & Django REST Framework" — and has covered, in order: project setup, serialization/deserialization, CRUD with `APIView`, generic views and mixins, concrete view classes, ViewSets and routers, authentication (basic, token, session, custom), permissions, filtering, pagination, and now throttling). The instructor's mention of JWT as a "pending topic" is a forward pointer beyond the recorded material available in this unit, not a lecture within it — there is no REST API Session 22 transcript in this set. With this session, the planned Django REST Framework curriculum is complete.

---

## Wrap-up

- **From video:** what throttling is and how it's similar to, but distinct from, permissions (rate/frequency of access vs. who's allowed at all); the official DRF definition of throttling quoted from the docs; the reasons multiple throttles might be combined (stricter-for-anonymous/looser-for-authenticated, resource-intensive endpoints, burst + sustained limits); a passing note that throttles can guard things other than raw request rate; the `AnonRateThrottle`/`UserRateThrottle` class names and their meaning; the global `DEFAULT_THROTTLE_CLASSES`/`DEFAULT_THROTTLE_RATES` settings and the rate-string format; a full build of a new `throttle_app` (model, admin, serializer) and a `StudentModelViewSet` first secured with `SessionAuthentication`/`IsAuthenticated` alone (unlimited access once logged in), then throttled with `AnonRateThrottle`/`UserRateThrottle` at demo rates (3/day anonymous, 4/hour authenticated), observed hitting `429` responses with a countdown wait time; building a custom `MohanRateThrottle` (a `UserRateThrottle` subclass with its own `scope`) and its own `DEFAULT_THROTTLE_RATES` entry, observed throttling at 2/minute; the closing note on a still-pending JWT topic.
- **Gap-filled:** the plain-language authentication-vs-permissions-vs-throttling comparison table; reconstruction of the `Student` model's fields and the settings/views code from garbled dictation; why the "expected available in N seconds" figures shrink between retries; the override-not-merge relationship between global and per-view throttle settings; the parallel between this lecture's custom-scope technique and DRF's built-in `ScopedRateThrottle`; and the closing observation that this is the final lecture of the entire 21-session DRF unit.
- **Researched:** the full quoted passages from the DRF throttling docs (temporary-state framing, burst+sustained example, non-rate-limit throttle use cases); the `SimpleRateThrottle`/`BaseThrottle` class hierarchy and cache-key mechanics underlying all three built-in throttle classes; the exact `429`/`Request was throttled...`/`Retry-After` response shape; the documented `ScopedRateThrottle` + `throttle_scope` pattern and how it differs from a hand-rolled scoped subclass; and industry pitfalls around cache-backend requirements in multi-process deployments, IP-based anonymous throttling on shared networks, and throttling never substituting for permissions.

Double-check against the completeness checklist: throttling concept and definition ✓; comparison to permissions ✓; DRF docs quote ✓; multiple-throttles rationale (strict/loose, resource-intensive endpoints, burst+sustained) ✓; non-rate-limit throttle use cases (bandwidth, records accessed) ✓; `AnonRateThrottle`/`UserRateThrottle`/`ScopedRateThrottle` ✓; `DEFAULT_THROTTLE_CLASSES`/`DEFAULT_THROTTLE_RATES` global settings and rate-string format ✓; new `throttle_app` build (model, admin, serializer, migrations, seeded via admin) ✓; `SessionAuthentication`+`IsAuthenticated` baseline demo (unlimited once logged in, `401` when logged out) ✓; `api-auth` browsable-API login/logout URLs ✓; adding `throttle_classes` and testing anonymous (3/day) and authenticated (4/hour) limits, `429` responses ✓; custom scoped throttle class (`MohanRateThrottle`) and its settings entry, tested at 2/minute ✓; internal cache-based mechanism, `429` status, `Retry-After` header ✓; global-vs-per-view configuration precedence ✓; best practices and pitfalls ✓; closing note on the pending JWT topic and end of the unit ✓. Nothing from the transcript appears to have been left out.
