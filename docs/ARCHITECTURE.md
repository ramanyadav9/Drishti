# Drishti - Complete System Architecture & Documentation

> **Drishti** (Sanskrit: "Vision") -- An AI-powered intelligent surveillance system with real-time detection, tracking, face recognition, zone monitoring, and a conversational RAG-based intelligence service.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Tech Stack](#2-tech-stack)
3. [System Architecture](#3-system-architecture)
4. [Directory Structure](#4-directory-structure)
5. [Database Schema](#5-database-schema)
6. [Backend Service (Port 8000)](#6-backend-service-port-8000)
7. [Intelligence Service (Port 8100)](#7-intelligence-service-port-8100)
8. [Frontend (Port 3000)](#8-frontend-port-3000)
9. [Data Flow & Communication](#9-data-flow--communication)
10. [AI Pipeline Deep Dive](#10-ai-pipeline-deep-dive)
11. [RAG Pipeline Deep Dive](#11-rag-pipeline-deep-dive)
12. [Query Processing Flow](#12-query-processing-flow)
13. [MCP Tools & Context Budget](#13-mcp-tools--context-budget)
14. [PDF Report System](#14-pdf-report-system)
15. [Authentication & Security](#15-authentication--security)
16. [How to Run](#16-how-to-run)
17. [Environment Variables](#17-environment-variables)
18. [API Reference](#18-api-reference)

---

## 1. Project Overview

Drishti is a full-stack AI surveillance platform built for real-time monitoring, event detection, and intelligent querying of surveillance data. The system processes live video feeds (RTSP, IP cameras, YouTube streams, webcams), detects people, recognizes faces, estimates poses, monitors zone violations, and logs everything into a PostgreSQL database.

On top of the detection pipeline, a **RAG-based Intelligence Service** lets operators ask natural language questions like _"How many people entered the server room today?"_ or _"Where was Raman last seen?"_ -- and get accurate answers backed by real surveillance data, complete with source citations.

### Core Capabilities

| Feature | Technology |
|---------|-----------|
| Object Detection | YOLOv8 (Ultralytics) |
| Multi-Object Tracking | ByteTrack (Supervision) |
| Face Recognition | InsightFace (buffalo_l) + FAISS |
| Pose Estimation | MediaPipe |
| Zone Monitoring | OpenCV polygon intersection |
| Real-time Streaming | SSE (Server-Sent Events) + MJPEG |
| Natural Language Queries | RAG (ChromaDB + Ollama Cloud LLM) |
| Report Generation | LLM-powered PDF reports (fpdf2) |
| Frontend | React + Vite + TailwindCSS |

---

## 2. Tech Stack

### Backend Service
| Component | Technology | Purpose |
|-----------|-----------|---------|
| Framework | **FastAPI** | Async REST API server |
| Database | **PostgreSQL** + asyncpg | Persistent event/camera/zone/face storage |
| ORM | **SQLAlchemy** (async) | Database models & queries |
| AI Detection | **YOLOv8** (Ultralytics) | Person/object detection |
| Tracking | **ByteTrack** (Supervision) | Multi-object tracking with persistent IDs |
| Face Recognition | **InsightFace** (buffalo_l) | Face embedding extraction |
| Face Matching | **FAISS** | Fast similarity search on face embeddings |
| Pose Estimation | **MediaPipe** | Standing/Sitting/Lying classification |
| Video Input | **OpenCV** + **yt-dlp** | RTSP, HTTP MJPEG, YouTube, webcam |
| Auth | **JWT** (python-jose) + **bcrypt** | Token-based authentication with RBAC |
| Streaming | **SSE** + **MJPEG** | Real-time event push + live video |

### Intelligence Service
| Component | Technology | Purpose |
|-----------|-----------|---------|
| Framework | **FastAPI** | Async REST API server |
| LLM | **Ollama Cloud** (devstral-2, qwen3.5) | Natural language generation |
| Vector Store | **ChromaDB** (persistent) | Semantic search over surveillance events |
| Embeddings | **all-MiniLM-L6-v2** (ONNX, local) | 384-dim embeddings, runs on CPU |
| LLM Framework | **LangChain** | RAG chain orchestration |
| PDF Generation | **fpdf2** | Branded intelligence reports |
| Database | **PostgreSQL** (read-only) | Source data for MCP SQL tools |

### Frontend
| Component | Technology | Purpose |
|-----------|-----------|---------|
| Framework | **React** 18.3 | UI components |
| Build Tool | **Vite** 5.1 | Dev server with proxy config |
| Styling | **TailwindCSS** 3.4 | Utility-first CSS |
| Charts | **Recharts** 2.12 | Analytics visualizations |
| Icons | **Lucide React** | UI iconography |
| Routing | **React Router DOM** 6.22 | Client-side navigation |

---

## 3. System Architecture

```
                    +----------------------------------------------+
                    |           USER BROWSER (Port 3000)           |
                    |         React + Vite + TailwindCSS           |
                    |                                              |
                    |  Pages: Dashboard, Cameras, Events, Zones,   |
                    |  Faces, Analytics, Chat, Settings, Login     |
                    +----------+---------------------+-------------+
                               |                     |
                      /api/v1 proxy           /intel proxy
                               |                     |
                               v                     v
                +-----------------------+  +---------------------------+
                |  BACKEND (Port 8000)  |  | INTELLIGENCE (Port 8100)  |
                |                       |  |                           |
                |  FastAPI + AI         |  |  FastAPI + RAG + LLM      |
                |                       |  |                           |
                |  +------------------+ |  |  +---------------------+  |
                |  | Auth (JWT)       | |  |  | Chat API            |  |
                |  | Camera CRUD      | |  |  | RAG Query API       |  |
                |  | Zone CRUD        | |  |  | Report API          |  |
                |  | Event Log        | |  |  +----------+----------+  |
                |  | Face Mgmt        | |  |             |             |
                |  | Analytics        | |  |  +----------v----------+  |
                |  | SSE Stream       | |  |  | RAG Pipeline        |  |
                |  | MJPEG Stream     | |  |  |                     |  |
                |  +--------+---------+ |  |  | Query Processor     |  |
                |           |           |  |  | MCP Tool Router     |  |
                |  +--------v---------+ |  |  | ChromaDB Search     |  |
                |  | AI Pipeline      | |  |  | Context Builder     |  |
                |  |                   | |  |  | LLM Generation      |  |
                |  | FrameGrabber     | |  |  +----------+----------+  |
                |  | YOLOv8           | |  |             |             |
                |  | ByteTrack        | |  |  +----------v----------+  |
                |  | InsightFace      | |  |  | Ollama Cloud API    |  |
                |  | MediaPipe        | |  |  | (devstral-2:123b)   |  |
                |  | ZoneChecker      | |  |  | (qwen3.5:397b)      |  |
                |  +--------+---------+ |  |  +---------------------+  |
                |           |           |  |                           |
                |  +--------v---------+ |  |  +---------------------+  |
                |  | Event Queue      | |  |  | ChromaDB            |  |
                |  | (thread-safe)    | |  |  | (all-MiniLM-L6-v2)  |  |
                |  +--------+---------+ |  |  | 384-dim vectors     |  |
                |           |           |  |  +---------------------+  |
                +-----------+-----------+  +------------+--------------+
                            |                           |
                            |    +----------------+     | (read-only)
                            +--->|  PostgreSQL    |<----+
                                 |  (Port 5432)   |
                                 |                |
                                 |  - users       |
                                 |  - cameras     |
                                 |  - zones       |
                                 |  - events      |
                                 |  - faces       |
                                 +----------------+
```

### Service Communication Summary

| From | To | Protocol | Purpose |
|------|-----|----------|---------|
| Frontend | Backend | HTTP REST (`/api/v1/*`) | CRUD, auth, analytics |
| Frontend | Backend | SSE (`/api/v1/events/stream`) | Real-time event push |
| Frontend | Backend | MJPEG (`/api/v1/streams/{id}/mjpeg`) | Live video |
| Frontend | Intelligence | HTTP REST (`/intel/*`) | Chat, reports |
| Backend | PostgreSQL | asyncpg | Read/write all tables |
| Backend | Intelligence | HTTP (port 8100) | Proxy chat/report requests |
| Intelligence | PostgreSQL | asyncpg (read-only) | MCP SQL tools + event sync |
| Intelligence | ChromaDB | Local (persistent) | Vector search |
| Intelligence | Ollama Cloud | HTTPS API | LLM chat completions |
| AI Pipeline | Event Queue | Thread-safe queue | Detection -> DB writes |

---

## 4. Directory Structure

```
drishti/
|-- services/
|   |-- backend/                              # Surveillance backend (port 8000)
|   |   |-- app/
|   |   |   |-- ai/                           # AI pipeline components
|   |   |   |   |-- detector.py               # YOLOv8 object detection
|   |   |   |   |-- tracker.py                # ByteTrack multi-object tracking
|   |   |   |   |-- face_recognition.py       # InsightFace + FAISS matching
|   |   |   |   |-- pose_detector.py          # MediaPipe pose estimation
|   |   |   |   |-- zone_checker.py           # Polygon zone intersection
|   |   |   |   +-- drishti_processor.py      # Main orchestrator (1 per camera)
|   |   |   |-- api/v1/                       # REST API endpoints
|   |   |   |   |-- health.py                 # GET /health
|   |   |   |   |-- cameras.py                # CRUD + start/stop/restart
|   |   |   |   |-- events.py                 # List + SSE stream
|   |   |   |   |-- zones.py                  # CRUD
|   |   |   |   |-- faces.py                  # List + update name
|   |   |   |   |-- streams.py                # MJPEG video stream
|   |   |   |   |-- analytics.py              # Summary, hourly, per-camera stats
|   |   |   |   |-- chat.py                   # Proxy to intelligence service
|   |   |   |   +-- reports.py                # Proxy to intelligence service
|   |   |   |-- core/
|   |   |   |   |-- database.py               # Async SQLAlchemy engine + sessions
|   |   |   |   +-- security.py               # JWT auth, bcrypt, RBAC
|   |   |   |-- models/
|   |   |   |   |-- camera.py                 # Camera SQLAlchemy model
|   |   |   |   |-- event.py                  # Event SQLAlchemy model
|   |   |   |   |-- zone.py                   # Zone SQLAlchemy model
|   |   |   |   |-- face.py                   # Face SQLAlchemy model
|   |   |   |   +-- track.py                  # Track (Pydantic, in-memory only)
|   |   |   |-- services/
|   |   |   |   |-- camera_manager.py         # Start/stop/restart camera processors
|   |   |   |   |-- event_processor.py        # Queue drain -> DB write + SSE broadcast
|   |   |   |   |-- event_aggregator.py       # SQL analytics queries
|   |   |   |   |-- track_state.py            # In-memory per-track state
|   |   |   |   |-- storage_service.py        # Snapshot/face image storage
|   |   |   |   +-- intelligence_client.py    # HTTP client to port 8100
|   |   |   |-- config.py                     # Pydantic BaseSettings
|   |   |   +-- main.py                       # FastAPI app entry point
|   |   |-- ai_models/                        # Pre-trained model weights
|   |   |   |-- yolov8n.pt                    # YOLOv8 nano
|   |   |   +-- buffalo_l/                    # InsightFace buffalo_l
|   |   |-- data/                             # Runtime data
|   |   |   |-- faces/                        # Face crops (JPEG)
|   |   |   |-- snapshots/                    # Event snapshots
|   |   |   +-- logs/                         # Application logs
|   |   +-- requirements.txt
|   |
|   +-- intelligence-service/                 # RAG + LLM service (port 8100)
|       |-- app/
|       |   |-- api/
|       |   |   |-- health.py                 # GET /health
|       |   |   |-- chat.py                   # POST /chat + session reports
|       |   |   |-- rag.py                    # POST /rag/query + sync
|       |   |   +-- reports.py                # Report generation + PDF
|       |   |-- rag/
|       |   |   |-- chromadb_client.py        # ChromaDB persistent client + embeddings
|       |   |   |-- embedding_service.py      # PostgreSQL -> ChromaDB sync
|       |   |   |-- pipeline.py               # Main RAG chain (MCP + vector + LLM)
|       |   |   |-- query_processor.py        # Intent detection + entity extraction
|       |   |   +-- context_builder.py        # Format docs for LLM context
|       |   |-- mcp/
|       |   |   |-- server.py                 # Intent-based tool router
|       |   |   +-- tools.py                  # 5 SQL-backed tools with context budget
|       |   |-- llm/
|       |   |   |-- ollama_client.py           # ChatOllama factory (LangChain)
|       |   |   |-- response_generator.py      # Generate response/report with fallback
|       |   |   +-- prompt_templates.py        # System prompts for RAG, reports, etc.
|       |   |-- reports/
|       |   |   |-- generator.py              # Report data gathering + LLM formatting
|       |   |   |-- pdf_export.py             # DrishtiPDF class (fpdf2)
|       |   |   +-- templates/                # Data gathering per report type
|       |   |       |-- daily_summary.py
|       |   |       |-- incident_report.py
|       |   |       +-- zone_report.py
|       |   |-- analytics/
|       |   |   |-- event_stats.py            # Event count/severity/peak queries
|       |   |   |-- person_timeline.py        # Person movement/sighting queries
|       |   |   +-- zone_analytics.py         # Zone entry/breach/loitering queries
|       |   |-- models/
|       |   |   |-- chat.py                   # ChatRequest/ChatResponse schemas
|       |   |   +-- reports.py                # ReportRequest/ReportResponse schemas
|       |   |-- config.py                     # Pydantic BaseSettings
|       |   |-- db.py                         # Async engine + session factory
|       |   +-- main.py                       # FastAPI app entry point
|       |-- data/
|       |   |-- chromadb/                     # Persistent vector store
|       |   +-- reports/                      # Generated PDF files
|       +-- requirements.txt
|
|-- frontend/                                 # React app (port 3000)
|   |-- src/
|   |   |-- pages/
|   |   |   |-- Dashboard/                   # Overview, KPIs, recent events
|   |   |   |-- Cameras/                     # Camera list, add, start/stop, live view
|   |   |   |-- Events/                      # Event log with filters
|   |   |   |-- Zones/                       # Zone list, polygon editor
|   |   |   |-- Faces/                       # Face gallery, register, identify
|   |   |   |-- Analytics/                   # Charts, report generator, saved reports
|   |   |   |-- Chat/                        # Conversational intelligence UI
|   |   |   |-- Settings/                    # App configuration
|   |   |   +-- Login.jsx                    # JWT login page
|   |   |-- components/                      # Shared UI components
|   |   |-- hooks/                           # Custom React hooks
|   |   |   |-- useChat.js                   # Chat session management
|   |   |   |-- useEvents.js                 # Event fetching
|   |   |   |-- useSSE.js                    # Real-time event subscription
|   |   |   |-- useCamera.js                 # Camera operations
|   |   |   +-- useZones.js                  # Zone operations
|   |   |-- services/                        # API client modules
|   |   |   |-- api.js                       # Base API client (auto mock fallback)
|   |   |   |-- chatApi.js                   # Intelligence service client (/intel)
|   |   |   |-- cameraApi.js                 # Camera CRUD
|   |   |   |-- eventApi.js                  # Event queries + SSE
|   |   |   |-- analyticsApi.js              # Analytics + report proxy
|   |   |   +-- mockData.js                  # Mock data for offline dev
|   |   |-- utils/
|   |   |   |-- constants.js                 # API_BASE, EVENT_TYPES, SEVERITY_COLORS
|   |   |   +-- helpers.js                   # Token management, formatting
|   |   |-- App.jsx                          # Root component with routes
|   |   +-- main.jsx                         # Vite entry point
|   |-- vite.config.js                       # Proxy: /api->8000, /intel->8100
|   |-- tailwind.config.js                   # Custom theme (drishti-* colors)
|   +-- package.json
|
|-- docs/                                    # Documentation
|-- docker/                                  # Docker compose files
|-- scripts/                                 # Utility scripts
+-- tests/                                   # Test files
```

---

## 5. Database Schema

PostgreSQL database: `drishti_db` on `localhost:5432`

### Tables

#### `users`
| Column | Type | Notes |
|--------|------|-------|
| id | Integer (PK) | Auto-increment |
| username | String (unique) | Login identifier |
| password_hash | String | bcrypt hash |
| role | String | `super_admin`, `admin`, `moderator` |
| building_id | Integer (nullable) | Multi-building support |
| is_active | Boolean | Account status |
| created_at | DateTime | Registration time |

#### `cameras`
| Column | Type | Notes |
|--------|------|-------|
| id | Integer (PK) | Auto-increment |
| name | String | Display name (e.g., "Lobby Cam") |
| rtsp_url | Text | Stream URL (RTSP, HTTP, YouTube, webcam index) |
| location | String (nullable) | Physical location |
| building | String (nullable) | Building name |
| block | String (nullable) | Block/wing |
| status | String | `running`, `stopped` |
| model_port | Integer (nullable) | GPU model assignment |
| use_ocr | Boolean | Enable OCR detection |
| use_anpr | Boolean | Enable license plate recognition |
| is_active | Boolean | Active flag |
| created_at | DateTime | Creation time |
| updated_at | DateTime | Last update |

#### `zones`
| Column | Type | Notes |
|--------|------|-------|
| id | Integer (PK) | Auto-increment |
| name | String | Zone name (e.g., "Server Room") |
| zone_type | String | `restricted` or `loitering` |
| points | JSON | `[[x1,y1], [x2,y2], ...]` normalized 0-1 coordinates |
| threshold | Integer | Loitering threshold in seconds |
| camera_id | Integer (FK) | Camera this zone belongs to |
| created_at | DateTime | Creation time |

#### `events`
| Column | Type | Notes |
|--------|------|-------|
| id | Integer (PK) | Auto-increment |
| event_type | String (indexed) | `detection`, `state_change`, `zone_entry`, `zone_exit`, `loitering_alert`, `restricted_breach`, `occupancy_summary` |
| timestamp | DateTime (indexed) | When the event occurred |
| camera_id | Integer (FK) | Source camera |
| zone_id | Integer (FK, nullable) | Related zone |
| zone_name | String (nullable) | Denormalized zone name |
| zone_type | String (nullable) | Denormalized zone type |
| face_id | String (indexed, nullable) | Face identifier (e.g., "F0001") |
| person_name | String (nullable) | Resolved person name |
| posture | String (nullable) | `standing`, `sitting`, `lying` |
| severity | Integer | 1-10 scale |
| description | Text (nullable) | Human-readable event description |
| snapshot_path | String (nullable) | Path to event snapshot image |
| loitering_seconds | Integer (nullable) | Duration in loitering zone |
| metadata_json | JSON (nullable) | Extra structured data |

#### `faces`
| Column | Type | Notes |
|--------|------|-------|
| id | Integer (PK) | Auto-increment |
| face_id | String (unique, indexed) | System ID (e.g., "F0001") |
| name | String | Person name |
| image_path | String (nullable) | Best face crop |
| embedding_index | Integer (nullable) | Position in FAISS index |
| first_seen | DateTime | First detection time |
| last_seen | DateTime | Most recent detection |
| is_registered | Boolean | True if manually registered |

---

## 6. Backend Service (Port 8000)

### Entry Point (`app/main.py`)

On startup:
1. Auto-creates all database tables via SQLAlchemy `metadata.create_all()`
2. Seeds default admin user (username: `admin`, password: `admin`)
3. Starts the event processor background task (drains event queue to DB)
4. Loads FAISS face index from disk (if exists)

### API Endpoints

#### Authentication
| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/auth/login` | Public | OAuth2 password flow, returns JWT token |

#### Cameras
| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/v1/cameras` | User | List all cameras |
| POST | `/api/v1/cameras` | Admin | Create camera |
| GET | `/api/v1/cameras/{id}` | User | Get camera details |
| PUT | `/api/v1/cameras/{id}` | Admin | Update camera |
| DELETE | `/api/v1/cameras/{id}` | Admin | Delete camera |
| POST | `/api/v1/cameras/{id}/start` | Admin | Start AI processing |
| POST | `/api/v1/cameras/{id}/stop` | Admin | Stop AI processing |
| POST | `/api/v1/cameras/{id}/restart` | Admin | Restart processor |

#### Events
| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/v1/events` | User | List events (filters: event_type, camera_id, severity_min) |
| GET | `/api/v1/events/alerts` | User | High-severity events only |
| GET | `/api/v1/events/stream` | Token (query param) | SSE real-time event stream |

#### Zones
| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/v1/zones` | User | List zones (optional camera_id filter) |
| POST | `/api/v1/zones` | Admin | Create zone |
| PUT | `/api/v1/zones/{id}` | Admin | Update zone |
| DELETE | `/api/v1/zones/{id}` | Admin | Delete zone |

#### Faces
| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/v1/faces` | User | List all known faces |
| GET | `/api/v1/faces/{face_id}` | User | Get face details |
| PUT | `/api/v1/faces/{face_id}` | User | Update person name |

#### Streams
| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/v1/streams/{camera_id}/mjpeg` | Token (query param) | Live MJPEG video stream |

#### Analytics
| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/v1/analytics/summary` | User | Total events, alerts, active cameras |
| GET | `/api/v1/analytics/hourly` | User | Hourly event breakdown (query: `date`) |
| GET | `/api/v1/analytics/cameras` | User | Per-camera event statistics |

#### Chat & Reports (Proxy to Intelligence Service)
| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/v1/chat` | User | Proxy to intelligence service `/chat` |
| POST | `/api/v1/reports/generate` | User | Proxy to intelligence service `/reports/generate` |
| GET | `/api/v1/reports/types` | User | Proxy to intelligence service `/reports/types` |

### Background Services

**Event Processor** (`services/event_processor.py`):
- Thread-safe queue (maxsize: 1000)
- Background asyncio task drains queue at 10 ticks/second
- Writes events to PostgreSQL in batches
- Broadcasts each event to SSE subscribers via `asyncio.Queue`

**Camera Manager** (`services/camera_manager.py`):
- Registry: `_processors[camera_id]` maps to `DrishtiProcessor`
- Each camera runs its own processing thread
- Methods: `start_camera()`, `stop_camera()`, `restart_camera()`, `refresh_zones()`

---

## 7. Intelligence Service (Port 8100)

### Entry Point (`app/main.py`)

On startup:
1. Creates `data/chromadb/` directory
2. Initializes ChromaDB persistent vectorstore
3. Runs initial event sync (PostgreSQL to ChromaDB)
4. Starts periodic sync task (every 60 seconds)

On shutdown:
1. Cancels periodic sync task
2. Disposes database engine

### API Endpoints

#### Health
| Method | Path | Description |
|--------|------|-------------|
| GET | `/health` | Service status + document count |

#### Chat
| Method | Path | Description |
|--------|------|-------------|
| POST | `/chat` | Conversational chat with RAG + session memory |
| POST | `/chat/sessions/{id}/report` | Generate PDF report from chat session |
| GET | `/chat/sessions/{id}/report/download` | Download generated PDF |
| DELETE | `/chat/sessions/{id}` | Delete chat session |

#### RAG
| Method | Path | Description |
|--------|------|-------------|
| POST | `/rag/query` | Direct RAG query (no conversation memory) |
| POST | `/rag/sync` | Manual event sync trigger |
| GET | `/rag/status` | Sync status (last_id, document count) |

#### Reports
| Method | Path | Description |
|--------|------|-------------|
| POST | `/reports/generate` | Generate report by type (JSON response) |
| POST | `/reports/generate/pdf` | Generate and return PDF directly |
| GET | `/reports/list` | List all saved PDF reports |
| GET | `/reports/download/{filename}` | Download specific PDF |
| GET | `/reports/types` | List available report types |

### Event Sync Process

```
PostgreSQL (events table)
         |
         | SELECT ... WHERE id > last_synced_id LIMIT 1000
         |
         v
   event_to_text(row)
   "[2026-03-07 19:09:51] Zone Entry event (severity 4) on Phone 1:
    Person entered Reception Desk.
    Face ID F0005 (Neha), posture standing,
    Zone: Reception Desk (loitering)."
         |
         v
   all-MiniLM-L6-v2 (local ONNX)
   -> 384-dimensional vector
         |
         v
   ChromaDB (persistent)
   Collection: "surveillance_events"
   Document ID: "event_{id}"
   Metadata: {event_type, severity, camera_id, zone_name, face_id, ...}
```

- Runs every 60 seconds
- Processes 1000 events per batch
- Tracks `_last_synced_id` to avoid re-embedding
- Total sync of 21K events takes ~21 cycles (~21 minutes)

---

## 8. Frontend (Port 3000)

### Vite Proxy Configuration

```javascript
// vite.config.js
server: {
  port: 3000,
  proxy: {
    '/api': {
      target: 'http://localhost:8000',   // Backend service
      changeOrigin: true,
    },
    '/intel': {
      target: 'http://localhost:8100',   // Intelligence service
      changeOrigin: true,
      rewrite: (path) => path.replace(/^\/intel/, ''),  // Strip /intel prefix
    },
  },
}
```

### Route Map

| Route | Page | Description |
|-------|------|-------------|
| `/` | Dashboard | KPI cards, recent events, camera previews |
| `/cameras` | Cameras | Camera grid with live streams, CRUD |
| `/events` | Events | Searchable event log with filters |
| `/zones` | Zones | Zone list + polygon drawing on camera feed |
| `/faces` | Faces | Face gallery, identity management |
| `/analytics` | Analytics | Recharts visualizations, report generator |
| `/chat` | Chat | Conversational intelligence with PDF export |
| `/settings` | Settings | Application configuration |
| `/login` | Login | JWT authentication |

### Key Frontend Patterns

**Mock Mode Fallback**: If the backend is unreachable (2s timeout), the frontend automatically switches to mock data so the UI remains functional during development.

**Inline Delete Confirmation**: All delete operations use a two-click pattern instead of browser `window.confirm()`:
1. First click: Button turns red and shows "Confirm?"
2. Second click: Executes the delete
3. Auto-resets after 3 seconds if not confirmed

**Chat Session Management** (`useChat` hook):
- Sessions stored in `localStorage` for persistence
- Each session tracks: `id`, `messages[]`, `createdAt`, `lastActivity`
- "Export as PDF Report" button appears when session has 2+ messages
- PDF generation calls intelligence service, LLM creates executive summary

---

## 9. Data Flow & Communication

### Flow 1: Live Video Detection to Event Storage

```
Camera Feed (RTSP/YouTube/Webcam)
    |
    v
FrameGrabber (thread, OpenCV)
    | frames -> queue (size 50)
    v
DrishtiProcessor (thread per camera)
    |
    |-> YOLOv8 Detection
    |   +-> Bounding boxes + confidence scores
    |
    |-> ByteTrack Tracking
    |   +-> Persistent track IDs across frames
    |
    |-> InsightFace + FAISS
    |   +-> Face crop -> 512-dim embedding -> match against index
    |   +-> New face? Auto-enroll with ID "F{N:04d}"
    |
    |-> MediaPipe Pose
    |   +-> Classify: standing / sitting / lying
    |
    |-> ZoneChecker
    |   +-> Check if bbox intersects zone polygon
    |   +-> Track loitering duration
    |   +-> Emit: zone_entry, zone_exit, loitering_alert, restricted_breach
    |
    +-> Event Queue (thread-safe, maxsize 1000)
            |
            v
    Event Processor (asyncio background task)
            |
            |-> Write to PostgreSQL (events table)
            +-> Broadcast via SSE to all subscribers
```

### Flow 2: User Chat Query to RAG Response

```
User types: "Where was Raman last seen?"
    |
    v
Frontend (React) -> POST /intel/chat
    | { message, session_id }
    v
Intelligence Service (port 8100)
    |
    |-> 1. Query Processor
    |   +-> Parse intent: PERSON_LOOKUP
    |   +-> Extract entities: person_name="Raman"
    |
    |-> 2. MCP Tool Router
    |   +-> Intent=PERSON_LOOKUP -> run tool_person_tracker("Raman")
    |   +-> SQL: SELECT from events WHERE person_name ILIKE '%Raman%'
    |   +-> Returns: "Raman (F0001): 377 sightings, last seen 2026-03-07 22:15:38"
    |   +-> Truncate to 1500 chars budget
    |
    |-> 3. ChromaDB Vector Search
    |   +-> Embed query -> similarity_search(k=8)
    |   +-> Returns top 8 matching event documents
    |
    |-> 4. Context Builder
    |   +-> Combine MCP tool output + vector search docs
    |   +-> Total context budget: 4000 chars
    |
    |-> 5. LLM Generation (Ollama Cloud)
    |   +-> System prompt + combined context + chat history + user query
    |   +-> Primary model: devstral-2:123b-cloud
    |   +-> Fallback: qwen3.5:397b-cloud (if primary fails)
    |
    +-> 6. Response
        +-> { answer, sources, model_used, session_id, timing_ms }
        +-> Save to in-memory session history
```

### Flow 3: Event Sync (PostgreSQL to ChromaDB)

```
Every 60 seconds:
    |
    |-> Query: SELECT * FROM events WHERE id > last_synced_id LIMIT 1000
    |
    |-> For each event row:
    |   |-> event_to_text(row) -> natural language string
    |   |-> _build_metadata(row) -> {event_type, severity, camera_id, ...}
    |   +-> ID: "event_{row.id}"
    |
    |-> Batch embed: all-MiniLM-L6-v2 (local ONNX, 384 dims)
    |
    |-> Upsert into ChromaDB collection "surveillance_events"
    |
    +-> Update _last_synced_id
```

### Flow 4: PDF Report Generation

```
User clicks "Export as PDF Report" in Chat
    |
    v
POST /intel/chat/sessions/{id}/report
    |
    |-> Retrieve session messages from memory
    |
    |-> Send conversation to LLM for executive summary
    |   +-> Prompt: "Summarize this surveillance Q&A session..."
    |
    |-> DrishtiPDF (fpdf2)
    |   |-> Header: DRISHTI INTELLIGENCE SERVICE + timestamp
    |   |-> Session details: ID, query count, date
    |   |-> Executive Summary (LLM-generated)
    |   |-> Session Transcript (Q&A pairs)
    |   +-> Footer: "Automatically generated by Drishti"
    |
    +-> Save PDF to data/reports/
        +-> Return download URL
```

---

## 10. AI Pipeline Deep Dive

### DrishtiProcessor -- The Main Orchestrator

Each camera gets its own `DrishtiProcessor` running in a dedicated thread. Here's the per-frame pipeline:

```python
# Simplified per-frame flow
frame = frame_grabber.get()            # OpenCV VideoCapture

detections = detector.detect(frame)     # YOLOv8 -> bounding boxes
detections = tracker.update(detections) # ByteTrack -> persistent IDs

for track_id, bbox in detections:
    # Face recognition
    face_crop = frame[y1:y2, x1:x2]
    face_id, name, conf = face_recognizer.process_crop(face_crop)

    # Pose estimation
    posture = pose_detector.classify(face_crop)  # standing/sitting/lying

    # Zone checking
    for zone in zones:
        if zone_checker.is_inside(bbox, zone.points):
            # Track entry/exit, loitering duration
            emit_zone_event(track_id, zone, ...)

    # State change detection (deduplicate events)
    if track_state.has_changed(track_id, face_id, posture, zones):
        emit_event(...)
```

### Video Source Support

| Source Type | URL Pattern | Notes |
|-------------|-------------|-------|
| RTSP | `rtsp://user:pass@ip:554/stream` | IP cameras, NVRs |
| HTTP MJPEG | `http://ip:port/video` | Web cameras |
| YouTube | `https://youtube.com/watch?v=...` | Via yt-dlp, max 720p |
| Webcam | `0`, `1`, `2` | Local camera index |

### Event Types & Severity

| Event Type | Severity | Trigger |
|-----------|----------|---------|
| `detection` | 2 | New person detected in frame |
| `state_change` | 3 | Identity/posture/zone changed |
| `zone_entry` | 4 | Person entered a monitored zone |
| `zone_exit` | 2 | Person left a monitored zone |
| `loitering_alert` | 7 | Person exceeded loitering threshold |
| `restricted_breach` | 9 | Person entered restricted zone |
| `occupancy_summary` | 1 | Periodic frame occupancy count |

---

## 11. RAG Pipeline Deep Dive

### Embedding Model

- **Model**: all-MiniLM-L6-v2 (via ONNX runtime)
- **Dimensions**: 384
- **Location**: Runs 100% locally on CPU -- no API calls
- **Cache**: `~/.cache/chroma/onnx_models/` (~79MB, downloads once)
- **Wrapper**: `_LangChainEmbeddingWrapper` adapts ChromaDB's `DefaultEmbeddingFunction` for LangChain `Chroma` compatibility

### Vector Store

- **Engine**: ChromaDB (persistent mode)
- **Collection**: `surveillance_events`
- **Storage**: `data/chromadb/` directory
- **Document Format**: Natural language text (e.g., _"[2026-03-07 19:09:51] Zone Entry event (severity 4) on Phone 1: Person entered Reception Desk. Face ID F0005 (Neha), posture standing, Zone: Reception Desk (loitering)."_)
- **Metadata**: `{id, event_type, severity, camera_id, zone_name, face_id, person_name, timestamp}`

### LLM Models

| Model | ID | Use Case |
|-------|-----|---------|
| Primary | `devstral-2:123b-cloud` | Fast responses (~3-5s) |
| Fallback | `qwen3.5:397b-cloud` | Powerful backup if primary fails |

- **Provider**: Ollama Cloud API (`https://api.ollama.com`)
- **Temperature**: 0.3 (focused, factual responses)
- **Timeout**: 120 seconds
- **Auth**: API key in `Authorization: Bearer` header

### Pipeline Modes

**Mode 1: MCP + RAG (default when MCP tools succeed)**
```
User Query -> Query Processor -> MCP Tools (SQL) + ChromaDB (vector)
                                       |
                              Combined Context (<=4000 chars)
                                       |
                              direct_query_prompt + LLM
                                       |
                                    Answer
```

**Mode 2: Pure RAG (fallback when MCP tools fail)**
```
User Query -> history_aware_retriever -> ChromaDB -> stuff_documents_chain -> LLM -> Answer
```

---

## 12. Query Processing Flow

### Intent Detection (`query_processor.py`)

The query processor classifies each user message into an intent and extracts relevant entities:

| Intent | Triggers | Example Queries |
|--------|----------|-----------------|
| `PERSON_LOOKUP` | Person names, face IDs, "where is", "track" | "Where was Raman last seen?" |
| `ZONE_QUERY` | Zone names, "restricted", "breach", "entry" | "Any breaches in the server room?" |
| `EVENT_SEARCH` | Event types, "show me", "list", "recent" | "Show recent loitering alerts" |
| `SUMMARY` | "summary", "overview", "total", "stats" | "Give me today's summary" |
| `GENERAL` | Everything else | "What's happening right now?" |

### Entity Extraction

From the query text, the processor extracts:
- **person_names**: Matched against known names in DB
- **face_ids**: Pattern `F\d{4}` (e.g., "F0001")
- **zone_names**: Matched against zone names in DB
- **event_types**: Matched against known event type keywords
- **time_from / time_to**: Relative time expressions ("last 24h", "today", "yesterday", "this week")

### Tool Routing (`mcp/server.py`)

Based on parsed intent, different tools are invoked:

| Intent | Tools Called |
|--------|-------------|
| PERSON_LOOKUP | `tool_person_tracker()` |
| ZONE_QUERY | `tool_zone_monitor()` + `tool_recent_events()` |
| EVENT_SEARCH | `tool_recent_events()` + `tool_event_stats()` |
| SUMMARY | `tool_event_stats()` + `tool_zone_monitor()` |
| GENERAL | `tool_event_stats()` + `tool_recent_events()` |

---

## 13. MCP Tools & Context Budget

### The Problem

LLMs have limited context windows. If you dump 21K events worth of data into the prompt, the model either truncates or loses focus. MCP tools solve this by running **precise SQL queries** that return only the relevant data.

### Context Budget System

```
Per-tool limit:    MAX_TOOL_CHARS   = 1500 characters
Per-tool rows:     MAX_ROWS_PER_TOOL = 10 rows
Total budget:      MAX_TOTAL_CONTEXT = 4000 characters across all tools
```

Each tool output is truncated to 1500 chars. When multiple tools run, their combined output is capped at 4000 chars via `_budget_combine()`.

### Available Tools

#### `tool_event_stats(time_from, time_to)`
SQL queries for event distribution:
- Count by event type
- Severity distribution
- Peak activity hours
- Total event count

Example output:
```
=== Event Statistics (last 24h) ===
Total events: 2,760
By type: zone_entry (890), detection (750), occupancy_summary (480), ...
Peak hours: 19:00 (340 events), 20:00 (280 events)
```

#### `tool_person_tracker(person_name, face_id, time_from, time_to)`
SQL queries for person movement:
- Sighting count and last seen time
- Zone visit history
- Movement timeline

Example output:
```
=== Person Tracker: Raman ===
Raman (F0001): 377 sightings
Last seen: 2026-03-07 22:15:38 on Phone 1
Zones visited: Main Entrance (45), Lobby Area (32), Reception Desk (28)
```

#### `tool_zone_monitor(zone_name, time_from, time_to)`
SQL queries for zone activity:
- Entry/exit counts
- Breach history
- Loitering statistics
- Busiest time periods

#### `tool_recent_events(event_type, zone_name, person_name, time_from, time_to, limit)`
SQL query for the most recent matching events (default limit: 10)

#### `tool_occupancy(time_from, time_to)`
SQL query for occupancy heatmap data by zone and time

### Helper: `parse_time_range()`

Converts natural language time references to datetime objects:
- `"last 24h"` -> `(now - 24h, now)`
- `"today"` -> `(midnight today, now)`
- `"yesterday"` -> `(midnight yesterday, midnight today)`
- `"last week"` -> `(now - 7d, now)`
- `"last hour"` -> `(now - 1h, now)`

---

## 14. PDF Report System

### Report Types

| Type | Template | Data Source |
|------|----------|-------------|
| `daily_summary` | `templates/daily_summary.py` | Event counts, top zones, active cameras |
| `incident_report` | `templates/incident_report.py` | Single event deep-dive |
| `zone_report` | `templates/zone_report.py` | Zone-specific activity analysis |
| `session_report` | (from chat session) | Chat Q&A transcript + LLM summary |

### DrishtiPDF Class

Custom `FPDF` subclass with branded styling:
- **Header**: "DRISHTI INTELLIGENCE SERVICE" + current UTC timestamp
- **Footer**: "This report was automatically generated by Drishti Intelligence Service" + page number
- **Methods**: `add_title()`, `add_subtitle()`, `add_section()`, `add_qa_block()`, `add_body_text()`, `add_key_value()`
- **Output**: Saved to `data/reports/` with timestamped filenames

### Session Report Flow

1. User chats with Drishti (2+ messages)
2. Clicks "Export as PDF Report"
3. Frontend calls `POST /intel/chat/sessions/{id}/report`
4. Service retrieves session messages from memory
5. Sends full conversation to LLM for executive summary generation
6. Builds PDF with:
   - Session metadata (ID, query count, date)
   - Executive summary (LLM-generated)
   - Full Q&A transcript
7. Saves PDF and returns download URL
8. Frontend triggers browser download

---

## 15. Authentication & Security

### JWT Authentication

- **Algorithm**: HS256
- **Token Expiry**: 8 hours (480 minutes)
- **Login**: `POST /auth/login` with OAuth2 password flow
- **Token Storage**: `localStorage` in frontend
- **Protected Routes**: All API endpoints except `/auth/login` and `/health`

### RBAC (Role-Based Access Control)

| Role | Capabilities |
|------|-------------|
| `super_admin` | Everything |
| `admin` | Camera/zone CRUD, start/stop cameras, face management |
| `moderator` | Read-only access to all data |

### Default Credentials

```
Username: admin
Password: admin
```

### SSE & MJPEG Auth

SSE event stream and MJPEG video stream use query parameter authentication:
```
GET /api/v1/events/stream?token=<jwt_token>
GET /api/v1/streams/{camera_id}/mjpeg?token=<jwt_token>
```

---

## 16. How to Run

### Prerequisites

- **Python** 3.12+
- **Node.js** 18+ with npm
- **PostgreSQL** 15+ running on `localhost:5432`
- **GPU** (optional): CUDA-compatible for faster YOLOv8 inference

### Step 1: Database Setup

```bash
# Create the database (PostgreSQL must be running)
createdb drishti_db

# Tables are auto-created on first backend startup
```

### Step 2: Backend Service

```bash
cd drishti/services/backend

# Install Python dependencies
pip install -r requirements.txt

# Start the backend
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

The backend will:
- Create all database tables
- Seed default admin user (admin/admin)
- Start event processor background task
- Be available at `http://localhost:8000`
- Swagger docs at `http://localhost:8000/docs`

### Step 3: Intelligence Service

```bash
cd drishti/services/intelligence-service

# Install Python dependencies
pip install -r requirements.txt

# Start the intelligence service
python -m uvicorn app.main:app --host 0.0.0.0 --port 8100 --reload
```

The intelligence service will:
- Initialize ChromaDB vectorstore
- Download embedding model (~79MB, first run only, cached at `~/.cache/chroma/onnx_models/`)
- Start syncing events from PostgreSQL to ChromaDB (1000/batch, every 60s)
- Be available at `http://localhost:8100`
- Swagger docs at `http://localhost:8100/docs`

### Step 4: Frontend

```bash
cd drishti/frontend

# Install dependencies
npm install

# Start development server
npm run dev
```

The frontend will:
- Start Vite dev server on `http://localhost:3000`
- Proxy `/api` requests to backend (port 8000)
- Proxy `/intel` requests to intelligence service (port 8100)
- Fall back to mock data if backend is unreachable

### Startup Order (Important)

```
1. PostgreSQL         (must already be running on port 5432)
2. Backend Service    (port 8000 - creates tables, starts AI pipeline)
3. Intelligence Service (port 8100 - starts event sync, needs DB data)
4. Frontend           (port 3000 - connects to both services via proxy)
```

### Quick Start (3 Terminals)

```bash
# Terminal 1: Backend
cd drishti/services/backend
uvicorn app.main:app --port 8000

# Terminal 2: Intelligence Service
cd drishti/services/intelligence-service
python -m uvicorn app.main:app --port 8100

# Terminal 3: Frontend
cd drishti/frontend
npm run dev
```

Then open `http://localhost:3000` in your browser. Login with `admin` / `admin`.

---

## 17. Environment Variables

### Backend (`services/backend/.env`)

| Variable | Default | Description |
|----------|---------|-------------|
| `DATABASE_URL` | `postgresql+asyncpg://postgres:12345@localhost:5432/drishti_db` | PostgreSQL connection string |
| `JWT_SECRET` | `change-me-jwt-secret` | JWT signing secret (change in production) |
| `JWT_ALGORITHM` | `HS256` | JWT algorithm |
| `JWT_EXPIRE_MINUTES` | `480` | Token validity (8 hours) |
| `YOLO_MODEL` | `ai_models/yolov8n.pt` | YOLOv8 model file path |
| `FACE_MODEL` | `buffalo_l` | InsightFace model name |
| `SIMILARITY_THRESHOLD` | `0.50` | Face matching confidence threshold |
| `TARGET_FPS` | `30` | Video processing frame rate |
| `DETECTION_LOG_INTERVAL_SEC` | `5.0` | Minimum seconds between duplicate events |

### Intelligence Service (`services/intelligence-service/.env`)

| Variable | Default | Description |
|----------|---------|-------------|
| `OLLAMA_API_KEY` | (required) | Ollama Cloud API key |
| `OLLAMA_BASE_URL` | `https://api.ollama.com` | Ollama API endpoint |
| `OLLAMA_PRIMARY_MODEL` | `devstral-2:123b-cloud` | Fast primary LLM model |
| `OLLAMA_FALLBACK_MODEL` | `qwen3.5:397b-cloud` | Powerful fallback LLM model |
| `EMBEDDING_MODEL` | `nomic-embed-text` | (unused -- local ONNX embedding used) |
| `CHROMADB_PATH` | `data/chromadb` | ChromaDB persistent storage directory |
| `DATABASE_URL` | `postgresql+asyncpg://postgres:12345@localhost:5432/drishti_db` | PostgreSQL (read-only access) |
| `INTELLIGENCE_PORT` | `8100` | Service port number |
| `SYNC_INTERVAL_SECONDS` | `60` | Event sync interval in seconds |

---

## 18. API Reference

### Chat API (Intelligence Service - Port 8100)

#### Send Message
```http
POST /chat
Content-Type: application/json

{
  "message": "How many people entered the server room today?",
  "session_id": "optional-uuid",
  "model_preference": "fast"
}
```

Response:
```json
{
  "answer": "Based on the surveillance data...",
  "sources": [
    {
      "content": "[2026-03-07 19:09:51] Zone Entry event...",
      "metadata": { "event_type": "zone_entry", "zone_name": "Server Room" }
    }
  ],
  "model_used": "devstral-2:123b-cloud",
  "session_id": "4b70ead8-a920-4145-b7c4-1d9961f103ec",
  "timing_ms": 3421.5
}
```

#### Generate Session Report
```http
POST /chat/sessions/{session_id}/report
```

Response:
```json
{
  "message": "Report generated",
  "filename": "session_report_20260309_193942_4b70ead8.pdf",
  "path": "data/reports/session_report_20260309_193942_4b70ead8.pdf"
}
```

#### Download Report PDF
```http
GET /chat/sessions/{session_id}/report/download
```
Returns: `application/pdf` binary

### RAG Query API

#### Direct Query (no session memory)
```http
POST /rag/query
Content-Type: application/json

{
  "text": "zone breaches in the last 24 hours",
  "top_k": 8
}
```

Response:
```json
{
  "answer": "In the last 24 hours, there were...",
  "documents": [
    {
      "content": "...",
      "metadata": { "event_type": "restricted_breach" },
      "relevance_score": 0.8542
    }
  ],
  "model_used": "devstral-2:123b-cloud",
  "timing_ms": 2150.3
}
```

#### Manual Sync
```http
POST /rag/sync
```

Response:
```json
{
  "synced": 1000,
  "total": 5000,
  "last_id": 5000
}
```

### Report API

#### Generate Report
```http
POST /reports/generate
Content-Type: application/json

{
  "report_type": "daily_summary",
  "date_from": "2026-03-07",
  "date_to": "2026-03-08"
}
```

#### List Saved Reports
```http
GET /reports/list
```

Response:
```json
{
  "reports": [
    {
      "filename": "session_report_20260309_193942_4b70ead8.pdf",
      "size_bytes": 45230,
      "created_at": "2026-03-09T19:39:42"
    }
  ]
}
```

#### Download Report
```http
GET /reports/download/{filename}
```
Returns: `application/pdf` binary

### Backend API (Port 8000)

#### Login
```http
POST /auth/login
Content-Type: application/x-www-form-urlencoded

username=admin&password=admin
```

Response:
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIs...",
  "token_type": "bearer"
}
```

All subsequent requests include:
```
Authorization: Bearer <access_token>
```

#### List Cameras
```http
GET /api/v1/cameras
Authorization: Bearer <token>
```

#### Create Camera
```http
POST /api/v1/cameras
Authorization: Bearer <token>
Content-Type: application/json

{
  "name": "Lobby Camera",
  "rtsp_url": "rtsp://192.168.1.100:554/stream",
  "location": "Building A Lobby"
}
```

#### Start Camera Processing
```http
POST /api/v1/cameras/{id}/start
Authorization: Bearer <token>
```

#### Get Events
```http
GET /api/v1/events?event_type=zone_entry&camera_id=1&severity_min=5
Authorization: Bearer <token>
```

#### SSE Event Stream
```http
GET /api/v1/events/stream?token=<jwt_token>
```
Returns: `text/event-stream` with real-time events

#### MJPEG Video Stream
```http
GET /api/v1/streams/{camera_id}/mjpeg?token=<jwt_token>
```
Returns: `multipart/x-mixed-replace` MJPEG stream

#### Analytics Summary
```http
GET /api/v1/analytics/summary
Authorization: Bearer <token>
```

Response:
```json
{
  "total_events": 21000,
  "total_alerts": 450,
  "active_cameras": 2,
  "total_cameras": 3
}
```

---

## Summary

Drishti is a production-grade AI surveillance system that combines:

1. **Real-time AI detection** (YOLOv8 + ByteTrack + InsightFace + MediaPipe) for continuous video analysis
2. **Event-driven architecture** with thread-safe queuing and SSE streaming for real-time alerts
3. **RAG-based intelligence** with MCP SQL tools + ChromaDB vector search for precise natural language querying
4. **Context-budget management** to keep LLM prompts focused and efficient
5. **Branded PDF reports** with LLM-generated executive summaries
6. **Modern React frontend** with live video streams, interactive zone drawing, and conversational AI chat

The system is designed to run on a single machine with optional GPU acceleration, processing multiple camera feeds simultaneously while maintaining a queryable intelligence layer over all historical surveillance data.

---

*Built with FastAPI, React, YOLOv8, ChromaDB, and Ollama Cloud LLM.*
