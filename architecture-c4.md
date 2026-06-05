# Visualisasi Arsitektur — C4 Model (Updated)

Project: **API SEMINAR TIF** (`hono-api-seminar`)  
Stack utama: **Bun**, **Hono/OpenAPIHono**, **Prisma**, **PostgreSQL**, **Redis** (cache + job queue), **OpenRouter** (AI), **Google Calendar API**, **Google Drive API**, **SMTP Gmail**, **Worker Job Queue (SSE)**.

Dokumen ini memvisualisasikan arsitektur backend Sistem Manajemen Seminar KP & Tugas Akhir menggunakan pendekatan **C4 Model** yang dibatasi pada 3 level: **Context**, **Container**, dan **Component**.

> **Catatan Update:** Arsitektur terbaru mengadopsi **dual-process pattern** (API Server + Background Worker), integrasi **Google Calendar API** untuk undangan jadwal, **Koordinator Dashboard**, dan **AI Constraint Chat dengan SSE streaming**.

---

## Level 1 — System Context

```mermaid
C4Context
    title System Context - API Seminar TIF

    Person(mahasiswa, "Mahasiswa", "Mendaftar seminar, upload berkas, melihat jadwal dan hasil")
    Person(dosen, "Dosen", "Melihat jadwal, mengatur constraint (termasuk via chat AI), menginput penilaian")
    Person(koordinator, "Koordinator", "Mengelola pendaftaran, jadwal, ruangan, dosen, template dokumen, approval draft, dan dashboard monitoring")

    System(api, "API Seminar TIF", "Backend REST API + Background Worker untuk manajemen seminar KP/TA Teknik Informatika")

    System_Ext(keycloak, "Keycloak", "Autentikasi & otorisasi (SSO JWT)")
    System_Ext(postgres, "PostgreSQL", "Database relasional utama")
    System_Ext(redis, "Redis", "Cache, rate limiting, dan job queue untuk background worker")
    System_Ext(openrouter, "OpenRouter", "LLM provider untuk AI scheduling & constraint chat")
    System_Ext(gcal, "Google Calendar API", "Pembuatan event & pengiriman undangan jadwal seminar ke dosen")
    System_Ext(gdrive, "Google Drive API", "Penyimpanan dokumen pendaftaran mahasiswa")
    System_Ext(smtp, "SMTP Gmail", "Pengiriman email/notifikasi pendaftaran dan jadwal")

    Rel(mahasiswa, api, "Mengakses endpoint mahasiswa/pendaftaran/upload/jadwal", "HTTPS + JWT")
    Rel(dosen, api, "Mengakses endpoint dosen/jadwal/penilaian/constraint", "HTTPS + JWT")
    Rel(koordinator, api, "Mengakses endpoint koordinator/dashboard/admin", "HTTPS + JWT")

    Rel(api, keycloak, "Validasi JWT token & otorisasi role", "OIDC / JWT")
    Rel(api, postgres, "Membaca/menulis data domain", "Prisma + PostgreSQL")
    Rel(api, redis, "Cache data, rate limiting, enqueue worker jobs", "ioredis")
    Rel(api, openrouter, "Generate draft jadwal & parse constraint chat", "HTTP API")
    Rel(api, gcal, "Buat/update event kalender & kirim undangan", "Google Calendar API v3")
    Rel(api, gdrive, "Upload/hapus file dokumen pendaftaran", "Google Drive API v3")
    Rel(api, smtp, "Kirim email pendaftaran/notifikasi", "Nodemailer SMTP")
```

### Keterangan Level 1

| Komponen | Deskripsi |
|----------|-----------|
| **Mahasiswa** | Pengguna utama yang mendaftar seminar, upload berkas, melihat jadwal |
| **Dosen** | Pembimbing/penguji yang mengatur ketersediaan (constraint), menginput penilaian |
| **Koordinator** | Staf prodi yang mengelola seluruh alur seminar dan monitoring dashboard |
| **API Seminar TIF** | Sistem utama — mencakup HTTP API Server **dan** Background Worker |
| **Keycloak** | Provider autentikasi enterprise (SSO), menghasilkan JWT token |
| **PostgreSQL** | Database utama (schema `public`) |
| **Redis** | Multi-purpose: cache query, rate limiting support, **dan** job queue untuk worker |
| **OpenRouter** | Gateway LLM untuk AI schedule generation dan constraint parsing |
| **Google Calendar API** | Membuat event kalender & mengirim undangan otomatis ke dosen terkait |
| **Google Drive API** | File storage untuk dokumen pendaftaran mahasiswa |
| **SMTP Gmail** | Email service untuk notifikasi pendaftaran |

