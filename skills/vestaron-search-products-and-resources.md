---
name: Search Vestaron products and resources
description: Find Vestaron product pages (SPEAR RC, SPEAR T, LEPROTEC, LEPROTEC WG) and downloadable resources through the site-wide search endpoint, then resolve results to full page objects and media files.
api: openapi/vestaron-content-openapi.yml
operations:
  - getSearch
  - getPages
  - getPagesById
  - getMedia
  - getMediaById
  - getTypes
generated: '2026-08-05'
method: generated
source: openapi/vestaron-content-openapi.yml + data-model/vestaron-data-model.yml
---

# Search Vestaron products and resources

Base URL: `https://vestaron.com/wp-json/wp/v2`

**Authentication: none.** All operations below are anonymously readable.

## 1. Search the site

`getSearch` — `GET /wp/v2/search?search={query}&per_page=20`

The index covers 128 objects (posts and pages). Each result is a thin projection:

```json
{"id":13380,"title":"…","url":"https://vestaron.com/…","type":"post","subtype":"post"}
```

Narrow with `type=post`, `subtype=post|page`. `X-WP-Total` on the response carries the hit count.

Search returns no body text — it is a **locator**, not a content endpoint. Use step 2 to fetch
the real object.

## 2. Resolve a result to the full object

Branch on `subtype`:

- `page` → `getPagesById` — `GET /wp/v2/pages/{id}`
- `post` → `getPostsById` — `GET /wp/v2/posts/{id}`

Each result also carries `_links.self[0].href`, which is the same URL pre-built; following it is
equivalent and avoids constructing paths yourself.

## 3. Browse product pages directly

`getPages` — `GET /wp/v2/pages?per_page=100&_fields=id,slug,link,title,parent,menu_order`

34 pages are published. The product set is:

| Product | Slug |
|---|---|
| SPEAR® RC | `spear-rc` |
| SPEAR® T Liquid Concentrate | `spear-t-liquid-concentrate-bioinsecticide-for-greenhouses-vestaron` |
| LEPROTEC® | `leprotec` |
| LEPROTEC® WG | `leprotec-wg` |
| Products overview | `overview` |

SPEAR® LEP and BASIN™ FLEX are named throughout the newsroom and on the products overview page
but have **no dedicated page object** of their own — do not construct `/spear-lep/` or `/basin/`
URLs. Search for them with `getSearch` and follow the `url` the API returns.

Company pages: `about-us`, `science`, `socialresponsibility` (Sustainability), `beefriendly`,
`resources`, `downloads`, `contact`. Legal pages nest under the `legal` parent and are
per-jurisdiction (US, EU, UK, CA, Australia, Brazil, South Africa) — always pick the page whose
slug matches the user's jurisdiction rather than assuming the US one.

Filter by slug directly: `GET /wp/v2/pages?slug=science&_fields=id,title,content`.

## 4. Find downloadable resources

`getMedia` — `GET /wp/v2/media?search={query}&per_page=50&_fields=id,title,mime_type,source_url,post`

829 attachments are published. Filter on `mime_type` (e.g. `application/pdf`) or `media_type`
(`image`, `file`). `post` gives the page or post the file is attached to, so you can cite the
page a document belongs to instead of linking a bare file.

## 5. Confirm what exists before assuming

`getTypes` — `GET /wp/v2/types` lists the registered content types on this host. Vestaron
registers **no custom post types** — there is no product, label, trial or crop entity in the API.
If a task needs structured agronomic, label or registration data, that data does not exist in
this API; say so rather than inferring it from marketing page HTML.

## Rules

- **Read only.** Never attempt a write; every write requires credentials.
- `title.rendered` and `content.rendered` are HTML with escaped entities — decode before display.
- Always set `_fields`; the unprojected page and media objects are large.
- Cap `per_page` at 100 — higher returns `400 rest_invalid_param`.
- This is the CMS content API. Product label rates, application instructions, EPA registration
  numbers and safety data are published as documents on `https://vestaron.com/resources/`, not as
  API fields. Cite the document, do not synthesize a value.
