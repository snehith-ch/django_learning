# REST API Session 1 — Introduction to REST APIs & Django REST Framework

Source: `transcripts/restapi/REST API-1.txt`
Covers: the first session of a new unit. This is the opening lecture of **Django REST Framework (DRF)**, starting right after Lecture 42 closed out the plain-Django material. No code is written yet — this session is pure theory (what an API is, what REST/RESTful means, what DRF is, why JSON/XML instead of HTML, the CRUD↔HTTP-method mapping, API types) plus the **project setup steps** for tomorrow's coding session (create a Django project, install DRF, register it in `INSTALLED_APPS`).

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official Django/DRF docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## 1. What is an API? **[From video]**

The instructor opens with the core definition:

> API stands for Application Programming Interface. It is a software that enables communication between two or more applications.

In plain language: an **API** is a `dfn`-worthy piece of software that lets one application talk to another — send it a request, get back a response — without either side needing to know how the other one is built internally.

The video's running example is a mobile app:

> When we are using a mobile application, your application connects to the internet and sends data to a server. Then the server retrieves the data, interprets it, performs the necessary actions, and sends it back to your phone.

Two more real-world examples given directly in the transcript:

- **Flipkart** (an e-commerce app) — your phone's Flipkart app talks to Flipkart's servers through an API. The API is described as acting like a **middleman / mediator**: it carries your request over, and carries the server's answer back.
- **A travel reservation portal** — a single "reservation system" website can route you to different backends depending on what you clicked: a railway booking click talks to IRCTC's systems, a bus booking click talks to a service like RedBus or AbhiBus. Each of those is a separate application, and the front-end site reaches all of them through APIs.
- **Banking apps** — every transaction you do "in our daily basis" through a banking app is, per the instructor, ultimately an API call under the hood.

> **[Gap-filled] — why this matters before touching any code.** Every button tap, page load, or "refresh" you've ever done on a phone or website that shows you data from somewhere else is an API call in disguise. Django REST Framework doesn't invent this concept — it gives Django a standard, well-tested way to expose *your* Django project's data as an API, so that other applications (a mobile app, a JavaScript front-end, another company's server) can request it in a predictable format instead of scraping HTML pages meant for humans.

### Homogeneous vs. heterogeneous communication **[From video]**

The transcript makes a specific point about *why* APIs matter beyond "two apps talking":

> Applications might be implementing different technologies. Sometimes a Python-based application can communicate with a Java-based application... The application need not implement on the same technology.

The instructor names two categories:

- **Homogeneous communication** — same technology talking to itself, e.g. a Java application talking to another Java application, or a .NET application talking to another .NET application.
- **Heterogeneous communication** — different technologies talking to each other, e.g. Python ↔ Java, Java ↔ .NET, Python ↔ .NET.

> APIs are the medium that makes heterogeneous communication possible — the whole reason a Python/Django backend can serve data to an iOS app written in Swift, or an Android app written in Kotlin, or a separate Java-based service, without any of them needing to share a programming language.

> **[Example]** A Django backend exposing a `/api/products/` endpoint doesn't care whether the thing requesting it is: a React web app (JavaScript), a Flutter mobile app (Dart), a curl command in a terminal, or another Django project. As long as the requester can send an HTTP request and parse a JSON response, it can talk to your API — that's the entire point of a shared, technology-neutral data format (covered below in Section 6).

---

## 2. Django REST Framework — what it is, and how it relates to Django **[From video]**

This is the single most important framing point of the lecture, stated multiple times in slightly different words:

> Django REST framework is not a separate [framework] — it's along with Django only. This is an extension of Django.

And again, phrased as a hard prerequisite:

> Before going to discuss REST API, you must be aware of Django project creation and how to deal with the Django MVT pattern — model, view, template. This is just an extension of Django only.

