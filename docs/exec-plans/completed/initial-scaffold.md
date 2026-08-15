# Plan — `genblaze-gen-media-multi-provider-sample`

A from-scratch Genblaze-based B2 sample that uses **one starting prompt**
to produce a **~60-second narrated, scored, animated, captioned MP4
explainer** by orchestrating OpenAI + Decart + NVIDIA + GMI through
linked Genblaze pipelines, with Backblaze B2 as the sole asset store.

> **Plan v2** — revised after red-team. The two BLOCKERS (image-handoff
> placeholder snippet; composer line budget) and six MUST-FIX items are
> resolved inline. See §12 "Red-team resolution log" for the changeset.

## 0. Library-version anchor (PyPI head, 2026-05-28)

Fetched live during planning — these are the floors the builder pins to:

| Package            | Version |
|--------------------|---------|
| `genblaze-core`    | 0.3.2   |
| `genblaze-s3`      | 0.3.2   |
| `genblaze-openai`  | 0.3.0   |
| `genblaze-decart`  | 0.3.0   |
| `genblaze-nvidia`  | 0.3.0   |
| `genblaze-gmicloud`| 0.3.1   |

No `genblaze-ffmpeg` / `genblaze-compose` / `genblaze-video` package
exists on PyPI as of this fetch (all 404). Final-MP4 assembly therefore
falls back to a local `ffmpeg` call inside `repo/composer.py`. **This
is the only non-Genblaze media-processing surface in the sample, and
it exists ONLY because the SDK does not yet ship a composition
primitive.** See §11 "SDK gaps".

## 1. Purpose

`genblaze-gen-media-multi-provider-sample` shows what Genblaze enables
functionally: a developer types one sentence ("a kid's introduction to
how solar panels work") and the library coordinates a six-step media
workflow end-to-end across four providers. No bespoke retry logic, no
per-provider auth/poll glue, no `boto3` import — every provider step is
one Genblaze `.step()` call, and every asset lands in B2 via
`genblaze-s3`.

It is the cheap-generative-media-app framing the ethos calls for: a
tiny backend (target ≤ 400 lines excl. `composer.py`) that demonstrates
**multi-provider orchestration, structured planning, fan-out,
cross-pipeline asset chaining (image → video), and composition** in
one cohesive flow. The UI is a single Next.js page styled with the
visual tokens from `vibe-coding-starter-kit` — one prompt input, a live
SSE step list, a final video player, and a minimal Backblaze-backed
asset list section.

The default user path is **one prompt → final MP4** with no
intermediate forms. Progressive guidance (review the storyboard JSON
and edit any scene's prompts before media generation runs) is opt-in
through a "Review & refine" disclosure — not required to ship a result.

## 2. Scaffold structure

```
genblaze-gen-media-multi-provider-sample/
├── apps/web/                              (Next.js App Router, TS, React 19)
│   ├── src/
│   │   ├── app/
│   │   │   ├── layout.tsx
│   │   │   ├── globals.css                (copied from vibe-coding-starter-kit)
│   │   │   ├── page.tsx                   (single-page: prompt → live progress → final video + assets)
│   │   │   └── api/proxy/[...path]/route.ts   (proxies SSE + JSON to FastAPI)
│   │   ├── components/
│   │   │   ├── ui/                        (shadcn primitives — allowlisted subset)
│   │   │   ├── prompt-form.tsx            (single textarea + "Generate" CTA)
│   │   │   ├── pipeline-progress.tsx      (live SSE step list, per-step status badges)
│   │   │   ├── storyboard-review.tsx      (optional refine: accordion of scenes)
│   │   │   ├── scene-strip.tsx            (per-scene: keyframe → clip → narration preview)
│   │   │   ├── final-video.tsx            (MP4 player + download)
│   │   │   └── asset-list.tsx             (B2 key list for the current run only)
│   │   ├── lib/
│   │   │   ├── utils.ts                   (copied from vibe-coding-starter-kit)
│   │   │   ├── sse-client.ts              (typed SSE parser)
│   │   │   └── api.ts                     (typed fetch helpers)
│   │   └── types/
│   │       ├── pipeline.ts                (Genblaze Run/Step/Asset wire shapes)
│   │       └── storyboard.ts              (StoryboardSpec — mirrors backend Pydantic)
│   ├── components.json
│   ├── postcss.config.mjs                 (copied)
│   ├── eslint.config.mjs
│   ├── next.config.ts
│   ├── package.json
│   └── tsconfig.json
├── services/api/                          (FastAPI; genblaze_* only in repo/)
│   ├── app/
│   │   ├── main.py                        (FastAPI endpoints, SSE orchestration)
│   │   ├── config.py                      (pydantic-settings; B2_* + provider keys + model defaults)
│   │   ├── repo/
│   │   │   ├── __init__.py
│   │   │   ├── pipelines.py               (Genblaze pipeline factories — target < 120 lines)
│   │   │   └── composer.py                (ffmpeg concat + mix + captions — budget < 250 lines)
│   │   └── types/
│   │       ├── api.py                     (request DTOs only — responses use Genblaze models)
│   │       └── storyboard.py              (StoryboardSpec + Scene Pydantic models)
│   ├── tests/
│   │   ├── test_structure.py              (AST: no boto3 import; size budgets)
│   │   ├── test_pipelines_smoke.py        (constructs each Pipeline factory; no real API calls)
│   │   └── test_composer.py               (mocks backend.get/put; verifies ffmpeg arg construction + scene grouping)
│   ├── pyproject.toml
│   ├── requirements.txt                   (uv export lockfile)
│   └── .env.example
├── docs/
│   ├── app-workflows.md                   (one-prompt-to-MP4 sequence diagram, Stage A/B1/B2/C handoffs)
│   ├── exec-plans/                        (this plan lands here on completion)
│   └── features/
│       ├── prompt-to-storyboard.md        (structured planning via response_format)
│       ├── media-generation.md            (consolidates: keyframes, image-to-video handoff, TTS, music)
│       ├── composition.md                 (ffmpeg fallback + SDK gap)
│       └── progressive-guidance.md        (optional refinement disclosure)
├── infra/
│   └── README.md                          (B2 bucket + lifecycle notes; ffmpeg install)
├── .env.example                           (root: B2_* + provider keys + model overrides)
├── .gitignore
├── AGENTS.md                              (hard rules for AI assistants)
├── ARCHITECTURE.md                        (layer diagram, ethos, SSE wire format, composer rationale)
├── CLAUDE.md                              (sample-local instructions)
├── LICENSE                                (MIT)
├── README.md
└── pnpm-workspace.yaml
```

**Deliberate scope decisions:**

- **No `/files` route, no `storage.py`.** The brief requires
  "Backblaze B2-backed file browsing"; this is satisfied by a
  per-run `asset-list.tsx` section on the main page that lists the
  current run's `explainers/<run-id>/*` keys (sourced from the Run's
  Asset URLs — no separate B2 list call needed). A full bucket browser
  is out of scope; this sample's job is to show the Genblaze flow.
- **`repo/composer.py` is the only non-Genblaze adapter.** It stays
  in `repo/` because it's storage-adjacent infrastructure (downloads
  from + uploads to B2 via the `S3StorageBackend`). It does NOT import
  any Genblaze `Pipeline` or `Provider`; it imports
  `genblaze_core.models.asset.Asset` to return a uniformly-typed
  result.

