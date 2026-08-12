---
name: lily-ai-product-attribute-search
description: >-
  Search a retailer subsidiary's catalog in the Lily AI platform and read the
  consumer-centric attribute tags and synonyms Lily has generated for each product,
  with faceted filters and page-number pagination.
api: LilyApp Middleware API
base_url: https://lilyapp-api-prd.pub.lilyai.net
operations:
  - SubsidiaryController_getSubsidiaryListUser
  - ConfigController_getReailerInfo
  - SearchPamController_getPrductTagsFiltersList
  - SearchPamController_getFiltersTableFormattedLocalApi
  - SearchPamController_getAllProductsFromPI
  - SearchTagsController_getProductTagAutocompleteApi
  - SearchTagsController_getPersonProductTypes
  - SearchPamController_exportAllProductsFromPI
generated: '2026-08-12'
method: generated
source: openapi/lily-ai-lilyapp-api-openapi.yml
---

# Search Lily-enriched product attributes

**Before you start.** This skill drives the middleware behind Lily AI's customer
application. It is publicly reachable and publicly documented (Swagger UI at
`/api`, contract at `/api-json`), but **there is no developer program and no
self-service credential path**. You need a bearer JWT issued by Lily AI's Azure AD
B2C tenant, obtained through the application sign-in flow. Without it every call
below returns `401 {"message":"null Token", ...}`.

## Conventions you must follow

- **Auth**: `Authorization: Bearer <jwt>` on every call. The contract does not
  declare this per-operation — it is still required. See
  `authentication/lily-ai-authentication.yml`.
- **Tenancy**: every read is scoped to a retailer *subsidiary*. The API accepts four
  spellings of the same id — `subsidiaryCode` (query), `subsidiaryID` (query),
  `subsidiary-id` (header), `x-subsidiary-id` (header). Check the operation you are
  calling; do not assume.
- **Pagination**: `pageNumber` / `pageSize` (defaults 1 / 100). Responses carry a
  `PaginationDto` with `total` and `totalPages`.
- **No idempotency**: this skill is read-only, so retries are safe, but note the API
  has no idempotency mechanism at all.
- **Rate limits**: 10 req/s, 50 req/10s, 100 req/60s, signalled on
  `X-RateLimit-*-{short,medium,long}`. Respect the `Remaining` counters rather than
  bursting.

## Steps

1. **Resolve the subsidiary you are working in.**
   `GET /subsidiary/list-subsidiaries` (`SubsidiaryController_getSubsidiaryListUser`)
   returns the subsidiaries the signed-in principal may access. Keep the `code` —
   most operations key off it.

2. **Read the retailer configuration.**
   `GET /config/retailer/info` (`ConfigController_getReailerInfo`) returns the
   `RetailerInfoDto` for the tenant. Use it to confirm you are pointed at the right
   catalog before issuing writes anywhere else in the platform.

3. **Discover the available filters.**
   `GET /search/subsidiary/tags` (`SearchPamController_getPrductTagsFiltersList`)
   lists the subsidiary's tag vocabulary; `GET
   /search/products/filters/tableFormatted`
   (`SearchPamController_getFiltersTableFormattedLocalApi`) returns the same surface
   as `FilterContainerDto` / `FilterDto` rows suitable for a table UI. Do not
   hard-code attribute names — Lily's taxonomy is per-subsidiary.

4. **Search products.**
   `POST /search/products` (`SearchPamController_getAllProductsFromPI`) with a
   `SearchPAMPayloadDto` body: `sku_id` and/or `subsidiary_tags[]`
   (`SubsidiaryTagsDto`). The response is a `ProductSearchDtoWithSynonyms` —
   `productsWithSynonym[]` of `ProductDtoWithSynonym`, each carrying `tagSections[]`
   → `tags[]` (`TagsWithSynonyms`), plus a `pagination` block.

5. **Resolve partial attribute terms.**
   `GET /search/tags/autocomplete`
   (`SearchTagsController_getProductTagAutocompleteApi`) when you have a fragment
   rather than an exact tag. `GET /search/tags/types`
   (`SearchTagsController_getPersonProductTypes`) returns the person/product type
   taxonomy that scopes tag lookups.

6. **Export when the result set is large.**
   `POST /search/products/export`
   (`SearchPamController_exportAllProductsFromPI`) instead of paging through
   thousands of products.

## Failure handling

| What you see | What it means | Do |
|---|---|---|
| `401 {"message":"null Token"}` | No/expired bearer token | Re-run the B2C sign-in flow |
| `403` | Token is valid but not entitled to that subsidiary | Re-check the subsidiary id you passed, in the spelling that operation expects |
| `404 {"name":"NotFoundException"}` | Path does not exist | Re-read `/api-json`; v1 and v2 paths both exist |
| `503` on `/health` | A platform dependency is down | Inspect `response.details[]` (`pi_api`, `lilyAppDB`, `productCopy`) before retrying |

Every auth-guard error carries a `correlationId` UUID. It is the only tracing handle
the API exposes — capture it before you retry.