---

## Level 2 — Container Diagram

```mermaid
C4Container
    title Container Diagram - API Seminar TIF (Dual-Process Architecture)

    Person(mahasiswa, "Mahasiswa")
    Person(dosen, "Dosen")
    Person(koordinator, "Koordinator")

    System_Boundary(system, "API Seminar TIF") {
        Container(apiServer, "HTTP API Server", "Bun + OpenAPIHono + TypeScript", "REST API, routing, middleware (auth/rate-limit/log), OpenAPI docs, error handling, SSE streams")
        Container(worker, "Background Worker", "Bun + Hono Worker + TypeScript", "Proses job queue satu-per-satu: log creation, email sending, AI scheduling, Google Calendar invitations, constraint chat parsing")
        Container(modules, "Feature Modules", "Handlers + Services + Repositories", "22 modul: jadwal, pendaftaran, penilaian, dosen, mahasiswa, ruangan, dokumen, constraint, dashboard, worker-job, dll.")
        ContainerDb(prismaClient, "Prisma Client", "Prisma 7 ORM + adapter-pg", "Data access abstraction — digunakan oleh repositories")
        Container(aiEngine, "AI Prompt Engine", "Markdown prompts + Zod schemas", "Persona, rules, task prompts, output schema validation untuk AI scheduling & constraint parsing")
    }

    ContainerDb(postgres, "PostgreSQL", "Relational DB", "Data dosen, mahasiswa, jadwal, penilaian, pendaftaran, dokumen, constraint, draft, audit log")
    ContainerDb(redis, "Redis", "Key-value store", "3 fungsi: (1) Cache query/dashboard (2) Rate-limit support/fallback (3) Worker job queue (BRPOP)")

    System_Ext(keycloak, "Keycloak", "SSO Auth Provider")
    System_Ext(openrouter, "OpenRouter API", "LLM Gateway — model fallback untuk AI scheduling & constraint chat")
    System_Ext(gcal, "Google Calendar API", "Event creation & undangan jadwal")
    System_Ext(gdrive, "Google Drive API", "File storage dokumen pendaftaran")
    System_Ext(smtp, "Gmail SMTP", "Email/notifikasi pendaftaran & jadwal")

    Rel(mahasiswa, apiServer, "HTTP requests", "HTTPS + JWT Bearer")
    Rel(dosen, apiServer, "HTTP requests", "HTTPS + JWT Bearer")
    Rel(koordinator, apiServer, "HTTP requests", "HTTPS + JWT Bearer")

    Rel(apiServer, keycloak, "Validasi JWT token", "OIDC")
    Rel(apiServer, modules, "Routes requests to feature handlers")
    Rel(apiServer, redis, "Cache read/write + Enqueue jobs (LPUSH)", "ioredis")

    Rel(worker, redis, "Dequeue jobs (BRPOP) + Update status", "ioredis")
    Rel(worker, modules, "Calls service methods untuk process job")
    Rel(worker, postgres, "Direct DB queries via Prisma", "Prisma adapter-pg")

    Rel(modules, prismaClient, "Queries/mutations")
    Rel(prismaClient, postgres, "SQL", "Prisma adapter-pg")
    Rel(modules, aiEngine, "Reads prompt files & Zod schemas")
    Rel(modules, openrouter, "Chat completion (scheduling + constraint chat)", "HTTP API")
    Rel(modules, gcal, "Create/update/delete calendar events", "Google Calendar API v3")
    Rel(modules, gdrive, "Upload/delete files", "Google Drive API v3")
    Rel(modules, smtp, "Send email", "Nodemailer SMTP")
```

### Diagram Alur Job Queue (Penjelasan Tambahan Level 2)

```mermaid
sequenceDiagram
    participant Client
    participant API as API Server
    participant Redis as Redis
    participant Worker as Background Worker
    participant Ext as External Services

    Client->>API: HTTP Request (POST jadwal / POST constraint chat)
    API->>API: Validasi input & proses bisnis
    API->>Redis: LPUSH job ke queue
    API-->>Client: 202 Accepted {job_id, status: queued}

    Client->>API: GET /worker/jobs/:job_id (SSE)
    API->>Redis: GET job status
    API-->>Client: SSE stream (heartbeat, status updates)

    Worker->>Redis: BRPOP ambil job dari queue
    Worker->>Worker: processJob() — execute berdasarkan job type
    Worker->>Ext: Call external API (OpenRouter/GCal/SMTP)
    Worker->>Redis: markCompleted() / appendProgress()
    Worker->>Worker: Update DB via Prisma

    API->>Redis: GET job status (updated)
    API-->>Client: SSE: job:completed / job:failed
```

