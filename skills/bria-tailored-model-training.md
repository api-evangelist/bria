---
name: bria-tailored-model-training
description: Train a tailored (fine-tuned) Bria image model on your own dataset and generate with it — create a project, create a dataset, upload and caption images, train a model, watch checkpoints, and generate tailored images. Use when the user wants a brand-consistent or style-consistent model rather than one-off generations.
api: openapi/bria-tailored-generation-openapi-original.yml
base_url: https://engine.prod.bria-api.com/v2
operations:
  - create-project
  - create-dataset
  - upload-image
  - bulk-upload-images
  - get-bulk-upload-status
  - update-image-caption
  - regenerate-all-captions
  - create-model
  - start-training
  - get-model
  - list-checkpoints
  - get-checkpoint
  - image-generate-tailored
  - structured-prompt-generate-tailored
  - get_status
generated: '2026-08-08'
method: generated
source: openapi/bria-tailored-generation-openapi-original.yml
---

# Train and use a tailored Bria model

Bria's Tailored Generation surface is the only genuinely stateful part of its API. Everything
else is a stateless transformation; this is a resource hierarchy —
**Project → Dataset → DatasetImage** and **Project → Model → Checkpoint** — with a long-running
training job in the middle.

## Before you start

- Authenticate with the `api_token` header on **every** request. Bria declares it as a plain
  header parameter, not an OpenAPI security scheme, so most generated clients will omit it —
  add it yourself. See `authentication/bria-authentication.yml`.
- Send `User-Agent: BriaPlatform/APIdocs/LLMsAgent`; Bria's own `llms.txt` states this is required.
- There is **no idempotency key**. Never blind-retry a POST — a retry creates a second project,
  dataset or training run and a second charge. Check state with a GET before retrying. See
  `conventions/bria-conventions.yml`.

## Steps

1. **Create the project.** `create-project` (`POST /tailored-gen/projects`). Keep the returned
   `project_id`; every dataset and model hangs off it.
2. **Create the dataset.** `create-dataset` (`POST /tailored-gen/datasets`), binding it to the
   `project_id`. Confirm with `get-datasets-by-project`.
3. **Load images.** For a handful, call `upload-image`
   (`POST /tailored-gen/datasets/{dataset_id}/images`) per image. For a real training set use
   `bulk-upload-images` (`POST /tailored-gen/datasets/{dataset_id}/images/bulk-upload`) and then
   poll `get-bulk-upload-status` until it settles — bulk upload is async and returns before the
   images exist.
4. **Fix the captions.** Caption quality drives tailored-model quality. Use `get-images` to read
   what Bria auto-captioned, `update-image-caption` to correct individual images, or
   `regenerate-all-captions` to redo the whole set after changing the dataset.
5. **Create the model.** `create-model` (`POST /tailored-gen/models`), bound to the project and
   the dataset.
6. **Start training.** `start-training`
   (`POST /tailored-gen/models/{model_id}/start_training`). This returns immediately; training
   is long-running. `stop-training` cancels it.
7. **Watch progress.** Poll `get-model` for model state and `list-checkpoints`
   (`GET /tailored-gen/models/{model_id}/checkpoints`) for training checkpoints. Inspect a
   specific one with `get-checkpoint`. Prune with `delete-checkpoint`.
8. **Generate.** Once a checkpoint is usable, call `image-generate-tailored`
   (`POST /image/generate/tailored`) or, for full VGL control,
   `structured-prompt-generate-tailored`. Both are async by default.
9. **Collect the result.** Every async call returns `request_id` + `status_url`. Poll `get_status`
   (`GET /v2/status/{request_id}`) until `status` is `COMPLETED` or `ERROR`, or supply
   `webhook_url` on the request and let Bria push the result. Prefer the webhook for training
   and video work.

## Reading results correctly

`get_status` returns **HTTP 200 even for a failed job**. Branch on the body's `status` field
(`IN_PROGRESS` / `COMPLETED` / `ERROR` / `UNKNOWN`), never on the HTTP status. `UNKNOWN` is
Bria's async equivalent of a 500 — stop polling and report the `request_id`.

## Errors worth handling

- `409` — cannot delete a project while its models are training. Call `stop-training` first.
- `413` — an image exceeds the 12MB limit. Downscale or pass a public URL instead of base64.
- `415` — only `jpeg`, `jpg`, `png`, `webp` are accepted.
- `429` — plan rate limit (10/min free, 60/min starter). Back off; there is no `Retry-After`
  header contract.
- `460` — Bria could not download the image at the URL you supplied. Confirm it is publicly
  reachable.

Full catalog: `errors/bria-problem-types.yml`.

## Cleanup

`delete-checkpoint`, `delete-model`, `delete-image`, `delete-dataset`, `delete-project` — in that
order. Deleting a project with models in training returns `409`.