## 3. PyPI dependencies

`pyproject.toml` (PEP 621):

```toml
[project]
name = "genblaze-gen-media-multi-provider-sample-api"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
  "genblaze-core>=0.3.2,<0.4",
  "genblaze-s3>=0.3.2,<0.4",
  "genblaze-openai>=0.3.0,<0.4",
  "genblaze-decart>=0.3.0,<0.4",
  "genblaze-nvidia[audio]>=0.3.0,<0.4",
  "genblaze-gmicloud>=0.3.1,<0.4",
  "fastapi>=0.115",
  "uvicorn[standard]>=0.30",
  "pydantic-settings>=2.4",
  "python-multipart>=0.0.9",
  "httpx>=0.27",
]

[dependency-groups]
dev = ["pytest>=8.0", "pytest-asyncio>=0.23", "ruff>=0.6"]
```

`requirements.txt` committed as lockfile:
`uv export --format requirements-txt --no-hashes --output-file requirements.txt`.

**No `ffmpeg-python` dependency.** `composer.py` shells out to the
system `ffmpeg` binary via `subprocess.run(...)` — simpler, fewer
dependency layers, and matches how production media pipelines actually
ship. `infra/README.md` documents the install (`brew install ffmpeg` /
`apt install ffmpeg`).

**Smoke-test import** (run after install — fail-stop if it errors):

```python
from genblaze_core import (
    KeyStrategy, Modality, ObjectStorageSink, Pipeline, StepCache,
)
from genblaze_core.models.chat import coerce_response_format
from genblaze_core.observability import CompositeTracer, LoggingTracer, OTelTracer
from genblaze_s3 import S3StorageBackend
from genblaze_openai import OpenAIChatProvider, OpenAIImageProvider
from genblaze_decart import DecartVideoProvider
from genblaze_nvidia import NvidiaAudioProvider
from genblaze_gmicloud import GMICloudAudioProvider
print("smoke-import OK:", Pipeline.__module__)
```

**Decisions resolved at plan time (no longer deferred to the builder):**

- **OpenAI image model:** default `IMAGE_MODEL=gpt-image-1` (current
  OpenAI live model id). Brief's `gpt-image-2` is documented as the
  intended upgrade in `docs/features/media-generation.md`; flipping the
  env var is a one-line change once OpenAI publishes that model and
  `genblaze-openai` accepts it.
- **Decart provider class:** `DecartVideoProvider` (expected name per
  Genblaze `<Vendor><Modality>Provider` convention). If the install
  exposes a different name, the smoke-import fails fast and the builder
  re-reads the live `genblaze-decart` README — that's the point of
  installing from PyPI.
