# Everdell: Result Screen Recognition and Data Storage

## Decision Summary

During the first stage, the game result screen will be the primary data source.
The optimal way to read it is not generic OCR over the entire image, but a
hybrid pipeline:

1. locate the result screen and transform it to a canonical size;
2. extract predefined regions from each row;
3. recognize names and numbers with an OCR model;
4. recognize characters by comparing them with a catalog of reference avatars;
5. validate the result with domain rules and store it together with confidence
   scores;
6. send uncertain results for manual review.

SQLite will be used as the database. Source images will remain as files, while
the database will store their relative paths, SHA-256 hashes, recognized values,
and technical information about parser runs.

Network interception was not selected as the primary approach. It could provide
more accurate data, but it is less universal and more fragile. If an official
API, a local log, or a stable JSON response is discovered later, it should be
implemented as a separate adapter that produces the same internal game model.

## What Can Be Read from the Existing Example

`data/photo_2026-10-10_10-57-49.jpg` contains four horizontal rows. Each row has
the following fields in stable positions:

- placement;
- circular character avatar;
- player name;
- final score;
- a number next to the event icon;
- a number next to the remaining-resources icon.

The example can be read manually as follows and can become the first regression
fixture:

| Place | Name | Score | Events | Resources | Character |
|---:|---|---:|---:|---:|---|
| 1 | Corviurge | 95 | 3 | 2 | frog avatar; canonical ID is not known yet |
| 2 | tkhelet | 79 | 1 | 1 | hedgehog avatar; canonical ID is not known yet |
| 3 | Ember | 65 | 2 | 16 | bird avatar; canonical ID is not known yet |
| 4 | Thistle | 64 | 2 | 7 | rat/mouse avatar; canonical ID is not known yet |

A character name cannot be inferred reliably from pixels without a reference
catalog. Until that catalog exists, the parser should save the avatar crop and
set `character_id = NULL` instead of guessing from its appearance.

## Why a Hybrid Approach Was Chosen

### Fixed Geometry and Region-Based OCR

The screen resembles a form rather than an arbitrary document. Once its
coordinates have been normalized, its rows and columns are known. Running text
detection over the entire image would only introduce errors from the heading,
button, and game board in the background. Each field should be sent to the model
as a separate crop:

- `player_name`: one line containing letters, digits, spaces, and ordinary
  nickname characters;
- `score`, `events_completed`, and `resources_left`: an integer only;
- `place`: recognize it as a validation field, but derive the primary value from
  the top-to-bottom row order.

The initial candidate for names is local PaddleOCR with an English recognition
model and without text detection because the geometry already provides a ready
text-line crop. It has a Python API, exposes confidence scores, and is better
suited to a nonstandard game font than handcrafted character-processing rules.
The same model should initially be used for numbers, with its output restricted
and validated in code. This keeps the application to a single OCR runtime.

Before committing to an OCR engine, run a short benchmark on an annotated
dataset comparing PaddleOCR with Tesseract. Tesseract offers useful single-line
and single-word modes and a digit whitelist, but requires a separate system
binary. Hide the engine behind a `TextRecognizer` interface so that replacing it
does not affect the parser. Choose the winner by exact field match rather than by
subjective output quality. The initial recommendation is PaddleOCR; Tesseract
remains a lighter fallback if it achieves equal or better accuracy on the real
dataset.

Useful primary sources:

