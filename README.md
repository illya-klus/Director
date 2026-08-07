# Director

**Voice-controlled video post-production for YouTube — automate editing and creator workflows with spoken commands embedded directly into your recording.**

> **You edit while you record.**

Director lets creators control post-production without interrupting the recording process.
Instead of stopping to add markers, remember mistakes, prepare thumbnails, or write down chapter timestamps, creators simply speak explicit `DIRECTOR` commands while recording.

Director recognizes those commands, converts them into a validated edit plan, and executes the result deterministically with FFmpeg.

---

## The Problem

Recording content and managing post-production are usually two separate workflows.

While recording, creators often need to remember things like:

* where a failed take happened;
* where a new chapter starts;
* which frame could make a good thumbnail;
* what needs to be edited later.

That creates interruptions and additional manual work after recording.

Director moves those instructions directly into the recording itself.

---

## The Idea

A creator can simply say:

```text
DIRECTOR RETAKE

DIRECTOR CHAPTER Authentication

DIRECTOR THUMBNAIL
```

Director later interprets those commands and turns them into real post-production actions.

```text
raw.mp4
   ↓
Director
   ↓
final.mp4
thumbnail candidates
chapter metadata
YouTube sync
```

The recording becomes its own post-production instruction stream.

---

# Supported Commands

The current version focuses on three commands.

## `DIRECTOR RETAKE`

Removes a failed take.

Example:

```text
Today we're going to configure authentication using...

DIRECTOR RETAKE

Today we're going to configure authentication using Google OAuth.
```

Director identifies the previous failed take and proposes a cut.

When confidence is high, the failed take and the spoken `DIRECTOR RETAKE` command are removed automatically.

When confidence is low, Director keeps the content instead of making a destructive guess.

---

## `DIRECTOR CHAPTER <title>`

Creates a YouTube chapter marker.

Example:

```text
DIRECTOR CHAPTER Authentication
```

Director:

1. detects the command;
2. removes the spoken command from the final video;
3. creates a chapter marker;
4. maps the marker from the raw timeline to the edited timeline;
5. generates YouTube-compatible chapter metadata;
6. optionally syncs it to an existing YouTube video description.

Director automatically synthesizes:

```text
00:00 Intro
```

so a recording with two spoken chapter commands can produce a valid three-chapter YouTube layout.

---

## `DIRECTOR THUMBNAIL`

Captures a thumbnail candidate during recording.

Example:

```text
DIRECTOR THUMBNAIL

😲
```

Director extracts a frame from the **raw recording** shortly after the command.

The spoken command and thumbnail pose are removed from the final edited video.

Multiple thumbnail commands produce multiple candidates:

```text
thumbnail_1.jpg
thumbnail_2.jpg
thumbnail_3.jpg
```

---

# Recording Protocol

For best results, pause briefly before and after a Director command.

```text
normal speech

~300–500 ms pause

DIRECTOR ...

~300–500 ms pause

continue speaking
```

These pauses make command removal and edit boundaries significantly more reliable.

---

# Architecture

Director follows a simple principle:

> **Speech recognition never edits media directly.**

The system first understands what was said, converts it into an inspectable edit plan, validates that plan, and only then executes media operations.

```text
RAW VIDEO
    ↓
INGEST
    ├── media metadata
    ├── transcript
    └── silence map
    ↓
UNDERSTAND
    ↓
Command Parser
    ↓
PLAN
    ↓
edit-plan.raw.json
    ↓
NORMALIZE
    ↓
VALIDATE / RESOLVE
    ↓
edit-plan.resolved.json
    ↓
TimelineMapper
    ↓
EXECUTE
    ↓
FFmpeg Renderer
    ↓
final.mp4
    +
thumbnail candidates
    ↓
YouTube metadata sync
```

---

# Deterministic Editing

Director does **not** use an LLM to decide how the video should be edited.

AI is only used for speech transcription.

```text
Speech
  ↓
faster-whisper
  ↓
Timestamped transcript
```

Everything after transcription is deterministic:

```text
Transcript
   ↓
Commands
   ↓
EditPlan
   ↓
Validation
   ↓
FFmpeg
```

This makes every editing decision inspectable and reproducible.

---

# EditPlan

One of Director's core concepts is the separation between what the system **understood** and what it actually **executed**.

## Raw Edit Plan

```text
edit-plan.raw.json
```

Contains:

* detected commands;
* command removal ranges;
* proposed retake cuts;
* raw chapter markers;
* thumbnail markers;
* confidence values.

This represents:

> **What Director heard.**

---

## Resolved Edit Plan

```text
edit-plan.resolved.json
```

Contains:

* normalized cuts;
* resolved conflicts;
* final chapter positions;
* thumbnail outputs;
* confidence decisions;
* warnings.

This represents:

> **What Director actually executed.**

No destructive media operation happens before the edit plan is resolved.

---

# Transcript Model

The transcript is represented as a first-class data model.

```text
Transcript
└── words[]
    ├── text
    ├── start
    └── end
```

This allows command parsing, RETAKE matching, payload extraction, and timeline operations to use the same normalized representation.

