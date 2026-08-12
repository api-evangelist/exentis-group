---
name: exentis-group-research-technology-pages
description: Answer questions about Exentis Group's 3D screen printing technology, materials, applications, certifications and sustainability by resolving a query to real page ids through the site search endpoint and fetching the structured page bodies, instead of scraping HTML.
api: exentis-group:exentis-group-search-api
operations:
  - searchSite
  - getPage
  - listPages
  - getBlogPost
  - listLanguages
generated: '2026-08-12'
method: generated
source: >-
  openapi/exentis-group-search-api-openapi.yml, openapi/exentis-group-pages-api-openapi.yml,
  openapi/exentis-group-blog-api-openapi.yml, data-model/exentis-group-data-model.yml
---

# Research Exentis Group's technology from the source

Exentis Group's substantive technical content — the screen-printing process, material families,
application segments, certifications, sustainability posture — lives in 119 WordPress pages and 16
blog entries, all readable as JSON. Use the API rather than fetching the rendered site: the HTML
homepage alone is ~230 KB, and the JSON page body is a fraction of that once you select fields.

Base URL: `https://www.exentis-group.com/wp-json`. No authentication.

## 1. Confirm the language tree first

```
GET /pll/v1/languages
```

Returns two languages: `de` (`de_CH`, **the default**) and `en` (`en_US`). If you omit `lang` on any
later call you get German. It also returns `page_on_front` and `page_for_posts` page ids per
language — the roots of each tree.

Beware: each language object embeds a base64 data-URI flag image. Ignore the `flag` field.

## 2. Resolve the question to ids

`searchSite` is the resolver. It searches every publicly readable object at once and tells you which
collection each hit belongs to.

```
GET /wp/v2/search?search=sintering&per_page=20&_fields=id,title,url,type,subtype
```

Each result carries:
- `id` — the object id
- `subtype` — `page`, `post` or `blog`
- `url` — the human page
- `type` — always `post` for content results; it is **not** the useful discriminator

**`subtype` decides your next call**, and getting it wrong is the most common mistake here:

| `subtype` | fetch from |
|---|---|
| `page` | `/wp/v2/pages/{id}` |
| `post` | `/wp/v2/posts/{id}` |
| `blog` | `/wp/v2/blogposts/{id}` |

506 objects were searchable at profiling time. Narrow with `subtype=page` when you only want
reference material rather than announcements.

```
GET /wp/v2/search?search=ceramics&subtype=page&per_page=20
```

## 3. Fetch the page body

```
GET /wp/v2/pages/{id}?_fields=id,slug,link,title,content,excerpt,parent,menu_order,lang,translations
```

`content.rendered` is HTML. `parent` gives you the section it sits under — the site is organized as
`/en/technology/...`, `/en/materials/...`, `/en/applications/...`, `/en/about-exentis/...`,
`/en/sustainability/...`, `/en/investors/...`, `/en/careers/...`.

## 4. Walk a whole section instead of searching

When you want everything under one topic, list by parent rather than guessing slugs:

```
GET /wp/v2/pages?lang=en&per_page=100&_fields=id,slug,title,parent,menu_order,link
```

119 pages fit in two requests at `per_page=100`. Build the tree from `parent`, order siblings by
`menu_order`, then fetch only the bodies you need.

## 5. Blog entries are a different shape

The `blog` custom post type is **not** shaped like a post. It has no `author`, no `excerpt` and no
`featured_media`. Code written against `/wp/v2/posts` will throw on missing keys.

```
GET /wp/v2/blogposts/{id}?_fields=id,slug,title,content,date,blogkategorie,lang,translations
GET /wp/v2/blogkategorie?per_page=100&_fields=id,slug,name,count
```

12 blog-category terms existed at profiling time.

## 6. Cross the language boundary

Every content object carries `translations`, a map from language slug to the sibling object's id.
Use it to move between the German and English versions of the same page — do not search twice.

## Rules that will bite you

- Always pass `_fields`. `yoast_head` and `yoast_head_json` are large on every object and are never
  what you want.
- `per_page` max is 100; 200 returns `400 rest_invalid_param` / `rest_out_of_bounds`.
- Totals live in the `X-WP-Total` / `X-WP-TotalPages` headers, never in the body.
- Errors use the WordPress envelope `{code, message, data:{status}}`, not RFC 9457, and `message` is
  always German. Branch on `code`.
- No rate-limit headers exist. Self-throttle.
- The `acf` field appears on every object and carries site-specific Advanced Custom Fields content
  with no published schema. Treat it as opaque; do not build a contract on it.
- `robots.txt` disallows `/*?` for generic agents and bans ~35 named AI crawlers by user-agent. Get
  an exemption before running this at scale.
- Do not use `_embed` and do not call `/wp/v2/users`. The author records are person data and are
  deliberately out of scope for every skill in this repository.
