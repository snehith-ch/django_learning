# Lecture 42 — A guided tour of two finished projects: Teacher Directory and MiniBlog

Source: `transcripts/Django42.txt`
Covers: unlike every earlier project lecture (build-alongs from an empty folder), this lecture is a **guided tour of two already-completed projects** — the presenter walks through pre-written code, explaining structure and demonstrating behavior in the browser, rather than typing it live. This is also the final Django-focused lecture in this run of transcripts: the video closes by announcing the course's move into Django REST Framework starting the next session — a distinct topic not covered by these notes.

Legend:
- **[From video]** — explained directly in the transcript
- **[Gap-filled]** — my own explanation, added because the video skipped or under-explained something
- **[Researched]** — pulled from official Django docs or other trusted sources
- **[Example]** — an extra worked example beyond what the video showed

---

## A note on this lecture's format **[Gap-filled]**

Every other project lecture in this course (the MySQL CRUD project, the image upload project) was built live, line by line, with every file's contents dictated on screen. This lecture is different: the presenter opens two **finished** projects in PyCharm and clicks through their file structure and running behavior, narrating what each piece does rather than writing it. Because of that, these notes describe the **architecture, features, and Django patterns being demonstrated** — the "what" and "why" of each project — rather than transcribing exact file contents that were never fully dictated on screen. Where the video does show a specific, complete snippet (the CSV/model shape, the filtering logic), that's included directly; where it only describes a file's purpose while scrolling past it, that's represented as a description, not invented code.

---

## 1. Teacher Directory — project overview **[From video]**

> A client has requested a directory application containing all the teachers in a given school. Each teacher should have: first name, last name, profile picture, email address, phone number, room number, and the subjects they teach.

Stated requirements, read directly from the project's own "About Us" page:

- Teachers can share a first/last name, but **email address must be unique**.
- A teacher can teach **no more than five subjects**.
- The directory should be **filterable** — by first name, last name, or subject.
- Clicking a teacher opens a **profile page** showing their full details.
- An **import module** lets teacher data be bulk-loaded from a CSV file (plus a ZIP of profile images) — but **only for logged-in users**; anonymous visitors cannot run the import.
- If no profile image is available for a teacher, a **default placeholder image** should be used instead.
- SQLite is explicitly called out as an acceptable backend "for simplicity" — this project doesn't need MySQL the way Lecture 39–40's project deliberately did.

> **[Gap-filled] — this is a worked example of writing a spec before building.** Nothing about this requirements list is Django-specific — it's a small, plain-English product spec of the kind a real client or product owner might actually hand a developer. Worth noticing as a category distinct from the code itself: before any model or view exists, the project defines *what* it needs to do and what rules govern the data (uniqueness on email, a five-subject cap, who's allowed to import). Translating a spec like this into models/views/forms is the actual day-to-day work of building a Django app from a real requirement, not just from a tutorial's step-by-step instructions.

## 2. Teacher Directory — architecture, as demonstrated **[From video]**

- **Two models**: a `Login` table (a simple admin-style username/password pair, separate from Django's built-in `auth_user` — the project rolls its own minimal login rather than using `django.contrib.auth`) and a `Teacher` model holding the fields from the spec above.
- **All URLs are project-level** (no app-level `urls.py` — a valid, if less scalable, choice for a project this size, contrasted with the app-owns-its-URLs convention established since Lecture 7).
- **Around ten views total**: a main/index view, a login-check view, a home view, an import-form view, a bulk-import-processing view, a "show all teachers" view, a filter view, a profile view, a profile-image view, and an add-teacher view.
- **A `media/` folder** holds both the uploaded/bundled profile pictures and a `teachers.csv` file (first name, last name, profile picture filename, email, phone number, and more, one row per teacher) — the same `MEDIA_ROOT` pattern from Lecture 41, now used for bulk-importable data rather than user-uploaded files.
- **Template inheritance** (Lecture 22): an `app_base.html` at the project root defines shared structure; page templates like `home.html` extend it. A separate `base.html` inside the app itself defines the filter form's shared layout, extended by other pages that need the same filtering UI.

### Login-gating the import feature

> Only logged-in users can run the import module — without login details, they cannot run it.

The video demonstrates this directly: visiting the import page while logged out redirects back to a login prompt; logging in with a valid admin username/password (stored in the project's own `Login` table, not Django's built-in auth) allows the import page to load.

> **[Gap-filled] — this is a hand-rolled version of `is_authenticated`.** Because this project uses its own `Login` model instead of `django.contrib.auth`, "is this user logged in" has to be tracked and checked manually (most likely via a session value set at login time, checked at the top of the import view) rather than the built-in `request.user.is_authenticated` pattern from Lecture 28. Both achieve the same *effect* — gating a feature behind login — but the built-in `django.contrib.auth` system (covered in depth in Lectures 26–29) gets you the same protection with substantially less custom code, and is generally the better default choice for a new project unless there's a specific reason to roll a custom login table.

### Filtering by first/last name or subject

The video demonstrates typing a subject name (e.g. "Python") into a filter form and getting back only teachers who teach it — the same `.filter()` mechanism from Lecture 37, most likely combined with `__icontains` (Lecture 38) so a partial, case-insensitive match works, applied to whichever field (first name, last name, or subject) the visitor chose to filter by.

## 3. MiniBlog — project overview **[From video]**

> This is closer to a real blog: any signed-up user can write posts, but only a **superuser** can edit or delete a post.

Unlike Teacher Directory, MiniBlog uses Django's **built-in authentication system** (`django.contrib.auth`) directly — the same `UserCreationForm`/`AuthenticationForm`/`login()`/`logout()` machinery covered in depth back in Lectures 27–28, applied to a real blogging use case.

### Demonstrated behavior

- **Sign-up** — a `SignUpForm` (the Lecture 27 pattern: a `UserCreationForm` subclass with extra fields) collects username, first/last name, email, and password; a successful sign-up shows a "Congratulations, you've become an author" message.
- **Password-strength validation** on sign-up surfaces the same built-in messages from Lecture 27 ("password too similar to username," "password too common") — confirming this project relies on the same `AUTH_PASSWORD_VALIDATORS` machinery, not custom validation logic.
- **Login** — the `AuthenticationForm` pattern from Lecture 28.
- **Adding a post** — any logged-in user (not just a superuser) can create a new post via an "Add Post" form and see it appear on their dashboard.
- **Editing a post** — the video shows a *regular* logged-in user successfully editing their own post's title/description.
- **Deleting a post** — attempting this as a regular (non-superuser) account shows **no delete option available at all** in the UI. Creating a superuser (`python manage.py createsuperuser`, Lecture 8) and logging in as that account instead reveals a delete action, which works.

> **[Gap-filled] — how this permission gating most likely works, connecting it to what's already been covered.** The video doesn't show the exact conditional, but the described behavior — a delete option that appears only for superusers — is the same shape as the `request.user.is_authenticated` gate from Lecture 28's profile view, extended one step further to `request.user.is_superuser` (the flag from Lecture 26's `auth_user` table walkthrough). A view or template checking `if request.user.is_superuser:` before showing/allowing the delete action is the standard, minimal way to implement exactly this behavior — reusing a field this course has already covered in detail, rather than anything new.

