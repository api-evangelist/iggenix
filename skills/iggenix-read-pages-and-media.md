---
name: iggenix-read-pages-and-media
description: Read iggenix.com pages, media and post-type registry through the WordPress REST API — the only reliable path to this content while the site's own HTML routing serves another company's home page.
api: iggenix:iggenix-pages-api
operations:
  - getPages
  - getPagesId
  - getMedia
  - getMediaId
  - getTypes
  - getRouteIndex
---

# Read IgGenix pages and media

## Why this skill exists

On 2026-08-22 the HTML front end at `iggenix.com` was misconfigured: the site root and **every**
unmatched path — including `/privacy-policy/` and `/the-iggenix-team/` — returned a byte-identical
76,626-byte page belonging to a different company (iECURE, `<title>Home | iECURE</title>`,
canonical `https://iecure.com/`). Only the three custom-post-type archives (`/publications/`,
`/abstracts/`, `/pressreleases/`) render IgGenix HTML.

The REST API is unaffected and returns the correct IgGenix content. If you need the privacy policy
or the team pages, **read them through the API**; a browser or naive crawler will silently give you
the wrong company. Verify what you fetched before you use it.

## Discover the surface first (`getRouteIndex`, `getTypes`)

- `GET /wp-json/` — the full route index: 148 routes, 7 namespaces, plus `name: "IgGenix"` and the
  advertised authentication scheme. This is the document every OpenAPI in this repository was
  derived from, and it is how you confirm you are talking to IgGenix and not to the catch-all.
- `GET /wp/v2/types` — the registered post types and their `rest_base` values. The IgGenix-specific
  ones are `publications`, `abstracts`, `pressreleases`, `careers`.

## Pages (`getPages`, `getPagesId`)

`GET /wp/v2/pages?per_page=100&_fields=id,slug,link,title,parent`

27 pages. They are **hierarchical** — the leadership pages nest under `the-iggenix-team`
(management, scientific co-founders, board of directors, scientific advisory board). Follow
`parent` to rebuild the tree.

Known ids worth having: `2` is the front page (currently carrying the wrong company's content —
this is the defect itself, visible in the data), `3` is the IgGenix privacy policy.

`GET /wp/v2/pages/{id}` returns `content.rendered` as an HTML string. Unescape entities and strip
tags before treating it as text.

## Media (`getMedia`, `getMediaId`)

`GET /wp/v2/media?per_page=100&_fields=id,date,slug,link,source_url,mime_type,title`

76 items, images and PDFs. The PDFs are the substantive ones — published conference posters such as
the GLP safety toxicology data ahead of Phase 1 initiation of IGNX001. Use `source_url` for the
file itself and `mime_type` to filter (`application/pdf`).

Filter by parent with `?parent={post_id}` to get the attachments of one abstract, or add `_embed`
to the abstract request and read `_embedded['wp:featuredmedia']` instead.

## Conventions that apply throughout

- `_fields` for sparse responses, `_embed` to resolve `_links` in one call, `context=view` only
  (anything else needs authentication).
- `X-WP-Total` / `X-WP-TotalPages` carry the counts; `Link` carries RFC 8288 next/prev.
- Errors use the WordPress envelope — branch on `code`.

## Out of scope

Writes (all authenticated, no idempotency key, and `DELETE /wp/v2/media/{id}` is irreversible),
`/wp/v2/settings` and `/wp-abilities/v1/*` (both `401` anonymously), and `/wp/v2/users` (personal
data).
