---
name: Read the Vestaron newsroom
description: Retrieve Vestaron press releases, trade articles, award recognitions and video/podcast entries from the vestaron.com WordPress content API, filtered by category, tag or date, with correct pagination.
api: openapi/vestaron-content-openapi.yml
operations:
  - getCategories
  - getTags
  - getPosts
  - getPostsById
  - getMediaById
generated: '2026-08-05'
method: generated
source: openapi/vestaron-content-openapi.yml + conventions/vestaron-conventions.yml
---

# Read the Vestaron newsroom

Base URL: `https://vestaron.com/wp-json/wp/v2`

**Authentication: none.** Every operation in this skill is anonymously readable. Do not send
credentials. If you receive `401 rest_forbidden_context`, you asked for `context=edit` — drop it.

## 1. Resolve the category you want

`getCategories` — `GET /wp/v2/categories?per_page=20&_fields=id,name,slug,count`

Four categories exist: `news` (64 posts), `articles` (16), `recognitions` (13), `multimedia`
(Video/Podcasts, 5). Take the numeric `id`; the posts collection filters on ids, not slugs.

Year tags are available the same way via `getTags` — `GET /wp/v2/tags?per_page=20` returns
`2015`, `2018` … `2025`.

## 2. List posts

`getPosts` — `GET /wp/v2/posts`

Useful parameters, all taken from the live route descriptor:

| Parameter | Notes |
|---|---|
| `categories` | comma-separated category ids from step 1 |
| `tags` | comma-separated tag ids |
| `search` | free-text match |
| `after` / `before` | ISO 8601 date bounds on the published date |
| `order` | `asc` or `desc` (default `desc`) |
| `orderby` | `date`, `modified`, `relevance`, `title`, `slug`, `id` |
| `per_page` | 1–100, default 10 — **101 or more returns `400 rest_invalid_param`** |
| `page` | 1-based |
| `_fields` | comma-separated projection; always set it to keep responses small |
| `_embed` | inlines author, featured media and terms into `_embedded` |

Example:
`GET /wp/v2/posts?categories=35&per_page=25&_fields=id,date,slug,link,title,excerpt`

## 3. Page correctly

Read the pagination signal from the response headers, not from the body:

- `X-WP-Total` — total matching items (94 across the whole newsroom)
- `X-WP-TotalPages` — total pages at your `per_page`
- `Link` — RFC 8288 `rel="prev"` / `rel="next"`

Stop when `page` exceeds `X-WP-TotalPages`; requesting beyond it returns `400 rest_invalid_param`.

## 4. Fetch one post and its image

`getPostsById` — `GET /wp/v2/posts/{id}` returns `title.rendered`, `content.rendered` and
`excerpt.rendered` as **HTML strings**; strip or render them, do not treat them as plain text.
HTML entities are escaped (`&#8220;`, `&#x2122;`) — decode before display.

`getMediaById` — `GET /wp/v2/media/{featured_media}` resolves the hero image to a `source_url`.
Skip this call entirely by adding `_embed` in step 2; the media object arrives in
`_embedded['wp:featuredmedia']`.

## 5. Error handling

Errors use the WordPress envelope, **not** RFC 9457 problem+json:

```json
{"code":"rest_post_invalid_id","message":"Invalid post ID.","data":{"status":404}}
```

| Code | Status | What to do |
|---|---|---|
| `rest_invalid_param` | 400 | read `data.params` — it names the offending parameter |
| `rest_post_invalid_id` | 404 | the id is not a published post; re-list the collection |
| `rest_no_route` | 404 | the path is not registered; re-read `https://vestaron.com/wp-json/` |
| `rest_forbidden_context` | 401 | remove `context=edit` |

Full catalog: `errors/vestaron-problem-types.yml`.

## Rules

- **Read only.** Every write operation on this API requires credentials you do not have. Never
  attempt `POST`, `PUT`, `PATCH` or `DELETE`.
- **No retries-as-idempotent.** This API has no idempotency key. A retried write would duplicate.
- **No rate-limit signal.** The host returns no `RateLimit-*` or `Retry-After` header. Self-limit
  to a modest request rate and always set `_fields`.
- This is Vestaron's CMS content API, not a crop-protection product API. It carries marketing and
  newsroom content. Do not present it as agronomic, regulatory or label data — for label and
  safety information use the documents on `https://vestaron.com/resources/`.
