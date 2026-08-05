---
name: Walk the Skydance Media library
description: Enumerate the skydance.com media library through the WordPress REST API and pull real asset URLs, MIME types and alt text — working around the empty-first-page quirk.
api: openapi/skydance-media-content-openapi.yml
base_url: https://skydance.com/wp-json/wp/v2
auth: none (anonymous read)
operations: [getMedia, getMediaId, getPagesId]
---

# Walk the Skydance Media library

`skydance.com` exposes 5,431 media-library attachments through the WordPress REST API —
images, posters and documents used across the label site. This is the largest readable
collection on the host by two orders of magnitude.

## Before you start

- Base URL: `https://skydance.com/wp-json/wp/v2`
- No authentication. No API key exists.
- These are Skydance Media's copyrighted assets. Reading metadata and linking to
  `source_url` is fine; redistributing the assets is not. Cite the `link`.

## The quirk you must handle first

`GET /wp/v2/media?per_page=1` returns **HTTP 200 with an empty array** while the response
headers report:

```
X-WP-Total: 5431
X-WP-TotalPages: 5431
```

This is not an error and not a rate limit. The default ordering is `date`/`desc`, and the
newest attachments are attached to custom post types that are not REST-registered, so they
are counted by the query and then filtered out of the payload.

**Always pass explicit ascending-id ordering:**

`GET /wp/v2/media?per_page=100&orderby=id&order=asc`

Verified 2026-08-05: with `orderby=id&order=asc` the collection returns real attachments
starting at id 22.

## Steps

### 1. Page the library — `getMedia`

`GET /wp/v2/media?per_page=100&page=1&orderby=id&order=asc&_fields=id,date,slug,link,title,alt_text,media_type,mime_type,source_url`

- `per_page` maximum is **100**. Loop `page` up to `X-WP-TotalPages`; a page beyond the last
  returns HTTP 400 `rest_invalid_param`, which is your stop condition, not a failure.
- Useful filters declared on the operation: `media_type` (`image`, `video`, `audio`,
  `application`, `file`, `text`), `mime_type`, `after`/`before`, `parent`, `search`, `slug`.
- Prefer `_fields` — the full payload carries a `media_details.sizes` block with every
  generated thumbnail size and is large.

### 2. Use `source_url`, never a constructed path

`source_url` is the real asset URL, e.g.
`https://skydance.com/wp-content/uploads/2015/05/slide1.jpg`. Do not build upload paths
yourself; WordPress rewrites and dedupes filenames.

### 3. Fetch one attachment — `getMediaId`

`GET /wp/v2/media/{id}` returns the full record including `media_details` (width, height and
every registered size with its own `source_url`) and `post` — the id of the object the
attachment hangs off.

### 4. Follow `post` back to its parent — `getPagesId`

When `post` is set, `GET /wp/v2/pages/{post}` resolves the parent page. If it returns
HTTP 404 `rest_post_invalid_id`, the parent is a custom post type that is not exposed —
expect this for most of the newer catalog artwork.

## Rules

- **Read-only.** `createMedia`, `updateMediaId` and `deleteMediaId` all require a WordPress
  Application Password over HTTP Basic that you do not have.
- **Do not trust `X-WP-Total` as a promise of a non-empty page.** It counts the query, not
  the payload. See `conventions/skydance-media-conventions.yml`.
- **Errors are not RFC 9457.** Branch on `code`. See
  `errors/skydance-media-problem-types.yml`.
- **Pace yourself.** 5,431 items at `per_page=100` is 55 requests. No rate limit is
  documented and Wordfence is installed on the origin.