Transcript results are cached so Director does not need to rerun Whisper while debugging or reprocessing the same file.

---

# Silence Detection

Director calculates a silence map once during ingest.

```text
silence-map.json
```

The same silence map is used for:

* command removal;
* RETAKE boundaries;
* thumbnail pose removal.

This keeps editing boundaries consistent across the system.

---

# RETAKE Resolution

`DIRECTOR RETAKE` is the most advanced command in the current version.

Director:

1. finds the corrected take after the command;
2. searches backward for the previous similar take;
3. considers text similarity and proximity;
4. uses silence boundaries to refine the cut;
5. calculates confidence;
6. returns a proposed retake cut.

Conceptually:

```text
corrected take
      ↓
backward transcript search
      ↓
similar previous fragment
      ↓
silence boundary snapping
      ↓
ProposedRetakeCut
```

The RETAKE resolver does not modify the EditPlan directly.

```text
edit_planner.py
      ↓
retake.resolve(...)
      ↓
ProposedRetakeCut
```

This keeps RETAKE isolated and easy to disable if needed.

---

# Safe Failure

Director follows a non-destructive philosophy.

```text
HIGH confidence
→ apply automatically

MEDIUM confidence
→ review / keep by default

LOW confidence
→ keep content
```

An unresolved RETAKE does not fail the entire processing pipeline.

Example:

```text
✓ Thumbnail captured
✓ Chapters generated
✓ Retake #1 removed
⚠ Retake #2 kept — low confidence

final.mp4 generated successfully
```

Uncertainty should never destroy user content.

---

# Timeline Mapping

Cuts change all timestamps after them.

Director therefore maps metadata from the raw timeline to the final edited timeline.

Conceptually:

```text
finalTime =
rawTime
-
duration of all cuts before rawTime
```

Before mapping, cuts are always:

```text
sorted
↓
merged
↓
validated
```

The TimelineMapper only accepts sorted, non-overlapping ranges.

---

# Conflict Resolution

Director resolves editing conflicts before rendering.

### Overlapping cuts

```text
10–15
13–19
```

become:

```text
10–19
```

### Chapter inside a removed range

```text
CUT:      10–18
CHAPTER:     14
```

The chapter is moved to the end of the removed range.

### Duplicate chapter timestamps

If multiple chapters collapse to the same final timestamp:

```text
keep first
drop duplicate
emit warning
```

### Thumbnail inside a removed range

This is intentionally **not a conflict**.

Thumbnail frames are always extracted from the raw recording.

---

# Rendering

Director renders the final video in one FFmpeg pass.

Instead of repeatedly editing and re-encoding:

```text
cut
render
cut
render
cut
render
```

Director first calculates all cuts:

```text
cuts
↓
normalize
↓
calculate keep segments
↓
one FFmpeg render
```

Example:

```text
CUTS
10–20
40–45
```

become:

```text
KEEP
0–10
20–40
45–end
```

The exact generated FFmpeg command is stored in:

```text
ffmpeg-command.txt
```

for debugging and reproducibility.

---

# YouTube Integration

Director can connect to YouTube using OAuth 2.0.

The current version intentionally does **not** upload videos through the YouTube API.

The workflow is:

```text
Director produces final.mp4
        ↓
Creator uploads it to YouTube
        ↓
Creator provides a video ID
        ↓
Director updates the video description
```

This keeps the hackathon version focused on Director's actual innovation: voice-controlled post-production.

---

## YouTube Chapters

Director generates YouTube-compatible timestamp lists.

Validation checks include:

* first chapter starts at `00:00`;
* at least three timestamps;
* timestamps are ordered;
* each chapter is at least 10 seconds long;
* timestamps use `MM:SS` or `HH:MM:SS`.

Example:

```text
00:00 Intro
00:18 Setup
00:43 Authentication
```

If a chapter layout is invalid, Director warns the user instead of sending a known-invalid chapter list.

---

# Tech Stack

### Backend / Media

* Python
* FastAPI
* faster-whisper
* RapidFuzz
* FFmpeg
* ffprobe
* Pydantic
* pytest

### YouTube

* YouTube Data API
* `google-api-python-client`
* `google-auth-oauthlib`

### Frontend

* React
* Vite
* TypeScript

### Storage

Hackathon version uses local filesystem storage and JSON artifacts.

No database or external queue is required.

---

# Project Structure

```text
director/
│
├── backend/
│   ├── app.py
│   │
│   ├── director/
│   │   ├── models.py
│   │   ├── transcription.py
│   │   ├── commands.py
│   │   ├── silence.py
│   │   ├── retake.py
│   │   ├── edit_planner.py
│   │   ├── plan_validator.py
│   │   ├── timeline.py
│   │   ├── ffmpeg.py
│   │   ├── renderer.py
│   │   └── youtube.py
│   │
│   └── tests/
│
├── frontend/
│
├── samples/
│   ├── happy-path.mp4
│   └── happy-path-script.md
│
├── README.md
└── .gitignore
```

---

# Runtime Artifacts