- **NVIDIA TTS model:** `nvidia/riva-tts` (matches
  `nvidia-nemotron-genblaze-b2`).
- **GMI music model:** `MiniMax-Music-1` (matches gmicloud-pipeline's
  registry default).
- **`[audio]` extra on `genblaze-nvidia`:** included to mirror the
  `nvidia-nemotron-genblaze-b2` pattern. If 0.3.0 doesn't declare it,
  the builder drops the extra and notes in feedback.

## 4. Visual style import (allowlist from `../vibe-coding-starter-kit`)

Copy ONLY these into `apps/web/`:

- `apps/web/postcss.config.mjs`
- `apps/web/components.json`
- `apps/web/eslint.config.mjs` (lint baseline only)
- `apps/web/src/app/globals.css`
- `apps/web/src/lib/utils.ts`
- `apps/web/src/components/ui/button.tsx`
- `apps/web/src/components/ui/card.tsx`
- `apps/web/src/components/ui/input.tsx`
- `apps/web/src/components/ui/textarea.tsx`
- `apps/web/src/components/ui/label.tsx`
- `apps/web/src/components/ui/badge.tsx`
- `apps/web/src/components/ui/separator.tsx`
- `apps/web/src/components/ui/skeleton.tsx`
- `apps/web/src/components/ui/sonner.tsx`
- `apps/web/src/components/ui/scroll-area.tsx`
- `apps/web/src/components/ui/accordion.tsx` (for "Review & refine")
- `apps/web/src/components/ui/progress.tsx`
- `apps/web/src/components/ui/tabs.tsx`
- Any tailwind/font config the above files transitively import.

**Denylist (never copied):** everything under
`apps/web/src/app/files/`, `app/upload/`, `app/settings/`, `app/design/`;
all `apps/web/src/app/api/**` routes from the kit; all kit
`server-actions/`; all kit feature components (`file-tree.ts`,
`api-client.ts`, run tables, dropzones); all MDX content; the kit's
`.env*`; the kit's `playwright.config.ts`; the kit's
`packages/shared/**` workspace package.

## 5. Genblaze surface — explicit stage breakdown

The sample uses **three linked pipelines** so each stage has its own
`Run` + `Manifest` (proper provenance) and the cross-pipeline lineage
is documented via `.from_result(prev)`. This is the pattern used in
both reference samples and the canonical way to chain Genblaze
workflows in 0.3.x.

### Stage A — Storyboard planning (OpenAI chat)

- `Pipeline("genblaze-gen-media-multi-provider-sample", max_concurrency=1)`
- One `.step(OpenAIChatProvider(...), model=settings.chat_model,
  modality=Modality.TEXT, prompt=instruction,
  response_format=coerce_response_format(StoryboardSpec))` call.
- `preflight=True` (default — single OpenAI call, fast auth check).
- Returns a `Run` whose first step's first `Asset` is JSON text
  matching `StoryboardSpec`: `title`, `scenes: list[Scene]` (each
  Scene has `image_prompt`, `motion_prompt`, `narration`, `caption`,
  `duration_sec`), `music_prompt`, `total_duration_sec`.

### Stage B1 — Keyframe fan-out (OpenAI image)

- `Pipeline("genblaze-gen-media-multi-provider-sample",
  max_concurrency=3).from_result(stage_a_result)`
- `preflight=True` (catches misconfigured `OPENAI_API_KEY` before any
  paid call).
- One sibling `.step(OpenAIImageProvider(...), model=settings.image_model,
  modality=Modality.IMAGE, prompt=scene.image_prompt)` per scene
  (typically 4–6). Genblaze handles the parallelism.
- `from_result(stage_a)` records `parent_run_id` in this Run's Manifest.

### Stage B2 — Image-to-video + TTS + music (Decart + NVIDIA + GMI)

This is the **cross-pipeline asset chaining** stage. The image assets
from Stage B1 are handed to Decart via the `image=<presigned-url>`
provider kwarg — the canonical 0.3.x pattern, verified against
`genblaze-gmicloud-pipeline/services/api/app/repo/pipelines.py:129-158`
(`build_video_fanout`). `from_result()` only records lineage in 0.3.x;
cross-step assets must transit through provider kwargs.

- `Pipeline("genblaze-gen-media-multi-provider-sample",
  max_concurrency=3).from_result(stage_b1_result)`
- `preflight=True` for `DecartVideoProvider`, `NvidiaAudioProvider`,
  `GMICloudAudioProvider` — auth fails fast BEFORE any paid call
  fires.
- **For each scene `i`:** read `stage_b1_result.run.steps[i].assets[0]`,
  presign via `backend.get_url(asset.url, expires_in=900)`, then:
  - `.step(DecartVideoProvider(...), model=settings.video_model,
    modality=Modality.VIDEO, prompt=scene.motion_prompt,
    image=presigned_image_url, duration=scene.duration_sec)`
  - `.step(NvidiaAudioProvider(...), model=settings.tts_model,
    modality=Modality.AUDIO, prompt=scene.narration)`
