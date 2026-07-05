# Mumbai Sentinel Backend Architecture

> Version: v1.0
>
> Status: Core Backend Platform Complete
>
> This document describes the architecture of the Mumbai Sentinel backend platform. It serves as the single source of truth for backend design decisions, system boundaries, service responsibilities, and data flow.

---

# Overview

Mumbai Sentinel is a distributed Security Operations Center (SOC) platform designed to monitor, manage, stream, and retrieve surveillance footage from Mini PCs and CP-Plus IP cameras deployed across multiple locations.

The backend follows a layered architecture that separates:

- Operational Asset Inventory
- Health Monitoring
- Dashboard Aggregation
- Streaming
- Recording
- Application Services

Each layer owns a single responsibility and communicates through service boundaries rather than directly accessing data sources.

---

# High-Level Architecture

```text
                          INSTALLATION
────────────────────────────────────────────────────────────────

      Mini PC
         │
         ├──────── Camera 1
         ├──────── Camera 2
         ├──────── Camera 3
         └──────── Camera 4

                │
                ▼

        master_sites.csv
        camera_map.json

                │
                ▼

        Ingestion Pipeline
        ├── Parser
        ├── Validator
        ├── Normalizer
        ├── Exporter
        └── Asset Generator

                │
                ▼

        generated/assets.json
      (Single Source of Truth)

                │
                ▼

        Inventory Domain

                │
                ▼

         Health Domain

                │
                ▼

       Dashboard Domain

                │
                ▼

       Streaming Domain

                │
                ▼

       Recording Domain

                │
                ▼

             FastAPI

        REST API + SSE

                │
                ▼

       React Frontend
      Terminal Console
```

---

# Backend Folder Structure

```text
backend/

├── api/
│   ├── inventory.py
│   ├── health.py
│   ├── dashboard.py
│   ├── streaming.py
│   ├── recording.py
│   ├── router.py
│   └── __init__.py
│
├── inventory/
│   ├── cache.py
│   ├── loader.py
│   ├── service.py
│   ├── watcher.py
│   └── exceptions.py
│
├── health/
│   ├── cache.py
│   ├── watcher.py
│   ├── service.py
│   └── exceptions.py
│
├── dashboard/
│   ├── service.py
│   └── models.py
│
├── streaming/
│   └── service.py
│
├── recording/
│   └── service.py
│
├── models/
│   ├── requests.py
│   └── responses.py
│
└── main.py
```

---

# System Layers

## 1. Infrastructure Layer

Purpose

Provides runtime infrastructure for every backend service.

Components

- Docker Compose
- FastAPI
- MariaDB
- MediaMTX
- Watchdog
- DigitalOcean Spaces
- FFmpeg
- rclone

Responsibilities

- Container orchestration
- Network communication
- Video streaming
- Archive retrieval
- Persistent storage

---

# 2. Ingestion Layer

Purpose

Convert deployment data into standardized operational assets.

Input

```
master_sites.csv

camera_map.json
```

Output

```
generated/assets.json
```

Responsibilities

- Parse deployment files
- Validate inventory
- Normalize names
- Detect duplicates
- Generate clean asset inventory

Modules

```
parser.py

validator.py

normalizer.py

exporter.py

ingest_assets.py
```

---

# 3. Inventory Domain

Purpose

Provide access to the operational inventory.

Source

```
generated/assets.json
```

Responsibilities

- Mini PC inventory
- Camera inventory
- Asset lookup
- Asset filtering
- Cache management

Components

```
Inventory Cache

Inventory Watcher

Inventory Service

Inventory REST API
```

Important Rule

Only the Inventory Service may access the inventory cache.

Routes never read JSON files.

---

# 4. Health Domain

Purpose

Provide operational health information.

Source

```
Watchdog

↓

live_status.json
```

Responsibilities

- Mini PC status
- Camera status
- Online / Offline
- Last Online
- Health summary

Components

```
Health Cache

Health Watcher

Health Service

Health REST API
```

---

# 5. Dashboard Domain

Purpose

Aggregate multiple backend services into UI-friendly models.

Consumes

```
Inventory Service

+

Health Service
```

Produces

- Dashboard summary
- Site overview
- GIS markers
- Alert list

Important

Dashboard Service owns no data.

It aggregates existing services.

---

# 6. Streaming Domain

Purpose

Provide live video information.

Responsibilities

- Camera stream lookup
- Stream availability
- MediaMTX abstraction
- HLS URLs
- Future WebRTC support

Streaming Service should never know where inventory comes from.

---

# 7. Recording Domain

Purpose

Provide archive playback and evidence retrieval.

Responsibilities

- Search footage
- Download TS segments
- Merge recordings
- Upload merged evidence
- Generate downloadable URLs

