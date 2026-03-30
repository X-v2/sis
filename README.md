# SIS: Floor Plan Parsing, Structural Reasoning, and Analysis Verification Platform

## Overview

This repository contains a multi-part system for converting architectural floor plan images into a normalized structural model, visualizing that model in an interactive 3D web interface, generating structural and material recommendations, persisting analysis records, and optionally anchoring analysis hashes on the Stellar test network for verification.

The project is composed of four principal layers:

1. A Python floor plan parser that extracts walls, openings, labels, slabs, structural nodes, and columns from an input image.
2. A Python FastAPI backend that exposes parsing, analysis persistence, and verification endpoints.
3. A Next.js frontend that validates uploaded models, renders an interactive 3D scene, runs recommendation workflows, and presents optimization suggestions.
4. A Soroban smart contract workspace used to store analysis hashes on Stellar for integrity verification.

This root README is the authoritative documentation for the repository. The `frontend/README.md` and `contract/README.md` files are legacy/generated module-level documents and do not fully describe the current implementation.

## Core Objectives

The system is designed to support the following workflow:

1. Accept a floor plan image or structured JSON model.
2. Convert the plan into a machine-readable geometric schema in meters.
3. Render walls, slabs, doors, windows, labels, nodes, and inferred columns in 3D.
4. Validate the uploaded geometry and identify structural or geometric issues.
5. Generate material recommendations and layout optimization suggestions.
6. Persist analyses locally in SQLite.
7. Optionally write a deterministic hash of each analysis to Stellar and verify the stored record against the on-chain value.

## Technology Stack

### Frontend

- Next.js 16 with the App Router
- React 19
- TypeScript
- React Three Fiber and Drei for 3D rendering
- Zustand for client state management
- Tailwind CSS 4 plus custom global CSS tokens/utilities

### Backend and Parser

- Python
- FastAPI and Uvicorn
- OpenCV, NumPy, scikit-image, Shapely, and EasyOCR
- SQLite for local persistence
- PyTorch-backed OCR/runtime dependencies

### Smart Contract Layer

- Rust
- Soroban SDK
- Stellar CLI

## High-Level Architecture

### End-to-End Flow

1. A user uploads either:
   - a raw floor plan image, or
   - a structured JSON scene model.
2. For image uploads, the frontend sends the file to `frontend/app/api/parse-image/route.ts`, which proxies the image to a parser backend endpoint.
3. The parser backend runs OCR, wall detection, door/window detection, scale extraction, slab derivation, and schema export.
4. The frontend normalizes and validates the returned schema, then renders it as an interactive 3D model.
5. The frontend calls `frontend/app/api/recommendations/route.ts` to produce:
   - material recommendation tables, and
   - heuristic layout optimization suggestions.
6. When an analysis is submitted to the backend, the backend:
   - computes a canonical SHA-256 hash,
   - optionally records the hash on Stellar,
   - stores the analysis and line items in SQLite.
7. Verification requests compare the stored database hash with the on-chain hash and update verification state accordingly.

### Architectural Responsibilities

| Layer | Responsibility |
| --- | --- |
| `parser/` | Computer vision and OCR pipeline that converts image data to normalized geometry |
| `backend/` | Public API, persistence, hashing, and verification |
| `frontend/` | Visualization, validation, recommendations, interaction, and optimization workflows |
| `contract/` | Soroban contract for analysis-hash storage and retrieval |

## Repository Structure

The following structure focuses on maintained source files and important artifacts. Generated folders such as `frontend/node_modules/` and `frontend/.next/` are intentionally omitted.