Every processing run creates an isolated directory.

```text
runs/{run-id}/

raw.mp4
audio.wav

transcript.json
silence-map.json

commands.json

edit-plan.raw.json
edit-plan.resolved.json

ffmpeg-command.txt
run.log

final.mp4

thumbnail_1.jpg
thumbnail_2.jpg

project.json
```

---

# Development Setup

## Requirements

Install:

* Python 3.11+
* Node.js
* FFmpeg
* Git

Make sure FFmpeg is available from your shell:

```bash
ffmpeg -version
ffprobe -version
```

---

## Backend

```bash
cd backend
```

Create a virtual environment:

```bash
python -m venv .venv
```

Activate it.

### Windows

```bash
.venv\Scripts\activate
```

### macOS / Linux

```bash
source .venv/bin/activate
```

Install dependencies:

```bash
pip install -r requirements.txt
```

Start the API:

```bash
uvicorn app:app --reload
```

---

## Frontend

```bash
cd frontend
npm install
npm run dev
```

---

# Environment Variables

Example:

```env
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
GOOGLE_REDIRECT_URI=http://localhost:8000/youtube/callback

WHISPER_MODEL=tiny.en

SILENCE_DB=-30

ENABLE_RETAKE=true
```

Do not commit OAuth credentials or tokens.

---

# Security

The repository must never contain:

```text
client_secret.json
credentials.json
token.json
.env
```

These should remain ignored by Git.

Example `.gitignore`:

```gitignore
runs/

.env

client_secret*.json
credentials*.json
token*.json

__pycache__/
*.pyc
```

---

# API

Current minimal API surface:

```text
POST /projects

POST /projects/{id}/process

GET /projects/{id}

GET /projects/{id}/artifacts

GET /youtube/connect

GET /youtube/callback

POST /projects/{id}/youtube/sync
```

Processing is synchronous in the MVP.

A progress streaming layer can be added later if needed.

---

# Testing

Highest-priority tests cover pure deterministic logic.

## Timeline

* raw → final timestamp mapping;
* multiple cuts;
* keep-segment calculation;
* final segment using video duration;
* invalid cut ordering.

## Plan Validation

* overlapping cuts;
* chapter inside a cut;
* duplicate chapters;
* invalid chapter layout;
* thumbnail-inside-cut behavior;
* missing chapter payload.

## RETAKE

* happy path;
* RETAKE as first utterance;
* low-confidence result;
* unresolved RETAKE remaining non-destructive.

Run tests:

```bash
pytest
```

---

# Happy Path Demo

The repository includes a prepared sample recording:

```text
samples/happy-path.mp4
```

The sample contains:

* one RETAKE;
* two spoken CHAPTER commands;
* one THUMBNAIL command;
* clear pauses around commands.

Director synthesizes the initial:

```text
00:00 Intro
```

allowing the demo to produce at least three valid chapter markers.

The corresponding script is stored in:

```text
samples/happy-path-script.md
```

---

# Demo Flow

A typical demo looks like this:

```text
failed sentence

DIRECTOR RETAKE

corrected sentence

DIRECTOR CHAPTER Setup

...

DIRECTOR CHAPTER Authentication

...

DIRECTOR THUMBNAIL

😲
```

After processing:

```text
✓ Retake removed
✓ 3 chapters prepared
✓ Thumbnail captured
```

The final video contains:

* no failed take;
* no spoken Director commands;
* no thumbnail pose.

Director also exposes:

```text
edit-plan.raw.json
        ↓
edit-plan.resolved.json
```

showing exactly how the recording was interpreted and edited.

---

# Why Director?

Most AI editing tools try to infer what the creator wanted after recording.

Director uses a different interaction model:

> **The creator tells the edit what to do while the recording is happening.**

That makes editing intent:

* explicit;
* timestamped;
* deterministic;
* inspectable;
* extensible.

Today the commands are:

```text
RETAKE
CHAPTER
THUMBNAIL
```

Tomorrow the same interaction model could support many more post-production and creator-workflow commands.

---

# Future Commands

Possible extensions include:

```text
DIRECTOR SHORT THIS

DIRECTOR NOTE

DIRECTOR B-ROLL

DIRECTOR HIGHLIGHT

DIRECTOR LINK

DIRECTOR CAPTION

DIRECTOR CENSOR

DIRECTOR CUT
```

The architecture is intentionally command-driven so new workflows can be added without changing the core interaction model.

---

# Philosophy

Director follows four principles:

### Explicit intent over AI guessing

Creators tell Director what should happen.

### Plan before execution

Every media operation is represented in an inspectable EditPlan first.

### Safe failure

Low-confidence editing decisions preserve content.

### Deterministic execution

The final edit is executed reproducibly using FFmpeg.

---

# Hackathon Goal

Director explores a simple question:

> **What if creators could control post-production without ever leaving the recording?**

Instead of switching context between creating and editing:

```text
Speak
  ↓
Understand
  ↓
Plan
  ↓
Validate
  ↓
Edit
  ↓
Sync
```

## **You edit while you record.**
