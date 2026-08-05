---
name: Read Skydance Media site content
description: Page through the corporate, legal and marketing pages skydance.com publishes through its WordPress REST API, and resolve the taxonomy behind them.
api: openapi/skydance-media-content-openapi.yml
base_url: https://skydance.com/wp-json/wp/v2
auth: none (anonymous read)
operations: [getPages, getPagesId, getPosts, getPostsId, getCategories, getCategoriesId, getTags, getTypes]
---

# Read Skydance Media site content

Skydance Media publishes no product API. This skill reads the **content** API that
skydance.com's WordPress install exposes. Use it to retrieve what Skydance has publicly
posted on its own site — corporate, legal and policy pages — not the film or television
catalog, which is not in this API (see **What you cannot get here** below).

## Before you start

- Base URL: `https://skydance.com/wp-json/wp/v2`
- **No authentication is required** for any step below. Every read operation is anonymous.
- **No API key exists.** Do not attempt to obtain one; there is no developer signup.
- `https://skydance.com/robots.txt` disallows nothing and sets no `Crawl-delay`. There is
  still no documented rate limit and no `X-RateLimit-*` or `Retry-After` header, and a
  Wordfence namespace is registered on the site, so pace yourself deliberately.

## Steps

### 1. List pages — `getPages`

`GET /wp/v2/pages?per_page=100&page=1&orderby=date&order=desc`

- `per_page` maximum is **100**; the default is 10. Asking for 500 returns HTTP 400
  `rest_invalid_param` with `data.details.per_page.code = rest_out_of_bounds`.
- Add `_fields=id,date,slug,link,title,parent,menu_order,template` to trim the payload when
  you do not need the full rendered `content`.
- 46 pages existed at harvest (2026-08-05).

Read the total from the response headers, not the body:

- `X-WP-Total` — total items matching the query
- `X-WP-TotalPages` — total pages at the current `per_page`

### 2. Fetch one page — `getPagesId`, or by slug

- `GET /wp/v2/pages/{id}` — by id
- `GET /wp/v2/pages?slug=patents` — by slug, when you know the URL but not the id

`title`, `content` and `excerpt` are objects with a `rendered` string containing HTML
entities (`&#8211;` for an en dash, `&#8220;` for a curly quote). Decode the entities and
strip the HTML before handing text to a model or a user.

A non-existent id returns HTTP 404 with `{"code":"rest_post_invalid_id"}`.

### 3. Read core posts — `getPosts`, `getPostsId`

`GET /wp/v2/posts?per_page=100`

Expect **one** result: the 2015 `hello-world` placeholder. This is not a failure and not a
sign you called it wrong — the site's editorial content lives in custom post types that are
not registered with the REST API.

### 4. Resolve taxonomy — `getCategories`, `getTags`, `getTypes`

- `GET /wp/v2/categories?per_page=100` — 7 terms: Available Now, Coming Soon, DVD/Blu-Ray,
  In Development, Now Available, Now Playing, Uncategorized. Their `count` values (8, 4, 18…)
  refer to objects in non-REST post types, so do not expect to fetch them.
- `GET /wp/v2/tags?per_page=100` — empty (0 terms).
- `GET /wp/v2/types` — the authoritative list of what this API exposes. All 11 entries are
  WordPress core. Call this first if you are unsure whether a content type is reachable.

## What you cannot get here

- **The title catalog.** Film, television, animation, interactive, sports, news and press
  releases are custom post types on skydance.com. None is REST-registered. They appear in
  `https://skydance.com/sitemap_index.xml` (film-sitemap.xml, tv-sitemap.xml,
  animation-sitemap.xml, interactive-sitemap.xml, sports-sitemap.xml, news-sitemap.xml,
  press_releases-sitemap.xml) and in rendered HTML at `https://skydance.com/film/` and
  siblings — walk the sitemap if you need them, and attribute the source.
- **Authors.** `getUsers` and `getUsersId` return HTTP 401 `rest_forbidden` anonymously.
  Do not call `getUsersMe`.

## Rules

- **Read-only.** Every write operation in this spec (`createPages`, `updatePagesId`,
  `deletePagesId`, and the equivalents on every other resource) requires a WordPress
  Application Password over HTTP Basic that you do not have and must not attempt to acquire.
  A write attempt returns HTTP 401 `rest_forbidden`.
- **No idempotency contract exists.** There is no idempotency key header or parameter on any
  route. Do not retry a non-GET request.
- **Errors are not RFC 9457.** The envelope is
  `{"code": "<slug>", "message": "<human string>", "data": {"status": <int>}}` with
  `Content-Type: application/json`. Branch on `code`, not on the message text. See
  `errors/skydance-media-problem-types.yml`.
- **Attribute the source.** Content returned here is Skydance Media's own published material.
  Cite the page `link` when you surface it.
