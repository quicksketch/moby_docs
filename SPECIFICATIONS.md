# Moby app specification

This document describes everything the Moby app does, so the Android and iOS
apps can be built and kept feature-equivalent. Both apps must behave as
described here. Where the platforms differ, the section **Platform mapping**
says how each feature is built on each one.

The Android app (`org.mobyapp.android`) is the reference implementation. When
this document and the Android app disagree, decide which is right, then fix the
other, and fix the iOS app to match.

The server side is the Moby module for Backdrop CMS. [`API.md`](API.md), next
to this file, is the authoritative description of the REST API, and this
document refers to it rather than repeating it.

## Contents

1. Purpose and principles
2. The API
3. Stored data
4. Navigation
5. Screens
   - Sites
   - Add site and log in
   - Feed
   - Post page
   - Post editor
   - Settings
6. Media handling
7. Tags and hashtags
8. Descriptions and HTML
9. Location
10. Dates
11. Sharing from other apps
12. Errors and session expiry
13. Security and privacy
14. Accessibility
15. Appearance
16. Text
17. Platform mapping
18. Parity checklist

---

## 1. Purpose and principles

Moby posts photos, videos, and short descriptions from a phone to a Backdrop
CMS site running the Moby module, much like a social media app posts to a
service. A person can connect several sites, browse each site's recent posts,
create posts, and edit or delete them.

These principles apply throughout:

- **The site decides.** The app reads the site's discovery document and adapts
  to it: the login form, the allowed file types and sizes, the number of media
  items, and which features (location, tags, drafts) are shown. The app never
  hard-codes what a particular site supports.
- **Hide what the site doesn't support.** A feature the site doesn't offer is
  not shown, rather than shown disabled.
- **Never lose work or create duplicates.** Uploads start immediately and can
  be retried. Creating a post uses a client ID so a retry never makes a second
  post. A failed load never lets a blank form be saved over an existing post.
- **Plain text is shown, never HTML.** Descriptions from the site are always
  displayed as plain text inside the app's own screens.
- **The token only goes to the site.** The API token is stored encrypted and is
  never sent to any other host.
- **Each site is separate.** Every site has its own login and its own settings.

## 2. The API

All requests go to `{site}/api/moby`, with the API version the app is built
for as the `api` query parameter, such as `{site}/api/moby/posts?api=1`. See
[`API.md`](API.md) for each endpoint, its request and response bodies,
and error codes, and **Versioning** below for how apps handle sites with a
different version.

### Versioning

The API has one version number, which goes up by one with every change to
the API. The site's module and the apps update separately, and apps usually
update first, so an app adapts to whatever version a site has.

- **What each app keeps** (`ApiVersion.kt` on Android, `ApiVersion.swift` on
  iOS):
  - `CURRENT`: the version the app is built for, sent as `?api=` on every
    request.
  - `MIN_SITE`: the oldest site version the app can work with.
  - A list of changes after version 1. Each has its version, a short label for
    users (or none, if they wouldn't notice it), and what the app does on sites
    without it.
- **The site's versions** come from discovery's `api_version` (the newest it
  implements) and `api_min_version` (the oldest it still answers). Sites
  without `api_version` count as version 1.
- **Compatibility**, worked out after discovery:

  | State | When | What the app does |
  |---|---|---|
  | Current | `api_version` ≥ the app's version | Nothing; a newer site answers as the app's version. |
  | Behind | `api_version` < the app's version | Uses each missing change's fallback, and shows the update notice if any missing change has a label. |
  | Site too old | `api_version` < `MIN_SITE` | Blocks the site: "This site's Moby module needs updating before the app can use it. Ask the site's administrator to update it." |
  | App too old | the app's version < `api_min_version`, or any `426 api_version_unsupported` response | Blocks the site: "Update the Moby app to post to this site." (or the site's own `426` message). |

- **Code that depends on a change** checks the site's version first
  (`supports(version)`), rather than whether a property is present.
- **The update notice:**
  - **Sites list:** a site that's behind shows "Site update available" under
    its login status, in the secondary color.
  - **Feed:** a card at the top: "**This site's Moby module is out of date.**
    Ask the site's administrator to update it to use: {labels}." with
    **Dismiss**.
  - **Dismissing** lasts until the site's or the app's version changes, so
    the notice comes back when the gap changes.
  - **Blocked sites** show their message on the Add site and Log in step, like
    setup issues.
- **Adding a change:** the module adds a row to API.md's changelog and bumps
  its version. An app that starts using the change raises `CURRENT`, adds the
  change to its list, and guards the code that uses it with `supports`. Both
  apps add the same change with the same label.

| Endpoint | Used for |
|---|---|
| `GET /` | Lists the API's URLs, like the root of GitHub's API. The apps don't use it; their paths are fixed for each API version. |
| `GET /meta` | Discovery: site name and favicon, auth mechanism, limits, media types and sizes, description format, features, setup issues. |
| `POST /auth/token` | Logging in. Sends the credential fields plus `device_name`. |
| `DELETE /auth/token` | Logging out (revokes the token). |
| `GET /me` | The logged-in user and their `permissions` (`create`, `publish`). |
| `POST /media` | Uploading a photo or video (multipart: `file`, optional `alt`, optional `thumbnail`). |
| `DELETE /media/{id}` | Deleting an upload that isn't attached to a post yet. |
| `GET /posts?page=&limit=` | The feed, 20 posts per page. |
| `GET /posts/{id}` | One post, for the post page and the editor. |
| `POST /posts` | Creating a post, with `client_id`. |
| `PATCH /posts/{id}` | Saving an edited post. |
| `DELETE /posts/{id}` | Deleting a post. |
| `GET /tags?search=&limit=8` | Tag suggestions while typing. |

Requests:

- Every request sends `Accept: application/json`.
- Authenticated requests send `Authorization: Bearer {token}`.
- JSON bodies are sent as `application/json`.
- Unknown properties in responses are ignored, so newer servers work with older
  apps. Properties added in later versions of the module have defaults, listed
  in the next table.

Timeouts: connect 20 seconds, read 60 seconds, write 5 minutes (large videos on
slow connections).

Defaults for properties that older servers may not send:

| Property | Default |
|---|---|
| `site.favicon` | none |
| `setup_issues` | empty |
| `limits.max_description_length` | none |
| `limits.max_alt_length` | 2000 |
| `limits.max_tags_per_post` | 50 |
| `limits.max_tag_length` | 255 |
| `media.video.thumbnail` | none (the site doesn't store thumbnails) |
| `description` | `{ format: null, html: false }` |
| `features.*` | all false |
| `display.order` | `date`, `media`, `description`, `location` |
| `user.permissions` | `create` and `publish` false |
| `post.permissions` | `edit` and `delete` false |
| `post.tags` | empty |
| `media.alt` | empty |

Errors have the shape `{ "error": { "code", "message", "field" } }`. The app
shows `message` to the user, and uses `field` to mark the input it applies to.
A response that isn't in this shape becomes code `http_{status}` with the
message "The site returned an unexpected error ({status})."

## 3. Stored data

### Sites

A list of saved sites, in the order added. Each has:

| Field | Meaning |
|---|---|
| `id` | A UUID made by the app. |
| `url` | The normalized base URL, such as `https://example.com`. |
| `name` | The site's name from discovery. |
| `favicon` | The favicon URL from discovery. |
| `customTitle` | A title the user chose, shown instead of `name`. |
| `userName` | The logged-in user's name. |
| `token` | The API token, **encrypted**. Absent when logged out. |

Rules:

- A site is **logged in** when it has a token.
- The **display name** is `customTitle`, else `name`, else the URL's host,
  else the URL.
- Sites are matched by URL, ignoring case. Logging in to a URL already in the
  list updates that site and keeps its ID and custom title.
- The ID of the **open site** is stored too, so it survives the app being
  stopped.
- Name and favicon are refreshed from discovery whenever it is loaded. This
  happens each time the Sites screen is shown, for every site, in parallel;
  sites that can't be reached keep what was saved.

### Per-site settings

Stored separately for each site. They are kept when logging out and deleted
when the site is removed.

| Setting | Default | Meaning |
|---|---|---|
| Include Location | on | New posts fill in the device's location when opened. |
| Publish by Default | on | New posts start as published; when off, as drafts. |

### Not stored

Discovery and the current user are cached in memory for the open site only,
and cleared when another site is opened or the user logs out. Posts, drafts in
progress, and media are not stored by the app.

## 4. Navigation

```
Sites ──► Add site / Log in ──► Feed ──► Post page ──► Editor (edit)
  │                              │  └──► Editor (new)
  └──► Settings (site menu)      └──► Settings
```

- The app opens on **Sites**.
- Opening a logged-in site opens its **Feed**, which replaces everything above
  Sites in the back stack, so Back from the Feed returns to Sites.
- Opening a logged-out site opens **Log in** for it.
- After logging in, the Feed opens.
- After a new post is saved, the editor closes, and the Feed reloads its first
  page and scrolls to the top.
- After an edit is saved, the editor closes. The Post page reloads, and the
  Feed reloads that one post in place.
- After a post is deleted, the Post page closes, and the Feed removes it
  without reloading or losing its scroll position. See **Feed** for how paging
  is kept right.
- Whenever the token is rejected (HTTP 401), the app logs out of the site
  locally and returns to Sites.

## 5. Screens

### Sites

Title: **Sites**.

**No sites:** the screen shows "No sites connected yet." and an **Add site**
button.

**With sites:** a list of cards, one per site, and a floating **Add site**
button.

Each card shows:

- **Icon:** the site's favicon in a 40-point rounded square. Until the favicon
  loads, or if it fails or there is none, the first letter or digit of the
  display name, uppercased, is shown on a tinted background ("?" if none).
  Favicons can be SVG, ICO, or PNG. The icon is decorative.
- **Title:** the display name, on one line.
- **Address:** the URL without its scheme, on one line.
- **Session:** a dot and a label. A green dot means "Logged in as {userName}"
  (or "Logged in"); a gray dot means "Logged out". The dot and label are read
  together.
- **Options menu**, labeled "Options for {name}", with:
  - **Settings**, which opens that site's Settings.
  - **Log out**, shown only when logged in.
  - **Remove site**.

Tapping a card opens the site, as described in **Navigation**.

**Log out** asks "Log out of {name}?" with **Log out** and **Cancel**. Logging
out revokes the token on the site (`DELETE /auth/token`), then forgets it
locally even if revoking failed. The site stays in the list.

**Remove site** asks "Remove {name}?" with **Remove** and **Cancel**:

- If logged in, the message is "You'll be logged out, and the app's settings
  for this site will be deleted. Your posts stay on the site."
- Otherwise: "The app's settings for this site will be deleted. Your posts stay
  on the site."

Removing logs out first if needed, then deletes the site and its settings.

The cards and their menus are disabled while a log out or removal is in
progress.

**While sharing** (see **Sharing from other apps**), a banner at the top says
"Choose a site to post the shared photos and videos to." with a **Cancel**
button, which discards the shared media.

### Add site and log in

Title: **Add site**, or **Log in** for a saved site.

**Step 1: the address.** This step is skipped for a saved site, which shows
"Connecting to {url}…" while it connects.

- A **Site address** field, with placeholder `example.com`, a URL keyboard and
  a Go action.
- Below it, the description "URL of a site that has the Moby module or plugin
  installed (Backdrop, Drupal, or WordPress)."
- A **Continue** button, which shows a spinner while connecting.

The address is normalized:

- Spaces at either end and trailing slashes are removed.
- An address that is empty or contains a space is rejected with "Enter the
  site's address."
- `https://` is added if there is no scheme.

The app then loads discovery:

- **If the site can be used**, the app moves to step 2. A site can be used
  when `auth` is present and no setup issue has severity `error`. A site
  without a content type lists the `not_configured` error.
- **If it can't, and it lists setup issues**, the app shows a card titled
  "This site isn't ready for posting from the app". The card has an error
  color and lists each issue's message with a warning icon, so the user can
  pass them on to an administrator.
- **If it can't, and it lists no issues** (older servers):
  - Without `auth`: "This site does not allow logging in from the app."
  - Otherwise: "This site has not finished setting up posting from the app."

**Step 2: credentials.**

- The heading is "Log in to {site name}".
- If the site has setup issues that are only warnings, a card titled "Some
  features aren't available on this site" lists them, with a note icon.
- One field per entry in `auth.fields`, in order:
  - The label comes from the field.
  - `password` fields are masked, with a password keyboard; `email` fields get
    an email keyboard.
  - The Next action moves between fields, and the last field has Done.
  - The form is built only from discovery, so new auth mechanisms need no app
    change.
- **Log in** button, with a spinner while working.
  - A required field left blank shows "Enter your {label, lowercase}." and
    marks that field.
  - API errors show their message, and mark the field named by `error.field`.
- **Forgot your password?** is shown if `auth.password_reset_url` is set, and
  opens that URL in the browser.
- **Use a different site** returns to step 1. It is shown only when adding a
  site.

Logging in sends the credentials plus `device_name`:

- **Android:** the manufacturer and model, such as "Google Pixel 8". The model
  alone is used if it already starts with the manufacturer.
- **iOS:** the device's model name, such as "iPhone 16 Pro". Use the user's
  device name only if the platform allows it without extra entitlements.

On success, the site is saved with its encrypted token and user name, and
opened. Discovery and the user from the login response are cached, and the Feed
opens.

### Feed

The title bar shows:

- The site's display name. It updates if it is changed in Settings.
- **Back** (labeled "Sites"), which returns to Sites.
- **Settings**.
- **Log out**, which asks "Log out of {name}?" and then revokes the token
  (see **Sites**) and returns to Sites.

A floating **New post** button opens the editor.

**Loading:**

- The feed loads discovery (if not cached), then page 0 with 20 posts.
- Pull to refresh reloads page 0.
- A **Load more** button at the end loads the next page, and reads "Loading…"
  while it loads. It is shown while `(page + 1) × limit < total`.
- Posts already shown are skipped when a page loads, in case new posts shifted
  the pages.
- An empty feed shows "No posts yet. Tap New post to share photos or videos."
- Errors are shown as a temporary message (snackbar or toast).

**Each post card** is tappable, and opens the Post page. Its parts (date,
media, description and location) are shown in the order discovery's
`display.order` gives, which is the order posts show them in on the site.
Older sites without it use date, media, description, location. Unknown parts
are ignored, and missing ones are added at the end in that default order. It
shows:

- The post's date, in the device's medium date and short time format, in the
  device's time zone.
- The location name, if any, on one line.
- A **Draft** badge, if `status` is `draft`.
- The description as plain text (see **Descriptions and HTML**), up to 8 lines.
- A horizontal row of media thumbnails, 120 points square, with rounded corners
  and cropped to fill:
  - **Photos** show `thumbnail.url`, or the full `url` if there is no
    thumbnail (such as HEIC uploaded elsewhere).
  - **Videos** show `thumbnail.url` if present, with a play icon on a
    translucent dark circle and the duration ("m:ss"). Without a thumbnail, the
    video is **not** downloaded to get a frame; only a play icon and the
    duration are shown.
  - The alt text is the image's accessible label.

**Keeping the feed right after changes:**

- **New post:** reload page 0 and scroll to the top.
- **Edited post:** fetch that post and replace it in place.
- **Deleted post:** remove it, and remember its ID so a response already on its
  way can't add it back. Because every later post moves up one place on the
  server, reload the last page loaded so that **Load more** doesn't skip a
  post. If a page is loading at the time, do this when it finishes.

### Post page

Title: **Post**, with Back.

The page shows the post's **real web page**, as the site renders it for the
logged-in user, in a web view. See "Viewing a post's page" in `API.md`.

- Load `GET /posts/{id}` first, then load the post's `url` in the web view.
- Add a unique query parameter `moby_view={current time in milliseconds}` to
  the URL each time, so the site's page cache is bypassed and drafts render.
- Send `Authorization: Bearer {token}` with the page request **only** if the
  URL has the same scheme, host, and port as the site.
- On a redirect, send the header again if the new URL is on the same site. A
  redirect to another host is followed without the token.
- JavaScript and DOM storage are on. File and content access are off. Caching
  is off, so the current version is always shown.
- The web view is transparent until the page draws, so there's no white flash
  in dark mode.
- Links:
  - A link to the same page (only the fragment differs) stays in the web view.
  - Every other link opens in the system browser. If nothing can open it, it is
    ignored.
- While the post or page loads, a spinner is shown.
- If the post or page fails to load (an HTTP error or a network error for the
  page itself), the screen shows "This post couldn't be shown.", the reason,
  and **Try again**.

A floating toolbar in the lower corner (trailing side) shows the following.
On iOS, it's hosted in its own UIKit view, since SwiftUI buttons drawn over a
web view don't receive taps.

- **Edit post**, if `permissions.edit`. It opens the editor for this post.
- **Delete post**, if `permissions.delete`. It asks "Delete this post?" / "This
  can't be undone." with **Delete** (in the error color) and **Cancel**.
  - Deleting calls `DELETE /posts/{id}`. A 404 counts as success, since the
    post is already gone, such as when retrying after a timeout.
  - The button is disabled while deleting.
  - On success, the page closes and the Feed removes the post.

The toolbar is hidden if the user may do neither.

### Post editor

Title: **New post** or **Edit post**, with Back. The action button is **Post**
(new) or **Save** (editing). It shows a spinner while saving, and is enabled
only when the editor has loaded without error and isn't saving.

Back leaves without saving and without asking. Uploads that aren't attached to
a post stay pending on the site, which deletes them after about six hours
(Backdrop's temporary file cleanup).

**Loading:**

- Load discovery and the user (for `permissions.publish`). When editing, also
  load the post. Then add any shared media.
- A spinner is shown while loading.
- If loading fails, the form is **not** shown, so it can't be saved. The screen
  shows "This post couldn't be loaded." (editing) or "The editor couldn't be
  loaded." (new), the reason, and **Try again**. Shared media waits until a
  load succeeds.

**Form, in order:**

1. **Description**
   - A multi-line field, at least 4 lines, labeled "What's happening?".
   - It is marked when an error applies to `description`.
   - If the post's description contains HTML markup, the note "This post has
     formatting, so it is edited as HTML." is shown (see **Descriptions and
     HTML**).

2. **Media.** See **Media handling** for the details.
   - A horizontal row of tiles, one per item.
   - **Add photos or videos**, with a plus icon. It is shown when
     `max_media_per_post` > 0, and disabled when the post is full. It opens
     the system photo picker for photos and videos: multiple selection when
     more than one item can be added, single selection when only one can.
   - The count, "{n} of {max}.", followed by " Touch and hold an item to drag
     it into place." when there are two or more items.

3. **Tags.** Shown only if `features.tags`; see **Tags and hashtags**.
   - A "#" icon and the heading "Tags".
   - Chips for the tags added. Tapping one removes it; it is labeled "Remove
     tag {tag}".
   - Disabled chips "#{tag}" for hashtags in the description that aren't
     already tags. They are labeled "Tag {tag}, from a hashtag in the
     description".
   - An **Add a tag** field with suggestions; see **Tags and hashtags**.
   - The hint "Hashtags in the description, like #hiking, are added as tags
     too."

4. **Date:** a calendar icon, the date and time (the same format as the feed),
   and **Change**.
   - **Change** opens a date picker. Its **Next** button opens a time picker
     with **Done**, and each picker has **Cancel**.
   - Choosing a time sets the seconds to 0.
   - New posts default to now.

5. **Location.** Shown only if `features.location`; see **Location**.
   - A pin icon, the heading "Location", and **Clear**.
   - A **Place name** field, shown only if `features.location_name`. It is
     marked when an error applies to `location.name`.
   - A **Use current location** button, which reads "Finding location…" and is
     disabled while working.
   - Coordinates are never shown or typed. They come only from the device, are
     sent with the post, and **Clear** removes them.

6. **Status:** a switch with a title and a hint.
   - **If the user may publish:**
     - The title is "Published" and the hint "Turn off to save as a draft.".
     - The switch is on when the post is published.
   - **If not:**
     - The title is "Save as draft" and the hint "Drafts are not shown on the
       site.".
     - The switch is on when saving as a draft.
   - **When editing a draft the user may not publish:**
     - The switch is disabled.
     - The hint reads "A site administrator needs to publish this post."
   - **Initial state:**
     - New posts follow the **Publish by Default** setting.
     - Edited posts reflect their status.

**Alt text dialog.** It opens from a tile's **Alt** button.

- Title: "Alt text".
- A multi-line field (3 lines) labeled "Describe this for people who can't see
  it". It is limited to `max_alt_length` characters; typing past the limit is
  ignored.
- The field is focused when the dialog opens, so the keyboard appears straight
  away.
- The dialog is compact and sits directly above the keyboard, without a large
  gap below the field.
- **Done** and **Cancel**. The text is kept however the dialog closes (Done,
  swiping it away, or tapping outside it); only **Cancel** discards changes.

Alt text changed after upload is saved with the post, in its media list.

**Saving** checks, in order, and stops at the first problem:

1. An upload is still in progress: "Wait for uploads to finish."
2. An upload failed: "Some uploads failed. Retry or remove them."
3. No description and no media: "Add a description, or a photo or video." The
   description field is marked.
4. The location is invalid; see **Location**.
5. Text left in the tag field is added as a tag, as if the user had finished
   it. If that fails (too long, too many), it stops with that error.

The body is then built:

- **`description`:**
  - Trimmed, then encoded for the site's format (see **Descriptions and
    HTML**).
  - When creating, it is sent only if not empty.
  - When editing, it is always sent, so it can be cleared.