```text
.
|-- backend/
|   |-- main.py
|   |-- stellar.py
|   `-- storage.py
|-- contract/
|   |-- Cargo.toml
|   |-- README.md
|   `-- contracts/
|       `-- analysis-contract/
|           |-- Cargo.toml
|           |-- Makefile
|           `-- src/
|               |-- lib.rs
|               `-- test.rs
|-- debug_layers/
|   `-- *.png
|-- frontend/
|   |-- app/
|   |   |-- api/
|   |   |   |-- parse-image/
|   |   |   |   `-- route.ts
|   |   |   `-- recommendations/
|   |   |       `-- route.ts
|   |   |-- favicon.ico
|   |   |-- globals.css
|   |   |-- layout.tsx
|   |   `-- page.tsx
|   |-- components/
|   |   |-- ColumnMesh.tsx
|   |   |-- ConsoleGuard.tsx
|   |   |-- ExplanationPanel.tsx
|   |   |-- MaterialPanel.tsx
|   |   |-- NodeMesh.tsx
|   |   |-- Openings.tsx
|   |   |-- SlabMesh.tsx
|   |   |-- Viewer3D.tsx
|   |   `-- WallMesh.tsx
|   |-- lib/
|   |   |-- explainer.ts
|   |   |-- geometry.ts
|   |   |-- layoutHeuristics.ts
|   |   |-- materialData.ts
|   |   |-- materialEngine.ts
|   |   |-- optimizationSolver.ts
|   |   |-- recommendation.ts
|   |   |-- sceneGraph.ts
|   |   |-- sceneInsights.ts
|   |   |-- types.ts
|   |   `-- validation.ts
|   |-- public/
|   |   |-- file.svg
|   |   |-- globe.svg
|   |   |-- next.svg
|   |   |-- vercel.svg
|   |   `-- window.svg
|   |-- store/
|   |   `-- useStore.ts
|   |-- package.json
|   |-- package-lock.json
|   `-- tsconfig.json
|-- parser/
|   |-- config.py
|   |-- debug.py
|   |-- doors.py
|   |-- image_io.py
|   |-- main.py
|   |-- rooms.py
|   |-- scale.py
|   |-- schema.py
|   |-- text.py
|   |-- walls.py
|   `-- windows.py
|-- analysis.db
|-- floorplan.png
|-- floorplan_test.png
|-- output.json
|-- requirements.txt
|-- sample.json
|-- test.py
`-- README.md
```

## Frontend

### Purpose

The frontend is an interactive structural reasoning viewer. It accepts uploaded JSON and image-based parser output, validates and normalizes the data, renders it in 3D, presents material recommendations, and provides guided layout optimization actions.

### Main Frontend Entry Points

#### `frontend/app/page.tsx`

This is the primary application shell and orchestration layer. It is responsible for:

- image upload and JSON upload handling,
- parse API invocation,
- recommendation API invocation,
- selection state and inspector state,
- validation issue presentation,
- optimization-action preview/confirm/revert flows,
- integration with the Zustand store,
- viewer-level focus and navigation behavior.

#### `frontend/app/layout.tsx`

Defines the root HTML layout, applies global styling, and mounts `ConsoleGuard`.

#### `frontend/app/globals.css`

Contains the global design tokens, theme variables, layout primitives, component utilities, badges, cards, status states, responsive layout rules, and animation helpers used across the interface.

### Frontend API Routes

#### `frontend/app/api/parse-image/route.ts`

Acts as a proxy for image uploads. It accepts an uploaded image from the UI and forwards it to an upstream parser endpoint.

Important implementation detail:

- The parser URL is currently hardcoded as:
  - `https://pulled-budapest-mind-peter.trycloudflare.com/parse`

This means the frontend does not automatically target the local FastAPI backend unless this route is updated to point to the local deployment.

#### `frontend/app/api/recommendations/route.ts`

Accepts the normalized scene plus the current selection and returns:

- `materialTable`
- `heuristics`

The route combines deterministic structural/material scoring with server-side heuristic reporting.

### Frontend State Management

#### `frontend/store/useStore.ts`

The Zustand store owns the normalized scene and derived interaction state:

- `rawInput`
- `scene`
- `issues`
- `selectedEntity`
- `structuralView`
- `debugOverlay`
- `recommendations`
- `explanation`

It also exposes the core state transitions:

- `loadRawInput`
- `selectEntity`
- `toggleStructuralView`
- `toggleDebugOverlay`
- `clearScene`

### Frontend Component Layer

#### 3D Visualization Components

- `Viewer3D.tsx`: the main 3D canvas, lighting, orbit controls, hover/selection handling, focus transitions, and render orchestration.
- `WallMesh.tsx`: renders wall bodies and wall segments around openings.
- `SlabMesh.tsx`: renders slab/room surfaces.
- `Openings.tsx`: renders doors and windows with semantic geometry.
- `NodeMesh.tsx`: renders graph nodes and support/junction markers.
- `ColumnMesh.tsx`: renders inferred or supplied columns.

