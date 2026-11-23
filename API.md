# Moby REST API

Status: **Draft.** Authentication (`/auth/token`), media (`/media`), creating
posts (`POST /posts`), listing posts (`GET /posts`), viewing a post
(`GET /posts/{id}`), updating a post (`PATCH /posts/{id}`), and deleting a post
(`DELETE /posts/{id}`) are implemented.

The API lets a mobile app publish short-form posts (a description, a date, a
location, and one or more photos or videos) to a Backdrop site, the way a post
is made in Twitter, Instagram, or Bluesky.

The API is the same on every site. How a post is stored (which content type,
which fields, which file types) is set per site at
`admin/config/services/moby`. Apps never see field names.

## Conventions

- **Base URL:** `https://example.com/api/moby`.
- **Version:** every request sends the API version the app was built for as
  the `api` query parameter, such as `GET /api/moby/posts?api=1`. See
  [Versioning](#versioning).
- **Format:** Request and response bodies are JSON (`Content-Type:
  application/json`), except media uploads, which use `multipart/form-data`.
- **Dates:** ISO 8601 with a required UTC offset: `Z`, or hours with
  optional minutes (`-06:00`, `-0600`, or `-06`), e.g.
  `2026-10-27T18:30:00-06:00` or `2026-10-28T00:30:00Z`. Seconds and
  fractional seconds are optional. Dates without an offset are rejected, so
  the time is never ambiguous. Apps should send the device's local time and
  offset. The server converts the date to a Unix timestamp and does not keep
  the offset. Responses give dates in the site's time zone, so the offset in a
  response may differ from the one sent, but it is the same moment.
- **Coordinates:** WGS 84 decimal degrees, as reported by iOS and Android
  location services.
- **IDs:** Integers. Post IDs are node IDs (nid). Media IDs are file IDs (fid).
- **Caching:** All responses are sent with `Cache-Control: no-store` and are
  never stored in Backdrop's page cache.

## Versioning

The API has a single version number, which goes up by one with **every**
change to the API, additive or breaking: a new endpoint, a new property,
changed behavior, or a newly allowed method. The Backdrop, Drupal, and
WordPress versions of Moby share the numbering, and a module claims version N
only once it implements every change up to N.

- **Requests:** apps send the version they were built for in the `api` query
  parameter on every request, such as `?api=1`. A request without one is
  treated as version 1.
- **Older app (`api` below the site's version):** the site answers as the
  requested version. Additive changes need nothing special, since apps ignore
  properties they don't know. For a breaking change, the site keeps the old
  behavior for requests below the version that changed it.
- **Newer app (`api` above the site's version):** the site answers as its own
  newest version. The app learns that version from `api_version` in `GET /meta`
  or the `Moby-API-Version` header, does without the changes the site lacks,
  and tells the user the site's module could be updated.
- **App too old (`api` below `api_min_version`):** every endpoint answers
  `426 api_version_unsupported`, with the oldest version served in
  `error.api_min_version`. Apps tell the user to update the app.
- **Every response** has a `Moby-API-Version` header with the version it was
  answered as.

### Changelog

Every change adds a row here and one to `MOBY_API_VERSION` in `moby_backdrop/moby.module`.

| Version | Change | Breaking |
|---------|--------|----------|
| 1 | The API as first released: the URL index, discovery (`GET /meta`), authentication, media, posts (including `POST /posts/{id}` as an alternative to `PATCH`), tags, and `display.order`. | — |

## Authentication

Requests are authenticated with a bearer token:

```
Authorization: Bearer <token>
```

Each device gets its own token, which works like a Bluesky "app password".
Tokens do not expire, are stored only as a SHA-256 hash, and can be revoked
individually. A token stops working if its account is blocked, and is deleted
if the account is deleted.

- An invalid or revoked token is treated as anonymous. Endpoints that require
  authentication then return `401 unauthorized`.
- Session cookies are only honored on `GET` and `HEAD` requests, which is
  useful for testing in a browser. On other methods they are ignored, which
  rules out CSRF. If a request has both a token and a session cookie, the
  token is used.
- The API never starts or saves a session, so it never sets a cookie.
- Tokens apply only to `/api/moby/*` paths, and to `GET` and `HEAD` requests for
  a post's own page (see [Viewing a post's page](#viewing-a-posts-page)). They
  do not log in to the rest of the site.

The user also needs the **Post via Moby** permission, plus
Backdrop's own create/edit/delete access for the configured content type.

### Authentication mechanisms

How credentials are checked is swappable. Modules declare mechanisms with
`hook_moby_auth_info()` (see `moby.api.php`), each one a class extending
`MobyAuthMechanism`. The site picks one at `admin/config/services/moby`. Token
issuing and validation are the same whichever mechanism is used, so changing
the mechanism does not log out existing devices.

The app learns which credentials to ask for from the `auth` property of
`GET /meta`, and builds its login form from `auth.fields`:

```json
"auth": {
  "mechanism": "password",
  "label": "Username and password",
  "fields": [
    { "name": "username", "label": "Username", "type": "text", "required": true },
    { "name": "password", "label": "Password", "type": "password", "required": true }
  ],
  "password_reset_url": "https://example.com/user/password"
}
```

`type` is `text`, `email`, or `password`. Each field's `name` is the property
to send to `POST /auth/token`. Other properties, such as
`password_reset_url`, depend on the mechanism.

The default `password` mechanism runs the user login form's own validation
handlers. That gives it the same rules as logging in on the site:

- The site's login method setting (username, email address, or either). The
  `username` field's label and type change to match.
- Blocked and unactivated accounts are refused.
- Flood control: failed attempts count toward the same per-IP and
  per-account limits as the login form.
- The same log messages are recorded.

### `POST /auth/token`

Exchanges credentials for a token. The body holds the active mechanism's
`auth.fields`, plus an optional `device_name`. The device name is shown to
the user when they manage their devices, and defaults to "Mobile device".

```json
{
  "username": "nate",
  "password": "********",
  "device_name": "Nate's iPhone"
}
```

`201 Created`

```json
{
  "token": "moby_xP3Lhna5sDUpbekLQ-grB0-HDbwMUWw89YsZbtL8IAw",
  "token_type": "Bearer",
  "user": {
    "id": 1,
    "name": "nate",
    "url": "https://example.com/user/1",
    "permissions": { "create": true, "publish": true }
  }
}
```

The token is returned only once, so the app should keep it in the platform's
secure storage (the iOS Keychain or Android Keystore). Tokens start with
`moby_`, which makes them easy for secret scanners to spot.

| HTTP | `code`                | When                                                    |
|------|-----------------------|---------------------------------------------------------|
| 400  | `invalid_json`        | The body is not a JSON object.                          |
| 401  | `invalid_credentials` | Unknown account, wrong password, or blocked account. `field` says which input to highlight. `message` is the login form's own message. |
| 403  | `forbidden`           | The credentials are valid, but the account lacks the **Post via Moby** permission, or permission to create the configured content type. No token is issued. `message` names the missing permission. |
| 422  | `validation_failed`   | A required credential is missing.                       |
| 429  | `too_many_requests`   | Flood control was triggered.                            |
| 503  | `auth_unavailable`    | The configured mechanism no longer exists.              |

### `DELETE /auth/token`

Revokes the token used for this request. Use it to log out. Returns
`204 No Content`, or `401 unauthorized` if the request was not made with a
valid token.

## Discovery

### `GET /`

Public. Lists the API's URLs, like the root of GitHub's API. Each is an
[RFC 6570](https://www.rfc-editor.org/rfc/rfc6570) URI template: `{/id}` is
an optional path segment, and `{?page,limit}` optional query parameters.
Clients add the `api` query parameter themselves. The apps don't read this,
since the paths are fixed for each API version. It is for people, debugging,
and other clients.

```json
{
  "meta_url": "https://quicksketch.org/api/moby/meta",
  "auth_token_url": "https://quicksketch.org/api/moby/auth/token",
  "current_user_url": "https://quicksketch.org/api/moby/me",
  "media_url": "https://quicksketch.org/api/moby/media{/id}",
  "posts_url": "https://quicksketch.org/api/moby/posts{/id}{?page,limit}",
  "tags_url": "https://quicksketch.org/api/moby/tags{?search,limit}"
}
```

### `GET /meta`

Public. Describes the site and its capabilities. The app calls this when the
user adds a site. It checks that the site runs Moby and adjusts the UI to the
site's limits.

```json
{
  "api_version": 1,
  "api_min_version": 1,
  "module": { "name": "moby", "platform": "backdrop", "version": "1.x-1.0.0" },
  "site": { "name": "quicksketch.org", "url": "https://quicksketch.org/", "favicon": "https://quicksketch.org/themes/basis/favicon.ico?v=1795066959" },
  "setup_issues": [],
  "auth": { "mechanism": "password", "label": "Username and password", "fields": [ "..." ] },
  "description": { "format": "filtered_html", "html": true },
  "display": { "order": ["date", "media", "description", "location"] },
  "limits": {
    "max_upload_size": 104857600,
    "max_media_per_post": 10,
    "max_description_length": null,
    "max_alt_length": 2000,
    "max_tags_per_post": 50,
    "max_tag_length": 255
  },
  "media": {
    "image": { "extensions": ["jpg", "jpeg", "png", "gif", "webp", "heic"], "max_size": 20971520 },
    "video": {
      "extensions": ["mp4", "mov", "m4v"],
      "max_size": 52428800,
      "thumbnail": { "extensions": ["jpg", "jpeg", "png", "webp"] }
    }
  },
  "features": { "location": true, "location_name": true, "location_coordinates": true, "drafts": true, "tags": true }
}
```

- `description` says how to encode post descriptions (see
  [Post object](#post-object)). `format` is the machine name of the text
  format they are saved in, or `null` if the field has no text formats.
  `html` is `true` when descriptions are HTML. It is `false` without text
  formats, or for a format that escapes all HTML.
- `display.order` is the order posts show their parts in on the site, so
  apps can show posts the same way, such as in a list of posts. It always has
  `date`, `media`, `description`, and `location`, ordered by the fields'
  weights in the content type's default display ("Manage display"). The date
  comes first when it is the node's authored on date, which themes show above
  the content. The location is placed by the lighter of its name and
  coordinates fields. Posts on the site use the same order.
- `api_version` is the newest API version the site implements, and
  `api_min_version` the oldest it still answers. See [Versioning](#versioning).
- `module` describes the module, for apps to show and for support. Its
  `version` is `null` when unknown. Apps never compare it; they use
  `api_version`.
- `auth` describes how to log in. See
  [Authentication mechanisms](#authentication-mechanisms).
- Until an administrator selects a content type, `setup_issues` includes
  `not_configured` and post endpoints return `503 not_configured`.
- `site.favicon` is the absolute URL of the favicon the site's pages use: the
  theme's or an uploaded one (any image type, including `.ico` and `.svg`),
  or Backdrop's default. For a file on the site, a `v` query parameter with
  its modification time is added, so a changed favicon has a new URL and is
  never shown from an older cached copy.
- `setup_issues` lists problems with the site's setup, so the app can show
  them when the user adds the site and the user can pass them on to an
  administrator. Each has a `code`, a `severity`, and a `message` for
  people. An `error` means posting from the app is not possible. A `warning`
  means posting works, but some of what the app sends is not saved. Field
  and permission names are not included, as discovery is public.

  | `code`                    | Severity | When                                     |
  |---------------------------|----------|------------------------------------------|
  | `auth_unavailable`        | error    | The configured authentication mechanism is missing. |
  | `not_configured`          | error    | No content type is selected, or it was deleted. The mapping is not checked. |
  | `description_not_mapped`  | warning  | No description field is selected.       |
  | `field_missing`           | warning  | A mapped description, date, or location field no longer exists on the content type, the coordinates field is not a Geofield, or the content type has no field for tags from the tag vocabulary. |
  | `geofield_unavailable`    | warning  | A coordinates field is selected, but the Geofield module is not enabled. Sites without Geofield and with no coordinates field selected have no issue; they just report `location_coordinates: false`. |
  | `media_not_mapped`        | warning  | No media field is selected, or it no longer exists. |
  | `media_extensions`        | warning  | The media field allows no photo extensions, no video extensions, or neither. |
  | `thumbnail_field_missing` | warning  | The video thumbnail field is not an image field on the video file type. |
  | `tags_unavailable`        | warning  | A tag vocabulary is selected, but it or the Taxonomy module is missing. |
  | `thumbnail_style_missing` | warning  | The thumbnail image style no longer exists, so thumbnails are returned at full size. |
- `max_upload_size` is in bytes. It is the smallest of PHP's
  `upload_max_filesize`, PHP's `post_max_size`, and the media field's maximum
  upload size. It is the most a single request may carry.
- `max_alt_length` is the longest alt text allowed, in characters.
- `max_tags_per_post` is the most tags a post may have, counting hashtags in
  its description. `max_tag_length` is the longest a tag may be, in
  characters.
- `media.*.extensions` is the media field's allowed extensions, split into
  photos and videos by MIME type. Allowed extensions that are neither (e.g.
  `pdf`) are left out. Both lists are empty until a media field is mapped.
- `media.image.max_size` is the most bytes a single image upload may have: a
  photo, or a video's thumbnail. It is `limits.max_upload_size`, or the video
  thumbnail field's maximum upload size if that is smaller. Apps should
  convert or reject larger photos, and shrink thumbnails to fit, before
  uploading.
- `media.video.max_size` is the most bytes a single video upload may have. It
  is the site's "Maximum video upload size" setting, and is never larger than
  `limits.max_upload_size` (it equals it when the setting is empty). Apps
  should transcode or compress videos to fit this limit before uploading.
- `media.video.thumbnail` is `null` when the site doesn't store video
  thumbnails. Otherwise apps should send a thumbnail with each video (see
  [`POST /media`](#post-media)), in one of `extensions` and at most
  `media.image.max_size` bytes.
- `features.location_coordinates` says whether posts on this site support
  `location.latitude` and `location.longitude`. It is `true` only when the
  optional Geofield module is enabled and a Geofield is mapped. When it is
  `false`, apps must hide coordinate inputs and not send coordinates.
- `features.location_name` says the same for `location.name`, and
  `features.location` is `true` if either is supported.
- `features.tags` says whether posts on this site support `tags`. It is `true`
  when a tag vocabulary is selected and the content type has a field for its
  tags. When it is `false`, apps should hide tag inputs. Hashtags in
  descriptions are then left as text.

### `GET /me`

Returns the authenticated user.

```json
{
  "id": 1,
  "name": "nate",
  "url": "https://example.com/user/1",
  "permissions": { "create": true, "publish": true }
}
```

`create` is whether the user may create posts: a content type is configured,
and the user has **Post via Moby** and permission to create
that content type. `publish: false` means the user can only create drafts (`status` is forced to
`"draft"`).

## Media

Media is uploaded one file per request **before** the post is created, the way
Bluesky's `uploadBlob` and Mastodon's `/media` work. On mobile networks this
gives per-file progress and per-file retries, and it keeps each request under
PHP's `post_max_size`.

An uploaded file is saved as a *temporary* Backdrop file (`status: pending`).
It becomes permanent when a post that references it is saved. Backdrop's cron
deletes pending files that are never attached, after about six hours. Apps
should therefore create the post soon after uploading its media.

Only files uploaded through this API are available at `/media/{id}`. Other
files on the site return `404`, even when the user could view them elsewhere.

### Media object

```json
{
  "id": 1001,
  "type": "image",
  "status": "pending",
  "filename": "IMG_1001.jpg",
  "mime_type": "image/jpeg",
  "size": 2457600,
  "width": 4032,
  "height": 3024,
  "duration": null,
  "url": "https://example.com/files/moby/IMG_1001.jpg",
  "thumbnail": {
    "url": "https://example.com/files/styles/large/public/moby/IMG_1001.jpg",
    "width": 480,
    "height": 360
  },
  "alt": ""
}
```

| Property    | Description                                                       |
|-------------|-------------------------------------------------------------------|
| `type`      | `image` or `video`, based on the MIME type. The site maps each to a Backdrop file type (usually `image` and `video`). |
| `status`    | `pending` (uploaded but not attached) or `attached`.              |
| `filename`  | The original filename. The file is saved under this name, renamed if it is already taken or for security (e.g. `shell.php.png` becomes `shell.php_.png`), so `url` may differ. |
| `mime_type` | Determined from the extension, as Backdrop does for all files.    |
| `width`, `height` | Display dimensions in pixels, or `null` if unknown. For images, they are read by core when the file is saved (JPEG, PNG, GIF, and WebP; not HEIC). For videos, they are read from the MP4/QuickTime header and take rotation into account, so a portrait phone video is taller than it is wide. |
| `duration`  | Video length in seconds (to the millisecond), from the MP4/QuickTime header. `null` for images, for other video formats such as WebM, and for fragmented MP4s whose header does not record a duration. |
| `url`       | The original, full-size file. |
| `thumbnail` | A smaller image for lists and previews, at the site's "Thumbnail image style": `{ "url": "...", "width": 480, "height": 360 }`. For a video, this is made from the image uploaded as its thumbnail. `width` and `height` are `null` if unknown, or if the image style's effects don't report them. `null` for photos the server can't process, such as HEIC, where apps should use `url`. Also `null` for videos uploaded without a thumbnail or on a site that doesn't store thumbnails, where apps should show a placeholder rather than download the video to make one. |
| `alt`       | Alt text. Copied to the file field item's description when the post is saved. |

### `POST /media`

`multipart/form-data` with these parts:

| Part   | Required | Description                     |
|--------|----------|---------------------------------|
| `file` | yes      | One photo or video. The filename's extension must be in `media.image.extensions` or `media.video.extensions`. |
| `alt`  | no       | Alt text, up to `max_alt_length` characters. It can also be set when the post is created. |
| `thumbnail` | no    | For videos only, an image to make the video's thumbnail from, such as a frame chosen by the user. Ignored if `media.video.thumbnail` is `null` in discovery. |

```
curl -H "Authorization: Bearer $TOKEN" \
  -F file=@IMG_1001.jpg -F "alt=Orange sky over the Flatirons" \
  https://example.com/api/moby/media
```

Returns `201 Created`, a `Location` header, and a media object.

| HTTP | `code`                   | When                                              |
|------|--------------------------|---------------------------------------------------|
| 413  | `payload_too_large`      | The file exceeds `media.image.max_size` for photos or `media.video.max_size` for videos, or the thumbnail exceeds `media.image.max_size` (`field` is `thumbnail`). This includes requests rejected by PHP's own limits. |
| 415  | `unsupported_media_type` | The file is not a photo or video, or its extension is not allowed by the media field. |
| 422  | `validation_failed`      | No `file` part, more than one file, an incomplete upload, `alt` too long, a `thumbnail` with a photo, or rejected by the media field's or thumbnail field's other validation. `field` is `file`, `alt`, or `thumbnail`. When the thumbnail is rejected, the video is not saved either. |
| 503  | `not_configured`         | No media field is mapped.                         |

The upload is saved the same way as through the media field's widget on the
node form:

- It goes into the field's upload directory (its "File directory" setting,
  with tokens replaced), on the field's storage scheme (public or private).
- It is checked against the field's allowed extensions and maximum upload
  size.
- Core's filename protections apply: dangerous extensions are munged.

Its Backdrop file type comes from the `image`/`video` file type mapping. If no
mapping is set, Backdrop picks the file type from the MIME type.

For videos, the width, height, and duration are read from the MP4 or
QuickTime header. They are stored in core's `{file_metadata}` table under the
same `width` and `height` keys core uses for images, plus `duration`. This is
done only for videos uploaded through the API. A video that can't be parsed is
still accepted, with `null` metadata.

**Video thumbnails.** Servers, especially shared hosting, usually can't
extract video frames, so the app provides the thumbnail. It is saved in the
image field chosen in the "Video thumbnail field" setting, on the video's own
file entity, so each video has its own thumbnail:

- It is validated like an upload through that image field: its allowed
  extensions (limited to formats the server can process), maximum upload
  size, and minimum dimensions. Images larger than the field's maximum
  dimensions are scaled down.
- It goes in the image field's upload directory. Its alt text is the video's
  alt text from the upload, if any.
- It is deleted with the video. If an unused upload expires, cron deletes
  both.

**Thumbnail sizes.** The image is created by the image style the first time its
URL is requested, so that first request can be slower. The site's "Thumbnail
image style" setting chooses the style for all media. Changing the setting
changes the `thumbnail` of existing media too, so apps should not store these
URLs for long.

The site adds the `.heic` and `.heif` extensions (`image/heic`,
`image/heif`), which Backdrop core does not recognize, so iPhone photos can be
uploaded without converting them.

### `GET /media/{id}`

Returns a media object. Only the uploader can see a media item while it is
pending. Once attached, it is also visible to anyone with view access to the
file. Otherwise the response is `404`.

### `DELETE /media/{id}`

Deletes a pending media item, e.g. when the user removes a photo from the
composer. The file is removed from disk. Returns `204 No Content`.

Errors: `403 forbidden` if the user is not the uploader (for attached media
they can view), and `409 media_attached` if the item is attached to a post.
Attached media can only be removed by updating the post.

## Posts

### Post object

```json
{
  "id": 3,
  "url": "https://example.com/node/3",
  "api_url": "https://example.com/api/moby/posts/3",
  "status": "published",
  "description": "Sunset hike up the ridge.",
  "date": "2026-10-27T18:30:00-06:00",
  "location": {
    "name": "Boulder, Colorado",
    "latitude": 40.01499,
    "longitude": -105.27055
  },
  "media": [ { "id": 31, "type": "image", "status": "attached", "...": "..." } ],
  "tags": ["hiking", "Colorado"],
  "created": "2026-10-27T18:42:10-06:00",
  "changed": "2026-10-27T18:42:10-06:00",
  "permissions": { "edit": true, "delete": true }
}
```

| Property      | Description                                                     |
|---------------|-----------------------------------------------------------------|
| `status`      | `published` or `draft` (the node's published status).          |
| `description` | Text in the site's description format, already encoded by the app, as the node form saves it. When discovery's `description.html` is `true`, it is HTML, so the app must escape `<`, `>`, and `&` in what the user typed (e.g. `Tom &amp; Jerry &lt;3`) and may use markup the format allows. When it is `false`, it is plain text. It is stored and returned exactly as sent (trimmed), and the text format filters it when displayed on the site. Line breaks may be sent as newlines, which formats like Filtered HTML turn into paragraphs. |
| `date`        | When the post happened, e.g. when the photo was taken. Defaults to the time the post is created. Converted to a timestamp and saved as the node's "Authored on" date (`created`), unless the site maps it to a date field. |
| `location`    | `null`, or an object. Every property is optional, but `latitude` and `longitude` must be provided together. |
| `media`       | Photos and videos together, in display order.                  |
| `tags`        | The post's tags, as names, in order. This includes tags added from hashtags in the description. An empty list when the site doesn't support tags. See [Tags](#tags). |
| `created`, `changed` | Server timestamps. Read only.                           |
| `permissions` | Whether the current user may update (`edit`) or delete (`delete`) this post, from Backdrop's node access. Apps should hide options the user can't use. Read only. |

### `GET /posts`

Lists the authenticated user's posts of the configured content type, newest
`date` first. Each post is a post object, the same as from `GET /posts/{id}`.

| Query   | Default | Description                  |
|---------|---------|------------------------------|
| `limit` | 20      | Posts per page (1–50).       |
| `page`  | 0       | Zero-based page number (0–100000). |

```json
{ "posts": [ { "...": "..." } ], "page": 0, "limit": 20, "total": 42 }
```

- `total` is the number of posts across all pages. A page past the end has
  an empty `posts` list.
- Every node of the configured type that the user authored is listed,
  including ones created on the site rather than through the API.
- The user's drafts are listed only if they have "View own unpublished
  content" or "Bypass content access control", as with `GET /posts/{id}`.
  Node access modules' grants also apply.
- Posts with the same `date` are listed newest first (by ID), so paging is
  stable. When `date` is mapped to a date field, posts with no value in that
  field are listed last.

| HTTP | `code`              | When                                                    |
|------|---------------------|---------------------------------------------------------|
| 422  | `validation_failed` | `limit` or `page` is not a whole number in range. `field` names it. |
| 503  | `not_configured`    | No content type is configured.                          |

### `POST /posts`

```json
{
  "client_id": "6f1c2a9e-5b3d-4e8a-9c07-2d1f0e4b7a11",
  "description": "Sunset hike up the ridge.",
  "date": "2026-10-27T18:30:00-06:00",
  "location": { "name": "Boulder, Colorado", "latitude": 40.01499, "longitude": -105.27055 },
  "media": [
    { "id": 1001, "alt": "Orange sky over the Flatirons" },
    { "id": 1002 }
  ],
  "tags": ["hiking", "Colorado"],
  "status": "published"
}
```

- Either `description` (not just whitespace) or at least one `media` item is
  required. Everything else is optional.
- `client_id` is an optional UUID generated by the app. If the app resends a
  request after a timeout with the same `client_id`, the server returns the
  existing post with `200 OK` and does not create a duplicate. This holds
  even if the first request saved the post but its response never arrived.
  The rest of a repeated request is ignored. Client IDs are scoped to the
  user, and are case-insensitive.
- `media` is a list of up to `max_media_per_post` items, in display order.
  Each `id` must be pending media uploaded by the same user. Media that has
  expired, belongs to someone else, or is already attached to a post is
  rejected. `alt`, if given, replaces the alt text sent with the upload.
- `tags` is a list of tag names. See [Tags](#tags).
- `location.latitude` and `location.longitude` must be JSON numbers (not
  strings), within ±90 and ±180. Location properties that the site has not
  mapped are ignored.
- `status` (`published` or `draft`) is honored for users with Backdrop's
  "Administer content" permission, and defaults to the site's "Publish posts
  by default" setting. Other users always get the content type's default
  publishing status, but may ask for a draft. `GET /me` reports which applies
  as `permissions.publish`.
- The node title is generated from the description. HTML descriptions are
  converted to plain text (tags removed, entities decoded), whitespace is
  collapsed, and it is cut at a word boundary to the site's "Generated title
  length". With no description, it is "Post from" followed by the date.
- The post is checked with the fields' own validation (e.g. a text field's
  maximum length) before it is saved.

**Saving media.** Pending media becomes permanent only when the post is
saved. Core marks the files permanent and records their usage by the node. If
the request fails for any reason, its media stays pending. The app can retry,
or the files expire after about six hours.

Returns `201 Created`, a `Location` header, and the post object.

| HTTP | `code`              | When                                                    |
|------|---------------------|---------------------------------------------------------|
| 400  | `invalid_json`      | The body is not a JSON object.                          |
| 403  | `forbidden`         | The user may not create content of the configured type. |
| 422  | `validation_failed` | Any invalid property. `field` names it, e.g. `date`, `location.latitude`, `media[1]`, or `media[0].alt`. |
| 503  | `not_configured`    | No content type is configured.                          |

### `GET /posts/{id}`

Returns a post object, the same as the response to `POST /posts`.

- Any node of the configured content type is a post, including ones created
  on the site rather than through the API.
- Access follows Backdrop's node access, as when viewing the node on the
  site. Anyone who may view a published post can read it through the API, not
  only its author.
- Unpublished posts need "View own unpublished content" (for one's own posts)
  or "Bypass content access control". Without one of those, a user cannot read
  back a draft they created, so grant it to roles that post from the app.
- A property whose field the user may not view (e.g. through a field
  permissions module) is returned empty, as `null`, `""`, or `[]`. Fields
  hidden only in the content type's display settings are still returned, so a
  published post's location is readable by anyone who can view the post.

`404 not_found` if the node does not exist, is not of the configured content
type, or the user may not view it. The API does not say which, so the
existence of unpublished posts is not revealed.

### Viewing a post's page

To show a post as the site renders it, with its theme and layout, an app loads
the post's `url` in a web view and sends its token in the `Authorization`
header of that request.

- The page is rendered for the token's user, so a draft they may view is shown.
  Without a valid token it is rendered as for an anonymous visitor.
- This only applies to `GET` and `HEAD` requests for `node/{id}` of the
  configured content type. Links on the page, and the images and scripts it
  loads, are not authenticated. Files in a private file system will not load.
- The response is sent with `Cache-Control: no-store`, and is never stored in
  Backdrop's page cache.
- The admin bar, the page's tabs (such as "View" and "Edit"), and contextual
  links are left out, as apps provide their own options using the post's
  `permissions`.
- Backdrop's page cache answers before tokens are checked. Apps should add a
  unique query string (e.g. `?moby_view=1793518770`) so that they get the page
  rendered for the user, rather than a copy cached for anonymous visitors,
  which for a draft would be the "Access denied" page.
- Apps must only send the token to the site's own address. A redirect drops the
  header, so apps should send it again if the redirect stays on the site.

### `PATCH /posts/{id}`

Also accepted as `POST /posts/{id}`, which does exactly the same, for clients
that can't send `PATCH` (such as Android's `HttpURLConnection`).

Partial update. It takes the same properties as `POST /posts`, and only the
properties included are changed. `client_id` can't be changed and is ignored.
An empty object changes nothing.

- **Access:** the user needs Backdrop's edit access to the node, such as "Edit
  own content". This works for any post of the configured content type,
  including ones created on the site. As on the node form, a field the user
  may not edit (e.g. through a field permissions module) keeps its value.
- `description: ""` clears the description. A post must still have a
  description or media after the update.
- `location`, when present, **replaces** the whole location. Parts left out of
  the object are cleared. `location: null` clears the location. Parts the site
  doesn't support are not affected.
- `tags`, when present, **replaces** the post's tags, and the hashtags in the
  description are added to them. If only the description changes, the tags
  of hashtags removed from it are removed, and those of new hashtags added.
- `null` is not accepted for `description`, `date`, `media`, `status`, or `tags`. Leave
  the property out to keep its value.
- `media`, when present, **replaces** the post's whole media list, in that
  order. To keep an existing item, include its `id`. Items already on the post
  may be kept whoever uploaded them. New items must be pending media uploaded
  by the same user. An item's `alt`, if left out, keeps its current text.
- **Removed media:** media left out of the list is removed from the post. If
  nothing else uses the file, it goes back to `pending`, and cron deletes it,
  along with any video thumbnail, after about six hours. If the content type
  creates a new revision by default, older revisions still use it, so it is
  kept.
- `status`: users with "Administer content" may set either status. Other users
  may always change a post to `draft`. They may change a draft to `published`
  only if `permissions.publish` is `true`, and otherwise get `403 forbidden`
  with `field` set to `status`.
- **Title:** the node title is regenerated only if the description changes, or
  if the date changes on a post with no description. A title written on the
  site is otherwise kept.
- **Revisions:** a new revision is created if the content type does so by
  default.
- There is no conflict detection. The last update wins.

Returns `200 OK` with the updated post.

| HTTP | `code`              | When                                                    |
|------|---------------------|---------------------------------------------------------|
| 400  | `invalid_json`      | The body is not a JSON object.                          |
| 403  | `forbidden`         | The user may view the post but not edit it, or may not publish it. |
| 404  | `not_found`         | As for `GET /posts/{id}`.                               |
| 422  | `validation_failed` | Any invalid property, as for `POST /posts`.             |

### `DELETE /posts/{id}`

Deletes the post (node), with all its revisions. Returns `204 No Content`.

## Tags

A post's tags are terms in the site's tag vocabulary (by default "Tags"),
saved in the content type's field for them. They come from two places:

- The `tags` list sent with a post.
- **Hashtags** in the description: `#backdrop` adds the tag `backdrop`. A
  hashtag starts with `#` and a letter, and continues with letters, numbers,
  `_`, and `-`. It must come at the start of the description or after a space,
  a line break, or an opening bracket, so a link like `/page#section` or an
  HTML entity like `&#39;` isn't one. The description itself is saved as
  written, with its hashtags.

Tags are matched to existing terms regardless of letter case, so `#Backdrop`
is the tag `backdrop` if that term exists. Terms that don't exist are added to
the vocabulary. A leading `#` in a listed tag is removed, spaces are collapsed,
and repeated tags are saved once.

To show hashtags as tags as the user types, apps can use the same rules. On
an existing post, apps should leave the description's hashtags out of the
`tags` they send, so that removing a hashtag from the description also
removes its tag.

### `GET /tags`

Suggests tags as the user types. Needs the same access as creating a post.

| Parameter | Description                                                        |
|-----------|--------------------------------------------------------------------|
| `search`  | What the user has typed. A leading `#` is ignored. Tags starting with it come first, then tags containing it. Left out, the most used tags are returned. |
| `limit`   | The most tags to return, from 1 to 20. Defaults to 10.            |

```json
{ "tags": ["backdrop", "BADCamp"] }
```

Within each group, the most used tags come first. The list is empty when the
site doesn't support tags.

- **Access:** the user needs Backdrop's delete access to the node, such as
  "Delete own content". Any post of the configured content type can be deleted,
  including ones created on the site.
- **Media:** media used only by this post, including by its older revisions,
  goes back to `pending`. Its files
  stay on disk and at their URLs until cron deletes them, after about six
  hours. Cron then deletes a video's thumbnail once it has also been unused for
  that long. Files still used elsewhere are kept.
- **Repeating a request:** deleting a post that is already gone returns `404`,
  which an app retrying after a timeout should treat as success.

| HTTP | `code`      | When                                                    |
|------|-------------|---------------------------------------------------------|
| 403  | `forbidden` | The user may view the post but not delete it.           |
| 404  | `not_found` | As for `GET /posts/{id}`.                               |

## Errors

Every error uses the same shape:

```json
{
  "error": {
    "code": "validation_failed",
    "message": "A post must have a description or at least one media item.",
    "field": "description"
  }
}
```

`message` is meant for people and may be translated. Apps should branch on
`code`. `field` is included when the error comes from a specific request
property.

| HTTP | `code`                   | Meaning                                         |
|------|--------------------------|-------------------------------------------------|
| 400  | `invalid_json`           | The body is not a JSON object.                  |
| 401  | `unauthorized`           | Missing, invalid, or revoked token. Sent with `WWW-Authenticate: Bearer`. |
| 401  | `invalid_credentials`    | Wrong username or password (`/auth/token`).     |
| 403  | `forbidden`              | Authenticated, but without access.             |
| 404  | `not_found`              | No such resource or endpoint.                  |
| 405  | `method_not_allowed`     | Sent with an `Allow` header.                   |
| 409  | `media_attached`         | Attached media cannot be deleted directly.     |
| 413  | `payload_too_large`      | The upload exceeds `limits.max_upload_size`, or `media.*.max_size` for its type. |
| 415  | `unsupported_media_type` | The file type is not allowed by the media field. |
| 426  | `api_version_unsupported` | The app is older than the oldest API version the site serves (`error.api_min_version`). |
| 422  | `validation_failed`      | The request is well formed but invalid.        |
| 429  | `too_many_requests`      | Flood control was triggered.                   |
| 501  | `not_implemented`        | The endpoint is not implemented yet.           |
| 503  | `auth_unavailable`       | The configured authentication mechanism is missing. |
| 503  | `not_configured`         | No content type, or for uploads no media field, is configured. |
| 503  | `site_offline`           | The site is in maintenance mode.               |

## Storage mapping

Configured at `admin/config/services/moby` and stored in `moby.settings`:

| API property          | Setting                              | Supported targets                                  |
|-----------------------|--------------------------------------|----------------------------------------------------|
| (post)                | `node_type`                          | Any content type                                   |
| `status`              | `default_status`                     | The node's published flag                          |
| (video uploads)       | `max_video_size` setting             | A size like `50 MB`, at most PHP's upload limit; empty uses PHP's limit |
| (title)               | `title_length`                       | Generated from the description, or from the date if there is no description |
| `description`         | `mapping.description.field`/`format` | `text`, `text_long`, `text_with_summary` (e.g. `body`) |
| `date`                | `mapping.date.field`                 | `created` (Authored on), or a date field           |
| `location.name`       | `mapping.location.name_field`        | `text`                                             |
| `location.latitude`, `location.longitude` | `mapping.location.coordinates_field` | A `geofield` field (needs the optional Geofield module), saved as a point |
| `media[]`             | `mapping.media.field`                | A multi-value `file` field                         |
| `media[].type`        | `mapping.media.file_types.image`/`video` | Any Backdrop file type                         |
| `media[].thumbnail` (videos) | `mapping.media.thumbnail_field` | An `image` field on the video file type           |
| `media[].thumbnail` (size)   | `thumbnail_style`            | Any image style. Defaults to `large`             |
| `tags[]`              | `mapping.tags.vocabulary`            | Any vocabulary. Defaults to `tags`. Saved in the content type's multi-value term reference field for only that vocabulary, which is added when the settings are saved if there is none |

## Open questions

- **Large videos:** Shared hosting caps uploads (100 MB here). For now the app
  must compress videos to fit `media.video.max_size`. Should a later version add
  resumable, chunked uploads (e.g. the tus protocol)?
- **HEIC:** The Android app converts HEIC photos to JPEG before uploading, and
  other apps should too. The API still accepts HEIC from any client if the
  media field allows it, but many browsers can't display it, and the server
  can't read its dimensions or make image styles from it.
- **EXIF:** Should GPS metadata be stripped from originals for privacy when no
  location is attached to the post?
- **Displaying descriptions:** `description` is returned unfiltered, as
  stored, which is what's needed for editing. Like the node form's raw value,
  it may contain markup the text format would strip on display. That includes
  other users' posts, which can be read with `GET /posts/{id}`. The app must
  sanitize it before rendering it as HTML (e.g. in a web view). Alternatively,
  should responses add a `description_html` property filtered by the text
  format with `check_markup()`, for display?
- **Tags/hashtags:** Should hashtags map to a taxonomy field?
- **Token management UI:** Should users see and revoke device tokens from a
  tab on their account page? `{moby_token}` already records each token's device
  name, creation time, and last use for this.
- **Password changes:** Should changing a password revoke all of that user's
  tokens? Revoking is safer if the password was compromised, but it logs out
  every device.
- **Token expiry:** Should unused tokens expire, e.g. after 90 days without
  use?