### Keterangan Level 2

| Container | Deskripsi |
|-----------|-----------|
| **HTTP API Server** | Menerima semua HTTP requests, menjalankan middleware, routing ke handlers, mengembalikan response |
| **Background Worker** | Proses **terpisah** yang berjalan independen dari API server. Mengambil job dari Redis queue (BRPOP), memprosesnya satu per satu, dan update status ke Redis + DB |
| **Feature Modules** | 22 modul bisnis dengan struktur konsisten: Route → Handler → Service → Repository |
| **Prisma Client** | ORM layer yang digunakan oleh repositories untuk query database |
| **AI Prompt Engine** | Koleksi prompt templates (Markdown) dan Zod schemas untuk validasi output AI |
| **Redis** | 3 fungsi utama: cache, rate limiting, **dan job queue** (list BRPOP/LPUSH) |

---

## Level 3 — Component Diagram: HTTP API Server

```mermaid
C4Component
    title Component Diagram - HTTP API Server

    Container_Boundary(api, "HTTP API Server (Bun + OpenAPIHono)") {
        Component(entry, "src/index.ts", "OpenAPIHono App", "Creates app, configures CORS, OpenAPI docs, global middleware, error handlers, mounts /api router")
        Component(apiRouter, "src/api.ts", "API Router", "Aggregates semua feature routes under /api")
        Component(authMw, "AuthMiddleware", "Middleware", "JWT bearer token extraction + role-based access (mahasiswa/dosen/koordinator)")
        Component(rateMw, "RateLimitMiddleware", "Middleware", "Tiered rate limiting: global(300/min), read(120/min), write(30/min), auth(10/15min), AI(10/10min)")
        Component(logMw, "LogMiddleware", "Middleware", "Structured request logging (method, path, status, duration)")
        Component(handlers, "Handlers", "Controller layer", "Maps HTTP request/response ke service calls; extract context & params")
        Component(services, "Services", "Business logic layer", "Validasi, rules, orchestration, cache interaction, external API calls")
        Component(repositories, "Repositories", "Data access layer", "Prisma queries — findAll, findById, create, update, destroy")
        Component(validators, "Validators (Zod)", "Zod schemas", "Validates params, query, body, form data sebelum handler")
        Component(utils, "Utils / Helpers", "Shared utilities", "APIError, logger, cache keys/invalidation, tahun ajaran, jadwal/dosen/mahasiswa helpers, crypto")
        Component(sseStream, "SSE Streaming", "Hono streamSSE", "Server-Sent Events untuk real-time job status & progress updates")
    }

    ContainerDb(postgres, "PostgreSQL", "Database")
    ContainerDb(redis, "Redis", "Cache + Job Queue")
    System_Ext(keycloak, "Keycloak", "SSO Auth")

    Rel(entry, apiRouter, "Mounts /api router")
    Rel(entry, authMw, "Route-level auth via authMiddleware()")
    Rel(entry, rateMw, "Applies /api/* global rate limiter")
    Rel(entry, logMw, "Structured request logging")
    Rel(apiRouter, handlers, "Routes HTTP to handlers")
    Rel(apiRouter, validators, "Attaches zValidator for request validation")
    Rel(handlers, services, "Calls service methods")
    Rel(handlers, sseStream, "Streams SSE for long-running jobs")
    Rel(services, repositories, "Uses repositories for data access")
    Rel(services, utils, "Uses helpers/errors/logging/cache utils")
    Rel(repositories, postgres, "Prisma queries", "Prisma adapter-pg")
    Rel(services, redis, "Cache: remember/get/set/invalidate\nQueue: WorkerJobService.enqueue()", "ioredis")
    Rel(authMw, keycloak, "Validasi JWT token signature", "OIDC")
    Rel(sseStream, redis, "Polls job status for SSE stream", "ioredis")
```

---

## Level 3 — Component Diagram: Background Worker