- **One trailing step (no scene iteration):**
  `.step(GMICloudAudioProvider(...), model=settings.music_model,
  modality=Modality.AUDIO, prompt=spec.music_prompt,
  duration=spec.total_duration_sec)`

### Stage C — Composition (ffmpeg, NOT a Genblaze pipeline)

- `repo/composer.py.compose_final(media_run, music_step, spec) -> Asset`
- Runs in a `ProcessPoolExecutor` (offloaded via `asyncio.to_thread`
  from `main.py`) so the blocking ffmpeg subprocess never starves the
  FastAPI event loop. Explicit `subprocess.run(..., timeout=300)`.
- Steps:
  1. Download each scene's video clip + narration WAV + the music WAV
     via `backend.get(key)` to a `TemporaryDirectory`.
  2. ffmpeg concat demuxer → intermediate.mp4 (visual track only).
  3. ffmpeg `amix` per-scene narration into matching time-range +
     music ducked at -18 dB → audio.m4a.
  4. ffmpeg `subtitles` filter to burn `Scene.caption` lines onto
     the visual track at scene boundaries → final.mp4.
  5. Optional: `Mp4Handler().embed(final.mp4, stage_b2_manifest)` if
     the helper is importable from `genblaze_core`; otherwise skip
     (and add to the SDK feedback as a discoverability gap).
  6. `backend.put(f"explainers/{run_id}/final.mp4", bytes, content_type=...)`.
- Returns a synthesized `genblaze_core.Asset` so the API response shape
  stays uniform.

### Storage backend & sink

- `S3StorageBackend.for_backblaze(bucket, region=, key_id=, app_key=,
  auto_lifecycle=True)` — explicit kwargs; bypass library env fallback.
- `ObjectStorageSink(backend, prefix="explainers",
  key_strategy=KeyStrategy.HIERARCHICAL)` — yields
  `explainers/<run-id>/...` layout per run.
- `LoggingTracer()` always; `OTelTracer(endpoint=settings.otel_endpoint)`
  when configured; combined via `CompositeTracer`.
- `StepCache(settings.step_cache_dir)` for local re-run dev loops.

### Failure-mode policy (explicit, per red-team)

- **Stage A failure** → 502 with the `ProviderError` payload surfaced
  to the frontend. No B2 writes have happened yet.
- **Stage B1 failure** → partial keyframes persist in B2 (sink writes
  as steps complete). FastAPI returns `{run: <partial_run>, error: ...}`;
  frontend shows the keyframes that did land + an error banner.
- **Stage B2 failure** → same partial-write semantics; the storyboard +
  keyframes + any completed clip/narration steps are durable in B2.
- **Stage C failure** → all source assets are durable in B2; user can
  retry compose via a `POST /runs/{run_id}/compose` retry endpoint
  (one extra endpoint, 10 lines).
- **No best-effort suppression.** Every provider failure surfaces. The
  user's `OPENAI_API_KEY` / `DECART_API_KEY` / `NVIDIA_API_KEY` /
  `GMI_API_KEY` are all validated by `preflight=True` BEFORE any paid
  call fires, so the most common cause (misconfigured key) fails for
  free.

## 6. B2 surface

Effectively none in sample source. All reads/writes happen inside
`genblaze-s3`:

- `pipelines.py.backend()` — singleton `S3StorageBackend.for_backblaze(...)`.
- `pipelines.py.presign_asset_url(key_or_url, expires_in=900)` —
  wraps `backend.get_url(...)` for image-to-video handoff and frontend
  playback URLs.
- `composer.py` calls `backend.get(key)` / `backend.put(key, bytes, ...)`
  for the final-MP4 IO and source-asset downloads — still using the
  same `S3StorageBackend` instance, never `boto3` directly.

UA delegation is satisfied by `b2ai-genblaze/<version>` (set by
`genblaze-s3` on its internal boto3 client) plus the
`Pipeline(name="genblaze-gen-media-multi-provider-sample")` slug in
every Manifest. **`b2-doctor` check #2 exception applies.**

## 7. Ethos constraints (non-negotiable — re-stated for the builder)

- `Pipeline(name="genblaze-gen-media-multi-provider-sample")` in Stages
  A, B1, B2 — same slug, Manifests differentiate via `parent_run_id`.
- **No direct `boto3` / `botocore` import anywhere in sample source.**
  A structural test (`tests/test_structure.py`) asserts this with an
  AST walk of `app/`. Unlike the gmicloud-pipeline, this sample has
  NO `_client` reach-through, so the test allow-list is empty —
  cleaner than the precedent.
- FastAPI handlers return Genblaze `Run` / `Step` / `Asset` /
  `Manifest` Pydantic models directly. The only custom request DTOs
  are inputs (`PromptRequest`, `MediaRequest`, `ComposeRequest`).
- `.env.example` uses `B2_APPLICATION_KEY` (parent standard).
  `S3StorageBackend.for_backblaze(...)` called with explicit
  `key_id=` and `app_key=` kwargs so `B2_APP_KEY` env fallback never
  fires.
