---
name: iggenix-read-the-record
description: Search and read IgGenix's public scientific and corporate record — press releases, peer-reviewed publications and conference abstracts — as structured JSON from the WordPress REST API at iggenix.com.
api: iggenix:iggenix-search-api
operations:
  - getSearch
  - getPressreleases
  - getPressreleasesId
  - getPublications
  - getPublicationsId
  - getAbstracts
  - getAbstractsId
---

# Read the IgGenix record

IgGenix is a clinical-stage antibody company (lead program IGNX001, a human IgG4 monoclonal
antibody for peanut allergy). It ships no product API. Its public record — announcements,
peer-reviewed papers, conference abstracts — is served as JSON by the WordPress REST API on its
own site. Base URL: `https://iggenix.com/wp-json`. No credential is required for anything below.

## Before you start

- The three content collections are **separate post types**, not one blog: `pressreleases` (20
  items), `publications` (9), `abstracts` (14). Core `posts` is registered but **empty**.
- Ids are unique across types but **not interchangeable between routes**. Asking for a publication
  id under `/wp/v2/posts/{id}` returns `404 rest_post_invalid_id`.
- An empty collection is `200` with `[]` and `X-WP-Total: 0`. Read the header; do not infer absence
  from the status code.
- Use `_fields` to keep responses small, and read `X-WP-Total` / `X-WP-TotalPages` for paging.
  `per_page` is capped at 100 — every collection here fits in one page.

## 1. Start with search (`getSearch`)

`GET /wp/v2/search?search=peanut`

This is the entry point, because it is the only operation that spans every post type. Each hit
returns `id`, `title`, `url`, `type`, `subtype`, and a `_links.self[0].href` you can follow
directly. **Branch on `subtype`** — it tells you which typed collection the item really lives in
(`pressreleases`, `publications`, `abstracts`, `page`).

## 2. Or walk a collection directly

- `getPressreleases` — `GET /wp/v2/pressreleases?per_page=100&orderby=date&order=desc&_fields=id,date,slug,link,title`
- `getPublications` — `GET /wp/v2/publications?per_page=100&_fields=id,date,slug,link,title`
- `getAbstracts` — `GET /wp/v2/abstracts?per_page=100&_fields=id,slug,link,title`

Useful filters declared on all three: `search`, `after`, `before`, `modified_after`,
`modified_before`, `include`, `exclude`, `slug`, `order`, `orderby`, `offset`.

## 3. Fetch one item

- `getPressreleasesId` — `GET /wp/v2/pressreleases/{id}`
- `getPublicationsId` — `GET /wp/v2/publications/{id}`
- `getAbstractsId` — `GET /wp/v2/abstracts/{id}`

Add `_embed` to inline the featured media (the abstracts frequently attach the conference poster
PDF) in the same round trip instead of following `_links` separately.

## Reading the payload

- `title.rendered`, `content.rendered` and `excerpt.rendered` are **HTML strings with HTML
  entities** (`&#8220;`, `&nbsp;`). Unescape and strip before using them as text.
- `date` is site-local (UTC−7); `date_gmt` is the one to compare across sources.
- `link` is the canonical permalink. Prefer it over constructing a URL from the slug.
- `acf` is present but empty on this site — there are no extra custom fields to read.

## Errors

The envelope is WordPress's, not RFC 9457:

```json
{"code":"rest_post_invalid_id","message":"Invalid post ID.","data":{"status":404}}
```

Branch on `code`, never on `message`. `rest_invalid_param` (400) puts the offending parameter names
in `data.params`. See `errors/iggenix-problem-types.yml`.

## Limits and etiquette

No rate limits are published and no `X-RateLimit-*` or `Retry-After` headers are returned. The
site's `robots.txt` asks for `Crawl-delay: 10`; honour it. Responses are edge-cached for ten
minutes, so polling faster than that gains nothing.

## What not to do

Do **not** call `/wp/v2/users`. It returns real staff members' names and Gravatar hashes to
anonymous callers. It is out of scope for this skill by design.