- **`date`:** ISO 8601 with the device's offset (see **Dates**).
- **`location`:**
  - Its `name` is sent if the site maps names and one was entered.
  - Its `latitude` and `longitude` are sent together, if the site maps
    coordinates and both are set.
  - It is left out when empty on create. On edit, it is sent as `null` when
    empty, which clears it.
- **`media`:** `[{ id, alt }]` in the tiles' order, with alt text trimmed.
  When editing, this list replaces the post's media.
- **`status`:**
  - If the user may publish: `published` or `draft`.
  - If not: `draft` when saving as a draft, or left out otherwise, so the site
    uses the content type's default.
- **`tags`:**
  - The tags added in the app, **without** those from hashtags, which the site
    adds itself.
  - Sent only if `features.tags`.
- **`client_id`** (create only): a UUID made once when the editor opens. It
  stays the same across retries, so the post is never created twice.

On success the editor closes (see **Navigation**). On failure the error message
is shown, and the field named by `error.field` is marked.

Messages and errors in the editor are shown as a temporary message (snackbar or
toast).

### Settings

Title: **Settings**, with Back. It opens from the Feed or from a site's menu on
Sites.

- The intro reads "These settings apply to {name}. Each site has its own."
- **Site** section:
  - A **Site title** field. It starts as the custom title, or else the site's
    name, and the site's name is the placeholder.
  - Changes are saved as the user types.
  - A blank title, or one equal to the site's name, removes the custom title.
  - The hint reads "Your own title, shown in the app instead of the site's."
    when a custom title is set. Otherwise it reads "The site's own title.
    Change it to show something else in the app."
  - When a custom title is set, a reset button labeled "Use the site's title"
    restores the site's name.