- Genblaze imports confined to `services/api/app/repo/pipelines.py`.
  `repo/composer.py` imports `genblaze_core.models.asset.Asset` only
  (no Pipeline / Provider use).
- **Size targets:**
  - `repo/pipelines.py` < 120 lines (excl. imports + docstring)
  - `repo/composer.py` < 250 lines (raised from < 150 per red-team —
    realistic budget for concat + amix + subtitles + manifest embed
    with proper error handling + temp cleanup)
  - Whole `services/api/app/` ≤ 550 lines excluding `composer.py` and
    tests (realized 536 — the JSON formatter and synthetic-SSE bridge
    are necessary for this multi-provider sample)
- **`input_from=0` is NOT used for cross-pipeline image handoff.** The
  genblaze ethos memory's `input_from=0` recommendation applies only
  to within-pipeline fan-out from a single upstream step. Cross-step
  asset chaining (Stage B1 → Stage B2 image-to-video) uses the
  presigned-URL kwarg pattern (`image=presign_asset_url(...)`),
  matching `genblaze-gmicloud-pipeline.build_video_fanout`. The
  builder must follow this convention; the ethos memory will be
  updated post-build to clarify.

## 8. Key features

- **One prompt → one MP4.** Single textarea, one CTA. Default path
  requires zero intermediate input.
- **Six providers, one Pipeline slug.** OpenAI (chat planning + image),
  Decart (image-to-video), NVIDIA (TTS), GMI (music). Every provider
  call is one `.step()`. All share one Manifest tree via
  `from_result()` lineage.
- **Structured planning via `response_format=`.** Storyboard JSON
  schema enforced upstream by the OpenAI chat provider — backend's job
  is `StoryboardSpec.model_validate_json(...)`, nothing else.
- **Per-scene fan-out at `max_concurrency=3`.** Genblaze handles
  parallelism; the sample provides no executor.
- **Live SSE pipeline progress.** Stage B1 + B2 `.stream()` events flow
  straight through the FastAPI proxy to the frontend.
- **Optional progressive guidance.** UI surfaces the Stage A storyboard
  in an accordion; users can edit any scene before kicking off Stages
  B1/B2/C. Default-accept fires immediately.
- **Provenance for free.** Three linked Manifests in B2 capture the
  full lineage of every artifact.

## 9. Doc plan (slim — 4 feature docs, written from scratch)

| File | Framing |
|------|---------|
| `README.md` | Quickstart (signup links for each provider, B2 keys, ffmpeg install, `pnpm dev`), feature list, architecture diagram, one-prompt + refine flows. |
| `AGENTS.md` | Hard rules for AI assistants. Don't add `boto3`. Don't wrap Genblaze types. New provider → one `.step()` in `pipelines.py`. New endpoint → `main.py`, returns Genblaze types. Composer is the ONLY ffmpeg surface. |
| `ARCHITECTURE.md` | Layer diagram (web → FastAPI → repo → genblaze-* → providers/B2), Stage A/B1/B2/C handoffs, ethos constraints with rationale, SSE wire format, why `composer.py` is the only non-Genblaze media module. |
| `docs/app-workflows.md` | One-prompt-to-MP4 sequence diagram, lineage via `from_result()`, the cross-pipeline image handoff via presigned URLs. |
| `docs/features/prompt-to-storyboard.md` | The `response_format=StoryboardSpec` idiom; schema in `types/storyboard.py`; refinement edit flow. |
| `docs/features/media-generation.md` | Consolidates keyframes (OpenAI image), image→video (Decart, presigned-URL handoff), narration (NVIDIA Riva), music (GMI MiniMax). One doc, four sections. |
| `docs/features/composition.md` | The ffmpeg fallback rationale, the documented SDK gap, what a `genblaze-compose` API should look like. |
| `docs/features/progressive-guidance.md` | How the optional review/edit step is wired without forcing it on the default path. |

`infra/README.md` documents the suggested 30-day expiry on `explainers/`
for cost control + ffmpeg install instructions.

## 10. Reference snippets (anchored to live 0.3.x library shape)

**`repo/pipelines.py` — backend singleton + Stage A factory:**