> **[Researched] — a more scalable alternative to hand-checking `is_superuser` everywhere.** Per the [Django docs](https://docs.djangoproject.com/en/stable/topics/auth/default/#permissions-and-authorization), for anything beyond "superuser vs. everyone else," Django's built-in **permissions framework** (`user.has_perm('app_label.permission_codename')`, and the `@permission_required` view decorator) lets specific, named permissions — like "can delete post" — be granted to specific users or groups individually, without requiring full superuser status. For a small project like this one, checking `is_superuser` directly is a reasonable, simple choice; a larger real-world project with several different roles (editors, moderators, admins) would more likely reach for named permissions instead.

## 4. MiniBlog — architecture, as demonstrated **[From video]**

- **One model**: `Post` (title, description) — the blog content itself, registered in the admin.
- **Static assets included directly in the project** (`static/css/bootstrap...`, `static/js/jquery...`) rather than loaded from a CDN — the local-file approach from Lecture 18's two ways to add Bootstrap/jQuery, contrasted with the CDN approach used in the image-upload and MySQL-CRUD projects from Lectures 39–41.
- **A `navigation.html`** partial, included into the shared `base.html`, builds its links using named URLs (`{% url 'dashboard' %}`, `{% url 'login' %}`, `{% url 'sign_up' %}`, `{% url 'logout' %}`, `{% url 'add_post' %}`) — the Lecture 21 named-URL pattern, now applied to an entire site's navigation bar rather than a single link.
- **Views**: home (lists every post via `Post.objects.all()`), about, contact (the video notes the contact form doesn't actually persist submissions anywhere — a deliberately unfinished/placeholder feature), dashboard (visible only to authenticated users), sign-up, login, logout, add-post, update-post, delete-post.

---

## 5. What this lecture is (and isn't) meant to teach **[Gap-filled]**

Both projects are explicitly offered as **downloadable, practice-by-reading-and-running** material — the video's closing instructions are to download the `.rar` archives, extract them, open each in PyCharm, and study the working code directly rather than follow written build steps. Every individual technique on display — model relationships, `ModelForm`, template inheritance, `django.contrib.auth`, CSV-based bulk import via Python's `csv`/`os` modules, image uploads — is something this course already introduced piece by piece in earlier lectures (Lectures 21–41). What's genuinely new about this lecture isn't any single technique; it's seeing several of them **combined into two complete, real-shaped applications**, which is a different (and useful) kind of learning from following a single new concept in isolation.

> **Industry best practice:** Reading a complete, working codebase end-to-end — even one you didn't write — is a genuinely valuable exercise distinct from following tutorials, precisely because real projects combine many small decisions (which permission model to use, where to put shared templates, whether to use app-level or project-level URLs) that a single-concept lesson never has to make. Downloading and running both of these projects locally, then deliberately trying to extend them (e.g. adding a "most recent posts" filter to MiniBlog, or a CSV export to Teacher Directory) is a natural next step beyond just reading the code.

---

## Wrap-up

- **From video:** a full walkthrough of Teacher Directory (requirements, a hand-rolled login-gated CSV/ZIP bulk-import feature, name/subject filtering, project-level-only URLs) and MiniBlog (built on `django.contrib.auth`, author-vs-superuser permission gating for edit/delete, a full nav-bar built from named URLs); both offered as downloadable practice projects; the course's announced transition to Django REST Framework starting the next session.
- **Gap-filled:** an explicit note on this lecture's different format (a code tour, not a build-along) and what that means for how these notes represent it; how MiniBlog's superuser-only delete gating most likely works, tying it back to `is_superuser` from Lecture 26; framing the requirements-first structure of Teacher Directory as a transferable habit, not just project trivia.
- **Researched:** Django's built-in permissions framework as the more scalable alternative to hand-checking `is_superuser` once a project needs more than two tiers of access.

Double-check: this lecture doesn't introduce any new Django API beyond what Lectures 21–41 already covered — its value is in seeing those pieces assembled into two complete, differently-shaped real applications (an admin-only directory vs. a multi-user blog), not in new syntax to learn. Django REST Framework, announced here as the next topic, is a distinct framework-on-top-of-Django and is not covered by these notes.
