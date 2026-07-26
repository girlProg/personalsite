---
title: "Inheriting a Time Bomb: Rewriting a Legacy Attendance System Without Downtime"
date: 2026-07-04
tags: [django, drf, legacy-migration, nginx, postgres, docker]
---

I inherited a production system I didn't build. There was no documentation and no handover, and the people who originally wrote it were long gone, so what I had to work with was a PHP and MySQL codebase and a list of things that kept breaking that only seemed to get longer.

The problems were structural. The database wasn't normalised, so the same facts were duplicated across several tables, and depending on which one you read you could get a different answer to the same question. Attendance was the worst of it, because attendance is what the whole programme runs on: it decides which students are eligible for payment, and eligibility numbers that disagree with each other are not a cosmetic bug in a system that moves money to real people.

I maintained it for a while, until I hit the point where every change I made caused more bugs than I could keep up with, with no tests to tell me what I'd broken and no documentation to tell me why the code was written the way it was. Code with no tests and no documentation is a time bomb, and the technical debt doesn't sit still, it builds faster than you can clear it. So I stopped patching and rewrote it.

This post is about how, because the interesting part isn't that I chose Django, it's the specific decisions that let me replace a live system without taking it down, and the ones I'd defend if someone asked me why I did it that way.

## The write path is the whole migration

The instinct for replacing a legacy system incrementally is the strangler fig: stand the new system up alongside the old one, move one slice across at a time, and let the old system shrink. The slice everyone reaches for first is the one they understand best, and for me that was attendance.

What made it tractable is that attendance is the only thing in the system that actually gets written. Everything else is read-only reporting, lookups, historical records that nobody edits. There is exactly one live write path, and it is attendance submission. That fact is what lets a new database become the source of truth for one bounded thing without disturbing everything that reads it.

The discipline that keeps this safe is one writer per table. Once Django owns attendance, PHP stops writing those tables entirely and reads them back through an endpoint if it still needs them, because the moment two systems write the same table you have recreated the exact drift the rewrite exists to kill, one layer down.

I want to be honest that I didn't end up doing the incremental version. The payments-eligibility view reads attendance, and if I migrated attendance while leaving eligibility reading the old table, I'd have two attendance tables with one going stale, which is the drift problem wearing a different hat. Attendance, eligibility and authentication were entangled tightly enough that once I was doing all three I'd built the hard eighty percent, and the rest was ordinary CRUD not worth running two systems to preserve. So it became a full rewrite-replace. The strangler-fig thinking still shaped it, because the write-path analysis is what told me which table had to move first and why the others could follow.

## The attendance model, and computing the average without letting it drift

Attendance is recorded weekly, not daily, so a record is one row per student per week holding four booleans for Monday through Thursday. Present, sick, and public holiday all count as attended, so each of those sets the day's boolean to true, and the weekly average is just the count of true days over four.

That average is read constantly on the frontend and needed for reporting across roughly a hundred thousand students, which is millions of weekly rows once cohorts accumulate. The obvious move is to store it in a column and keep it updated, and the obvious way to keep it updated is a signal on save. That is the move I deliberately didn't make, because signals leak in exactly the ways that matter here: they don't fire on `bulk_update`, or on a queryset `.update()`, or on raw SQL, or on the bulk operations my legacy-data migration would use to move millions of rows. A signal-maintained column would be wrong on most of those rows silently, which is the same "a stored copy disagrees with its source" disease I was rewriting the system to cure.

What I used instead is a database-level generated column, so the database itself computes and stores the value from the source columns on every write, and it cannot drift because application code never touches it:

```python
attendance_average = GeneratedField(
    expression=(
        Case(When(monday=True, then=1), default=0, output_field=IntegerField())
        + Case(When(tuesday=True, then=1), default=0, output_field=IntegerField())
        + Case(When(wednesday=True, then=1), default=0, output_field=IntegerField())
        + Case(When(thursday=True, then=1), default=0, output_field=IntegerField())
    )
    * 25,
    output_field=DecimalField(max_digits=5, decimal_places=2),
    db_persist=True,
)
```

Four booleans, each worth twenty-five percent on a fixed denominator, so the stored value is a clean percentage. `db_persist=True` is the part that matters at this scale, because on Postgres a stored generated column is the one you can index, and the reporting queries that hurt are the ones that scan the whole table looking for students below a threshold. The value is right on every row by construction, including the rows loaded by a bulk migration that never ran a line of my Python, which is the whole point.

## The auth boundary I got wrong first