```mermaid
C4Component
    title Component Diagram - Background Worker

    Container_Boundary(worker, "Background Worker (Bun)") {
        Component(workerEntry, "src/worker.ts", "Worker Entry Point", "Initialize Prisma + Redis, poll queue (BRPOP), dispatch jobs ke processor")
        Component(jobProcessor, "Job Processor", "Switch by WorkerJobType", "Route job ke service handler yang sesuai berdasarkan tipe job")
        Component(workerJobService, "WorkerJobService", "Job Management", "Enqueue, dequeue, markRunning/Completed/Failed, appendProgress, get status")
        Component(progressTracker, "Progress Tracker", "Event Emitter", "Emit real-time progress events yang disimpan ke Redis untuk SSE streaming")
    }

    Container_Boundary(jobTypes, "Job Type Processors") {
        Component(logJob, "log.create", "Log Processor", "Membuat audit log entries di database")
        Component(emailPendaftaran, "pendaftaran.email.send", "Email Pendaftaran", "Kirim email notifikasi pendaftaran (created/updated/validated)")
        Component(emailJadwal, "jadwal.email.send", "Email Jadwal", "Kirim email notifikasi jadwal + sync Google Calendar invitation")
        Component(aiDraft, "jadwal-draft.generate", "AI Draft Scheduler", "Generate batch draft jadwal via OpenRouter AI + validasi constraint")
        Component(constraintChat, "constraint-dosen.chat", "Constraint Chat", "Parse pesan natural language dosen → array constraint via OpenRouter AI")
        Component(constraintChatUpdate, "constraint-dosen.chat-update", "Constraint Chat Update", "Parse pesan natural language → update existing constraint via OpenRouter AI")
    }

    ContainerDb(postgres, "PostgreSQL", "Database")
    ContainerDb(redis, "Redis", "Job Queue + Status Store")
    System_Ext(openrouter, "OpenRouter", "LLM Gateway")
    System_Ext(gcal, "Google Calendar API", "Calendar invitations")
    System_Ext(smtp, "Gmail SMTP", "Email sending")

    Rel(workerEntry, redis, "BRPOP job queue + update status", "ioredis")
    Rel(workerEntry, jobProcessor, "Dispatch job by type")
    Rel(jobProcessor, logJob, "Route LOG_CREATE")
    Rel(jobProcessor, emailPendaftaran, "Route PENDAFTARAN_EMAIL_SEND")
    Rel(jobProcessor, emailJadwal, "Route JADWAL_EMAIL_SEND")
    Rel(jobProcessor, aiDraft, "Route JADWAL_DRAFT_GENERATE")
    Rel(jobProcessor, constraintChat, "Route CONSTRAINT_DOSEN_CHAT")
    Rel(jobProcessor, constraintChatUpdate, "Route CONSTRAINT_DOSEN_CHAT_UPDATE")

    Rel(logJob, postgres, "Insert log entry")
    Rel(emailPendaftaran, smtp, "Send email via Nodemailer")
    Rel(emailJadwal, smtp, "Send jadwal email")
    Rel(emailJadwal, gcal, "Create/update calendar event & invitation")
    Rel(aiDraft, openrouter, "Generate draft jadwal via LLM")
    Rel(aiDraft, postgres, "Persist draft jadwal")
    Rel(aiDraft, redis, "Cache AI context (constraint + rules)")
    Rel(constraintChat, openrouter, "Parse NL → structured constraints")
    Rel(constraintChatUpdate, openrouter, "Parse NL → update constraint")
    Rel(constraintChatUpdate, postgres, "Update constraint record")

    Rel(workerJobService, redis, "Read/write job status + progress events")
```

---

## Level 3 — Component Diagram: Feature Modules