- **Defaults** section, with toggles. The whole row toggles, and the switch is
  read as part of it.
  - **Include Location:** "Fill in your current location on new posts."
  - **Publish by Default:** "Start new posts as published. When off, they start
    as drafts."

## 6. Media handling

### Picking and checking

Each picked item is checked before it is added. A rejected item shows its
message and isn't added.

- **Too many items:** if more items are picked than there is room for, only the
  first ones that fit are added, with "A post may have up to {max} photos and
  videos. Only the first {n} were added." If `max_media_per_post` is 0: "This
  site doesn't accept photos or videos from the app yet."
- **Type:** only photos and videos. Otherwise: "Only photos and videos can be
  added." An unknown type gives "This file's type could not be determined."
- **Filename:** the original name without its extension, or `upload`, plus an
  extension taken from the file's **actual MIME type**. The site checks
  extensions.
  - `jpeg` becomes `jpg` if the site allows `jpg` but not `jpeg`.
  - If the extension isn't allowed: "This site does not accept .{ext} files.
    Allowed: {list}."
- **Size:** an unknown size gives "This file's size could not be determined."
  - Photos over `media.image.max_size` give "This photo is {size}. The site accepts
    up to {max}."
  - Videos are checked later, because they can be compressed.
- **HEIC/HEIF photos** (by MIME type `image/heic`, `image/heif`, or their
  `-sequence` forms, or else by a `.heic` or `.heif` filename) are converted to
  JPEG.
  - Their own type and size aren't checked; the JPEG's are.
  - If the site accepts neither `jpg` nor `jpeg`: "This photo needs to be
    converted to JPEG, but this site does not accept JPEG files."

Sizes are shown as "{n} bytes", "{n} KB", "{n.n} MB", or "{n.n} GB" (1024-based).

