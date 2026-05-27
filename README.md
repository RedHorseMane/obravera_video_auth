# ObraVera Video Authentication

**Version:** 0.1.0
**Status:** Active Development — Fingerprint Bench Phase
**Architecture:** CLI Benchmark Harness + Phased Path to Provenance Authentication

---

## Overview

Video provenance and authentication system, currently in its **fingerprint robustness benchmarking** phase. ObraVera Video evaluates how perceptual video fingerprints (ISCC video, vPDQ) survive the everyday transformations that platform pipelines, editing tools, and adversaries apply to video — re-encoding, scaling, frame-rate changes, clipping, blur, brightness shifts, noise, AI regeneration. The bench produces versioned JSON + Markdown reports comparing fingerprint robustness across a transform matrix.

The benchmark is the foundation. Once the survivability envelope is mapped objectively, ObraVera Video extends to a full multi-layer authentication stack analogous to its sibling [ObraVera Audio](https://github.com/RedHorseMane/obravera_audio_auth) — combining fingerprints with C2PA manifests, watermarking, registry-based recovery, and Ed25519-signed credentials.

## Key Features

### Multi-Algorithm Fingerprint Benchmark
- **ISCC Video (ISO 24138)** — International Standard Content Code via MP7 frame signatures. Produces a 64-bit raw hash from per-frame ternary signatures extracted by ffmpeg's built-in `signature` filter, fed through `iscc_core.soft_hash_video_v0`. Hamming distance is the comparison metric.
- **vPDQ (Meta)** — Video Perceptual Discriminator Quotient. Per-frame 256-bit PDQ hashes via Meta's `vpdq` library, stored as a packed `(quality, frame_number, hash)` sequence per clip. Distance is the average best-match per-frame Hamming.
- **Side-by-side comparison** — Every transform × clip × fingerprint cell is run through both algorithms so reports show comparative robustness, not just absolute pass/fail.

### Three-Tier Transform Matrix (31 transforms)

**Tier 1 — ffmpeg-based (16 transforms):** Pipeline-level transformations that platforms and editing tools routinely apply.
- Re-encoding: `h264_crf23`, `h264_crf28`, `h265`, `vp9`
- Resolution: `scale_720p`, `scale_480p`
- Frame rate: `fps_30_to_24`, `fps_30_to_60`
- Container/audio: `container_swap_mp4_to_mov`, `audio_strip`
- Letterbox: `crop_letterbox_add`, `crop_letterbox_remove` (invertible crop→pad pair)
- Clip extraction: `clip_first_10s`, `clip_middle_10s`, `clip_last_10s`
- Effects: `fade_in_out`

**Tier 2 — Python frame-level + ffmpeg complex filters (11 transforms):** Pixel-level manipulations applied via Pillow/numpy.
- Blur: `gaussian_blur_per_frame`
- Brightness: `brightness_shift_+15`, `brightness_shift_-15`
- Contrast: `contrast_shift_+20`
- Rotation: `mild_rotation_2deg`
- Aspect/layout: `aspect_pad_to_square`, `text_overlay_corner`
- Noise: `noise_injection_low` (σ=5), `noise_injection_high` (σ=15) — **deterministically seeded** so reports are reproducible
- Speed: `speed_1.1x`, `speed_0.9x`

**Tier 3 — ML-based regeneration attacks (4 stubs, GPU-required):** Currently registered as `Outcome.SKIPPED` cells in reports so the gap is named objectively rather than silently omitted.
- `sdxl_regen_per_frame` — SDXL img2img per-frame regeneration (the canonical "regen attack" that defeats most watermarks)
- `video_diffusion_regen` — Stable Video Diffusion full-clip resynthesis
- `neural_frame_interp` — RIFE/FILM neural frame interpolation
- `ai_upscale_4x` — Real-ESRGAN 4× upscaling

### Honest Failure Mode Taxonomy

Every bench cell resolves to one of:
- `match` — Distance ≤ threshold (algorithm recovered identity)
- `no_match` — Distance > threshold band
- `near_miss` — Distance inside the ±10% band around threshold (rounded outward to integers)
- `transform_error` — The transform itself broke (cell isolation prevents one bad transform from killing the run)
- `fingerprint_error` — The fingerprint extractor broke
- `skipped` — Tier 3 stub or explicitly excluded

The classifier is pure: given `(distance, threshold, near_miss_pct)` it returns the outcome deterministically. No hidden heuristics.

### Reproducible Reports
- **Dated JSON output** — `<date>-bench-report.json` with full schema (environment, corpus SHA-256s, thresholds with source citations, per-cell distance/outcome/timing)
- **Markdown summary** — Same data rendered as a human-readable matrix with per-transform breakdowns and an errors section
- **Versioned artifacts** — Reports live in `docs/testing/` and are committed alongside the code that produced them
- **Deterministic transforms** — Seeded noise injection means the same input video produces identical noise patterns across runs

### CLI Tool (`obravera-video-bench`)
- **`list`** — Enumerate fingerprints, transforms, and corpus clips
- **`validate`** — Verify environment (ffmpeg + library imports) and corpus integrity (SHA-256 against manifest)
- **`run`** — Execute the bench matrix; filter by `--fingerprints`, `--tiers`, `--clips`, `--trials`; `--smoke` mode for fast iteration

### Cell-Level Error Isolation
The bench runner caches the original fingerprint per `(clip, fingerprint)` so it's only computed once. Each subsequent step (transform application, transformed fingerprint, distance computation) catches exceptions independently — one broken cell never kills a multi-hour bench run.

## Quick Start

### Installation

```bash
# Installation options are still under development and will be released when the bench harness is finalised.
```

### Basic Usage

#### CLI — Environment Check

```bash
# Verify ffmpeg + fingerprint libraries are present, corpus SHA-256s match
obravera-video-bench validate
```

#### CLI — Inventory

```bash
# List registered fingerprints, transforms, and corpus clips
obravera-video-bench list
```

#### CLI — Smoke Run

```bash
# Single clip, Tier 1 only, fail-fast; produces a report in docs/testing/
obravera-video-bench run --smoke --output-dir docs/testing
```

#### CLI — Full Bench

```bash
# Full matrix: all clips × ISCC + vPDQ × Tier 1 + Tier 2 transforms
obravera-video-bench run --output-dir docs/testing

# Include Tier 3 ML stubs as SKIPPED cells in the report
obravera-video-bench run --tiers 1,2,3 --output-dir docs/testing
```

## CLI Reference

### Commands

#### `obravera-video-bench list`
List corpus clips, fingerprints, and transforms.

```bash
obravera-video-bench list

Output:
  - Registered fingerprints (iscc-video, vpdq)
  - All Tier 1 + Tier 2 + Tier 3 transform ids
  - Corpus clips (verified) and pending acquisitions
```

#### `obravera-video-bench validate`
Verify environment + corpus integrity. Exits 0 if ready, 1 otherwise.

```bash
obravera-video-bench validate

Checks:
  - ffmpeg on PATH
  - iscc-video extractor importable
  - vpdq library importable
  - All corpus clips have valid SHA-256 against manifest
  - Reports pending (unacquired) clips separately

Examples:
  # Standard environment + corpus check
  obravera-video-bench validate
```

#### `obravera-video-bench run`
Execute the bench matrix and write JSON + Markdown reports.

```bash
obravera-video-bench run [options]

Options:
  --fingerprints LIST   Comma-separated fingerprint ids (iscc, vpdq). Default: iscc,vpdq
  --tiers LIST          Comma-separated tier numbers (1, 2, 3). Default: 1,2
  --clips LIST          Comma-separated clip ids (empty = all clips)
  --trials N            Trials per cell (default: 1)
  --smoke               Smoke mode: 1 clip, Tier 1 only, trials=1, fail-fast
  --output-dir PATH     Where to write reports (default: docs/testing)
  --help                Show this message and exit

Examples:
  # Smoke run on first clip, Tier 1 only
  obravera-video-bench run --smoke

  # ISCC only, Tier 1 + 2, three trials per cell
  obravera-video-bench run --fingerprints iscc --tiers 1,2 --trials 3

  # Restrict to a single clip
  obravera-video-bench run --clips outdoor_motion

  # Include Tier 3 ML stubs (skipped, but visible in report)
  obravera-video-bench run --tiers 1,2,3
```

## Fingerprint Algorithms Explained

### ISCC Video (ISO 24138)
- **Standard**: International Standard Content Code, ISO 24138:2024
- **Pipeline**: ffmpeg `signature` filter produces MP7 frame signatures (380-dim ternary vectors per frame) → `iscc_core.soft_hash_video_v0` aggregates them into a 64-bit raw hash
- **Distance**: Hamming distance over the 64-bit hash
- **Robustness profile**: Designed for content identification across re-encoding and re-distribution. Stable across most platform transcodes.
- **Strength**: Standards-track, audit-friendly, low storage cost (8 bytes per clip)

### vPDQ (Meta Video PDQ)
- **Origin**: Meta's vPDQ — extension of the still-image PDQ algorithm to video
- **Pipeline**: `vpdq.computeHash` produces one 256-bit PDQ hash per sampled frame; stored as `(quality, frame_number, hash_bytes)` sequence
- **Distance**: For each frame in clip A, find the minimum Hamming distance to any frame in clip B; average across A. Simplified vs Meta's match-rate metric but sufficient for match/no_match/near_miss classification.
- **Robustness profile**: Survives heavy temporal edits (clip extraction, fps changes) because per-frame matching is order-independent.
- **Strength**: Empirically robust against the regen attacks that broke earlier image-PDQ deployments.

### Why Both?
The bench is a comparison harness. ISCC and vPDQ have different sensitivity profiles — ISCC tracks global content identity, vPDQ tracks per-frame perceptual stability. By running them side-by-side across the full transform matrix, the bench reveals which algorithm survives which class of transformation, and where they fail together (the genuine gap in the state of the art).

## Transform Tier Strategy

### Tier 1 — Platform Transforms (ffmpeg)
What happens to videos when they pass through YouTube, Vimeo, Instagram, or any standard video editor. These are the **transformations a fingerprint MUST survive** to be useful for real-world content identification.

If a fingerprint fails Tier 1, it's broken for the platform use case.

### Tier 2 — Pixel-Level Edits (Pillow/numpy + ffmpeg filters)
Manipulations applied by editors, colourists, and casual users — blur, colour shifts, rotation, overlays, noise. These represent **legitimate creative edits** and **light adversarial pressure**.

A fingerprint that survives Tier 1 + Tier 2 is robust enough for practical provenance.

### Tier 3 — ML Regeneration (Stubs, GPU-Required)
The state-of-the-art adversarial attacks: SDXL img2img per-frame, Stable Video Diffusion, neural frame interpolation, ESRGAN upscaling. **Most perceptual hashes fail here** — this is the gap that motivates the whole bench.

Tier 3 is registered as stubs so reports honestly say `skipped (requires GPU + SDXL pipeline)` rather than silently pretending the attack class doesn't exist. When GPU access is available, the stubs become real implementations and the bench picks them up automatically.

## Failure Mode Taxonomy

Every `(clip, fingerprint, transform)` cell resolves to exactly one `Outcome`:

| Outcome | Meaning |
|---|---|
| `match` | Distance ≤ threshold. The fingerprint recovered the identity. |
| `no_match` | Distance above the near-miss band. Fingerprint failed. |
| `near_miss` | Distance in the band `[floor(t·(1−p)), ceil(t·(1+p))]` around threshold. Worth investigating. |
| `transform_error` | The transform itself raised an exception. Cell isolated; bench continues. |
| `fingerprint_error` | The fingerprint extractor failed. Cell isolated; bench continues. |
| `skipped` | Tier 3 stub raised `NotImplementedError`, or cell explicitly excluded. |

Reports surface all six categories. The `errors` and `skipped` sections of the Markdown report exist so failures are never silently swept into `no_match`.

## Report Format

### JSON (Canonical)
```json
{
  "schema_version": "1.0",
  "bench_run": {
    "started_at": "2026-05-26T17:42:13+00:00",
    "finished_at": "2026-05-26T18:14:08+00:00",
    "duration_seconds": 1915,
    "smoke_mode": false,
    "trials_per_cell": 1
  },
  "environment": {
    "os": "Darwin 25.5.0",
    "python": "3.13.0",
    "ffmpeg": "7.1",
    "iscc_core": "1.3.0",
    "vpdq": "0.2.5",
    "obravera_video": "0.1.0",
    "platform": "darwin/arm64"
  },
  "corpus": [...],
  "thresholds": {
    "iscc-video": {"match_max_distance": 8, "source": "..."},
    "vpdq": {"match_max_distance": 31, "source": "Meta vPDQ paper, 256-bit"}
  },
  "cells": [
    {
      "clip_id": "outdoor_motion",
      "fingerprint": "iscc-video",
      "transform": "t1_reencode_h264_crf23",
      "tier": 1,
      "outcome": "match",
      "distance_raw": 2,
      "distance_threshold": 8,
      ...
    }
  ],
  "summary": {
    "by_fingerprint_by_tier": {
      "iscc-video": {"tier1": {"match": 14, "no_match": 1, "near_miss": 1, "errors": 0, "skipped": 0}, ...}
    }
  }
}
```

### Markdown (Human-Readable)
- Top-level summary table (Tier 1 match/total, Tier 2 match/total, T3 skipped)
- Per-tier breakdown table (Transform × Clip × Fingerprint × Outcome × Distance × Threshold)
- Skipped section (Tier 3 stubs with their gap reasons)
- Errors section

## Project Structure

```
obravera_video/                       # Bench harness
├── pyproject.toml                    # Project configuration
├── README.md                         # Internal dev docs
├── CLAUDE.md                         # Project conventions
│
├── src/obravera_video/
│   ├── __init__.py
│   ├── cli.py                        # obravera-video-bench CLI
│   ├── bench.py                      # BenchRunner orchestrator
│   ├── classification.py             # Outcome enum + classify()
│   ├── report.py                     # BenchResult + JSON/MD renderers
│   ├── corpus.py                     # Manifest loader + SHA-256 verify
│   │
│   ├── fingerprints/
│   │   ├── base.py                   # Fingerprint protocol + Hash type
│   │   ├── iscc.py                   # ISCC video via MP7 + iscc_core
│   │   └── vpdq.py                   # vPDQ via Meta library
│   │
│   └── transforms/
│       ├── base.py                   # Transform protocol + FfmpegTransform
│       ├── tier1_ffmpeg.py           # 16 platform transforms
│       ├── tier2_python.py           # 11 pixel-level transforms
│       └── tier3_stubs.py            # 4 ML stubs (SKIPPED in reports)
│
├── corpus/                           # Bundled CC0 test clips
│   ├── README.md                     # Manifest + SHA-256s
│   ├── outdoor_motion.mp4            # Handheld outdoor footage
│   ├── talking_head.mp4              # Single speaker, static camera
│   ├── screen_recording.mp4          # UI + text + scrolling
│   └── low_light.mp4                 # Dim/noisy footage
│
├── tests/                            # pytest suite (≥90% coverage target)
│   ├── conftest.py                   # Synth video fixtures
│   ├── test_corpus.py
│   ├── test_classification.py
│   ├── test_report.py
│   ├── test_bench.py
│   ├── test_cli.py
│   ├── test_fingerprints.py
│   ├── test_synth_fixtures.py
│   ├── test_transforms_tier1.py
│   ├── test_transforms_tier2.py
│   └── test_transforms_tier3.py
│
└── docs/
    ├── superpowers/
    │   ├── specs/                    # QA-DDD design specs
    │   └── plans/                    # Phased implementation plans
    └── testing/                      # Versioned bench report artifacts
```

## Performance Characteristics

> ### The Data Below is Subject to Change as Comprehensive Testing is Still Ongoing

### Transform Speed (M2, no GPU, single-clip)
- **Tier 1 transforms**: 0.5s – 6s per cell (re-encode bound)
- **Tier 2 transforms**: 8s – 12s per cell (frame extraction + Python loop + re-encode)
- **Fingerprint extraction**: 2s – 5s per clip (ISCC via ffmpeg signature filter), <1s (vPDQ)

### Bench Run Scale
- **Smoke (1 clip × 16 T1 × 2 fp)**: ~60s
- **Full T1 + T2 (4 clips × 27 transforms × 2 fp × 1 trial)**: ~30–60 min
- **Full + 3 trials**: ~2–3 hours

### Cell Isolation
Original fingerprint is computed once per `(clip, fingerprint)` pair and cached for the duration of the run. Each transform/fingerprint/distance step is independently wrapped — one broken cell costs the time of that cell only, not the run.

## Development

### Setup Development Environment

```bash
# Clone repository
git clone https://github.com/RedHorseMane/obravera_video_auth.git
cd obravera_video_auth

# Once the bench harness repo is integrated, install with dev dependencies:
uv sync --extra dev

# Run tests
uv run pytest -v

# Type checking
uv run mypy src/

# Linting
uv run ruff check src/ tests/
uv run ruff format src/ tests/
```

### Running Tests

```bash
# Default suite (excludes slow + tier3 meta-tests)
uv run pytest

# Include slow tests (CLI smoke test, full bench end-to-end)
uv run pytest -m "slow or not slow"

# Tier 3 meta-tests (shape-of-stubs, not behaviour)
uv run pytest tests/test_transforms_tier3.py --override-ini="addopts="

# With coverage
uv run pytest --cov=obravera_video --cov-report=html
```

### Test Discipline

- **QA-DDD** governs the *measurements* the bench produces (corpus selection, transform matrix, threshold sources, failure-mode taxonomy). Documented in `docs/superpowers/specs/`.
- **TDD** governs the *harness machinery* (transform wrappers, classifier, report renderer, BenchRunner). Test first, fail, implement, pass, commit.
- **Synth fixtures** test the machinery — they MUST NOT be used to draw fingerprint conclusions (degenerate input for perceptual hashes).
- **Real corpus** is what the bench actually measures. Byte-pinned via SHA-256 in `corpus/README.md`.

A passing pytest run says nothing about fingerprint robustness. A bench report says nothing about whether the report is correctly computed. Both are needed.

## Requirements

### Python Dependencies (Bench Harness)
- **iscc-core** (≥1.0,<2.0) — ISCC video hash computation
- **vpdq** (≥0.2.5,<0.3) — Meta video PDQ fingerprinting
- **click** — CLI framework
- **rich** — Terminal formatting
- **pydantic** — Data validation
- **pillow** — Image manipulation for Tier 2 transforms
- **numpy** — Frame processing + seeded RNG for noise

### System Dependencies

- **ffmpeg ≥ 6.0** — All transforms + MP7 signature extraction for ISCC

  ```bash
  # macOS
  brew install ffmpeg

  # Ubuntu/Debian
  sudo apt-get install ffmpeg
  ```

  ffmpeg must include the `signature` filter (standard in modern builds — verify with `ffmpeg -filters | grep signature`).

### Optional (Future / Tier 3)
- **GPU access** (CUDA / MPS / Metal) — Required for Tier 3 ML stubs when they become real implementations
- **diffusers + torch** — SDXL img2img regeneration attack
- **Real-ESRGAN** weights — AI upscale attack
- **RIFE / FILM** model weights — Neural frame interpolation

## Documentation

- **[Design Specification]()**: QA-DDD spec for the bench (transform matrix, thresholds, methodology)
- **[Implementation Plan]()**: Phased TDD roadmap (Phase 0 → Phase 8)
- **[Bench Reports]()**: Versioned JSON + Markdown reports per run

## Known Limitations

### Tier 3 (ML Attacks) Are Stubs
- **Honest gap**: All 4 Tier 3 transforms (`sdxl_regen_per_frame`, `video_diffusion_regen`, `neural_frame_interp`, `ai_upscale_4x`) currently raise `NotImplementedError` and surface as `Outcome.SKIPPED` cells.
- **Why stubs rather than silent omission**: The skip is named in every report. If a fingerprint passes Tiers 1 and 2 perfectly but you ignore Tier 3, you do not yet know whether it survives the attacks that matter most for AI-era provenance.
- **Path forward**: GPU access + model weights → each stub's `apply()` gains a real implementation → bench picks them up automatically.

### Single-Algorithm Confidence
- The bench measures ISCC and vPDQ in isolation. **No fingerprint alone is a complete provenance solution.** The bench's purpose is to map where each survives and where each fails — the answer to "is this content authentic" requires combining a survivable fingerprint with C2PA, watermarking, and a registry layer (the audio sibling project demonstrates the full stack).

### Threshold Calibration
- `DEFAULT_THRESHOLDS` in the CLI (ISCC=8, vPDQ=31) are **placeholders** drawn from published guidance, not from observed identity-clip distances on the real corpus. Phase 8 of the implementation plan recalibrates them against the actual bundled clips.

### Corpus Size
- Bundled CC0 corpus is 4 clips × ~20s. Sufficient for the methodology but **not statistically powerful** — bench results are robustness *indicators*, not population estimates. A larger licensed corpus is on the roadmap.

## Roadmap

### Phase 0–7 (Complete)
- ✅ Project scaffold + uv standalone build
- ✅ Synth video fixtures + corpus loader with SHA-256 verification
- ✅ ISCC video fingerprint wrapper (MP7 frame signatures via ffmpeg)
- ✅ vPDQ fingerprint wrapper (per-frame PDQ + average best-match distance)
- ✅ 16 Tier 1 ffmpeg transforms (re-encode, scale, fps, container, letterbox, clip, fade)
- ✅ Pure-function classifier with integer-rounded near-miss band + tie-break rules
- ✅ BenchResult dataclasses + JSON + Markdown renderers
- ✅ BenchRunner with cell-level error isolation + original-fingerprint caching
- ✅ CLI: `list`, `validate`, `run` (with `--smoke` fail-fast)
- ✅ 11 Tier 2 Python frame-level transforms (blur, brightness, contrast, rotation, aspect, overlay, seeded noise, speed)
- ✅ 4 Tier 3 ML stubs with TODO-named gap reasons; `NotImplementedError` → `Outcome.SKIPPED`

### Phase 8 (In Progress — pending CC0 corpus acquisition)
- 📋 First real bench run against the bundled corpus
- 📋 Recalibrate `DEFAULT_THRESHOLDS` from observed identity-clip distances
- 📋 Commit dated JSON + Markdown report as the project's first versioned artifact
- 📋 Final whole-project code review

### Future Phases
- 📋 Tier 3 implementation: SDXL img2img regeneration (highest priority — the attack that defeats most watermarks)
- 📋 Tier 3 implementation: Stable Video Diffusion, RIFE neural interp, Real-ESRGAN upscale
- 📋 Extended corpus (10+ clips, multiple licences, broader content range)
- 📋 Multi-algorithm ensemble fingerprint (combine ISCC + vPDQ + neural variants)
- 📋 Video watermarking integration (parallel to AudioSeal in the audio sibling)
- 📋 C2PA video manifest support
- 📋 Registry-based credential recovery for stripped metadata
- 📋 GUI application (analogous to ObraVera Audio)

## Privacy & Security

### Local Processing
- All bench operations run on your machine. Video files never leave the local filesystem.
- Reports are written to local paths; no telemetry.
- Corpus SHA-256s detect tampering on disk and across distribution.

### Reproducibility
- Seeded RNG for noise injection means two runs on identical inputs produce byte-identical processed frames.
- Reports record full environment (ffmpeg, iscc-core, vpdq versions + OS + arch) so any divergence is traceable.

## Code Quality & Standards

### PEP 8 Compliance
This project follows [PEP 8](https://peps.python.org/pep-0008/) style guidelines:
- **Line Length**: 88 characters (Ruff)
- **Formatting**: Enforced via `ruff format`
- **Linting**: Checked with `ruff check`

### Automated Enforcement

```bash
# Check code quality
uv run ruff check src/ tests/

# Auto-format
uv run ruff format src/ tests/

# Type check
uv run mypy src/

# Full test suite
uv run pytest
```

### Test Coverage
- **Target**: ≥90% line coverage on `src/obravera_video/`
- **Synth fixtures**: 5 deterministic video generators (mandelbrot + noise) for harness tests
- **Real corpus**: SHA-256-pinned CC0 clips for the bench itself

## Relationship to ObraVera Audio

ObraVera Video is the parallel project to [ObraVera Audio](https://github.com/RedHorseMane/obravera_audio_auth). The two share the same fundamental thesis: **content authentication requires multi-layer defence in depth, and each layer's survival envelope must be measured honestly before it can be relied upon.**

The audio project's multi-layer architecture (metadata + C2PA + watermark + fingerprint + signature + registry) is the destination for video too. The bench harness is the prerequisite — without knowing which video fingerprints survive which attacks, the layered design cannot be assembled responsibly.

## Contributing

***Contributions not yet open***

## License

TBD

## Contact

TBD

---

**Status:** Active development. Fingerprint bench harness fully implemented (Phases 0–7); first real bench run pending CC0 corpus acquisition. Tier 3 ML attacks are honest stubs awaiting GPU implementation.