```mermaid
C4Component
    title Component Diagram - Feature Modules (Updated)

    Person(mahasiswa, "Mahasiswa")
    Person(dosen, "Dosen")
    Person(koordinator, "Koordinator")

    Container_Boundary(features, "Feature Modules (22 modules)") {

        Component(jadwal, "Jadwal Module", "Route + V + H + S + R", "CRUD jadwal seminar + conflict detection + Google Calendar invitation enqueue")
        Component(jadwalDraft, "Jadwal Draft AI Module", "Route + V + H + S + R", "Generate batch draft jadwal via AI, approve/reject workflow, validasi constraint otomatis")
        Component(pendaftaran, "Pendaftaran Module", "Route + V + H + S + R", "Pengajuan & pengelolaan pendaftaran seminar, EAV data_pendaftaran, status tracking")
        Component(penilaian, "Penilaian Module", "Route + V + H + S + R", "Input & manajemen penilaian seminar, detail komponen nilai per role")
        Component(dosenMod, "Dosen Module", "Route + V + H + S + R", "CRUD data dosen (NIP, nama, email)")
        Component(mahasiswaMod, "Mahasiswa Module", "Route + V + H + S + R", "CRUD data mahasiswa (NIM, nama, status aktif)")
        Component(ruangan, "Ruangan Module", "Route + V + H + S + R + Helper", "Data ruangan + deteksi konflik jadwal per ruangan")
        Component(constraint, "Constraint Dosen Module", "Route + V + H + S + R", "CRUD constraint dosen + AI chat (NL parsing via OpenRouter) + SSE streaming progress")
        Component(koordinatorDashboard, "Koordinator Dashboard Module", "Route + H + S", "Statistik dashboard, semester stats, recent activity, lecturer workload monitoring")
        Component(masterData, "Master Data Modules", "Jenis Seminar + Tahun Ajaran + Bobot Penilai + Bidang Keahlian + Komponen Penilaian", "Konfigurasi master: jenis seminar, tahun ajaran, bobot penilai, bidang keahlian, komponen penilaian")
        Component(dokumenTemplate, "Dokumen Template Module", "Route + V + H + S + R", "Template dokumen pendaftaran (FILE_UPLOAD/TEXT/URL/BOOLEAN/DATE/SELECT)")
        Component(requirementDokumen, "Requirement Dokumen Module", "Route + V + H + S + R", "Requirement dokumen per jenis seminar + tahun ajaran")
        Component(upload, "Upload Module", "Route + H + S", "Upload/delete dokumen pendaftaran via Google Drive API")
        Component(log, "Audit Log Module", "Route + H + S + R", "Polymorphic audit log (entity_type + entity_id) untuk semua perubahan data")
        Component(workerJob, "Worker Job Module", "Route + H + S + SSE", "Monitor job status via polling + SSE streaming real-time progress")
    }

    ContainerDb(postgres, "PostgreSQL", "Relational DB")
    ContainerDb(redis, "Redis", "Cache + Job Queue")
    System_Ext(openrouter, "OpenRouter", "LLM AI service")
    System_Ext(gdrive, "Google Drive", "File storage")
    System_Ext(gcal, "Google Calendar", "Calendar invitations")
    System_Ext(smtp, "Gmail SMTP", "Email service")

    Rel(mahasiswa, pendaftaran, "Mengajukan pendaftaran")
    Rel(mahasiswa, jadwal, "Melihat jadwal seminar")
    Rel(mahasiswa, upload, "Upload dokumen pendaftaran")
    Rel(dosen, jadwal, "Melihat jadwal seminar")
    Rel(dosen, penilaian, "Input penilaian nilai")
    Rel(dosen, constraint, "CRUD constraint + chat AI (NL)")
    Rel(koordinator, jadwal, "Mengelola jadwal seminar")
    Rel(koordinator, jadwalDraft, "Generate/approve/reject draft AI")
    Rel(koordinator, pendaftaran, "Verifikasi pendaftaran")
    Rel(koordinator, koordinatorDashboard, "Monitoring statistik & workload")
    Rel(koordinator, masterData, "Mengelola master data")
    Rel(koordinator, dokumenTemplate, "Mengelola template dokumen")
    Rel(koordinator, ruangan, "Mengelola ruangan")
    Rel(koordinator, log, "Melihat audit log")

    Rel(jadwal, postgres, "Read/write jadwal")
    Rel(jadwal, redis, "Enqueue Google Calendar job via WorkerJobService")
    Rel(jadwalDraft, openrouter, "Generate draft jadwal via LLM")
    Rel(jadwalDraft, redis, "Cache AI scheduling context")
    Rel(jadwalDraft, postgres, "Persist draft jadwal")
    Rel(pendaftaran, postgres, "Read/write pendaftaran + EAV data")
    Rel(pendaftaran, smtp, "Enqueue email notifikasi via WorkerJobService")
    Rel(penilaian, postgres, "Read/write penilaian + detail penilaian")
    Rel(upload, gdrive, "Upload/delete files")
    Rel(upload, postgres, "Simpan metadata file")
    Rel(constraint, postgres, "Read/write constraint dosen")
    Rel(constraint, openrouter, "Parse NL → structured constraints (via WorkerJobService)")
    Rel(constraint, redis, "Enqueue constraint chat job + SSE progress")
    Rel(koordinatorDashboard, postgres, "Query stats, workload, activity")
    Rel(koordinatorDashboard, redis, "Cache dashboard stats")
    Rel(log, postgres, "Persist audit log")
    Rel(workerJob, redis, "Poll job status + SSE stream progress events")
```

