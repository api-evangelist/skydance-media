---
name: Search skydance.com and resolve results
description: Run a cross-content-type search against the skydance.com WordPress REST API and resolve each lightweight result pointer into the full page or post object.
api: openapi/skydance-media-content-openapi.yml
base_url: https://skydance.com/wp-json/wp/v2
auth: none (anonymous read)
operations: [getSearch, getPagesId, getPostsId, getTypes, getTypesType, getStatuses]
---

# Search skydance.com and resolve results

The search endpoint is the fastest way to find a specific page on skydance.com without
walking the whole collection. It returns **pointers**, not documents — you always need a
second call to get the content.

## Before you start

- Base URL: `https://skydance.com/wp-json/wp/v2`
- No authentication. No API key exists.
- Search only covers what the REST API exposes: pages and the single core post. It does not
  reach the film, television, animation, interactive or sports catalog.

## Steps

### 1. Search — `getSearch`

`GET /wp/v2/search?search=<term>&per_page=20&page=1`

Each result is a lightweight pointer:

```json
{"id": 7208, "title": "Skydance Patents", "url": "https://skydance.com/patents/",
 "type": "post", "subtype": "page"}
```

- `type` is the **search type** (always `post` here — WordPress's generic object type), not
  the content type. Do not branch on it.
- `subtype` is the real content type: `page` or `post`. **Branch on `subtype`.**
- Each result also carries a `_links.self[0].href` that is the exact resolve URL. Prefer
  following that link over constructing one.

Optional parameters declared on the operation: `type`, `subtype`, `exclude`, `include`,
`per_page`, `page`, `search`.

### 2. Resolve each result

- `subtype == "page"` → `GET /wp/v2/pages/{id}` (`getPagesId`)
- `subtype == "post"` → `GET /wp/v2/posts/{id}` (`getPostsId`)

Add `_embed` if you want the featured media inlined. The author relation will be missing —
`/wp/v2/users` is 401-gated, so `_embed` silently omits it for anonymous callers.

### 3. Sanity-check the surface — `getTypes`, `getStatuses`

If a search for something you can see on the website returns nothing, confirm the content
type is even exposed:

- `GET /wp/v2/types` — lists every REST-registered post type. There is no `film`, `tv`,
  `animation`, `interactive`, `sports`, `news` or `press_releases` entry, so those pages will
  never appear in search results.
- `GET /wp/v2/statuses` — the statuses the API will return. Anonymous callers only ever see
  `publish`.

When search misses, fall back to `https://skydance.com/sitemap_index.xml`, which does list
every custom post type, and fetch the HTML page directly.

## Rules

- **Read-only.** No write operation is reachable without a WordPress Application Password.
- **Errors are not RFC 9457.** A bad `per_page` returns HTTP 400 `rest_invalid_param`; a
  missing id returns HTTP 404 `rest_post_invalid_id`. Branch on `code`. See
  `errors/skydance-media-problem-types.yml`.
- **Decode before you display.** `title.rendered` carries HTML entities
  (`&#8220;`, `&#8211;`, `&#8217;`).
- **Attribute the source.** Cite the result `url` when you surface it.
