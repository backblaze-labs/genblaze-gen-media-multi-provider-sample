# Exec plan — provider switchboard

Turn the curated 5-provider explainer pipeline into an **interactive
provider switchboard**: the user picks a vendor + model per modality
(chat, image, video, tts, music) in the UI; pipeline factories resolve
provider instances from a registry. Covers every installed `genblaze-*`
provider and is extensible as new vendor packages land.

## Verified SDK facts driving the design

1. **`provider.list_models()` returns `[]` for every provider.** Providers
   validate model slugs against regex families (`^veo-`, `^imagen-`, …) at
   call time; they do NOT enumerate concrete model IDs. → dropdowns CANNOT
   be SDK-sourced. Each catalog entry carries a curated `default_model` +
   `suggested_models[]`, and the UI allows a **free-text override**.
2. **`provider.models.validate(slug)` is a cheap, no-API guard.** Used in the
   DTO validator to reject a malformed slug before any paid call.
3. Provider classes per modality (installed + the 7 newly-released packages):
   - image: `ImagenProvider`, `DalleProvider`, `DecartImageProvider`,
     `GMICloudImageProvider`, `NvidiaImageProvider`, **`ReplicateProvider`**
   - video: `VeoProvider`, `SoraProvider`, `DecartVideoProvider`,
     `GMICloudVideoProvider`, `NvidiaVideoProvider`, **`RunwayProvider`**,
     **`LumaProvider`**, **`ReplicateProvider`**
   - audio (tts + music): `OpenAITTSProvider`, `NvidiaAudioProvider`,
     `GMICloudAudioProvider`, **`ElevenLabsTTSProvider`**, **`LMNTProvider`**,
     **`HumeTTSProvider`**, **`ReplicateProvider`** (MusicGen-class music)
   - chat: `genblaze_openai.chat()` (a function, not a provider — stays
     special) + `NvidiaChatProvider`
   - NOT wired (don't fit the generate-media pipeline): `AssemblyAIProvider`
     (speech-TO-text — no transcription stage), `ElevenLabsSFXProvider`
     (sound effects — composer has no SFX track). Note both as future
     feature hooks, not switchboard entries.

4. **Image handoff is overwhelmingly `external_inputs`.** Runway
   (`route_images(slots=("prompt_image",))`), Luma
   (`route_keyframes(frames=("frame0",))`), Replicate
   (`route_by_media_type`), and GMICloud Kling all take the keyframe as an
   `external_inputs` Asset. Only Decart's retired path used `image=`. So the
   catalog `image_handoff` field defaults to `"external_inputs"`; `"image_kwarg"`
   is the rare legacy case.

## Scope decisions (locked with the user)

- **Expose all** providers per modality. Untested vendor×modality combos
  fall into the EXISTING Stage B2 best-effort degradation (failed video →
  keyframe still; failed audio → silent/dropped) + a `notice`. Essential
  stages (chat, image B0/B1) still fail loud at `preflight=True`.
- **No cost guardrail.** Treated as a dev test harness.

## Prerequisite: coordinated version bump

All 7 new packages require `genblaze-core>=0.3.4` (assemblyai: `>=0.3.0`).
Current pins are core/s3 0.3.2, providers 0.3.0–0.3.1.

- `pyproject.toml`: add the new provider deps; the `genblaze-core>=0.3.2,<0.4`
  constraint already admits 0.3.4 but bump existing providers to their 0.3.1
  releases for parity. New deps: `genblaze-replicate`, `genblaze-runway`,
  `genblaze-luma`, `genblaze-elevenlabs`, `genblaze-lmnt`, `genblaze-hume`.
  (Skip `genblaze-assemblyai` — no fit.)
- Re-pin `requirements.txt` via
  `uv export --format requirements-txt --no-hashes --output-file requirements.txt`.
- New config keys: `ELEVENLABS_API_KEY`, `LMNT_API_KEY`, `HUME_API_KEY`,
  `LUMAAI_API_KEY`, `REPLICATE_API_TOKEN`, Runway key (confirm
  `RUNWAYML_API_SECRET`). Add to `config.py` + `.env.example` + the
  `/health` and startup provider dicts.

## Backend changes

### 1. `app/repo/provider_catalog.py` (NEW — becomes the genblaze-provider import surface)

Registry keyed by `(Modality, vendor)`:

```python
@dataclass(frozen=True)
class CatalogEntry:
    vendor: str                       # "google" | "openai" | "nvidia" | "decart" | "gmicloud"
    modality: Modality
    make: Callable[[], BaseProvider]  # factory; reads the right settings.<x>_api_key
    env_key: str                      # settings attr gating availability
    default_model: str
    suggested_models: tuple[str, ...]
    # quirk hooks (default no-op) so pipelines.py stays branch-free:
    snap_duration: Callable[[float], float] | None  # Kling 5/10s grid
    audio_registry: Callable[[], ModelRegistry] | None  # MiniMax instrumental override
    image_handoff: Literal["image_kwarg", "external_inputs"]  # video providers only
    preflight: bool = True

CATALOG: dict[tuple[Modality, str], CatalogEntry] = { ... }

def entries_for(modality) -> list[CatalogEntry]
def resolve(modality, vendor) -> CatalogEntry      # raises on unknown
def key_available(entry) -> bool                   # bool(getattr(settings, entry.env_key))
def validate_model(entry, model) -> bool           # entry.make().models.validate(model)
```

The existing quirks move here as data:
- GMICloud video → `snap_duration` = nearest of (5,10), `image_handoff="external_inputs"`.
- Decart video → `image_handoff="image_kwarg"`, no snap.
- GMICloud music → `audio_registry=_instrumental_music_registry`.
- B2-only video/audio entries → `preflight=False`.

### 2. `app/repo/pipelines.py` (refactor)

- Delete `_imagen()`, `_resolve_video_provider()`, the hardcoded `tts`/`music`
  construction, and the standalone `snap_scene_durations` provider check.
- Each `build_*` factory takes the resolved `CatalogEntry` (or the
  `selection` dict) and calls `entry.make()`. Quirks read off the entry:
  `snap_scene_durations(spec, video_entry)`, image-handoff branch from
  `entry.image_handoff`, music registry from `entry.audio_registry`.
- Stage A chat stays a narrow resolver: `vendor=="openai"` → `chat()`;
  `vendor=="nvidia"` → `NvidiaChatProvider` one-shot. (Chat is the one
  modality that can't ride `Pipeline.step()` uniformly.)
- `genblaze_*` provider imports move OUT of pipelines.py and INTO
  provider_catalog.py. pipelines.py imports the catalog + `genblaze_core`
  types only.

### 3. `app/config.py`

Keep all five `*_api_key` fields. Per-modality default-model fields stay as
fallbacks (catalog `default_model` references them). Add nothing new unless
a vendor needs a key not already present.

### 4. `app/types/api.py`

```python
class ProviderChoice(BaseModel):
    vendor: str
    model: str | None = None   # None → entry.default_model

class Selection(BaseModel):
    chat: ProviderChoice  = default openai/gpt-4.1-nano
    image: ProviderChoice = default google/imagen-4.0-generate-001
    video: ProviderChoice = default gmicloud/Kling-Image2Video-V2.1-Master
    tts: ProviderChoice   = default nvidia/nvidia/magpie-tts-multilingual
    music: ProviderChoice = default gmicloud/minimax-music-2.5

class MediaRequest(BaseModel):
    prompt: str = Field(min_length=4, max_length=2000)
    spec: StoryboardSpec | None = None
    selection: Selection = Field(default_factory=Selection)  # defaults == today
```

A `model_validator` checks each choice against the catalog (`resolve` +
`validate_model`) so a bad vendor/slug 422s before any paid call. Defaults
reproduce today's pipeline EXACTLY (regression-safe).

### 5. `app/main.py`

- New `GET /providers` (sync `def` — constructs providers, no I/O): returns
  `{modality: [{vendor, default_model, suggested_models, key_available}]}`.
  Add to the `must_be_sync` structural set.
- `stream_media` threads `req.selection` into the `build_*` factories and
  the duration-snap call.
- `startup` + `/health` provider dicts: leave as-is (already per-vendor).

## Frontend changes

- `lib/models.ts` → replace static map with a `/providers`-fed catalog
  (fetch once, cache in a query). `lookupModel` resolves label/provider from
  the live matrix; unknown slug falls back to raw id (keep this behavior).
- New per-modality selector component (vendor `<Select>` + model `<Select>`
  with free-text/Combobox override; vendors with `key_available=false`
  disabled). Mount in the studio sidebar.
- `lib/api-client.ts` + SSE start: add `selection` to the media POST body.
- `types/pipeline.ts`: add `Selection` / `ProviderChoice` types.

## Tests

- `test_structure.py`:
  - extend `test_pipelines_is_the_only_genblaze_provider_consumer` →
    allow provider imports in `provider_catalog.py` too (rename to
    `test_genblaze_provider_imports_confined`).
  - add `provider_catalog.py` line budget; bump pipelines.py budget if needed.
  - add `get_providers` to `must_be_sync`.
- `test_provider_catalog.py` (NEW, offline): every entry `make()` constructs,
  reports the declared modality, and `validate_model(entry, entry.default_model)`
  passes. This is the "conformance" sweep — no network.
- `test_request_dtos.py`: extend for `Selection` defaults + bad-vendor /
  bad-model 422.
- `test_providers_endpoint.py` (NEW): `/providers` shape + key_available logic.
- `test_pipelines_smoke.py`: parametrize across a couple of selections.

## Docs (same PR)

- `AGENTS.md` Rule 3 → genblaze provider imports live in
  `provider_catalog.py` (pipelines.py consumes it). Rule 4 → "adding a
  provider = one `CatalogEntry`" (replaces "one `.step()`").
- `ARCHITECTURE.md` provider table + layer diagram (new catalog module).
- `docs/features/media-generation.md` provider table → the full matrix;
  document the curated-default + free-text-override model story and the
  quirk-as-data move.
- `.env.example`: no change (all five keys already documented).

## Risks / notes

- **Quirk god-object risk:** keep quirks as small per-entry hooks, NOT a
  plugin framework. Don't abstract for vendors that don't exist yet.
- **Chat asymmetry:** chat stays a separate narrow resolver — do not force
  it into the uniform catalog path.
- **Untested combos WILL fail at runtime** for some users; that's accepted
  (degrade + notice). Essential stages still fail loud for $0 via preflight.
- **Regression safety:** default `Selection` must reproduce today's run
  byte-for-byte in wiring; `test_pipelines_smoke` guards the default path.
</content>
</invoke>
