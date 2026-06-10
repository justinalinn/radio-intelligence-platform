# Architecture

The Radio Intelligence Platform is a two-service containerized system that turns trunked radio audio into transcribed, geocoded, operationally structured intelligence in real time. This document describes the processing pipeline and service topology at a conceptual level; implementation source is maintained in a private repository.

## Processing Pipeline

```
trunk-recorder (SDR decoder)
    │  upload script — POST after each call completes
    ▼
Ingestion  (multipart: audio + metadata, feed-key validated)
    ├── persist audio to the shared recordings volume
    ├── record clip in SQLite
    ├── generate an RMS waveform
    ├── bandpass tone detection → emit alert on match
    ├── push new_clip to all clients (sub-second)
    └── enqueue transcription (in-process FIFO)
              ▼
Transcription → Normalization
    ├── ASR via ElevenLabs / Deepgram / OpenAI
    ├── retain both raw and normalized transcript
    ├── push clip_updated to all clients
    └── keyword + tone alert evaluation → Telegram
              ▼
Incident Detection
    ├── ML-assisted extraction + geocoding
    ├── geofence + per-talkgroup boundary filtering
    ├── unit-identity correlation to open incidents
    ├── upsert incident; push incident_updated
    └── forward clip / incident to mobile connector

Browser (React, served from the same process)
    ├── session-authenticated gate
    ├── real-time event stream over Socket.IO
    ├── shared singleton audio engine
    └── Console · Incident CAD · Optimization · Diagnostics
```

## Service Topology

| Service | Role |
|---|---|
| **Decoder** | Decodes SDR RF into per-call audio and invokes the ingestion endpoint on completion |
| **Radio Intelligence** | API, real-time transport, transcription and intelligence workers, and the served front end |

The two services share a single named recordings volume. Operational diagnostics that stream or restart the decoder use the host container socket, mounted read-only.

## Design Principles

- **Non-blocking ingestion.** The upload path never waits on transcription or intelligence; heavy work is queued and processed asynchronously so audio reaches operators immediately.
- **Deterministic, auditable enrichment.** Normalization and learned corrections are rule-based and inspectable; ML is used for extraction and recommendations, with every run recorded for traceability.
- **Single source of truth for configuration.** Runtime configuration is watched and hot-reloaded; changes propagate to every connected client without a restart.
- **Defense in depth.** Authentication is enforced server-side on every API and audio endpoint, real-time connections require a valid session, and operational modes (read-only, kill switch) provide explicit ingestion and write controls.
- **Multi-tenant by design.** Per-system configuration overlays and scoped feed keys allow a single deployment to serve multiple radio systems and distributed decoder operators.