So: **Django REST Framework (DRF)**, also referred to in the transcript as "TRF"/"RF" (a transcription artifact — the correct expansion is **DRF**), is **not** a rival or replacement for Django. It's a Python package that sits *on top of* an ordinary Django project and adds tools specifically for building APIs, while reusing everything Django already provides: the ORM, models, the project/app structure, `settings.py`, URL routing, and (as later lectures will show) class-based views.

The video is explicit that **both Python and Django are mandatory prerequisites**:

> To work with REST API — that is Django REST framework — compulsorily, Python is mandatory, and Django framework is mandatory. Without Python and without Django, we cannot go for REST API.

> **[Gap-filled] — restating this as a dependency chain.** DRF depends on Django, and Django depends on Python. You cannot `pip install djangorestframework` into a project that has no Django installed and expect it to work — DRF's core building blocks (serializers, views, routers, as later lectures cover) are Python classes that inherit from or wrap Django's own classes (e.g. DRF's `APIView` extends Django's own `View`). This course's earlier lectures (project setup, virtual environments, MVT, models, views, URLs) are therefore not optional background — they're the foundation DRF is built on.

---

## 3. What "REST" means **[From video]**

> REST stands for **Representational State Transfer**. It is used for web-based architecture for data communication.

Two important clarifications the instructor makes, both worth remembering because they're commonly confused:

1. **REST is an architectural style, not a protocol.**
   > REST is not a kind of protocol — it is an architectural style, or a design pattern, or an architectural pattern for communication purposes.
   In plain terms: REST is a *set of conventions and principles* for designing web-based communication (using HTTP, resources identified by URLs, standard verbs like GET/POST, etc.) — not a fixed technical standard with a formal specification the way HTTP or TCP/IP is. Different frameworks in different languages can all "do REST" in their own way, as long as they follow the same conventions.

2. **REST is not exclusive to Django.**
   > REST is not reserved only for Django — it's a common service, a common architectural pattern for any [technology]. Even Java programmers will use RESTful services. Even .NET programmers will use web APIs. But terminologies are different — purpose is the same.

   The instructor gives the cross-language terminology mapping directly:

   | Ecosystem | What they call it |
   |---|---|
   | Django | REST API (via DRF) |
   | .NET | Web API |
   | Java | RESTful services |

   All three describe the same underlying idea — exposing an application's data over HTTP using REST conventions — just with ecosystem-specific naming and tooling.

### "REST API" vs. "RESTful API" **[From video]**

The transcript spends real time making sure this isn't misread as two different things:

> Rest API or RESTful API — don't get confused here. Rest API, RESTful API are not different things, both are the same thing only. When web services use REST architecture, they are called RESTful API or REST API.

So **REST API** and **RESTful API** are two names for the same thing: a web service (an application exposing functionality over the web) that follows the REST architectural style. "RESTful" is just the adjective form — "an API built the REST way."

