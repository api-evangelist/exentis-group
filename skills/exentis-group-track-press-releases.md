---
name: exentis-group-track-press-releases
description: Track Exentis Group news and press releases over time from the company's own WordPress REST API — pull the latest items in a chosen language, page through the full 371-item archive, and detect what changed since a previous run.
api: exentis-group:exentis-group-posts-api
operations:
  - listPosts
  - getPost
  - listCategories
generated: '2026-08-12'
method: generated
source: openapi/exentis-group-posts-api-openapi.yml + conventions/exentis-group-conventions.yml
---

# Track Exentis Group press releases

Exentis Group publishes no press API and no developer program. What it does serve, from its own
marketing host, is the WordPress REST API — and that is a perfectly good way to watch an
additive-manufacturing equipment vendor's announcements without scraping HTML.

Base URL: `https://www.exentis-group.com/wp-json`. No key, no signup, no auth header.

## 1. Pull the most recent releases

Use `listPosts`. Always constrain the fields — an unfiltered post object carries a multi-kilobyte
`yoast_head` SEO string you do not want.

```
GET /wp/v2/posts?lang=en&per_page=20&orderby=date&order=desc
    &_fields=id,slug,date,modified,link,title,excerpt,categories
```

- `lang` accepts `en` or `de`. Omitting it returns the German default tree. 185 English posts and
  186 German posts existed at profiling time; they are near-parity but not identical, so pick one
  and stay in it.
- `per_page` is capped at **100**. Asking for more returns `400 rest_invalid_param` with sub-code
  `rest_out_of_bounds`.
- `orderby` accepts only: `author, date, id, include, modified, parent, relevance, slug,
  include_slugs, title`. Anything else returns `400` / `rest_not_in_enum`.

## 2. Page through the archive correctly

The totals are **only in response headers** — nothing in the body tells you how many there are.

- `X-WP-Total` — total items matching the query
- `X-WP-TotalPages` — total pages at the current `per_page`
- `Link` — RFC 8288 `rel="next"` / `rel="prev"`

Follow `Link: rel="next"` until it is absent. Do not compute page counts yourself.

```
GET /wp/v2/posts?lang=en&per_page=100&page=1&_fields=id,slug,date,modified,title
```

At `per_page=100` the English archive is two pages.

## 3. Detect change since a previous run

Filter server-side rather than diffing the whole archive:

```
GET /wp/v2/posts?lang=en&modified_after=2026-07-01T00:00:00&per_page=100
    &_fields=id,slug,date,modified,link,title
```

`after` / `before` filter on publication date; `modified_after` / `modified_before` filter on last
edit. Use `modified_after` to catch silent revisions to an already-published release — a real thing
with corporate announcements.

Store the highest `modified` you have seen and pass it back on the next run.

## 4. Resolve categories

`listPosts` returns `categories` as an array of term ids, not names. Resolve them once and cache:

```
GET /wp/v2/categories?per_page=100&_fields=id,slug,name,count
```

27 terms existed at profiling time.

## 5. Fetch one release in full

```
GET /wp/v2/posts/{id}?_fields=id,slug,date,modified,link,title,content,excerpt,categories,translations
```

`translations` maps to the sibling-language version of the same item by id — that is how you jump
from the English release to the German one without a second search.

## Rules that will bite you

- **Errors are not RFC 9457.** The envelope is `{code, message, data:{status, params, details}}` with
  content-type `application/json`. Branch on `code` only.
- **Messages are German.** `message` is localized to `de_CH` even on `lang=en` requests. Never
  string-match it.
- **There is no rate-limit signal.** No `X-RateLimit-*`, no `RateLimit-*`, no `Retry-After`, and no
  observed 429. Self-throttle — a request per second is courteous and sufficient.
- **Conditional requests do not work.** No `ETag`, no `Last-Modified`, and `Expires` equals the
  response date. Use `modified_after` (step 3) as your change detector instead of HTTP caching.
- **`robots.txt` says `Disallow: /*?`.** Every call in this skill carries a query string. If you
  operate under a robots-honoring policy, you need an explicit exemption or the site owner's
  agreement before running it on a schedule. Roughly 35 named AI crawlers — including `Claude-User`,
  `CCBot` and `OAI-SearchBot` — are disallowed outright by user-agent.
- **Do not use `_embed`.** It inlines the author object, which is person data. Everything this skill
  needs is available without it.

## Errors

| status | code | what to do |
|---|---|---|
| 400 | `rest_invalid_param` | Read `data.params` — it names the offending parameter. Fix and retry. |
| 401 | `rest_forbidden_context` | You asked for `context=edit`. Drop it; use the default `view`. |
| 404 | `rest_post_invalid_id` | The id does not exist or is not public. Re-resolve via `listPosts`. |
| 404 | `rest_no_route` | Wrong path or method. Re-read `https://www.exentis-group.com/wp-json/`. |