### Uploading

Each item uploads as soon as it is added, with these steps in order:

1. **HEIC conversion**, if needed. The photo is decoded as displayed (rotated,
   converted to sRGB) and saved as JPEG at quality 90, then 75, then 60, until
   it fits `media.image.max_size`.
   - If none fits: "This photo is {size} as a JPEG. The site accepts up to
     {max}."
   - If decoding fails: "This HEIC photo could not be converted to JPEG on this
     device."
   - The JPEG has **no metadata**, including GPS.
   - One conversion runs at a time.
2. **Video compression**, if needed; see the next section.
3. **Video thumbnail**, if the site stores thumbnails
   (`media.video.thumbnail`):
   - A frame at 1 second, or the midpoint of videos shorter than 2 seconds.
   - Scaled to fit 1280 pixels on its longest side, and rotated as the video is
     shown.
   - Encoded in the first format the site accepts, in the order jpg, jpeg,
     webp, png.
   - JPEG and WebP use quality 85, then 70, then 50, until it fits
     `media.image.max_size`. PNG is tried once.
   - If no frame or format works, the video is uploaded without a thumbnail.
   - The thumbnail is kept for retries.
4. **Upload** with `POST /media`:
   - The file is streamed, never read into memory whole.
   - `alt` is sent if one has been entered.
   - Progress is reported as bytes sent ÷ total.

Temporary converted files are kept only to retry a failed upload, and deleted
once the upload succeeds, the item is removed, or the editor closes.

### Video compression

Compressed videos are H.264 video and AAC audio in MP4. They are only made if
the site accepts `mp4`; otherwise the video can't be compressed.

Video information is read first: duration, and width and height **as
displayed**, with the two swapped for a 90° or 270° rotation.

**Decision:**

| Situation | Decision |
|---|---|
| Larger than `media.video.max_size`, and can be compressed to fit | Compress |
| Larger than `media.video.max_size`, and can't be compressed | Reject: "This video is {size}. The site accepts up to {max}." |
| Larger than `media.video.max_size`, and 720p wouldn't fit | Reject: "…, and compressing it to 720p wouldn't be enough." |
| Fits, and 200 MB or smaller | Upload the original |
| Fits, over 200 MB, and compressing would save at least 20% | Ask the user |
| Otherwise | Upload the original |

**Asking the user:**

- The dialog can't be dismissed without choosing.
- Title: "Compress this video?"
- Message: "This video is {size}. Compressing it to {tier} should make it
  about {estimate}, so it uploads faster, with some loss of quality."
- Buttons: **Compress** and **Upload original**.
- Choosing **Upload original** is remembered for retries of that item.
- If several videos are waiting, one dialog is shown at a time.

**Plan.** The tiers are tried from the largest to the smallest, and the first
that fits is used:

| Tier | Short side | Target bitrate | Minimum bitrate |
|---|---|---|---|
| 1080p | 1080 | 8 Mbps | 3 Mbps |
| 720p | 720 | 5 Mbps | 1.5 Mbps |

- A tier is skipped if the video's short side is no larger than the next
  tier's. For example, a 720p video only tries 720p.
- Available video bitrate = `media.video.max_size × 0.93 ÷ overshoot ÷ 1.02 × 8 ÷
  duration − 192 kbps`.
  - 0.93 is a safety margin.
  - 1.02 allows for MP4 overhead.
  - 192 kbps is the audio.
  - `overshoot` starts at 1.
- A tier fits if the available bitrate is at least its minimum. The bitrate
  used is the lower of the target and the available.
- Estimated size = `(video bitrate + 192 kbps) × duration ÷ 8 × 1.02 ×
  overshoot`.
- Output size: the short side is scaled to the tier, keeping the aspect ratio,
  and both sides are rounded down to even numbers. Videos are never scaled up.
- HDR is tone-mapped to SDR.
- Videos already in H.264 are re-encoded at the requested bitrate, not copied.

**Checking the result.** Compression can be tried up to 3 times:

- If the file is still over `media.video.max_size`, multiply `overshoot` by `actual
  size ÷ estimate`, plan again, and retry.
- After 3 tries: "This video couldn't be compressed to fit the site's limit of
  {max}."

Other compression errors:

- Before starting, there must be free space of at least 1.1 × the estimate.
  Otherwise: "There isn't enough free storage to compress this video."
- If encoding fails: "This video couldn't be compressed on this device."

One compression runs at a time. While compressing, the tile shows "Compressing"
and the compression progress; then upload progress.

### Tiles

Each tile is about 120 points square, with rounded corners. It shows:

- **Preview:** the local file while uploading, then the site's thumbnail, or a
  photo's full `url` if it has no thumbnail.
- **Play icon** on videos.
- **While uploading:** a progress bar at the bottom, with "Compressing" above
  it while compressing.
- **On error:** a dark overlay with a warning icon and **Retry**, with the error
  message below the tile.
- **Alt button** (top leading): "Alt", with a check mark once alt text is set.
  It is labeled "Add alt text" or "Edit alt text".
- **Remove button** (top trailing): an × labeled "Remove".
  - Removing cancels the upload.
  - It deletes the upload on the site (`DELETE /media/{id}`) only if its
    `status` is `pending`, meaning not attached to a post yet. This is done
    quietly, and failures are ignored.
  - Attached media is removed from the post when the post is saved.
- **Tap:** opens the full-screen viewer. It is labeled "View photo" or "Play
  video".

### Reordering

- **Touch and hold, then drag** to reorder, when there are two or more items.
  - Haptic feedback when the drag starts.
  - The item follows the finger, raised with a shadow.
  - It swaps with another item once its middle passes over that item.
  - Dragging near either end scrolls the row.
  - Releasing doesn't also count as a tap.
- **Screen readers** get the actions **Move earlier** and **Move later**
  instead.

### Full-screen viewer

- A black background, with a close button (×, "Close") at the top leading
  corner.
- Photos are shown fitted to the screen. Videos play automatically, with
  standard controls but no next or previous buttons.
- The local file is shown while it's on the device, so the original is shown
  rather than a smaller copy. Otherwise the site's `url` is used.
- Video playback pauses when the app goes to the background.
- The viewer closes if its item is removed.

## 7. Tags and hashtags

The site turns hashtags in a description into tags itself. The app finds them
by **the same rules** so it can show them while the user types. These rules
must match the module's `moby_hashtags()` exactly:

- **Pattern:** `(?:^|(?<=[\s(\[]))#(\p{L}[\p{L}\p{M}\p{N}_-]*)`. That is a "#"
  at the start of the text or after whitespace, "(" or "[", then a letter, then
  letters, marks, numbers, "_" and "-".
- So the "#" in `a#b`, a URL fragment, or an HTML entity like `&#39;` is not a
  hashtag.
- A trailing "-" is trimmed from each hashtag.
- The hashtags are returned without the "#", each once, in order.

**Cleaning tags** works as on the site:

- Trim.
- Remove a leading "#", then trim again.
- Collapse whitespace to single spaces.
- Drop empty tags.
- Remove duplicates, keeping the first.

Tags are **case-insensitive**: "Mobile" and "mobile" are the same tag.

**In the editor:**

- **Loading a post:** tags that match a hashtag in the description are left out
  of the added-tags list. They are shown from the description instead, so
  removing the hashtag removes the tag.
- **The tag field:**
  - Typing a comma ends the tag before it and adds it.
  - Enter (Done), the add button (labeled "Add tag"), or choosing a suggestion
    adds the tag.
  - The field doesn't auto-capitalize.
- **Suggestions:**
  - Requested 250 ms after typing stops, with `GET /tags?search={text without
    a leading #}&limit=8`.
  - Each new keystroke cancels the pending request.
  - Tags the post already has, added or from hashtags, are left out.
  - Shown in a dropdown under the field while it is focused.
  - Failures are ignored.
- **Adding a tag:**
  - The text is cleaned, and the field and suggestions are cleared.
  - It does nothing if the post already has the tag.
  - Over `max_tag_length`: "Tags can be at most {max} characters."
  - When added tags plus hashtags already reach `max_tags_per_post`: "A post
    can have at most {max} tags."
- **The field** is marked when an error's `field` starts with `tags`.

## 8. Descriptions and HTML

The site may store descriptions as HTML (`description.html` in discovery).

- **Sending:**
  - When `html` is true, what the user typed is HTML-encoded: `&`, `<`, `>`
    and `"` become entities, as Backdrop's node form does.
  - When it is false, the text is sent as typed.
- **Displaying:** descriptions are always shown as **plain text**, never
  rendered as HTML. When `html` is true:
  - `<br>` and the ends of `p`, `div`, `li`, `h1`–`h6` and `blockquote`
    become line breaks.
  - Other tags are removed.
  - Entities are decoded: named (`amp`, `lt`, `gt`, `quot`, `apos`, `nbsp`),
    decimal and hex.
  - Non-breaking spaces become spaces, and the result is trimmed.
- **Editing:** when `html` is true and the stored description contains markup
  (`<` followed by a letter, `/` or `!`), it is edited **as raw HTML** and sent
  back as is, so formatting from the website isn't lost. The editor shows a
  note saying so. Without markup, entities are decoded for editing and the text
  is encoded again when saving.

## 9. Location

Shown only if `features.location`. The place name is editable only if
`features.location_name`. Coordinates are used only if
`features.location_coordinates`.

**Finding the device's location:**

- Location permission is requested when needed; approximate location is
  enough.
- Speed matters more than accuracy, since roughly 100 m is plenty.
  - Use a balanced-power, network-based fix.
  - Give up after **15 seconds**, then fall back to the most recent known
    location.
  - Android doesn't use Google Play services, so the app works on devices
    without them.
- Coordinates are rounded to 6 decimal places.
- The place name comes from reverse geocoding: the city (or else the county)
  and the state or region, such as "Boulder, Colorado". Duplicate parts are
  dropped. It is empty if the lookup fails.
- **Applying the result:**
  - The name replaces the place name only if one was found.
  - Coordinates are set only if the site maps them.
  - If the site maps only names and none was found: "A place name could not be
    found for your location."

**Automatic location ("Include Location" setting):**

- Applies to new posts only, when the setting is on.
- Runs once, when the editor finishes loading. It asks for permission if
  needed.
- It is silent: failures and a refused permission show nothing.
- It doesn't overwrite a name or coordinates the user entered while waiting.

**Use current location button:**

- Failures show "Your location could not be found."
- A refused permission shows "Location permission was not granted."

**Clear** empties the name and coordinates.

**Validation on save**, when the site maps coordinates:

- Only one of latitude and longitude set: "Enter both a latitude and a
  longitude, or neither."
- Latitude not a number from −90 to 90: "The latitude must be a number between
  -90 and 90."
- Longitude not a number from −180 to 180: "The longitude must be a number
  between -180 and 180."

These errors mark `location.latitude` or `location.longitude`. They're only
possible if a site's response gives bad values, since coordinates aren't typed.

## 10. Dates

- Sent as ISO 8601 with the device's UTC offset, to the second, such as
  `2026-10-27T18:30:00-06:00`.
- Dates from the API are parsed with their offset, then shown in the device's
  time zone.
- Display uses the device's localized medium date and short time format.
- A date that can't be parsed is shown as the raw string.

## 11. Sharing from other apps

Photos and videos can be shared to Moby from any app. One or several items can
be shared, and only image and video types are accepted.

- **Android:** only `content://` URIs are accepted. `file://` URIs are ignored,
  so another app can't get Moby to upload Moby's own private files.
  Duplicates are removed, and the order shared is kept.
- **Routing:**
  - **Exactly one site logged in:** that site is opened and a new post opens
    with the media.
    - If that site's Feed is already open, the new post opens on top, and any
      post being edited is kept underneath.
  - **Otherwise:** Sites is shown with the "Choose a site…" banner.
    - Choosing a logged-in site opens the new post.
    - Choosing a logged-out site logs in first, then opens it.
    - **Cancel** discards the share.
- Shared media is added to the editor once it loads, with the usual checks.
- Shared media that hasn't reached an editor yet survives the app being
  recreated, such as on rotation or when the process is killed. Media already
  given to an editor isn't added again.
- Sharing while the app is open works the same way.

## 12. Errors and session expiry

Messages for failures:

| Failure | Message |
|---|---|
| API error | The error's `message` |
| Response not understood (bad JSON) | "The site's response was not understood. Is this a Moby site?" |
| Network failure | "Couldn't reach the site. Check the address and your connection." |
| Anything else | The error's own message, or "Something went wrong." |
| A file that can't be opened (permission) | "This file could not be opened." |