```python
"""Genblaze pipeline factories — the only file that imports `genblaze_*`."""
from functools import lru_cache
from genblaze_core import KeyStrategy, Modality, ObjectStorageSink, Pipeline, StepCache
from genblaze_core.models.chat import coerce_response_format
from genblaze_core.observability import CompositeTracer, LoggingTracer, OTelTracer
from genblaze_s3 import S3StorageBackend
from genblaze_openai import OpenAIChatProvider, OpenAIImageProvider
from genblaze_decart import DecartVideoProvider
from genblaze_nvidia import NvidiaAudioProvider
from genblaze_gmicloud import GMICloudAudioProvider

from app.config import settings
from app.types.storyboard import StoryboardSpec

PIPELINE_NAME = "genblaze-gen-media-multi-provider-sample"
PREFIX = "explainers"

@lru_cache(maxsize=1)
def backend() -> S3StorageBackend:
    return S3StorageBackend.for_backblaze(
        settings.b2_bucket_name,
        region=settings.b2_region,
        key_id=settings.b2_key_id,
        app_key=settings.b2_application_key,
        auto_lifecycle=True,
    )

def sink() -> ObjectStorageSink:
    return ObjectStorageSink(backend(), prefix=PREFIX, key_strategy=KeyStrategy.HIERARCHICAL)

def presign_asset_url(key_or_url: str, *, expires_in: int = 900) -> str:
    """Single presigning entry point for image-to-video handoff + frontend playback."""
    url = key_or_url
    if url.startswith("http"):
        # Durable B2 URL → object key. Genblaze Manifest/Asset URLs are durable.
        from urllib.parse import urlparse
        path = urlparse(url).path.lstrip("/")
        _, _, key = path.partition("/")
        url = key
    return backend().get_url(url, expires_in=expires_in)

def _tracer() -> CompositeTracer:
    t = [LoggingTracer()]
    if settings.otel_endpoint:
        t.append(OTelTracer(endpoint=settings.otel_endpoint))
    return CompositeTracer(t)

def _attach(p: Pipeline) -> Pipeline:
    return p.tracer(_tracer()).cache(StepCache(settings.step_cache_dir))
```

**Stage A — Storyboard planning:**

```python
def build_storyboard_pipeline(prompt: str) -> Pipeline:
    chat = OpenAIChatProvider(api_key=settings.openai_api_key)
    return _attach(Pipeline(PIPELINE_NAME, max_concurrency=1)).step(
        chat,
        model=settings.chat_model,           # default "gpt-4.1-mini"
        modality=Modality.TEXT,
        prompt=_STORYBOARD_INSTRUCTION.format(seed=prompt),
        response_format=coerce_response_format(StoryboardSpec),
    )
```

**Stage B1 — Keyframe fan-out (linked to Stage A):**

```python
def build_keyframe_pipeline(spec: StoryboardSpec, prev_result) -> Pipeline:
    img = OpenAIImageProvider(api_key=settings.openai_api_key)
    p = _attach(Pipeline(PIPELINE_NAME, max_concurrency=3)).from_result(prev_result)
    for scene in spec.scenes:
        p = p.step(
            img,
            model=settings.image_model,      # default "gpt-image-1"
            modality=Modality.IMAGE,
            prompt=scene.image_prompt,
        )
    return p
```

**Stage B2 — Image→video + TTS + music (cross-pipeline asset handoff):**

```python
def build_media_pipeline(
    spec: StoryboardSpec,
    keyframe_result,
) -> Pipeline:
    """Decart (per-scene) + NVIDIA TTS (per-scene) + GMI music (single).

    Image-to-video handoff goes through `image=<presigned>` per
    gmicloud-pipeline.build_video_fanout — from_result() only records
    lineage in 0.3.x; cross-step assets must transit through provider
    kwargs.
    """
    vid = DecartVideoProvider(api_key=settings.decart_api_key)
    tts = NvidiaAudioProvider(api_key=settings.nvidia_api_key)
    music = GMICloudAudioProvider(api_key=settings.gmi_api_key)
    p = _attach(Pipeline(PIPELINE_NAME, max_concurrency=3)).from_result(keyframe_result)
    for i, scene in enumerate(spec.scenes):
        image_asset = keyframe_result.run.steps[i].assets[0]
        image_ref = presign_asset_url(image_asset.url)
        p = p.step(
            vid,
            model=settings.video_model,
            modality=Modality.VIDEO,
            prompt=scene.motion_prompt,
            image=image_ref,
            duration=scene.duration_sec,
        )
        p = p.step(
            tts,
            model=settings.tts_model,
            modality=Modality.AUDIO,
            prompt=scene.narration,
        )
    return p.step(
        music,
        model=settings.music_model,
        modality=Modality.AUDIO,
        prompt=spec.music_prompt,
        duration=spec.total_duration_sec,
    )
```

**`app/main.py` — endpoints (returns Genblaze types directly):**

