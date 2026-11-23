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
- **[specifications.md](specifications.md): the app specification.** It
  describes everything the apps do, screen by screen: sites and login, the
  feed, the post page, the editor, media handling and video compression, tags,
  location, dates, sharing, errors, security, accessibility, and appearance.
  Both apps follow it so they stay feature-equivalent. It ends with a
  platform mapping and a parity checklist.

## Projects

| Folder | What it is | README |
|--------|------------|--------|
| `../moby_backdrop` | The Moby module for Backdrop CMS (`moby`). It provides the API, the settings and setup wizard, and the carousel for showing posts. | [README](../moby_backdrop/README.md) |
| `../moby_android` | The Android app (`org.mobyapp.android`), built with Gradle. It is the reference implementation of the specification. | [README](../moby_android/README.md) |
| `../moby_ios` | The iOS app (`org.mobyapp.ios`), built on Ubuntu with xtool. | [README](../moby_ios/README.md) |

The module is used on quicksketch.org through a symlink,
`public_html/modules/custom/moby`, which points to `../moby_backdrop`.

## How the documents relate

- **The API comes first.** API.md defines what a site offers, and
  specifications.md refers to it rather than repeating it. The apps adapt to
  each site's discovery document (`GET /meta`) and never hard-code what a site
  supports.
- **Android is the reference app.** When the specification and the Android
  app disagree, decide which is right, fix the other one, and then make the
  iOS app match.
- **Every API change gets a version.** Any change to the API, additive or
  breaking, adds a row to the changelog in API.md and adds one to
  `MOBY_API_VERSION` in `moby_backdrop/moby.module`. The apps record the
  version they're built for in `ApiVersion.kt` and `ApiVersion.swift`. See
  "Versioning" in both documents.
- **Shared rules stay in step.** Hashtags, HTML descriptions, dates, site
  addresses, media checks, and video compression must give the same results
  on the server and in both apps. The specification's "Shared test cases"
  lists cases both apps' unit tests cover.

## Status

Everything is under development. API.md lists what the server has implemented,
and the iOS README lists what that app still lacks compared with the
specification.