**Session expiry.** A 401 from the Feed, the Post page, or a delete means the
token was revoked or expired. The app forgets the token locally, keeping the
site, and returns to Sites, where the site shows "Logged out". Tapping it logs
in again. The editor doesn't log out on a 401: it shows the error, so the user
doesn't lose their post.

## 13. Security and privacy

- **Token storage:**
  - **Android:** AES-GCM with a key in the Android Keystore. The key can't be
    exported, and app backups are disabled.
  - **iOS:** the Keychain, with accessibility "after first unlock, this device
    only", so tokens aren't restored to another device.
- **Token use:** the token is sent only to the site's own scheme, host and
  port: in API requests, and in the Post page's main request and same-site
  redirects. It is never put in a URL.
- **HTTPS only** in release builds.
  - Debug builds also allow plain HTTP to `*.ddev.site`, `10.0.2.2`, and
    `localhost` for local testing.
  - On iOS, App Transport Security exceptions for these domains go in a
    debug-only configuration.
- **The web view:** file and content access off, and links to other pages open
  in the system browser.
- **Descriptions** are never rendered as HTML in the app's own screens.
- **Removing a site** deletes its token and settings. Logging out revokes the
  token on the site.
- **Converted HEIC photos** carry no metadata (GPS included). Original files
  are uploaded as they are, so their metadata is kept.
- **Permissions requested:** internet, approximate and precise location (only
  when location is used), and photo access through the system picker. The
  picker needs no permission on Android; on iOS, use PHPicker, which needs none
  either.

## 14. Accessibility

- Every icon button has a label: Back, Sites, Settings, Log out, Options for
  {site}, Edit post, Delete post, Remove, Close, Add tag, and "Use the site's
  title".
- Decorative icons (favicons beside names, section icons) are hidden from
  screen readers.
- A site's session dot and label are read as one item.
- Toggle rows are one control, read as a switch.
- Media tiles have an action label ("View photo" or "Play video"), and their
  images use the alt text.
- Reordering media offers **Move earlier** and **Move later** actions.
- Tag chips are labeled "Remove tag {tag}" or "Tag {tag}, from a hashtag in the
  description".
- Fields show errors on the field itself, not only in a temporary message.
- Every control has a touch target of at least 44 × 44 points (48 dp on
  Android), even when its icon is smaller.
- Back buttons show the name of the screen they return to: "Sites", "Site"
  (the feed), "Post", and so on. iOS uses its native back button,
  which does this itself. Android shows its back arrow followed by the name,
  such as "← Sites".
- Titles in the top bar are bold and centered (17 points/sp).
- Edit and Delete on the post page are labeled ("Edit", "Delete" in the error
  color), and spaced well apart so they aren't confused.
- Support dynamic type / font scaling, dark mode, and right-to-left layouts.

## 15. Appearance

Each app uses its platform's standard controls (Material 3 on Android, UIKit
and SwiftUI on iOS), but both use the **same colors**, and match each other in
anything that is easy to align: button shapes and placement, labels, cards and
spacing. Native differences are fine where matching would mean replacing a
platform control, such as date and time pickers, dialogs, switches' shapes,
the platforms' own icon sets, and title bar styles.

- **App name:** Moby.
- **Icon:** the whale with an upload arrow (`branding/moby-whale.svg`), on blue
  `#1565C0`. Android also has a monochrome themed icon. iOS should use the same
  artwork, with dark and tinted variants made from it.