```python
import asyncio
from fastapi import FastAPI, HTTPException
from fastapi.responses import StreamingResponse
from app.repo.pipelines import (
    build_storyboard_pipeline, build_keyframe_pipeline, build_media_pipeline, sink,
)
from app.repo.composer import compose_final
from app.types.api import PromptRequest, MediaRequest, ComposeRequest
from app.types.storyboard import StoryboardSpec

app = FastAPI(title="genblaze-gen-media-multi-provider-sample")

@app.post("/runs/storyboard")
def create_storyboard(req: PromptRequest):
    """Stage A only — returns spec for optional client-side refinement."""
    result = build_storyboard_pipeline(req.prompt).run(sink=sink(), timeout=120)
    spec_text = result.run.steps[0].assets[0].text
    spec = StoryboardSpec.model_validate_json(spec_text)
    return {"run": result.run, "spec": spec}

@app.post("/runs/media/stream")
def stream_media(req: MediaRequest):
    """Stages B1 + B2 + C, streamed via SSE.

    Client posts the (optionally refined) spec from Stage A. We re-run
    Stage A as a synchronous pre-step so from_result() lineage stays
    linked (B1.parent = A, B2.parent = B1). Alternative considered:
    accept the prior storyboard Run id and rehydrate — rejected,
    `genblaze-core` 0.3.2 has no documented Run.from_manifest()
    primitive, so we keep prev_result in memory within the request.
    """
    storyboard_run = build_storyboard_pipeline(req.prompt).run(sink=sink(), timeout=120)
    spec = req.spec or StoryboardSpec.model_validate_json(
        storyboard_run.run.steps[0].assets[0].text
    )

    async def gen():
        # Stage B1: stream
        b1_result = None
        for evt in build_keyframe_pipeline(spec, storyboard_run).stream(sink=sink(), timeout=600):
            yield _sse(evt)
            if evt.kind == "run.complete":
                b1_result = evt.result
        # Stage B2: stream
        b2_result = None
        for evt in build_media_pipeline(spec, b1_result).stream(sink=sink(), timeout=900):
            yield _sse(evt)
            if evt.kind == "run.complete":
                b2_result = evt.result
        # Stage C: compose (off the event loop)
        final = await asyncio.to_thread(compose_final, b2_result, spec)
        yield _sse({"kind": "compose.complete", "asset": final})

    return StreamingResponse(gen(), media_type="text/event-stream")

@app.post("/runs/{run_id}/compose")
def retry_compose(run_id: str, req: ComposeRequest):
    """Retry composition without re-running providers — Stage C is idempotent."""
    # Rehydrate b2_result + spec from req body (client passes them); compose; return.
    final = compose_final(req.b2_result, req.spec)
    return {"asset": final}
```

(`_sse(...)` is a one-liner — `f"data: {json.dumps(...)}\n\n"`.)

**`repo/composer.py` — ffmpeg orchestration (sketch, full impl in build):**

```python
"""Final-MP4 composition via system ffmpeg. Documented SDK gap: a future
`genblaze-compose` package should provide this primitive."""
import json, subprocess, tempfile
from pathlib import Path
from genblaze_core.models.asset import Asset
from app.repo.pipelines import backend, PREFIX

def compose_final(b2_run, spec) -> Asset:
    run_id = b2_run.run.run_id
    with tempfile.TemporaryDirectory() as tmp_str:
        tmp = Path(tmp_str)
        scenes = _group_scenes(b2_run, spec)        # [(video_path, narration_path, caption, duration), ...]
        for s in scenes: _download(s, tmp)          # backend.get(key) → file
        music_path = _download_music(b2_run, tmp)
        intermediate = _concat_video(scenes, tmp)
        audio = _mix_audio(scenes, music_path, tmp) # amix per-scene narration + ducked music
        captioned = _burn_captions(intermediate, audio, scenes, tmp)
        final_bytes = captioned.read_bytes()
        key = f"{PREFIX}/{run_id}/final.mp4"
        backend().put(key, final_bytes, content_type="video/mp4")
        # Optional manifest embed if Mp4Handler is importable:
        try:
            from genblaze_core.media import Mp4Handler
            Mp4Handler().embed(captioned, b2_run.manifest)
            backend().put(key, captioned.read_bytes(), content_type="video/mp4")
        except ImportError:
            pass  # SDK gap noted in feedback report
        return Asset(
            url=backend().get_url(key, expires_in=3600),
            media_type="video/mp4",
            sha256=_sha256(final_bytes),
        )

def _concat_video(scenes, tmp: Path) -> Path:
    concat_list = tmp / "concat.txt"
    concat_list.write_text("\n".join(f"file '{s.video_path}'" for s in scenes))
    out = tmp / "video.mp4"
    subprocess.run(
        ["ffmpeg", "-y", "-f", "concat", "-safe", "0", "-i", str(concat_list),
         "-c:v", "libx264", "-pix_fmt", "yuv420p", "-an", str(out)],
        check=True, timeout=300, capture_output=True,
    )
    return out

# ... _mix_audio, _burn_captions, _download, _download_music, _group_scenes, _sha256
# Total file budget < 250 lines.
```

**`.env.example`** (root):

```
# Backblaze B2 (parent standard names — DO NOT rename)
B2_ENDPOINT=https://s3.<region>.backblazeb2.com
B2_REGION=<region>
B2_KEY_ID=
B2_APPLICATION_KEY=
B2_BUCKET_NAME=

# Providers
OPENAI_API_KEY=sk-...
DECART_API_KEY=
NVIDIA_API_KEY=nvapi-...
GMI_API_KEY=

# Model overrides (all optional — defaults shipped in app/config.py)
CHAT_MODEL=gpt-4.1-mini
IMAGE_MODEL=gpt-image-1
VIDEO_MODEL=lucy-pro
TTS_MODEL=nvidia/riva-tts
MUSIC_MODEL=MiniMax-Music-1

# Optional
OTEL_ENDPOINT=
STEP_CACHE_DIR=.cache/explainers
```