#### UI Support Components

- `MaterialPanel.tsx`: shows ranked material recommendations, structural concerns, and scoring formulas.
- `ExplanationPanel.tsx`: displays human-readable reasoning for the selected element.
- `ConsoleGuard.tsx`: mounted globally to manage console behavior during the client session.

### Frontend Domain Libraries

#### `frontend/lib/types.ts`

Defines the canonical TypeScript contracts for:

- raw input payloads,
- normalized scene data,
- validation issues,
- recommendation payloads,
- heuristic reports,
- optimization actions,
- geometric primitives.

This file acts as the frontend-side schema contract.

#### `frontend/lib/validation.ts`

This is one of the most important frontend modules. It:

- validates incoming JSON,
- normalizes incomplete or inconsistent inputs,
- infers missing wall types and defaults,
- parses slabs, labels, openings, nodes, and columns,
- detects opening clashes,
- repairs certain clash conditions,
- computes overall scene readiness (`valid`, `partial`, or `invalid`).

The frontend depends on this module to make parser output and manual JSON uploads renderable and inspectable.

#### `frontend/lib/geometry.ts`

Provides geometric primitives and utilities for:

- distances,
- midpoints,
- angles,
- 2D-to-3D conversion,
- polygon area and centroid calculations,
- room span computation,
- wall splitting around openings,
- scene bounds generation,
- room context derivation.

#### `frontend/lib/sceneGraph.ts`

Transforms normalized scene data into render-ready nodes for the viewer. It builds:

- room render nodes,
- wall render nodes,
- opening render nodes,
- structural graph nodes,
- columns,
- the aggregate scene graph consumed by the 3D layer.

#### `frontend/lib/materialEngine.ts`

Implements deterministic wall classification and local wall-level material scoring. It classifies walls as:

- load-bearing,
- partition, or
- semi-structural.

It also associates walls with room context and ranks suitable materials.

#### `frontend/lib/recommendation.ts`

Builds formal material recommendation tables. Its logic includes:

- structural system classification,
- adequacy-gate filtering,
- weighted tradeoff scoring,
- element-specific recommendation rows,
- optional Gemini-generated rationale enrichment.

If a Gemini API key is not available, the recommendation pipeline falls back to deterministic reasoning only.

#### `frontend/lib/layoutHeuristics.ts`

Produces structural advisory suggestions such as:

- span corrections,
- alignment fixes,
- load-path gaps,
- missing intermediate column/support signals,
- material optimization opportunities,
- potential wall-removal opportunities.

These suggestions are advisory and do not directly mutate the model.

#### `frontend/lib/optimizationSolver.ts`

Converts validation issues and heuristic suggestions into concrete UI actions. It supports:

- opening clash fixes,
- alignment snaps,
- guarded wall removals,
- preview-line generation for pending actions.

#### `frontend/lib/materialData.ts`

Contains the internal material catalog used for scoring. The catalog includes options such as RCC, brick variants, AAC blocks, steel systems, precast systems, and several lightweight partition materials.

#### `frontend/lib/explainer.ts`

Generates concise natural-language explanations for the currently selected wall or structural element.

#### `frontend/lib/sceneInsights.ts`

Generates readiness labels and suggested clarifying questions based on scene quality and detected issues.

### Frontend Data Expectations

The current frontend is built around the following preferred payload sections:

- `meta`
- `walls`
- `slabs`
- `labels`
- `doors`
- `windows`
- `openings`
- `graphNodes`
- `columns`

Important note:

- `rooms[]` is treated as deprecated by the frontend validation layer.
- The provided `sample.json` still uses `rooms[]`, so it should be considered a legacy example rather than the preferred current schema format.

## Backend

### Purpose

The backend exposes the parsing and persistence services used by the platform. It performs three major jobs:

1. parse uploaded images into normalized structural JSON,
2. persist full analysis records and itemized cost lines,
3. verify persisted analyses against an on-chain hash.

### `backend/main.py`

This is the FastAPI application entry point. It defines the following endpoints:

#### `GET /health`

Simple health-check endpoint returning `{"status": "ok"}`.

#### `POST /parse`

Accepts an uploaded image and returns parser output as JSON. The endpoint:

