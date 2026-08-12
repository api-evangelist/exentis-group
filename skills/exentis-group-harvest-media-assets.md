---
name: exentis-group-harvest-media-assets
description: Locate and retrieve Exentis Group brand, product and trade-fair imagery from the company's public media library — search by keyword, filter by MIME type, and pick the right generated size variant instead of downloading full-resolution originals.
api: exentis-group:exentis-group-media-api
operations:
  - listMedia
  - getMediaItem
  - getPage
generated: '2026-08-12'
method: generated
source: openapi/exentis-group-media-api-openapi.yml + data-model/exentis-group-data-model.yml
---

# Harvest Exentis Group media assets

The company's media library — 1,155 items at profiling time — is readable as JSON without
credentials, with rendered source URLs, MIME types and every generated size variant. This is how the
logo used on the API Evangelist profile for Exentis Group was located.

Base URL: `https://www.exentis-group.com/wp-json`. No authentication.

## 1. Find the asset

```
GET /wp/v2/media?search=logo&per_page=20&_fields=id,title,source_url,mime_type,media_type,alt_text
```

`search` matches the attachment title, filename, caption and description. Real examples from this
library: `logo`, `Formnext`, `Ceramitec`, `EMO`, `PowderMet`, `Interpack` — the trade-fair assets are
titled after the event.

Narrow by type when the library is noisy:

```
GET /wp/v2/media?media_type=image&mime_type=image/svg%2Bxml&per_page=100&_fields=id,title,source_url
```

- `media_type` — `image`, `video`, `audio`, `application`, `file`
- `mime_type` — an exact MIME string; URL-encode the `+` in `image/svg+xml`

## 2. Pick the right rendition

`listMedia` gives you `source_url`, which is the **original** — often multi-megabyte and sometimes
`-scaled`. Fetch the single item to see the generated variants:

```
GET /wp/v2/media/{id}?_fields=id,title,source_url,mime_type,filesize,media_details
```

`media_details.sizes` is a map of rendition name to `{source_url, width, height, mime_type}`. Choose
by the width you actually need rather than downloading the original.

SVG attachments have no `media_details.sizes` — vectors are not resized. For those, `source_url` is
the only URL and it is the correct one.

## 3. Find what an asset belongs to

`post` is the attachment's parent object id. It points into the shared WordPress post-id sequence, so
it may resolve to a page, a post or a blog entry — you cannot tell from the id alone.

```
GET /wp/v2/media/{id}?_fields=id,post,_links
```

Read `_links` rather than guessing the collection: the `about` and `up` hrefs are absolute URLs to
the parent's real endpoint.

Going the other way, `featured_media` on a page or post is a media id — `0` means none.

## 4. Sweep the library

```
GET /wp/v2/media?per_page=100&orderby=date&order=desc&_fields=id,title,source_url,mime_type,filesize
```

At `per_page=100` the full library is 12 pages. Follow `Link: rel="next"`; the totals are in
`X-WP-Total` and `X-WP-TotalPages`.

`orderby` here accepts `date`, `id`, `include`, `modified`, `parent`, `relevance`, `slug`,
`include_slugs`, `title`, `author`. Anything else returns `400` / `rest_not_in_enum`.

## Rules that will bite you

- **Media objects have no `lang` field.** The media library is not translated, unlike pages, posts
  and terms. Do not pass `lang` here expecting it to filter.
- `per_page` max is 100.
- Errors are the WordPress envelope, `message` is German, branch on `code`.
- Asset URLs live under `https://www.exentis-group.com/app/uploads/YYYY/MM/...` — note `app/uploads`,
  not the usual `wp-content/uploads`; this is a Bedrock-style layout. Do not hand-construct these
  paths, always read `source_url`.
- No rate-limit headers and no ETag/Last-Modified. Self-throttle, and cache by `id` + `modified`.
- **Licensing is not granted by accessibility.** These are Exentis Group's copyrighted brand and
  product assets, and several are third-party trade-fair logos (Formnext, Ceramitec, EMO, Interpack,
  PowderMet, TCT) owned by their organizers. The API being open is not permission to republish.
  Check `https://www.exentis-group.com/en/terms-of-use/` and ask.
- Do not use `_embed`; it inlines the author record, which is person data.