> **[Researched]** Per the [DRF documentation](https://www.django-rest-framework.org/) and general REST literature, the defining traits of a RESTful design (beyond "uses HTTP") are: **statelessness** (each request from a client contains all the information the server needs — the server doesn't remember previous requests), **resource-based URLs** (a URL identifies a "thing," like `/api/students/5/`, not an action), and **using HTTP methods to express the action** (GET/POST/PUT/PATCH/DELETE, covered in Section 6) rather than encoding the action into the URL itself (e.g. `/api/deleteStudent?id=5` would *not* be considered RESTful — `DELETE /api/students/5/` would be). The transcript doesn't use the word "stateless" explicitly, but this is the formal property behind why REST APIs work well for many different, independent clients (web apps, mobile apps, other servers) hitting the same endpoints.

---

## 4. What DRF provides, and why you'd reach for it **[From video]**

The instructor lists DRF's advantages somewhat rapid-fire; reorganized here as a checklist:

> DRF is open source, flexible, and a fully featured library with a modular and customizable architecture... DRF allows the flexibility to extend and customize the framework tools according to the programmer's demand. It will reduce development time greatly, and it also provides support for testing and debugging.

- **Open source, flexible, modular** — you can use as much or as little of DRF as you need; it's built to be extended/customized rather than used as an all-or-nothing black box.
- **Reduces development time** — because, per the video, DRF ships pre-built ("generic") classes that handle the repetitive parts of building a CRUD API, so you don't hand-write the same request-parsing/response-formatting logic for every model.
- **Built-in support for testing and debugging** — including (as a later lecture in this unit covers) a **browsable API** — a human-friendly, clickable HTML interface DRF auto-generates for testing your API in a browser, in addition to the raw JSON responses real clients would get.
- **Powerful serialization mechanism** — the video calls this out specifically:
  > A powerful serialization mechanism is there, and also deserialization mechanism is there. The serialization engine is compatible with ORM and non-ORM data sources.
  This is introduced only by name here — **serialization** and **deserialization** are the actual subject of REST API Sessions 2–6. For now, the one-line version: serialization converts Python/Django objects (like a model instance from the database) into JSON so they can be sent to a client; deserialization is the reverse — converting incoming JSON from a client back into Python data Django can validate and save.
- **Works with ORM and non-ORM data** — DRF's serializers aren't hard-wired to Django's ORM/models; the video notes they're "compatible with ORM and non-ORM data sources," meaning you *can* build a DRF API around data that isn't sitting in a Django model at all (though the vast majority of this course's examples, like every other Django project so far, will use the ORM).
- **Easy customization, validations, and authentication systems** — DRF ships built-in support for validating incoming data and for authenticating who's making a request (both explored in depth much later in this unit — REST API Sessions 13–17 cover authentication and permissions specifically).
- **Generic, class-based views for CRUD** — the video specifically highlights:
  > Mostly generic class-based views are there in the REST API for CRUD operations with the database. Clean and simple web resources using Django's [class-based views] are available here.
  "CRUD" is spelled out in the transcript as "grad operations" — a transcription error for **CRUD** (**C**reate, **R**ead, **U**pdate, **D**elete), the four basic operations any data-backed application needs to perform. DRF's generic class-based views (covered starting REST API Session 9) are pre-built view classes that implement these four operations with very little code, once a serializer exists for the model.

> **[Gap-filled] — why "reduces code" matters concretely.** The instructor promises this pays off later in the course, and it's worth setting the expectation now: without DRF, building a CRUD API in plain Django means manually parsing `request.body` as JSON, manually converting model instances to dictionaries field-by-field, manually validating incoming data, and manually setting the right HTTP status code and `Content-Type: application/json` header on every response — for every single model. DRF's serializers and generic views absorb almost all of that repetitive work, which is exactly what "10–15 lines now, 2–3 lines later" (mentioned in Section 9 below) is describing.

---

## 5. Types of APIs: Private, Partner, Public **[From video]**

> There are different types of APIs — private API, partner API, public API.

- **Private API** — usable only *within* a particular organization. Meant for a company's own internal systems to talk to each other, not exposed to outside developers.
- **Partner API** — shared *between* specific business partners. The video's framing: multiple organizations can deal with the same service/customers and share the same API, so different organizations' systems can communicate through it — access is deliberately opened up to a known, limited set of partners rather than the whole internet.
- **Public API** — usable by *any* third-party developer. The instructor notes this is the most common in real-world, consumer-facing systems ("maximum in real-time applications, public APIs are there, because services are completely free to the customers"), specifically calling out banking as an example sector that relies heavily on public APIs.

> **[Example]** A concrete way to tell these apart: a company's internal tool that lets its own payroll system query its own HR system is a **private API** (never exposed outside the company). A payment gateway like Razorpay or Stripe exposing an API *specifically* to registered, approved merchants under a partnership agreement is a **partner API**. A public weather service (e.g. OpenWeatherMap) that anyone can sign up for an API key and query is a **public API**.