- validates that the upload is an image,
- reads the file bytes,
- constructs a `ParserConfig` with `debug_enabled=False`,
- calls `parse_floorplan_bytes(...)`,
- returns the normalized model.

#### `POST /analyses`

Accepts an analysis payload containing:

- `totalCost`
- `totalArea`
- `costPerM2`
- `lineItems`
- `modelJson`

The backend then:

- generates a unique `analysisId`,
- normalizes line-item identifiers,
- computes derived totals if needed,
- builds a canonical hash payload,
- hashes the payload,
- optionally records the hash on Stellar,
- stores the analysis and its line items in SQLite.

#### `GET /analyses`

Returns all stored analyses ordered by `created_at` descending.

#### `GET /analyses/{analysis_id}`

Returns a single stored analysis by identifier.

#### `GET /verify/{analysis_id}`

Loads the stored analysis, retrieves the stored on-chain hash, compares it to the database hash, updates verification state, and returns the verification result.

### `backend/storage.py`

This module manages SQLite persistence through `analysis.db`.

It defines two tables:

#### `analyses`

Stores the top-level analysis record, including:

- identifiers and timestamps,
- total cost and area metrics,
- serialized model JSON,
- canonical data hash,
- Stellar recording metadata,
- verification status and last verification time.

#### `analysis_items`

Stores itemized line items with:

- item ID,
- parent analysis ID,
- element type,
- material,
- quantity,
- unit,
- unit rate,
- subtotal,
- justification.

### `backend/stellar.py`

This module is responsible for analysis hashing and optional Stellar integration.

It provides:

- deterministic canonical hashing via SHA-256,
- optional Soroban contract invocation for hash recording,
- on-chain hash retrieval,
- transaction-hash extraction and safe logging.

The hash pipeline normalizes:

- float precision,
- dictionary ordering,
- list ordering for itemized records.

This keeps verification stable across repeated serializations of the same logical data.

### Current Stellar Runtime Assumptions

The backend currently assumes:

- a Windows installation path for the Stellar CLI:
  - `C:\Program Files (x86)\Stellar CLI\stellar.exe`
- environment-driven contract configuration
- Stellar writes are disabled by default

If `STELLAR_ENABLE_WRITE` is not set to `1`, the backend returns a prototype/unconfigured recording status instead of writing on-chain.

## Parser Engine

### Purpose

The parser is the computer vision and schema export engine of the project. It converts raster floor plan imagery into a normalized structural model measured in meters.

### `parser/main.py`

The parser pipeline proceeds in the following order:

1. load and threshold the image,
2. detect OCR text,
3. build a text mask,
4. detect the main plan region and wall candidates,
5. inspect and detect doors,
6. inspect and detect windows,
7. merge window-host walls and reassign openings,
8. split walls by outside adjacency,
9. detect scale,
10. build the final export schema,
11. optionally write debug images.

The module supports both:

- file-based parsing via `parse_floorplan()`, and
- byte-based parsing via `parse_floorplan_bytes(...)`.

When the parser is run directly, it reads `floorplan_test.png` and writes `output.json`.

### `parser/config.py`

Defines the parser configuration object and the default values used throughout the pipeline, including:

- default wall, door, and window dimensions,
- thresholding parameters,
- wall extraction kernels,
- door/window candidate thresholds,
- default scale fallback,
- OCR configuration,
- debug-output directory and toggles.

### `parser/image_io.py`

Loads input images and converts them into:

- color image,
- grayscale image,
- inverted binary mask.

### `parser/text.py`

Uses EasyOCR to detect text, classifies the detected text, and builds a mask so text does not corrupt wall extraction. It is also involved in room-label and scale-text handling.

### `parser/walls.py`

This is the primary structural extraction module. It:

- isolates the plan region,
- extracts thick wall components,
- merges horizontal and vertical segments,
- recovers short missing wall sections,
- classifies walls as outer or internal,
- splits walls at intersections,
- segments walls based on outside adjacency,
- renders wall masks for downstream operations.

### `parser/doors.py`

Detects door openings using wall-gap analysis and symbol interpretation. It also infers swing direction for exported door objects.

### `parser/windows.py`

Detects window candidates, filters them against doors and wall geometry, merges host-wall context, and exports normalized window objects.

