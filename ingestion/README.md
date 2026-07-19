# Ingestion pipeline

Python ingestion for Pilot 01: turn volunteer field submissions into a trustworthy image set for ML.

> Implementation remains in a private org repo. This page covers problem, design, and decisions.

---

## Problem

Training data for agricultural ML rarely arrives clean. Field submissions come via Kobo (CSV + photos), with variable quality and real risk of bad or duplicate inputs entering the active dataset.

A weak ingestion step contaminates annotation, training, and evaluation downstream.

**Goal:** raw submissions → **training-ready** images, with explicit failure handling.

---

## Role

**Data Engineer — ML Dataset Ingestion & Validation** (volunteer)

Built and hardened a pipeline that validates images/metadata, quarantines failures, deduplicates by content hash, and writes structured audit outcomes per submission.

---

## Pipeline overview

```text
Kobo export (CSV + photos)
        │
        ▼
┌───────────────────┐
│  Schema / CSV     │  required columns, robust parsing
│  checks           │
└─────────┬─────────┘
          ▼
┌───────────────────┐
│  Per-submission   │  dual photo slots (e.g. 45° + top-down)
│  image acquisition│  local copy or download
└─────────┬─────────┘
          ▼
┌───────────────────┐
│  Validation       │  size, format (magic numbers), pixels,
│  chain            │  corruption, filename/text sanitisation,
│                   │  EXIF strip + normalize to clean JPEG
└─────────┬─────────┘
          ▼
     ┌────┴────┐
     │         │
  accept    reject
     │         │
     ▼         ▼
 Processed/  Quarantine/
 + tracking  + manifest
     │
     ▼
 Audit log (disposition per submission + run summary)
```

Accepted images only enter the active dataset tracker. Rejected items never mix into that set.

---

## Design decisions

### Fail closed
Validation failures do not enter the active set. Issues are quarantined and recorded.

### Separate output concerns

| Artifact | Purpose |
|----------|---------|
| Active tracking CSV | Accepted images only |
| Quarantine + manifest | Rejected / problematic files and reasons |
| Hash registry | First-seen SHA-256 for exact duplicate detection |
| Audit log | Per-submission disposition + run summary |

### Validation order (high level)
Size and format before heavy image work → corruption checks → content hash for dedup → normalize (orientation applied, EXIF stripped) → clean JPEG outputs.

### Dispositions
Examples of explicit per-submission outcomes:

- `accepted` — expected images passed and were stored  
- `partial` — some images OK, some failed  
- `rejected` — hardening / validation failure  
- `failed` — technical / metadata failure  
- `skipped` — already processed on re-import  

### Hardening mindset
Volunteer uploads are untrusted input: extension spoofing, oversized files, decompression bombs, unsafe filename metadata, and duplicate bytes are first-class cases.

---

## Stack

Python · pandas · Pillow · requests · CSV manifests / audit trails

---

## Status

Pilot-stage ingestion with P0-style hardening for a trusted volunteer collection context. This logic is intended to feed the broader [orchestrator](../) (coming soon).
