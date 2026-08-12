---
name: lily-ai-product-copy-batch
description: >-
  Create a product-copy batch in the Lily AI platform, generate AI product
  descriptions for the products in it, review and edit the generated copy, and move
  it through its approval status — using the v2 controllers.
api: LilyApp Middleware API
base_url: https://lilyapp-api-prd.pub.lilyai.net
operations:
  - ProductBatchesControllerV2_getAllProductCopyBatchesFromPI
  - ProductBatchesController_createProductBatch
  - ProductBatchesController_createProductBatchCopiesAndProducts
  - ProductBatchesControllerV2_getAllProductCopiesForBatch
  - ProductCopyController_generateDescription
  - ProductCopyControllerV2_getProductCopyTags
  - ProductCopyControllerV2_getProductCopyImages
  - ProductCopyControllerV2_updateProductCopyDetails
  - ProductCopyControllerV2_updateProductCopyStatus
  - ProductBatchesController_exportAllProductCopiesForBatch
  - ProductBatchesControllerV2_deleteProductBatch
generated: '2026-08-12'
method: generated
source: openapi/lily-ai-lilyapp-api-openapi.yml
---

# Run a product-copy batch

**Before you start.** Same access reality as every Lily AI skill: bearer JWT from
Lily's Azure AD B2C tenant, no self-service credential, no public developer program.
Read `conventions/lily-ai-conventions.yml` first.

**Choose a generation and stay in it.** Two generations of these controllers run
side by side with no deprecation marker: `/productcopy/*`
(`ProductBatchesController`, `ProductCopyController`) and `/productcopyV2/*`
(`ProductBatchesControllerV2`, `ProductCopyControllerV2`). Prefer **v2** for reads
and updates. Batch *creation* and *export* only exist on v1 — that split is a real
gap in the contract, not an error in this skill.

## Critical: this flow is not idempotent

`PUT /productcopy/batches/add` and `POST /productcopy/batches/add/products/{id}` have
**no idempotency key**. A retried create makes a second batch. Before retrying any
failed create, re-list batches with
`GET /productcopyV2/batches` and check whether your batch already landed.

## Steps

1. **List existing batches.**
   `GET /productcopyV2/batches`
   (`ProductBatchesControllerV2_getAllProductCopyBatchesFromPI`). Returns
   `ProductCopyBatches`. Do this first, always — see the idempotency warning above.

2. **Create the batch.**
   `PUT /productcopy/batches/add` (`ProductBatchesController_createProductBatch`).
   v1 only.

3. **Attach products to the batch.**
   `POST /productcopy/batches/add/products/{id}`
   (`ProductBatchesController_createProductBatchCopiesAndProducts`), where `{id}` is
   the batch id from step 2.

4. **Read the batch contents.**
   `POST /productcopyV2/batches/{id}`
   (`ProductBatchesControllerV2_getAllProductCopiesForBatch`) with a
   `SearchProductCopyBodyDto` body (`skuId`, `subsidiary_tags[]`). Note this is a
   POST that reads — the filter payload is in the body. The response is a
   `ProductCopySearchDtoV2`: `products[]` of `ProductCopyDtoV2` plus a `pagination`
   block.

5. **Generate a description for a product.**
   `POST /productcopy/products/generate/{productId}`
   (`ProductCopyController_generateDescription`) with a
   `DescriptionGenerationPayloadData` body. Carry the `session_id` through the
   review loop so the generation is traceable.

6. **Read what backs the copy.**
   `GET /productcopyV2/products/details/tags/{productId}`
   (`ProductCopyControllerV2_getProductCopyTags`) for the attribute tags driving the
   generation, and
   `GET /productcopyV2/products/details/images/{productId}`
   (`ProductCopyControllerV2_getProductCopyImages`) for the imagery.

7. **Edit the copy.**
   `PUT /productcopyV2/products/details/{productId}`
   (`ProductCopyControllerV2_updateProductCopyDetails`) with a
   `ProductDetailsUpdatePayloadDto`.

8. **Move it through review.**
   `PUT /productcopyV2/products/status/{productId}`
   (`ProductCopyControllerV2_updateProductCopyStatus`) with a
   `ProductStatusUpdatePayloadDto` carrying a `ProductCopyStatus`.

9. **Export, then clean up.**
   `POST /productcopy/batches/export/{id}`
   (`ProductBatchesController_exportAllProductCopiesForBatch`), then
   `DELETE /productcopyV2/batches/{id}`
   (`ProductBatchesControllerV2_deleteProductBatch`) when the batch is finished.
   Deletion is destructive and unconfirmed by the contract — export first.

## Failure handling

- `401 {"message":"null Token"}` — token missing or expired. Re-authenticate.
- `403` — the token is not entitled to this subsidiary. Every operation here is
  subsidiary-scoped; confirm which spelling (`subsidiaryCode` query,
  `subsidiary-id` / `x-subsidiary-id` header) the operation takes.
- `503` — check `GET /health`. `productCopy` is one of the named dependencies and has
  been observed down; do not queue generations against a degraded dependency.
- Capture the `correlationId` from any error body before retrying.
- Back off on the `X-RateLimit-Remaining-short` counter (10 req/s ceiling) — batch
  generation loops hit it first.