Pipeline

```
Search

↓

Download

↓

Merge

↓

Upload

↓

Download URL
```

---

# 8. Application Layer

Future application services.

Includes

- Authentication
- Zones
- Events
- Favorites
- Audit Logs
- Notifications

---

# Data Flow

## Asset Flow

```text
master_sites.csv
camera_map.json

        │

        ▼

Ingestion Pipeline

        │

        ▼

generated/assets.json

        │

        ▼

Inventory Cache

        │

        ▼

Inventory Service

        │

        ▼

REST API
```

---

## Health Flow

```text
Mini PCs

        │

Watchdog

        │

live_status.json

        │

Health Cache

        │

Health Service

        │

REST API
```

---

## Dashboard Flow

```text
Inventory Service

        │

        ├──────────────┐

        ▼              │

Dashboard Service      │

        ▲              │

        └──────────────┘

        │

Health Service

        │

REST API
```

---

# REST API Architecture

Every request follows the same dependency chain.

```text
Browser

↓

FastAPI Route

↓

Service Layer

↓

Cache

↓

Data Source

↓

Response Model

↓

JSON
```

Routes never access files directly.

---

# Current REST Endpoints

## Inventory

```
GET /api/assets

GET /api/minipcs

GET /api/minipcs/{id}

GET /api/cameras

GET /api/cameras/{id}
```

---

## Health

```
GET /api/health/summary

GET /api/health/minipcs

GET /api/health/cameras
```

---

## Dashboard

```
GET /api/dashboard/summary

GET /api/dashboard/sites

GET /api/dashboard/map

GET /api/dashboard/alerts
```

---

## Streaming (Planned)

```
GET /api/streams

GET /api/streams/{camera_id}
```

---

## Recording (Planned)

```
POST /api/evidence/jobs

GET /api/evidence/jobs

GET /api/evidence/jobs/{id}

GET /api/playback
```

---

# Backend Features

## Infrastructure

- Docker Compose deployment
- FastAPI
- MediaMTX
- MariaDB
- Watchdog
- FFmpeg
- DigitalOcean Spaces
- rclone

---

## Inventory

- Mini PC inventory
- Camera inventory
- Inventory cache
- Automatic reload
- Validation
- Normalization

---

## Health

- Mini PC online status
- Camera online status
- Last Online timestamp
- Health summary
- Automatic refresh

---

## Dashboard

- Summary cards
- GIS markers
- Site list
- Operational alerts

---

## Streaming

- Live stream lookup
- HLS playback
- MediaMTX integration

---

## Recording

- Footage retrieval
- Archive playback
- Evidence generation
- Footage merge
- Upload
- Download links

---

# Design Principles

## Single Source of Truth

Operational inventory is generated only by the ingestion pipeline.

```
generated/assets.json
```

Everything reads from it.

Nothing writes to it except ingestion.

---

## Layer Separation

Routes

↓

Services

↓

Caches

↓

Files

Never bypass layers.

---

## Domain Ownership

Inventory Domain

Owns

- Mini PCs
- Cameras

Health Domain

Owns

- Online state
- Last Online

Dashboard Domain

Owns

- Aggregated UI models

Streaming Domain

Owns

- Stream metadata

Recording Domain

Owns

- Archive retrieval

---

## Dependency Direction

```text
API

↓

Service

↓

Cache

↓

Storage
```

Never reverse this dependency.

---

# Current Development Status

| Component | Status |
|------------|--------|
| Docker Infrastructure | ✅ Complete |
| MediaMTX | ✅ Complete |
| Watchdog | ✅ Complete |
| Ingestion Pipeline | ✅ Complete |
| Asset Generation | ✅ Complete |
| Inventory Domain | ✅ Complete |
| Health Domain | ✅ Complete |
| Inventory Cache | ✅ Complete |
| Health Cache | ✅ Complete |
| Inventory Service | ✅ Complete |
| Health Service | ✅ Complete |
| Inventory REST API | ✅ Complete |
| Health REST API | ✅ Complete |
| Dashboard Domain | 🚧 Planned |
| Streaming Domain | 🚧 Planned |
| Recording Domain | 🚧 Planned |
| Authentication | 📅 Future |
| Zones | 📅 Future |
| Events | 📅 Future |

---

# Architectural Goals

The backend is designed around four principles:

- Single Source of Truth for operational assets.
- Clear separation between infrastructure, services, and APIs.
- Backend owns business logic; clients only consume API contracts.
- Modular domains that can evolve independently without breaking existing integrations.

This architecture provides a scalable foundation for approximately 155 Mini PCs and hundreds of cameras while remaining extensible for future capabilities such as authentication, GIS zones, event management, real-time updates, and advanced analytics.