- [PaddleOCR: recognition pipeline](https://www.paddleocr.ai/main/en/quick_start.html)
- [Tesseract: image-quality improvements and page segmentation modes](https://tesseract-ocr.github.io/tessdoc/ImproveQuality.html)
- [OpenCV: image processing](https://docs.opencv.org/4.x/d2/d96/tutorial_py_table_of_contents_imgproc.html)

### Character Recognition Is Classification, Not OCR

An avatar contains no text. It requires a local catalog:

```text
character_id, canonical_name, game_version, reference_image
```

After the avatar crop has been normalized to a standard size, compare it with
the references. Start with a simple, explainable method: perceptual hashing plus
SSIM or normalized correlation. This is robust to JPEG compression and minor
scaling. If the game includes animation, different frames, or skins, the next
step should be image embeddings with nearest-reference search. A fully trained
classifier only becomes worthwhile after an annotated dataset has been
collected.

An unknown avatar or one recognized with insufficient confidence must remain
`unknown` instead of being forced to the nearest character.

### Why a VLM Should Not Be the Primary Parser

A multimodal model could return nearly complete JSON from a single example, but
reproducibility matters more for a statistical database:

- the model and its responses may change;
- it is harder to guarantee the response format and identical interpretation of
  identical frames;
- it adds cost, latency, a network dependency, and transmission of player names
  to an external service;
- confidence scores for individual fields are usually less well calibrated;
- errors are harder to isolate and test.

A VLM can be useful for initial annotation and as an aid during manual review,
but it should not produce automatically accepted records.

### Why Packet Interception Should Not Be the Foundation

If a stable structured response is available, it would be the best source: OCR
and classification would be unnecessary, and values would arrive without loss
of accuracy. In practice, however, TLS, certificate pinning, proprietary
protocols, endpoint changes, different clients, and service rules may all get in
the way. Interception also cannot process screenshots that have already been
saved.

Recommended source priority in the future:

1. official export or API;
2. documented local logs or client-accessible JSON;
3. result-screen images;
4. network protocol reverse engineering, only as a separate investigation after
   checking the service rules.

Every source should produce the same `ParsedGame`, so adding a better source
later will not require changing the database or analytics.

## Proposed Image Pipeline

```text
Image
  -> source fingerprint / deduplication
  -> verify that this is a Victory screen
  -> locate the frame and normalize geometry
  -> locate player rows
  -> crop fields
  -> OCR text and numbers + match avatar
  -> normalize values
  -> cross-field validation
  -> accepted | needs_review | rejected
  -> SQLite
```

### 1. Ingestion and Deduplication

- Calculate the source file's SHA-256 before any transformation.
- Reimporting the same file should return the existing record.
- Do not rely on the file name: messaging applications rename and recompress
  images.
- Do not modify the source file.

### 2. Detecting the Required Screen

During the first stage, the CLI accepts an already selected image, but the core
parser should immediately be able to reject an irrelevant frame. Signals
include:

- the upper `VICTORY` heading or anchor is found;
- two to four similar horizontal rows are found;
- a score circle and the two expected icons are found on the right of each row;
- element geometry is consistent across rows.

OCR of the word `VICTORY` alone is insufficient. The decision should combine
multiple signals so that a future scanner over a general image directory does
not create false game records.

### 3. Geometry Normalization

- Locate the outer frame or a set of stable anchors.
- Correct scale, offset, and perspective when necessary.
- Transform the working area into a canonical coordinate system.
- Store the layout as versioned configuration, such as
  `everdell_digital_v1`, instead of scattering numeric constants through the
  code.
- Initially support straight digital screenshots. Treat monitor photos with
  strong perspective distortion as a separate quality tier.

This layer will allow one parser to handle fullscreen screenshots, differently
sized windows, and future realtime capture.

### 4. Extracting Rows and Fields

Prefer detecting row y-coordinates from the repeating panels and then applying
relative x-ranges for fields. Hard-coded pixel coordinates are acceptable only
after the image has been warped to the canonical size.

Create several inexpensive variants of each OCR crop: original, enlarged 2–3x,
and grayscale/high-contrast. Select the variant with the highest score that also
passes validation. Do not apply aggressive binarization to every field because
thin strokes in the game font may disappear.

### 5. Normalization and Validation

The parser should preserve both the raw and normalized values. Validate that:

- the player count matches the number of detected rows;
- placements are unique and form a sequence from 1 to N;
- every numeric field is a nonnegative integer;
- names are not empty;
- scores do not increase from top to bottom, except for tie behavior that must
  be investigated separately;
- all three numeric fields are populated in every row;
- confidence and geometric consistency exceed configured thresholds.

Do not introduce unverified hard maximums for scores or resources: expansions
may increase the ranges. Suspicious values can produce warnings instead.

Overall game confidence should combine field confidence scores with penalties
for geometry or domain-rule violations. Calibrate its threshold on a validation
set. Any correction made by a user must be stored separately from the machine
output.

## Data Model

SQLite is suitable for a local application: it is a single file, provides
transactions and unique constraints, supports ordinary SQL for metrics, and
requires no separate server. If the project becomes a multi-user web service,
the repository layer can later be moved to PostgreSQL.

Minimal normalized schema:

```text
source_images
  id, sha256 UNIQUE, relative_path, width, height,
  captured_at NULL, imported_at, metadata_json

parse_runs
  id, source_image_id, parser_version, layout_version,
  started_at, status, confidence, diagnostics_json

games
  id, source_image_id UNIQUE, played_at NULL,
  player_count, review_status, created_at, updated_at

game_results
  id, game_id, place, player_name_raw, player_name_normalized,
  character_id NULL, score, events_completed, resources_left,
  row_confidence,
  UNIQUE(game_id, place)

characters
  id, canonical_name, game_version NULL, enabled

character_templates
  id, character_id, relative_path, perceptual_hash,
  template_version

field_observations
  id, parse_run_id, result_id NULL, field_name,
  raw_value, normalized_value, confidence, crop_relative_path NULL,
  accepted, correction_value NULL
```

Notes:

- A separate `players` table is not needed initially: identical nicknames may
  belong to different people, and one person may change their nickname.
  `game_results` stores the string visible on the screen. Explicit identity
  merging can be added later without rewriting history.
- Do not store source JPEG files as BLOBs. Doing so would inflate SQLite and make
  the images harder to inspect.
- `metadata_json` and `diagnostics_json` are intended for unstable technical
  details; primary analytical fields remain separate typed columns.
- Schema migrations are needed from the first version. Numbered SQL migrations
  are enough for a small project; Alembic is justified only if SQLAlchemy is
  selected.
- The standard `sqlite3` module and a small repository layer are sufficient for
  the initial application. An ORM does not yet provide a meaningful benefit.

## Core Parser Contract

The recognition model must not depend on the future method used to acquire the
frame:

```python
@dataclass(frozen=True, slots=True)
class ParsedResult:
    place: int
    player_name: str
    character_id: str | None
    score: int
    events_completed: int
    resources_left: int
    confidence: float


@dataclass(frozen=True, slots=True)
class ParsedGame:
    results: tuple[ParsedResult, ...]
    confidence: float
    warnings: tuple[str, ...]
```

In addition to this convenient domain model, a technical report should preserve
all raw OCR values, bounding boxes, scores, model versions, and reasons for a
`needs_review` decision.

## Implementation Plan

### Stage 0. Collect and Annotate a Dataset

- Collect at least 30–50 result screens, including games with 2, 3, and 4
  players.
- Include the resolutions, window/fullscreen modes, JPEG quality levels,
  languages, and UI versions that will actually be used.
- Annotate expected fields in simple JSON files next to the fixtures.
- Build a separate character catalog and confirm canonical character names.
- Split images into development and validation sets before tuning thresholds.

This stage is mandatory. One clean image can prove that a prototype works, but
it cannot select the OCR engine or establish reliability.

### Stage 1. Vertical Prototype for One Image

- Set up the Python 3.12 package and a `parse-image PATH` CLI command.
- Implement the `everdell_digital_v1` layout for the current example.
- Extract ROIs, recognize fields, and produce structured JSON output.
- Compare PaddleOCR and Tesseract on the annotated dataset.
- Save a diagnostic image with bounding boxes only when an explicit debug flag
  is provided.
- Add unit tests for normalization and a regression test for the current image.

Completion criterion: accuracy is measured for all text and numeric fields in
the validation set; recognition errors and failures never become silently
incorrect values.

### Stage 2. Characters and SQLite

- Add the character catalog and reference templates.
- Implement matching and an `unknown` threshold.
- Create the SQLite schema, migrations, transactional writes, and SHA-256
  deduplication.
- Add `import-image PATH` and `show-game ID` CLI commands.
- Allow a new parser run without deleting the preceding `parse_run`.

Completion criterion: repeated imports are idempotent, corrections are
auditable, and unknown avatars never receive a false ID.

### Stage 3. Robustness and Manual Review

- Support multiple sizes and crops of the same UI.
- Calibrate confidence scores and thresholds on the validation set.
- Add a minimal review flow for uncertain fields.
- Record exact-match accuracy for each field, the percentage of entirely
  correct games, the false-accept rate, and the `needs_review` rate.

Proposed initial quality gates:

- 100% accuracy for numeric fields among automatically accepted games;
- at least 99% exact name match among automatically accepted fields;
- 100% correct character IDs among automatically accepted results;
- any case below the threshold is sent to `needs_review`.

These thresholds are goals, not promises before the dataset exists. For a
statistics system, requesting confirmation more often is better than silently
recording incorrect data.

### Stage 4. Future Sources, Not Implemented Now

- General image-directory scanner: filter by extension and date, use an
  inexpensive screen classifier, and then call the same `import-image`
  operation; keep a checkpoint to avoid rescanning old files.
- Realtime capture: capture a selected window or region, use a perceptual hash
  to detect frame changes, debounce until the screen is stable, and then invoke
  the same parser.
- Structured adapter: import an official API, log, or JSON response into
  `ParsedGame`.
- Analytics and visualizations over the stable schema.

## Scope of the First Implementation

Do not include the following in the first working version:

- recursive scanning of a user's image library;
- continuous screen monitoring or a Snipping Tool-like interface;
- network reverse engineering;
- a web UI or dashboards;
- automatic model training;
- cloud OCR or a VLM as a required dependency.

First, prove the accuracy of the pure `image -> ParsedGame + report` operation
on a representative dataset. Directory scanning and realtime capture then
become simple image providers rather than separate recognition implementations.

## Open Questions Before Implementation

These questions should preferably be answered with additional examples rather
than guesses:

1. Which platforms and modes are required: Windows/Steam, browser, or mobile;
   windowed or fullscreen?
2. Will all inputs be digital screenshots, or must monitor photographs also be
   supported?
3. Which interface languages and nickname characters are allowed?
4. Are there result screens for 1, 5, or 6 players, ties, or expansion variants?
5. What exactly does “character” mean: the avatar's canonical name, the animal
   species, or is preserving the image sufficient?
6. Can a complete avatar catalog with one or two references per character be
   collected?
7. Is fully offline operation required, and which operating systems will later
   run realtime capture?

None of these questions prevents work on Stage 0 or a prototype for the current
layout, but their answers will define the boundaries of the promised accuracy.