The frontend is a standalone single-page app, so it authenticates against the API directly with a token rather than a session, and wiring that up taught me a distinction I'd been fuzzy on. Django's `AUTHENTICATION_BACKENDS` and DRF's `DEFAULT_AUTHENTICATION_CLASSES` sound like the same thing and are not. The backends answer "given a username and password, how do I verify them and find the user," and they run once, at login. The DRF authentication classes answer "this incoming API request is carrying a token instead of credentials, how do I trust it," and they run on every request.

I started by trying to add JWT to the backends, which is the wrong layer entirely. JWT is request authentication, so it belongs in the DRF classes, and the backends stay exactly as they were:

```python
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": (
        "rest_framework_simplejwt.authentication.JWTAuthentication",
        "rest_framework.authentication.SessionAuthentication",
    ),
```

JWT is first so the bearer token on an API call is what gets checked, and I kept SessionAuthentication so the browsable API still works while I'm clicking around logged in through the admin. I dropped the default token auth the project shipped with, because for a system I control end to end, fewer authentication mechanisms is fewer things to reason about. SimpleJWT specifically, rather than the framework's other options, because the frontend was already built around an access and refresh pair, and matching the shape the client expected was less work than reshaping the client.

## One nginx, many apps, two environments

The whole coexistence lives in one host nginx acting as the single front door, and it is a smaller amount of config than it sounds. The main site routes by path prefix: the API, admin, and static go to the Django container, and everything else goes to the frontend container.

```nginx
location ^~ /static/ {
  proxy_pass http://127.0.0.1:8000;
  proxy_set_header Host $http_host;
}
location ^~ /api/ {
  proxy_pass http://127.0.0.1:8000;
  proxy_set_header Host $http_host;
  proxy_set_header X-Forwarded-Proto $scheme;
}
location / {
  proxy_pass http://127.0.0.1:8090;   # frontend container
  proxy_set_header Host $http_host;
}
```

The `^~` on `/static/` is load-bearing and I found out the hard way. The inherited config had a regex location matching every static file extension and serving it from the PHP root, and a plain prefix location loses to a regex match, so my SPA's own JavaScript and CSS were being routed into PHP and coming back wrong. The `^~` tells nginx that if this prefix matches, stop looking at regex locations, which hands the SPA's assets back to the frontend container.

The old PHP app moved to a subdomain rather than a path. Serving it under something like `/legacy/` would have broken it, because its internal links and redirects all assume they sit at the root, so `legacy.kadagile.online` serves it at root unchanged and reuses its original working config behind the proxy. Getting there also taught me that adding a subdomain server block isn't enough on its own: after restructuring the config, nginx needed a full restart rather than a reload before it would rebind the port the PHP block listens on, which cost me a confusing stretch of 502s that a reload wasn't fixing.

Staging is the same pattern pointed at a second set of containers. It runs the full production settings on `staging.kadagile.online`, proxying to a Django container on a different port and its own frontend container, with its own Postgres so it never touches production data. The two stacks coexist on one box, and the thing that actually keeps them separate is giving each Docker Compose project an explicit name, which I learned by bringing production down: without distinct project names, a `compose up` in the staging directory decided the production containers belonged to it and recreated them.

## Why not Next, and why not the reverse proxy the template shipped

Two forks I get asked about. I didn't use Next.js because the entire application sits behind a login, so there is no SEO to win and no first paint for anonymous visitors to optimise, and server-side rendering plus a second runtime to keep alive would be machinery in service of nothing. The backend already exists behind DRF, which leaves the frontend as a client for an API, and a client is a single-page app.

The Django template I started from ships Traefik as its reverse proxy and TLS manager, and I left it out, because the host nginx already terminates TLS and routes three apps, and two things cannot own ports eighty and four-four-three at once. Adopting Traefik would have meant rebuilding working routing in a second tool's config model to replace something already running. The one convenience I gave up is automatic certificate renewal, so I run certbot against the host nginx myself, which is a scheduled task rather than a feature I had to design.

## Where it stands

The system is in production use, with real data collectors on it, and the migrated attendance matches the legacy system, which was the thing I most needed to verify because eligibility and everything downstream of it depends on that being exactly right. The staging environment now sits in front of production as the rehearsal space, so the next changes get tested on a production-shaped stack before they reach anyone.

The lesson I didn't expect going in was how much of good architecture is just refusing to let a value exist in two places that can disagree. The unnormalised tables, the average I could have kept in sync with a signal, the attendance table two systems might both have written, the eligibility rule that a downstream service could have recomputed for itself: every one of those is the same mistake, and the whole rewrite is really a series of decisions to have one source of truth and make the database, not my memory, the thing that enforces it. I inherited the cost of code that didn't do that, and the clearest thing the experience left me with is that a system shouldn't break just because the person who understood it isn't in the room.