# Moby documentation

Moby posts photos, videos, and short descriptions from a phone to a website,
much like a social media app posts to a service. A post has a description, a
date, a location, tags, and one or more photos or videos. The website runs the
Moby module, which provides a REST API. The iOS and Android apps use that API
to log in, upload media, and create, list, edit, and delete posts.

This folder holds the documentation that applies across all of the Moby
projects. Each project's own README covers how to build and test that project.

## Documents

- **[API.md](API.md): the Moby REST API.** It is the authoritative description
  of every endpoint, with its request and response bodies, authentication,
  errors, and versioning. It also covers how posts map to a site's content type
  and fields, and lists open questions. Every server module implements it, and
  both apps use it.
- **[SPECIFICATIONS.md](SPECIFICATIONS.md): the app specification.** It
  describes everything the apps do, screen by screen: sites and login, the
  feed, the post page, the editor, media handling and video compression, tags,
  location, dates, sharing, errors, security, accessibility, and appearance.
  Both apps follow it so they stay feature-equivalent. It ends with a
  platform mapping and a parity checklist.

## Projects

| Repository | What it is |
|------------|------------|
| [backdrop-contrib/moby](https://github.com/backdrop-contrib/moby) | The Moby module for Backdrop CMS (`moby`). It provides the API, the settings and setup wizard, and the carousel for showing posts. |
| [quicksketch/moby_android](https://github.com/quicksketch/moby_android) | The Android app (`org.mobyapp.android`), built with Gradle. It is the reference implementation of the specification. |
| [quicksketch/moby_ios](https://github.com/quicksketch/moby_ios) | The iOS app (`org.mobyapp.ios`), built on Ubuntu with xtool. |

## How the documents relate

- **The API comes first.** API.md defines what a site offers, and
  SPECIFICATIONS.md refers to it rather than repeating it. The apps adapt to
  each site's discovery document (`GET /meta`) and never hard-code what a site
  supports.
- **Android is the reference app.** When the specification and the Android
  app disagree, decide which is right, fix the other one, and then make the
  iOS app match.
- **Every API change gets a version.** Any change to the API, additive or
  breaking, adds a row to the changelog in API.md and adds one to
  `MOBY_API_VERSION` in the module's
  [`moby.module`](https://github.com/backdrop-contrib/moby/blob/1.x-1.x/moby.module).
  The apps record the version they're built for in
  [`ApiVersion.kt`](https://github.com/quicksketch/moby_android/blob/main/app/src/main/java/org/mobyapp/android/data/ApiVersion.kt)
  and [`ApiVersion.swift`](https://github.com/quicksketch/moby_ios/blob/main/Sources/MobyCore/ApiVersion.swift).
  See "Versioning" in [API.md](API.md#versioning) and
  [SPECIFICATIONS.md](SPECIFICATIONS.md#versioning).
- **Shared rules stay in step.** Hashtags, HTML descriptions, dates, site
  addresses, media checks, and video compression must give the same results
  on the server and in both apps. The specification's "Shared test cases"
  lists cases both apps' unit tests cover.

## Status

Everything is under development. API.md lists what the server has implemented,
and the iOS README lists what that app still lacks compared with the
specification.