- **Colors:**
  - Both apps use the fixed colors below, not Android's wallpaper colors.
  - **Backgrounds:** black (`#000000`) with cards and grouped rows in
    `#1C1C1E` in dark mode; `#F2F2F7` with white cards in light mode (iOS's
    system backgrounds).
  - **Actions, links and icons** (back, edit, settings, log out, add, and so
    on) are very light gray (`#E3E3E3`) in dark mode and dark gray
    (`#1F1F1F`) in light mode, as Android's toolbar icons and text buttons
    are.
  - **Delete** is red (the platform's error/destructive red).
  - **Filled buttons and switches** use the brand blue: `#1E5AA8` with white
    text in light mode, `#A8C8FF` with `#0A2A55` text in dark mode. Switches
    have a white thumb.
  - **Floating buttons** (New post, Add site) are fully rounded pills in the
    brand blue, in the lower right. **Post** and **Save** are fully rounded,
    filled brand-blue buttons in the upper right.
  - **Continue** (Add site) is a fully rounded brand-blue button as wide as
    its label. **Log in** is the same, but full-width.
  - **Section headings** (Tags, Location, Site, Defaults) are bold and in
    sentence case on both platforms.
  - **Feed cards** have 16 points of padding inside and 12 between them.
  - **Confirmation dialogs** use text buttons; destructive ones (Delete,
    Remove) are red.
  - **Feed posts** are shown as rounded cards.
- **Contrast:** text and icons must have at least 4.5:1 contrast with their
  background in both light and dark mode (WCAG AA).
- **Light and dark mode** follow the system.
- **"Logged in" green:** `#2E9E4F`.

## 16. Text

All user-visible text is quoted in the sections above. Both apps must use the
same wording, so documentation and support apply to both. When text changes,
change it in this document and in both apps.

The apps currently ship in English only. When translation is added, both apps
should use the same source strings.

## 17. Platform mapping

How each feature is built on Android, and the suggested iOS equivalent.

| Feature | Android | iOS |
|---|---|---|
| Minimum OS | Android 8.0 (API 26) | iOS 17 (suggested) |
| UI | Jetpack Compose, Material 3 | SwiftUI |
| Navigation | Navigation Compose | `NavigationStack` |
| HTTP | OkHttp | `URLSession` |
| JSON | kotlinx.serialization, ignoring unknown keys | `Codable` with optional properties and defaults |
| Upload progress | Streaming request body | `URLSession` upload task from a file, with task progress |
| Images and favicons | Coil, with SVG and video frame decoders | `AsyncImage` or Nuke, with SVG support for favicons |
| Token storage | Android Keystore AES-GCM | Keychain |
| Settings | SharedPreferences per site | `UserDefaults` per site (key prefixed with site ID) |
| Photo picker | Photo Picker (`PickVisualMedia`) | `PhotosPicker` / PHPicker |
| HEIC to JPEG | `ImageDecoder`, then `Bitmap.compress` | ImageIO (`CGImageDestination`), without metadata |
| Video info | `MediaMetadataRetriever` | `AVAsset` (duration, `naturalSize` with `preferredTransform`) |
| Video compression | Media3 Transformer (H.264, AAC, HDR tone-mapping) | `AVAssetWriter` with an explicit H.264 bitrate and size (`AVAssetExportSession` presets can't target a bitrate) |
| Video thumbnail | `MediaMetadataRetriever` frame | `AVAssetImageGenerator` |
| Full-screen viewer | Dialog with ExoPlayer | Full-screen cover with `AVPlayerViewController` |
| Location | `LocationManager` (fused/network/GPS) | `CLLocationManager.requestLocation()`, with reduced accuracy allowed |
| Place name | `Geocoder` (locality, admin area) | `CLGeocoder` (locality, administrative area) |
| Post page | `WebView` | `WKWebView`, with a navigation delegate for links and redirects |
| Sharing in | `ACTION_SEND` / `SEND_MULTIPLE` intent filter | Share extension, passing media to the app through an App Group container and opening it with a URL scheme; or the share extension posts directly |
| Haptics | `HapticFeedback.LongPress` | `UIImpactFeedbackGenerator` |
| Reorder by drag | Custom long-press drag in a lazy row | `.draggable` / `.dropDestination`, or a custom gesture |
| Screen reader actions | Custom accessibility actions | `.accessibilityAction(named:)` |
| Temporary messages | Snackbar | A temporary banner (no system equivalent) |

Notes for iOS:

- **Share extension.** It runs in its own process with tight memory limits.
  Copy the shared files to an App Group container rather than holding them in
  memory. Keychain access must use a shared access group if the extension reads
  tokens.
- **Photos from the picker.** Load them as files, not data, so large videos are
  never held in memory. Name each file after its actual type, as on Android.
  For HEIC, either request a JPEG from the picker or convert with ImageIO, as
  long as the result has no metadata and meets the same size rules.
- **Background uploads.** These aren't required for parity, since Android
  uploads only while the editor is open. If the iOS app adds them, add them to
  Android too.

## 18. Parity checklist

Use this list to check both apps before a release. Each item must work the same
on both.

**Sites and login**

- [ ] An empty list shows "No sites connected yet." and **Add site**.
- [ ] Each site card shows its favicon (with the letter fallback), title,
  address, and login status.
- [ ] The site menu has Settings, Log out (only when logged in), and Remove
  site, each with its confirmation.
- [ ] Logging out keeps the site, and tapping it logs in again without asking
  for the address.
- [ ] Site names and favicons refresh when Sites is shown.
- [ ] The address is normalized (`https://` added, trailing slashes removed).
- [ ] Setup issues are listed: errors block login, and warnings are shown with
  the form.
- [ ] The login form is built from `auth.fields`, and "Forgot your password?"
  appears when the site gives a reset URL.
- [ ] Errors mark the field named by `error.field`.

**Feed and post page**

- [ ] Pull to refresh, 20 posts per page, **Load more**, and the empty message.
- [ ] Cards show the date, place, Draft badge, an 8-line description, and
  thumbnails (videos: play icon, duration, and no download without a
  thumbnail).
- [ ] A new post reloads the feed and scrolls to the top; an edit updates the
  post in place; a delete removes it without skipping a post on the next page.
- [ ] The post page shows the site's real page, including drafts, with
  `moby_view` added, and the token sent only to the same site.
- [ ] Links open in the browser.
- [ ] Edit and Delete appear according to the post's permissions, and deleting
  confirms first and treats a 404 as success.
- [ ] A 401 returns to Sites with the site logged out.

**Editor**

- [ ] A failed load shows **Try again**, and the form can't be saved.
- [ ] Media is checked for type, extension (named after its actual type), and
  size, with HEIC converted to JPEG.
- [ ] Uploads start immediately, with progress, retry, and remove (which
  deletes pending uploads on the site).
- [ ] Videos are compressed or rejected by the size rules, with the offer
  dialog for videos over 200 MB, and up to 3 attempts.
- [ ] Video thumbnails are made in the site's accepted format and size.
- [ ] Alt text, the full-screen viewer, and drag to reorder (with screen reader
  move actions) all work.
- [ ] Tags: suggestions after 250 ms, commas and Enter add tags, hashtags show
  as disabled chips, and the length and count limits apply.
- [ ] Hashtags are found by the same rules as the site (run the shared test
  cases below).
- [ ] The date and time pickers work, and the date is sent with the device's
  offset.
- [ ] Location: the automatic fill is silent and doesn't overwrite typing, the
  button reports errors, Clear works, and the name and coordinates follow the
  site's features.
- [ ] The status switch is right for publishers and non-publishers, and a
  locked draft can't be changed.
- [ ] Saving checks in the order listed, sends `client_id` on create, clears
  the location on edit, and sends HTML-encoded descriptions when the site's
  format is HTML.
- [ ] Descriptions with markup are edited as HTML, with the note shown.

**Settings and sharing**

- [ ] The site title can be changed, reset, and cleared.
- [ ] Include Location and Publish by Default apply per site, and are deleted
  with the site.
- [ ] Sharing one or several photos and videos from another app opens a new
  post on the only logged-in site, or asks which site.

### Shared test cases

Both apps should have unit tests for the logic that must match exactly. The
Android tests are in `app/src/test`. iOS should port them with the same inputs
and expected results:

| Area | Android test | Covers |
|---|---|---|
| Hashtags | `HashtagsTest` | The pattern, what isn't a hashtag (`a#b`, `&#39;`, URL fragments), the trailing "-", cleaning, and case-insensitive duplicates |
| HTML text | `HtmlTextTest` | Encoding, entity decoding, plain text for display, and raw HTML detection |
| Dates | `IsoDatesTest` | Formatting with the offset, and parsing with time zones |
| Login address | `LoginUrlTest` | Normalizing site addresses |
| Request bodies | `PostRequestsTest` | Create and update bodies: empty description, location null on update, status, tags |
| API client | `MobyApiTest` | Requests, headers, errors, tag suggestions, tags in posts |
| Media checks | `DeviceMediaReaderTest` | HEIC detection, the JPEG extension, file sizes |
| Video compression | `VideoCompressionTest` | Decisions, tier choice, bitrates, output sizes, overshoot |
| Shared media | `SharedMediaTest` | Allowed URI schemes, and encoding and decoding |