### `parser/scale.py`

Attempts to determine the image scale from OCR-extracted text and plan context. When scale cannot be extracted confidently, the parser falls back to configured defaults.

### `parser/schema.py`

Builds the exported JSON schema consumed by the frontend and backend. It converts pixel-space geometry into metric coordinates and constructs:

- `meta`
- `walls`
- `slabs`
- `labels`
- `doors`
- `windows`
- `openings`
- `graphNodes`
- `columns`

This module also derives slabs from free regions, labels, and wall masks rather than relying solely on an externally supplied room list.

### `parser/debug.py`

Writes visual debug layers to `debug_layers/`.

### `parser/rooms.py`

Contains room-detection utilities, but the current export path relies primarily on slab derivation in `schema.py`. It should be understood as auxiliary or legacy support logic rather than the sole room-generation path.

### Debug Artifacts

The `debug_layers/` directory contains generated diagnostic images such as:

- binary masks,
- wall masks,
- text masks,
- door candidate overlays,
- window candidate overlays,
- label overlays,
- scale overlays,
- combined structural previews.

These outputs are particularly useful while tuning parser heuristics.

## Smart Contract Layer

### Purpose

The `contract/` directory contains a Soroban workspace for recording and retrieving analysis hashes on Stellar.

### Workspace Structure

- `contract/Cargo.toml`: workspace definition for Soroban contracts.
- `contract/contracts/analysis-contract/Cargo.toml`: contract package definition.
- `contract/contracts/analysis-contract/src/lib.rs`: current contract implementation.
- `contract/contracts/analysis-contract/src/test.rs`: contract test scaffold.
- `contract/contracts/analysis-contract/Makefile`: convenience targets for build, test, format, and clean operations.

### Contract Behavior

The current contract provides three operations:

- `create(env, analysis_id, analysis_hash)`
- `get(env, analysis_id)`
- `delete(env, analysis_id)`

It stores the analysis hash keyed by analysis ID in contract instance storage.

### Important Contract Note

`src/test.rs` does not currently match the implemented contract interface in `src/lib.rs`. It still references a `hello` method that is not present in the live contract code. The contract test file should therefore be updated before relying on `cargo test` as a validation step for the current contract behavior.

## Data Model

### Parser and Viewer Schema

The normalized model used across the parser, viewer, and persistence workflows is centered on the following concepts:

#### `meta`

Contains model-wide units and default dimensions:

- `unit`
- `wallHeight`
- `defaultWallThickness`

#### `walls`

Each wall includes:

- `id`
- `start`
- `end`
- `thickness`
- `height`
- `type`

#### `slabs`

Represents rooms/floor surfaces as polygons and centroids.

#### `labels`

Represents room or semantic labels with 3D positions.

#### `doors` and `windows`

Represents semantic openings attached to walls.

#### `openings`

Provides a normalized unified opening list for downstream consumers.

#### `graphNodes`

Represents structural connection points or wall-network endpoints/junctions.

#### `columns`

Represents explicit or inferred vertical structural supports.

## API Summary

### Backend API

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/health` | Health check |
| `POST` | `/parse` | Parse an uploaded image into normalized JSON |
| `POST` | `/analyses` | Create and persist an analysis record |
| `GET` | `/analyses` | List stored analyses |
| `GET` | `/analyses/{analysis_id}` | Retrieve one analysis |
| `GET` | `/verify/{analysis_id}` | Compare stored hash with on-chain hash |

### Frontend Internal API

| Method | Path | Purpose |
| --- | --- | --- |
| `POST` | `/api/parse-image` | Proxy image uploads to the parser backend |
| `POST` | `/api/recommendations` | Generate material recommendations and layout heuristics |

## Local Development and Setup

### Prerequisites

The repository spans Python, Node.js, and Rust toolchains. A local development environment typically needs:

- Python for the parser and FastAPI backend
- Node.js and npm for the Next.js frontend
- Rust and Cargo for the Soroban contract workspace
- Stellar CLI for on-chain contract build/invocation work

### Python Backend and Parser Setup

From the repository root:

```bash
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
pip install python-dotenv
```

Notes:

- `backend/stellar.py` imports `python-dotenv`, but `requirements.txt` does not currently list it.
- `test.py` imports `requests`, so install `requests` as well if you intend to run that utility script.
- The pinned PyTorch packages in `requirements.txt` target CUDA-enabled builds. If the local machine is CPU-only or uses a different CUDA/runtime profile, those requirements may need to be adapted.

To run the API:

```bash
uvicorn backend.main:app --reload --port 8000
```

To run the parser directly against the default test image:

```bash
python -m parser.main
```

### Frontend Setup

```bash
cd frontend
npm install
npm run dev
```

The frontend development server starts on the default Next.js port unless otherwise configured.

### Full Local Integration Note

If the goal is to run the entire stack locally, align the frontend parser proxy with the local FastAPI backend. At present, `frontend/app/api/parse-image/route.ts` points to a hosted tunnel URL rather than `http://127.0.0.1:8000/parse`.