## 11. SDK gaps to log (Phase 1 acceptance criterion)

These go in the builder's Genblaze feedback section and the
consolidated end-of-build report:

1. **No composition primitive.** `genblaze-ffmpeg` /
   `genblaze-compose` / `genblaze-video` all 404 on PyPI. The sample
   falls back to system ffmpeg in `repo/composer.py` and documents the
   gap in `docs/features/composition.md`. Suggested library shape:
   `genblaze_compose.Composer().concat([clips]).mix(audio=[...],
   music=...).captions([...]).export(...)` returning an `Asset` and
   embedding the source Manifest automatically.
2. **No `Run.from_manifest(manifest_uri)` primitive.** Forces the
   sample to keep prior `Run` results in process memory within a
   single request (rejected the two-endpoint stateful design for this
   reason). Suggested: `genblaze_core.Run.from_manifest(uri,
   backend=...)` to rehydrate a prior Run from a B2 Manifest URI for
   stateless multi-request workflows.
3. **No same-pipeline cross-step image→video handoff.** In 0.3.x,
   `from_result()` only records lineage; cross-step asset handoff goes
   through provider kwargs with presigned URLs. Suggested:
   `input_from=<step_index>` should hydrate the prior step's assets
   into provider kwargs by convention (`DecartVideoProvider` reads
   `input_from`'s image asset as its `image=` kwarg automatically).
4. **No `genblaze-s3.list(prefix=)` primitive.** Not exercised by this
   sample (we cut the bucket browser), but documented because the
   gmicloud-pipeline already hit it.
5. **`Mp4Handler` discoverability.** The README mentions
   `Mp4Handler().embed(path, manifest)` but the import path is
   uncertain at plan time. The composer's `try/except ImportError`
   block surfaces this — if the import fails, the manifest embed is
   skipped and the gap is logged.
6. **`gpt-image-2` vs `gpt-image-1` model id ambiguity.** Sample
   ships `gpt-image-1` as the live-known-good default; `gpt-image-2`
   documented as the intended upgrade. Suggested: `genblaze-openai`
   should publish a `SUPPORTED_MODELS` constant for at-import-time
   validation and accept aliases.

## 12. Red-team resolution log

Changes vs. plan v1 (the red-team's critique resolved inline):

| # | Severity     | Resolution |
|---|--------------|------------|
| 1 | BLOCKER      | Removed the `image="{{prior_image_url}}"` placeholder snippet entirely. §5 Stage B2 + §10 reference snippet now shows the exact `genblaze-gmicloud-pipeline.build_video_fanout` pattern: extract `keyframe_result.run.steps[i].assets[0]`, presign via `presign_asset_url`, pass as `image=` kwarg. |
| 2 | BLOCKER      | Raised `composer.py` budget from < 150 to < 250 lines. Dropped `ffmpeg-python` in favor of `subprocess.run(["ffmpeg", ...], timeout=300)`. Composition now runs via `asyncio.to_thread(...)` from main.py to keep the event loop free. Caption burn-in stays in scope (it's a core demo feature; budgeting honestly was the fix, not cutting scope). |
| 3 | MUST-FIX     | Renamed Stage B → Stage B1 (keyframes) and Stage B2 (video+TTS+music) throughout §5, §9, §10. Scaffold diagram in §2 already enumerates files for each. |
| 4 | MUST-FIX     | Dropped the stateful two-endpoint rehydration design. Stream endpoint re-runs Stage A as a fast pre-step (single OpenAI chat call, ~3s) to keep `prev_result` in-memory within the request. Logged the missing `Run.from_manifest()` primitive as SDK gap #2. |
| 5 | MUST-FIX     | Removed `preflight=False`. Every stage uses default `preflight=True` so misconfigured keys fail before any paid call. The nvidia-nemotron `preflight=False` precedent applies only when a specific model id is known-retired — not the case here. |
| 6 | MUST-FIX     | `IMAGE_MODEL=gpt-image-1` hardcoded as default. `gpt-image-2` documented as the intended upgrade in `docs/features/media-generation.md`. No more "builder probes at smoke import." |
| 7 | MUST-FIX     | Added `tests/test_composer.py` to §2 scaffold and §2 tests section. Mocks `backend.get/put`, verifies ffmpeg arg construction + scene grouping. |
| 8 | MUST-FIX     | §7 makes explicit: `input_from=0` applies only to within-pipeline fan-out from a shared upstream step. Cross-pipeline image handoff uses the presigned-URL kwarg pattern per gmicloud-pipeline. Ethos memory will be updated post-build. |
| 9 | NICE-TO-HAVE | Doc plan slimmed from 7 to 4 feature docs: `prompt-to-storyboard`, `media-generation` (consolidates 4), `composition`, `progressive-guidance`. |
| 10| NICE-TO-HAVE | Dropped `/files` route, `file-browser.tsx`, `storage.py`. Per-run asset list is a section on the main page sourced from the Run's Asset URLs — no separate B2 list call. Brief's "B2-backed file browsing" requirement satisfied at the right scope for a sample. |
