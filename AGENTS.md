# AGENTS.md — hard rules for AI assistants

This file is the source of truth for anyone (human or model) modifying
this sample. If something here conflicts with a generic instinct, this
file wins.

## Sample-local instructions

Follow [parent CLAUDE.md](../CLAUDE.md) and the engineering working
agreement at all times. The following rules are specific to this sample:

### Rule 1: never add `boto3` or `botocore`

Storage is fully delegated to `genblaze-s3`. A structural test
(`services/api/tests/test_structure.py::test_no_direct_aws_client_imports`)
fails CI if either module is imported anywhere under
`services/api/app/`. There is no allow-list — this sample has no
`_client` reach-through.

If you need a B2 capability `genblaze-s3` doesn't expose (e.g. a new
list/copy/permission helper), file an SDK gap in the build feedback
section of the next sample build and use the existing primitives in
the meantime. Do not add a side-channel client.

### Rule 2: never wrap Genblaze response types

FastAPI handlers return `Run`, `Step`, `Asset`, `Manifest` instances
directly. The only custom DTOs in this codebase are request bodies
(`PromptRequest`, `MediaRequest`). If you need a field a Genblaze
model doesn't have, file SDK feedback — don't shadow the type with a
mirrored Pydantic class.

### Rule 3: provider classes live in `app/repo/provider_catalog.py`

This sample is a provider switchboard — any provider can drive any modality,
chosen per-run. The provider-import surface is the catalog, not pipelines.

- `provider_catalog.py` is the only file that imports provider CLASSES
  (`ImagenProvider`, `RunwayProvider`, `ReplicateProvider`, etc.). It exposes
  a `CatalogEntry` per `(slot, vendor)` with a `make()` factory + quirks.
- `pipelines.py` imports only `genblaze_openai.chat` (the standalone
  storyboard function — not a provider) and `genblaze_core` types/`Pipeline`.
  It resolves a `CatalogEntry` (passed by the caller) and calls `entry.make()`.
- `composer.py` may import `genblaze_core` *types* only
  (`Asset`, `Manifest`, `Mp4Handler`) — no `Pipeline`/`Provider` use.
- `main.py` / `app/types/**` may not import from any `genblaze_*` package.
  `main.py` consumes the catalog via `app.repo.provider_catalog`.

The structural test (`test_genblaze_provider_imports_confined`) enforces this:
provider-package imports are allowed only in `provider_catalog.py` and
`pipelines.py`.

### Rule 4: adding a new provider = one `CatalogEntry`

When the next provider lands on PyPI as `genblaze-<vendor>`:

1. Add the dependency to `services/api/pyproject.toml` and re-pin
   `requirements.txt` with
   `uv export --format requirements-txt --no-hashes --output-file requirements.txt`.
   (New adapters require `genblaze-core>=0.3.4`.)
2. Import the provider class in `app/repo/provider_catalog.py` (no other file)
   and add ONE `CatalogEntry` to the relevant slot in `CATALOG`: the `make()`
   factory (with the right key kwarg — `api_key`/`api_token`/`api_secret`/
   `auth_token`), `env_key`, a curated `default_model` + `suggested_models`,
   and any quirks (`image_handoff` for video, `snap_durations` for a clip
   grid, registry overrides baked into `make()`).
3. If it needs a NEW API key, add the field to `config.py`, `.env.example`,
   and the startup/`/health` provider dicts in `main.py`.
4. The pipeline slug (`PIPELINE_NAME`) does NOT change — Manifest lineage
   differentiates via `parent_run_id`. `pipelines.py` needs NO edit.
5. If the provider produces assets the composer reads differently, extend
   `_group_scenes()` in `composer.py`.
6. Add the default model to the conformance test's expectations (it auto-runs
   over every `CATALOG` entry) and update `docs/features/media-generation.md`.

### Rule 5: cross-pipeline asset handoff style is data on the catalog entry

Genblaze 0.3.x `from_result()` only records lineage; it does NOT hydrate
prior step assets into provider kwargs. The Stage B1 → Stage B2
image-to-video handoff goes through `presign_asset_url(...)` plus the style
declared by `video_entry.image_handoff`:

- `"external_inputs"` (the DEFAULT for nearly every provider — Kling, Runway,
  Luma, Replicate, Veo, Sora): the keyframe is passed as an
  `external_inputs=[Asset(...)]`; these providers route the image from step
  inputs and would DROP a bare `image=` kwarg.
- `"image_kwarg"` (legacy, Decart only): the presigned URL is passed as
  `image=`.

`pipelines.build_media_pipeline` branches on `image_handoff`; adding a video
provider means setting the right value on its `CatalogEntry`, not editing the
branch. Do not introduce `input_from=<step_index>` for cross-pipeline
handoffs — `input_from=` is for within-pipeline fan-out from a shared upstream
step (this sample doesn't use it; the fan-out is `max_concurrency=3` across
sibling `.step()` calls within a single Pipeline).

### Rule 6: composer is the ONLY ffmpeg surface

`repo/composer.py` is the lone non-Genblaze media-processing module
because the SDK ships no composition primitive. Do not add ffmpeg calls
elsewhere. Do not add `ffmpeg-python` (the explicit `subprocess.run([...])`
shape is intentional — one fewer dependency layer, matches production
media pipelines). Call the composer from `main.py` via
`asyncio.to_thread(...)` so the FastAPI event loop never blocks.

### Rule 7: preflight stays on for the ESSENTIAL pipelines

The essential pipelines — B0 (reference) and B1 (keyframes) — are
constructed with the default `preflight=True`. This catches a misconfigured
`OPENAI_API_KEY` before any paid call fires. Do not set `preflight=False` on
these for dev convenience.

**Stage B2 (video + TTS + music) is the deliberate exception:** it is built
`preflight=False` and run `fail_fast=False, raise_on_failure=False` because
video, narration, and music are best-effort — a DEAD/failing model must be
contained as one FAILED step (then degraded by the composer) rather than
aborting the run at preflight, which validates *every* step. See
ARCHITECTURE.md §"Ethos constraints" #5. Keep this divergence intentional and
documented; do not "restore" `preflight=True` on B2.

Stage A (`genblaze_openai.chat()`) has no preflight surface; a bad
`OPENAI_API_KEY` will surface as a `ProviderError` from `chat()` itself
on the first call (classified by `app/errors.py`).

### Rule 8: env var names follow the parent standard

`.env` uses `B2_REGION`, `B2_KEY_ID`, `B2_APPLICATION_KEY`,
`B2_BUCKET_NAME` — no aliases. Never use `B2_APP_KEY` (that's the
Genblaze env fallback; we pass explicit `app_key=` kwargs into
`S3StorageBackend.for_backblaze()`).

**Deliberate divergence from parent CLAUDE.md §3:** this sample has no
`B2_ENDPOINT`. `S3StorageBackend.for_backblaze()` derives the endpoint
from `region` internally, and carrying a dead env var would mislead
readers into thinking the sample consumes it. Parent rule predates the
Genblaze library handling endpoint derivation.

### Rule 9: docs change in the same PR as code

If you touch `pipelines.py`, update `docs/features/media-generation.md`
or `docs/features/composition.md` in the same change. If you touch
`composer.py`, update `docs/features/composition.md`. The structural
tests do not catch doc drift; reviewers do.

## Test commands

```bash
# Backend
cd services/api && uv run pytest tests/ -x

# Frontend type-check
cd apps/web && pnpm typecheck
```

Always run the structural tests before opening a PR; they're cheap.