> **[Researched]** This three-way split (private/partner/public) is a common API-management industry categorization — it isn't a Django- or DRF-specific concept, and DRF itself doesn't have a "type" setting for this. Whether an API you build with DRF ends up "private," "partner," or "public" is entirely a matter of how you configure **authentication and permissions** (who is allowed to call it at all — REST API Sessions 13–17) and **network/deployment access** (e.g. keeping it on an internal network vs. exposing it on the public internet), not a distinct DRF feature.

---

## 6. How an API request/response cycle works **[From video]**

The video walks through a diagram (delivered as a separate PDF, not reproducible here) and narrates it step by step:

> Generally, client sends HTTP request to API. API will interact with the web application, and web application will interact with the database if it is required. Web application provides the required data to the API. API returns data to the client.

Broken into an ordered sequence:

1. A **client** (the video's examples: a Python app, a Java app, a Django app — "any app") sends an **HTTP request** to the API.
2. The **API** receives that request and forwards it to the **web application** (the Django/DRF project itself — the code that actually knows what to do with the request).
3. If the request needs data from storage, the web application talks to the **database**.
4. The database returns data to the web application.
5. The web application hands the result back to the API.
6. The API sends the response back to the original client.

> The instructor reiterates that the API's role throughout this chain is as a **middleman / mediator**: "API responsibility is — whatever the client is sending as a request, that will take, and the same request will transfer to the web application. And when we get the response from the database, the web application is required to give the response to the API. API will give response to the client only."

> **[Gap-filled] — a text version of the diagram, since the PDF isn't part of this transcript.**
> ```
> Client (Python app / Java app / Django app / browser / mobile app)
>       │  HTTP request (GET/POST/PUT/PATCH/DELETE)
>       ▼
>   REST API  ───────────────►  Web application (your Django + DRF project)
>       ▲                              │
>       │  JSON/XML response           │ queries, if data is needed
>       │                              ▼
>       └────────────────────────  Database
> ```
> In a DRF project specifically, "the web application" in this diagram *is* your Django project: URLs route the incoming request to a view, the view (with help from a serializer) talks to the database through the ORM, and the same view sends back a response — DRF doesn't introduce a separate physical layer, it's the layer of Django that formats that response as JSON instead of HTML.

---

## 7. Why JSON/XML instead of HTML templates **[From video]**

This section connects directly back to material from earlier in the plain-Django course (Django Template Language), which makes it a genuinely useful "before vs. after" comparison:

> In Django, whenever we send a request to the server, server-side code will execute — Python views, class-based or function-based. Once execution is complete, we have to give a response to the client. The response comes in Django Template Language (DTL) format... because the client — the browser — can understand only template/HTML format. Client cannot understand server-side code.

> But here also, in the REST API case — while getting the response from the server, inside the client, we are using JSON or XML format, because the server can give a response to the client in JSON or XML. This client also cannot understand server-side code or API code... DTL can be understood only by a Django application, but a user can interact with any other application, like a Java application, a Python application, any different type of application. DTL cannot be understood by other applications.

The core contrast, restated plainly:

| | Plain Django view | DRF (API) view |
|---|---|---|
| What executes | Python view code | Python view code |
| What it returns | An HTML page, rendered with DTL (Django Template Language) | Data, formatted as JSON (or XML) |
| Who can read the response | A web browser (renders HTML for a human) | Any application at all — a browser, a mobile app, another server, a script |

> **[Gap-filled] — why this is the whole reason DRF exists.** A regular Django view returning an HTML page is fine when the "client" is a human sitting at a browser — the browser knows how to turn `<div>`/`<table>`/etc. into a visible page. But an HTML page is *useless* to a mobile app or another backend service trying to consume your data programmatically — it would have to scrape text out of `<div>` tags, which is fragile and not how real integrations work. JSON (and XML) are **data formats**, not **presentation formats**: they carry pure structured data with no visual styling baked in, so *any* program in *any* language can parse them and decide for itself how (or whether) to display the data. That's the direct answer to "why not just return HTML from an API" — a plain Django view already can, but nothing except a browser can make sense of what it returns.

### JSON **[From video / Researched]**

> JSON stands for JavaScript Object Notation. JSON is an open standard file format and data-interchange format that uses human-readable text to store and transmit data objects, consisting of attribute-value pairs... based on JavaScript object syntax.

**JSON** (**J**ava**S**cript **O**bject **N**otation) is a lightweight, text-based way of representing structured data as nested key–value pairs, lists, and primitive values (strings, numbers, booleans, `null`). Despite the name, it isn't tied to JavaScript at runtime — nearly every programming language has a library to read and write JSON — which is exactly why it's the de facto standard format for API responses, including DRF's.

> **[Example]** A `Trainer` model instance with `id=1, first_name="Anita", last_name="Rao", subject="Python"` serialized to JSON would look like:
> ```json
> {
>   "id": 1,
>   "first_name": "Anita",
>   "last_name": "Rao",
>   "subject": "Python"
> }
> ```
> A list of trainers (as you'd get from a "get all trainers" endpoint) would be wrapped in a JSON array:
> ```json
> [
>   {"id": 1, "first_name": "Anita", "last_name": "Rao", "subject": "Python"},
>   {"id": 2, "first_name": "Rahul", "last_name": "Verma", "subject": "Java"}
> ]
> ```
> This is the shape of data DRF's serializers produce — the actual mechanics of generating this from a model are REST API Session 2's topic.

### XML **[From video / Researched]**

> XML is also open standard. XML stands for eXtensible Markup Language. It is used to describe data. The XML standard is a flexible way to create information formats and electronically share structured data via the public internet, as well as via corporate networks.

**XML** (**e**Xtensible **M**arkup **L**anguage) is an older, tag-based way of representing structured data (`<trainer><first_name>Anita</first_name></trainer>`, roughly), conceptually similar to HTML but designed for data rather than page layout. The instructor is explicit that this course will focus on JSON in practice:

> In our case, I'm not going to use any XML — I'm going to use mostly JSON format only.

> **[Researched]** DRF supports both out of the box — JSON via its default `JSONRenderer`, and XML through optional add-on packages (e.g. `djangorestframework-xml`) since XML isn't in DRF's default renderer set the way JSON is. In modern practice, JSON has become the overwhelming default for new REST APIs because it's more compact and easier to parse than XML; XML tends to show up mainly in older enterprise systems (e.g. SOAP-based services) that predate JSON's popularity. This course's choice to focus on JSON matches current industry norms.

---

## 8. HTTP methods and the CRUD mapping **[From video]**

> These are the important HTTP methods, especially which we are going to deal with in REST API only: GET method, POST method, PUT method, PATCH method, DELETE method.
>
> GET means to retrieve the data from the database. POST means we can store or send the data from client to server. PUT means fully update. PATCH means partial update — sometimes only some of the fields... will be updated, then PATCH is required, and sometimes the full record needs to be updated, then PUT is required. DELETE means deleting or removing.

| HTTP method | CRUD operation | What it does |
|---|---|---|
| `GET` | **Read** | Retrieve/fetch existing data — doesn't change anything on the server |
| `POST` | **Create** | Insert/store new data (send data from client to server) |
| `PUT` | **Update** (full) | Replace an entire existing record with new data |
| `PATCH` | **Update** (partial) | Update only some fields of an existing record, leaving the rest untouched |
| `DELETE` | **Delete** | Remove an existing record |

The video sums this up as the direct CRUD mapping:

> Create means we can POST the record — creating or posting or inserting. Reading means GET only — reading, retrieving, or getting. Update means PUT/PATCH — PUT means fully update, PATCH means partial update, both are required, both ways we have to do all operations, especially in CRUD operations. Delete means to delete or remove the records from the database only.

> **[Gap-filled] — the difference between PUT and PATCH, made concrete.** Say a `Trainer` record has `first_name`, `last_name`, and `subject`. If you only want to correct a typo in `subject` without touching the name fields, you'd send a **PATCH** request with just `{"subject": "Python"}` — DRF updates only that field and leaves the rest alone. If you sent that same partial payload as a **PUT** request, the REST convention is that PUT expects the *entire* resource — a naive full-replace implementation could wipe out `first_name`/`last_name` (setting them to blank/default) because you didn't include them. In practice, DRF's own generic `UpdateAPIView`/`ModelViewSet` machinery (covered in REST API Sessions 9–12) requires all model fields for a PUT unless they're optional, and is more forgiving for PATCH. The safe habit: use PUT only when sending a complete representation of the object, and PATCH whenever you're changing a subset of fields.

> **[Researched] — idempotency, a formal property worth knowing.** Per REST conventions (and the [MDN HTTP methods reference](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods)), `GET`, `PUT`, and `DELETE` are expected to be **idempotent** — calling them multiple times with the same input produces the same end state as calling them once (fetching the same data again, replacing a record with the same data again, deleting an already-deleted record again — nothing changes further). `POST` is **not** idempotent by convention — calling it twice with the same payload is expected to create two separate records. `PATCH` is technically not guaranteed idempotent either, though in most simple field-update cases it behaves as if it were. This isn't enforced by Django/DRF automatically — it's a design convention you follow when building your views, and DRF's generic views/viewsets are built assuming you'll follow it.

---

## 9. API resource anatomy: base URL, naming convention, endpoint **[From video]**

> Generally, when we're working with the Django framework, on the URL section we'll find a URL like `http://localhost/website-name`. This is the general scenario of API resource. In this resource: number one is the **base URL** — the base URL means which you are going to communicate, e.g. `xyz.com`, for example, `/api`. Number two is **naming convention** — this is optional, not compulsory; every time API is not required. Number three, very important, is the **endpoint** or **API resource** — that is `/trainers`. Trainers is my endpoint or API resource. "API" [the literal word in the URL] is not compulsory, but the endpoint/resource must be there.

Broken down, a typical API URL has up to three parts:

1. **Base URL** — the domain you're talking to, e.g. `https://xyz.com`.
2. **Naming convention** (optional) — a segment like `/api` that signals "this path leads to API endpoints, not regular pages." Many real projects include this (`/api/v1/...`) as a convention, but it isn't a technical requirement.
3. **Endpoint / resource** — the actual "thing" being addressed, e.g. `/trainers` or `/students`. This part is mandatory — without it, there's nothing to identify what data the request is about.

Put together: `https://xyz.com/api/trainers/` — base URL (`xyz.com`) + naming convention (`api`) + endpoint/resource (`trainers`).

> **[Example]** A few endpoint URLs for a hypothetical `Trainer` resource, following this same anatomy, and the HTTP method + CRUD action each one represents:
> - `GET /api/trainers/` → list (read) all trainers
> - `GET /api/trainers/5/` → read one trainer (id 5)
> - `POST /api/trainers/` → create a new trainer
> - `PUT /api/trainers/5/` → fully update trainer 5
> - `PATCH /api/trainers/5/` → partially update trainer 5
> - `DELETE /api/trainers/5/` → delete trainer 5
>
> Notice the URL itself barely changes across all six — it's the **HTTP method** that changes what action happens, which is exactly the REST principle from Section 3 ("use HTTP methods to express the action, not the URL").

> **[Researched] — resource naming convention.** Per common REST API design guidance (and DRF's own tutorial), endpoint/resource names are conventionally **plural nouns** representing a collection (`/trainers/`, not `/trainer/` or `/getTrainer/`), with an individual item addressed by appending its identifier (`/trainers/5/`). DRF's `DefaultRouter` (used with ViewSets, covered in REST API Session 12) generates exactly this pattern automatically once you register a viewset.

---

## 10. Setting up a Django REST Framework project **[From video]**

This is the one hands-on part of the session — the instructor sets up a project live in PyCharm, ready for coding to begin "tomorrow." The steps, reconstructed from the (heavily garbled) transcript and cross-checked against how every earlier project in this course was set up:

> First of all, I strongly recommend to create a Django project only... After creating [the] project only, then we have to install [DRF]... `django-admin startproject` [project name] `rest_project`... inside the project we have to create an application: `python manage.py startapp rest_app`.

1. **Create a Django project**, exactly as in every earlier project in this course:
   ```bash
   django-admin startproject rest_project
   cd rest_project
   ```
2. **Create a Django app inside it** — the video names it `rest_app`:
   ```bash
   python manage.py startapp rest_app
   ```
   > At this point, nothing here is REST-specific yet — it's the same `startproject`/`startapp` sequence used for every plain-Django project so far in this course. The instructor is explicit about this being the same process: "It's not a different project... it's same as Django only."

3. **Install Django REST Framework itself**, via `pip`:
   > To install DRF: `pip install djangorestframework`... This is compulsory, [and] required — after that, include `rest_framework` under `INSTALLED_APPS` of your `settings.py`.
   ```bash
   pip install djangorestframework
   ```
   The video notes it got a "requirement already satisfied" message because DRF was already installed on the instructor's machine from a previous session — a first-time install will actually download and install the package.

4. **Register `rest_framework` in `INSTALLED_APPS`**, in `settings.py`:
   > Under `INSTALLED_APPS`, we have to include your application name (`rest_app`) — the application also needs to be installed — and after that, you have to include `rest_framework` also... whenever we install [DRF] in your project, then automatically [it's understood] `rest_framework` is compulsory, required — every application from now on, along with your application, including `rest_framework`, is required because you are working with REST API things now onwards, along with the Django project.

   ```python
   # rest_project/settings.py
   INSTALLED_APPS = [
       'django.contrib.admin',
       'django.contrib.auth',
       'django.contrib.contenttypes',
       'django.contrib.sessions',
       'django.contrib.messages',
       'django.contrib.staticfiles',

       'rest_framework',   # <- DRF itself; required for any DRF feature to work
       'rest_app',         # <- this project's own app, same as any normal Django app
   ]
   ```

> **[Gap-filled] — why Step 4 doesn't happen automatically.** `pip install djangorestframework` only puts the DRF Python package on disk / in your virtual environment — it does not touch your project's `settings.py`. Django only "knows about" installed apps (and loads their models, template tags, admin registrations, etc.) if they're explicitly listed in `INSTALLED_APPS`. This is the exact same reason every app this course has ever created (`startapp` output) also had to be manually added to `INSTALLED_APPS` before Django would recognize it — DRF is no different, it's just a third-party app instead of one you wrote yourself. Forgetting this step is a common first-time mistake: DRF's classes will still *import* fine, but certain DRF features (like the browsable API's templates, or DRF's default settings) won't be wired in correctly until `rest_framework` is listed.

> **[Researched] — DRF's official installation requirements.** Per the [DRF documentation](https://www.django-rest-framework.org/#installation), the two hard requirements are Django itself and Python — matching exactly what the video says are "compulsory." The official install command is `pip install djangorestframework`, and the docs likewise confirm the same "add `'rest_framework'` to `INSTALLED_APPS`" step as the only other mandatory setup action to get a minimal DRF install running. (Optional extras the docs mention — like `django-filter` for filtering, covered in REST API Session 18, or `markdown` for nicer docstring rendering in the browsable API — are not part of this lecture and aren't required to start.)

> **Industry best practice:** Use a **virtual environment** (as this course already established when Django itself was first installed) before running `pip install djangorestframework`, so DRF and its version get tracked per-project rather than installed globally — and pin the version in a `requirements.txt` (e.g. `djangorestframework==3.15.2`) once you've picked one, so the project's dependencies are reproducible on another machine.

---

## 11. What's next **[From video]**

The instructor closes by previewing the next several sessions and setting expectations about code volume:

> Tomorrow we will understand what is serialization and deserialization. How to create your first API, how to test API — also we will discuss tomorrow... Slowly, once we reach day by day sessions of REST API, finally we will reduce a lot of code — drastically — because by using built-in REST API classes, finally the same operations we will do within two or three lines. But first, initially, for every operation 10–15 lines of code is required, because a lot of conversions are there — like serialization technique, deserialization technique will come into the picture.

Also mentioned: the instructor is preparing a **PDF reference document** (with the diagrams and definitions referenced throughout this session) to be shared alongside the recorded video, sourced largely from DRF's own official website, `https://www.django-rest-framework.org/`.

> **[Gap-filled]** The "10–15 lines now, 2–3 lines later" comment is a direct preview of the arc this whole unit follows: REST API Sessions 2–6 build API views mostly "by hand" (writing serializers and function-based views explicitly, to learn what's actually happening), then REST API Sessions 7–12 progressively introduce DRF's generic views, mixins, concrete view classes, and finally ViewSets — each layer doing more automatically and requiring less code, exactly as promised here.

---

## Wrap-up

- **From video:** the definition of an API and real-world examples (mobile apps, Flipkart, banking, travel-reservation portals); homogeneous vs. heterogeneous communication; the framing of DRF as an extension of Django, not a separate framework, with Python + Django as hard prerequisites; the meaning of REST (an architectural style, not a protocol) and the REST-API-vs-RESTful-API non-distinction; DRF's advantages (serialization/deserialization, ORM & non-ORM compatibility, generic class-based views, testing/debugging support); the three API types (private, partner, public); the client → API → web application → database → API → client request/response cycle; why JSON/XML are used instead of Django's own HTML templates; definitions of JSON and XML; the GET/POST/PUT/PATCH/DELETE ↔ CRUD mapping; the base-URL/naming-convention/endpoint anatomy of an API URL; and the live project setup (`startproject`, `startapp`, `pip install djangorestframework`, adding `rest_framework` to `INSTALLED_APPS`).
- **Gap-filled:** the DRF-depends-on-Django-depends-on-Python dependency chain; why HTML responses don't work for non-browser clients (the core justification for DRF's existence); a text rendition of the request/response diagram (the video's own diagram was delivered as a separate PDF, not part of this transcript); the PUT-vs-PATCH difference made concrete with a field-level example; why registering `rest_framework` in `INSTALLED_APPS` is a required, separate step from `pip install`; and how this lecture's "code will shrink over time" comment maps onto the unit's actual lecture-by-lecture arc.
- **Researched:** the formal REST traits of statelessness and resource-based URLs; idempotency of GET/PUT/DELETE vs. POST/PATCH; DRF's official installation requirements and optional extras (django-filter, markdown) per the DRF docs; conventional plural-noun resource naming; and the industry best practice of virtual environments + pinned versions for installing DRF.

Double-check against the completeness checklist: API definition ✓, real-world API examples ✓, homogeneous/heterogeneous communication ✓, DRF-as-Django-extension framing ✓, Python/Django prerequisites ✓, REST definition & architectural-style clarification ✓, REST API vs. RESTful API ✓, cross-language terminology (Django/​.NET/​Java) ✓, DRF advantages/features list ✓, serialization/deserialization named (detail deferred to REST API Session 2+) ✓, private/partner/public API types ✓, request/response cycle diagram ✓, JSON vs. XML vs. DTL comparison ✓, HTTP methods ↔ CRUD mapping ✓, API resource/endpoint anatomy ✓, project setup steps (startproject/startapp/pip install/INSTALLED_APPS) ✓, closing preview of next session ✓. Nothing from the transcript was left uncovered.
