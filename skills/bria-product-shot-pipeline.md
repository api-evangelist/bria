---
name: bria-product-shot-pipeline
description: Turn a raw product photo into e-commerce-ready imagery with Bria — cut the product out, build a clean packshot, add a consistent shadow, then place it in a generated lifestyle scene by text or by reference image. Includes the automotive variant for vehicle photography. Use when the user has product photos and wants catalog, PDP or ad-ready shots.
api: openapi/bria-product-shot-editing-openapi-original.yml
base_url: https://engine.prod.bria-api.com/v1
operations:
  - product-cutout
  - product-cutout-v2
  - product-packshot
  - product-shadow
  - product-lifestyle-shot-by-text
  - product-lifestyle-shot-by-image
  - product-integrate
  - consistent-product-shots
  - contextual-keyword-extraction
  - vehicle-segmentation
  - vehicle-shot-by-text
  - vehicle-shot-by-image
  - vehicle-generate-reflections
  - vehicle-refine-tires
  - vehicle-harmonize
  - get_status
generated: '2026-08-08'
method: generated
source: openapi/bria-product-shot-editing-openapi-original.yml
---

# Build a product shot with Bria

## Before you start

- `api_token` header on every request. It is a plain header parameter in Bria's spec, not a
  declared security scheme, so generated clients drop it — add it explicitly.
- Send `User-Agent: BriaPlatform/APIdocs/LLMsAgent` as Bria's `llms.txt` requires.
- Every `image` parameter accepts either a **public URL** or **base64** data. v2 unified the old
  `image_url` / `image_file` pair into a single `image`.
- No idempotency key exists. Each retry is a new billable job.

## The core pipeline

1. **Cut out the product.** `product-cutout` (`POST /product/cutout`) — or `product-cutout-v2`
   (`POST /image/edit/product/cutout`), the newer path. Returns the product isolated from its
   background.
2. **Make a packshot.** `product-packshot` (`POST /product/packshot`) produces a precise,
   catalog-style cutout on a clean field.
3. **Ground it with a shadow.** `product-shadow` (`POST /product/shadow`) adds a consistent,
   customisable shadow. Skip this and composited products float.
4. **Place it in a scene.** Choose one:
   - `product-lifestyle-shot-by-text` (`POST /product/lifestyle_shot_by_text`) — describe the
     background in words.
   - `product-lifestyle-shot-by-image` (`POST /product/lifestyle_shot_by_image`) — supply a
     background image to match an existing brand set.
   - `product-integrate` (`POST /image/edit/product/integrate`) — place the product into an
     existing scene at precise coordinates.
5. **Keep a set coherent.** `consistent-product-shots`
   (`POST /products/consistent_shots`) generates a set that holds together across SKUs, which is
   what a catalog actually needs — running step 4 per SKU will drift.
6. **Optional — mine the scene copy.** `contextual-keyword-extraction`
   (`POST /product/contextual_keyword_extraction`) pulls keywords from the generated context,
   useful for PDP metadata.

## Automotive variant

Vehicles get their own operations because a car is reflective and wheel geometry gives away a
bad composite:

1. `vehicle-segmentation` (`POST /product/vehicle/segment`) — isolate the vehicle.
2. `vehicle-shot-by-text` or `vehicle-shot-by-image` — place it in a showroom or location.
3. `vehicle-generate-reflections` (`POST /product/vehicle/generate_reflections`) — the step that
   makes a composite read as real.
4. `vehicle-refine-tires` (`POST /product/vehicle/refine_tires`).
5. `vehicle-harmonize` (`POST /product/vehicle/harmonize`) — final colour/light match between
   vehicle and scene. Run this last.

## Chaining correctly

Each step returns a result image. Feed the previous step's returned URL into the next step's
`image` parameter rather than re-uploading base64 — it is faster and avoids the 12MB payload
limit (`413`).

These operations are on the `/v1` path. Bria has published a v1→v2 migration table for Image
Editing and states Product Shot Editing migrates in a later phase, with no date published. Pin
nothing to `/v1` that you cannot move — see `lifecycle/bria-lifecycle.yml`.

## Async and results

Bria v2 is async by default (`sync` defaults to `false`). Take the `request_id` + `status_url`,
then either poll `get_status` (`GET /v2/status/{request_id}`) or pass `webhook_url` on the
request. `get_status` returns **200 for failed jobs too** — read the body's `status` field.

## Errors worth handling

`413` payload over 12MB · `415` only jpeg/jpg/png/webp · `429` plan rate limit · `460` Bria could
not download your public image URL · `506` the input is not supported. Full catalog in
`errors/bria-problem-types.yml`.