### Contract Setup

At the workspace level:

```bash
cd contract
cargo build
```

At the contract level:

```bash
cd contract\contracts\analysis-contract
stellar contract build
```

The provided Makefile supports:

```bash
make build
make test
make fmt
make clean
```

Because `src/test.rs` is out of sync with the current contract code, update the test first before using `make test` or `cargo test` as a reliable verification path.

## Environment Variables

### Backend / Stellar

| Variable | Purpose |
| --- | --- |
| `STELLAR_ENABLE_WRITE` | Enables on-chain writes when set to `1` |
| `STELLAR_ANALYSIS_CONTRACT_ID` | Soroban contract identifier |
| `STELLAR_SOURCE_SECRET` | Source account secret used by Stellar CLI invocation |

### Frontend Recommendation Service

| Variable | Purpose |
| --- | --- |
| `GEMINI_API_KEY` | Enables LLM-generated recommendation rationales |
| `GEMINI_MODEL` | Overrides the default Gemini model (`gemini-2.5-flash`) |
| `RECOMMENDATION_COUNT` | Controls the number of ranked options returned, clamped in code |

### Current Configuration Limitation

The parser upstream URL used by `frontend/app/api/parse-image/route.ts` is currently a source constant rather than an environment variable.

## Example Assets and Working Artifacts

The repository includes several useful local artifacts:

- `floorplan.png` and `floorplan_test.png`: image inputs for parser development and testing
- `output.json`: example parser output
- `sample.json`: example structured model input, using a legacy `rooms[]` format
- `analysis.db`: SQLite database file used by the backend
- `debug_layers/`: generated parser diagnostics

## Ancillary Files

### `requirements.txt`

Defines the Python dependency set for the parser and backend layers.

### `test.py`

This is a standalone utility script that queries an external construction-materials API and prints example steel and cement rates. It is not part of the core frontend-backend-contract execution path, but it is relevant as an experimental helper for material-price exploration.

## Recommended Development Workflow

1. Start the FastAPI backend.
2. Start the Next.js frontend.
3. Confirm whether the frontend parser proxy should point to the hosted parser or the local backend.
4. Upload a floor plan image and verify the returned model.
5. Inspect validation issues, recommendations, and optimization suggestions in the viewer.
6. Persist a completed analysis through the backend.
7. Enable Stellar writes only after the contract ID, source account, and CLI installation are configured.

## Current Implementation Notes

The repository is functional as a prototype platform, but several implementation details are important for maintainers:

1. The frontend parser proxy is hardcoded to a hosted URL rather than environment-driven local configuration.
2. The frontend prefers `slabs[]` and `labels[]`; `sample.json` still reflects an older `rooms[]` format.
3. The backend uses `python-dotenv`, but it is not listed in `requirements.txt`.
4. The Soroban contract test scaffold is outdated relative to the current contract implementation.
5. The Stellar CLI path in the backend is currently Windows-specific.

## Summary

This repository is a full-stack structural analysis prototype that combines computer vision, geometry normalization, 3D visualization, heuristic reasoning, material recommendation logic, persistent analysis storage, and optional blockchain-backed integrity verification.

Its strongest architectural pattern is the separation of concerns between:

- image parsing,
- normalized model validation,
- interactive visualization,
- analysis persistence,
- verification and auditability.

For future maintenance, the most valuable areas to standardize are:

- schema versioning,
- environment-based endpoint configuration,
- contract test alignment,
- and dependency/runtime documentation for GPU versus CPU parser deployments.