---

## Level 3 — Component Diagram: Integrasi Google Calendar

```mermaid
C4Component
    title Component Diagram - Google Calendar Integration Flow

    Container_Boundary(jadwalSystem, "Jadwal Module") {
        Component(jadwalService, "JadwalService", "Business Logic", "CRUD jadwal + enqueue Google Calendar invitation saat create/update")
        Component(workerJobSvc, "WorkerJobService", "Job Queue", "Enqueue job tipe JADWAL_EMAIL_SEND ke Redis")
    }

    Container_Boundary(workerSystem, "Background Worker") {
        Component(emailJadwalJob, "JADWAL_EMAIL_SEND Processor", "Job Handler", "Ambil data jadwal, kirim email, sync Google Calendar event")
        Component(gcalInfra, "GoogleCalendarService", "Infrastructure", "Create/update/delete Google Calendar events, kirim undangan ke dosen via email")
    }

    System_Ext(gcal, "Google Calendar API", "Event & invitation management")
    System_Ext(smtp, "Gmail SMTP", "Email notification")
    ContainerDb(postgres, "PostgreSQL", "Database")
    ContainerDb(redis, "Redis", "Job Queue")

    Rel(jadwalService, workerJobSvc, "enqueueGoogleCalendarInvitation(jadwalId, action)")
    Rel(workerJobSvc, redis, "LPUSH job ke queue", "ioredis")
    Rel(emailJadwalJob, redis, "BRPOP dequeue job", "ioredis")
    Rel(emailJadwalJob, gcalInfra, "syncJadwalInvitation(jadwal, action)")
    Rel(gcalInfra, gcal, "events.insert() / events.update() / events.delete()", "Google Calendar API v3")
    Rel(gcalInfra, smtp, "Kirim email undangan ke dosen")
    Rel(emailJadwalJob, postgres, "Read jadwal + penilaian data")
```

---

## Catatan Pembacaan Diagram

### Struktur C4 yang Digunakan
- **Level 1 (System Context):** Aktor → Sistem Utama → External Systems — untuk non-teknis stakeholder
- **Level 2 (Container):** API Server + Background Worker (dual-process) + shared infrastructure — untuk technical architect
- **Level 3 (Component):** Detail internal komponen per container — untuk developer

### Perubahan dari Versi Sebelumnya
1. **Background Worker** ditambahkan sebagai container terpisah (dual-process architecture)
2. **Google Calendar API** ditambahkan sebagai external system (terpisah dari Google Drive)
3. **Worker Job Queue** (Redis BRPOP/LPUSH) menjadi jembatan antara API Server dan Worker
4. **Koordinator Dashboard Module** ditambahkan sebagai feature module baru
5. **AI Constraint Chat + SSE** streaming ditambahkan di Constraint Dosen Module
6. **Google Calendar Integration Flow** ditambahkan sebagai Level 3 diagram terpisah untuk menunjukkan integrasi kompleks
7. **Redis** diupdate fungsinya: cache + rate limit **+ job queue**
8. **Jadwal Module** diupdate: sekarang meng-enqueue Google Calendar invitation saat create/update

### Tools untuk Render Diagram
- **Mermaid C4 syntax** — gunakan Mermaid versi ≥ 10 dengan plugin C4
- **Renderers:** Mermaid Live Editor, GitHub/GitLab Markdown, VS Code (Mermaid extension), Notion, dll.
- Jika C4 syntax tidak ter-render, gunakan `C4Context`, `C4Container`, `C4Component` tags yang sesuai

### Diagram yang Tersedia
| Level | Diagram | File Location |
|-------|---------|---------------|
| Level 1 | System Context | `docs/architecture-c4.md` — Section "Level 1" |
| Level 2 | Container Diagram | `docs/architecture-c4.md` — Section "Level 2" |
| Level 2 | Job Queue Sequence | `docs/architecture-c4.md` — Section "Level 2" (bonus) |
| Level 3 | HTTP API Server Components | `docs/architecture-c4.md` — Section "Level 3: HTTP API Server" |
| Level 3 | Background Worker Components | `docs/architecture-c4.md` — Section "Level 3: Background Worker" |
| Level 3 | Feature Modules Components | `docs/architecture-c4.md` — Section "Level 3: Feature Modules" |
| Level 3 | Google Calendar Integration | `docs/architecture-c4.md` — Section "Level 3: Google Calendar Integration" |
