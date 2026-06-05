     1|# Visualisasi Arsitektur — C4 Model (Updated)
     2|
     3|Project: **API SEMINAR TIF** (`hono-api-seminar`)  
     4|Stack utama: **Bun**, **Hono/OpenAPIHono**, **Prisma**, **PostgreSQL**, **Redis** (cache + job queue), **OpenRouter** (AI), **Google Calendar API**, **Google Drive API**, **SMTP Gmail**, **Worker Job Queue (SSE)**.
     5|
     6|Dokumen ini memvisualisasikan arsitektur backend Sistem Manajemen Seminar KP & Tugas Akhir menggunakan pendekatan **C4 Model** yang dibatasi pada 3 level: **Context**, **Container**, dan **Component**.
     7|
     8|> **Catatan Update:** Arsitektur terbaru mengadopsi **dual-process pattern** (API Server + Background Worker), integrasi **Google Calendar API** untuk undangan jadwal, **Koordinator Dashboard**, dan **AI Constraint Chat dengan SSE streaming**.
     9|
    10|---
    11|
    12|## Kode Warna Universal
    13|
    14|| Warna | Representasi |
    15||-------|-------------|
    16|| 🔵 Biru | Person / Aktor Pengguna |
    17|| 🟢 Hijau | System Internal (Sistem Utama) |
    18|| 🟠 Oranye | External System / Third-party |
    19|| 🟣 Ungu | Database / Data Store |
    20|| 🔴 Merah Muda | Middleware / Security |
    21|| 🟡 Kuning | Container / Infrastructure |
    22|| 🩵 Tosca | Background Process / Worker |
    23|
    24|---
    25|
    26|## Level 1 — System Context
    27|
    28|```mermaid
    29|C4Context
    30|    %%{init: {"theme": "default", "themeVariables": {"c4Actor":"#1565C0","c4ActorBorder":"#0D47A1","c4Container":"#2E7D32","c4ContainerBorder":"#1B5E20","c4System":"#2E7D32","c4SystemBorder":"#1B5E20","c4SystemBoundary":"#F1F8E9","c4Person":"#1565C0","c4PersonBorder":"#0D47A1","c4External":"#E65100","c4ExternalBorder":"#BF360C","c4Db":"#6A1B9A","c4DbBorder":"#4A148C","c4Rel":"#455A64","c4Arrow":"#455A64","c4Boundary":"#F1F8E9","c4BoundaryLabel":"#1B5E20"}}}%%
    51|    title System Context - API Seminar TIF
    52|
    53|    Person(mahasiswa, "👨‍🎓 Mahasiswa", "Mendaftar seminar, upload berkas, melihat jadwal dan hasil")
    54|    Person(dosen, "👨‍🏫 Dosen", "Melihat jadwal, mengatur constraint (termasuk via chat AI), menginput penilaian")
    55|    Person(koordinator, "👩‍💼 Koordinator", "Mengelola pendaftaran, jadwal, ruangan, dosen, template dokumen, approval draft, dan dashboard monitoring")
    56|
    57|    System(api, "📋 API Seminar TIF", "Backend REST API + Background Worker untuk manajemen seminar KP/TA Teknik Informatika")
    58|
    59|    System_Ext(keycloak, "🔐 Keycloak", "Autentikasi & otorisasi (SSO JWT)")
    60|    System_Ext(postgres, "🗄️ PostgreSQL", "Database relasional utama")
    61|    System_Ext(redis, "⚡ Redis", "Cache, rate limiting, dan job queue untuk background worker")
    62|    System_Ext(openrouter, "🤖 OpenRouter", "LLM provider untuk AI scheduling & constraint chat")
    63|    System_Ext(gcal, "📅 Google Calendar API", "Pembuatan event & pengiriman undangan jadwal seminar ke dosen")
    64|    System_Ext(gdrive, "📁 Google Drive API", "Penyimpanan dokumen pendaftaran mahasiswa")
    65|    System_Ext(smtp, "📧 SMTP Gmail", "Pengiriman email/notifikasi pendaftaran dan jadwal")
    66|
    67|    Rel(mahasiswa, api, "Mengakses endpoint mahasiswa/pendaftaran/upload/jadwal", "HTTPS + JWT")
    68|    Rel(dosen, api, "Mengakses endpoint dosen/jadwal/penilaian/constraint", "HTTPS + JWT")
    69|    Rel(koordinator, api, "Mengakses endpoint koordinator/dashboard/admin", "HTTPS + JWT")
    70|
    71|    Rel(api, keycloak, "Validasi JWT token & otorisasi role", "OIDC / JWT")
    72|    Rel(api, postgres, "Membaca/menulis data domain", "Prisma + PostgreSQL")
    73|    Rel(api, redis, "Cache data, rate limiting, enqueue worker jobs", "ioredis")
    74|    Rel(api, openrouter, "Generate draft jadwal & parse constraint chat", "HTTP API")
    75|    Rel(api, gcal, "Buat/update event kalender & kirim undangan", "Google Calendar API v3")
    76|    Rel(api, gdrive, "Upload/hapus file dokumen pendaftaran", "Google Drive API v3")
    77|    Rel(api, smtp, "Kirim email pendaftaran/notifikasi", "Nodemailer SMTP")
    78|
    79|    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
    80|```
    81|
    82|### Keterangan Level 1
    83|
    84|| Komponen | Warna | Deskripsi |
    85||----------|-------|-----------|
    86|| **Mahasiswa** 🔵 | Biru | Pengguna utama yang mendaftar seminar, upload berkas, melihat jadwal |
    87|| **Dosen** 🔵 | Biru | Pembimbing/penguji yang mengatur ketersediaan (constraint), menginput penilaian |
    88|| **Koordinator** 🔵 | Biru | Staf prodi yang mengelola seluruh alur seminar dan monitoring dashboard |
    89|| **API Seminar TIF** 🟢 | Hijau | Sistem utama — mencakup HTTP API Server **dan** Background Worker |
    90|| **Keycloak** 🟠 | Oranye | Provider autentikasi enterprise (SSO), menghasilkan JWT token |
    91|| **PostgreSQL** 🟣 | Ungu | Database utama (schema `public`) |
    92|| **Redis** 🟣 | Ungu | Multi-purpose: cache query, rate limiting support, **dan** job queue untuk worker |
    93|| **OpenRouter** 🟠 | Oranye | Gateway LLM untuk AI schedule generation dan constraint parsing |
    94|| **Google Calendar API** 🟠 | Oranye | Membuat event kalender & mengirim undangan otomatis ke dosen terkait |
    95|| **Google Drive API** 🟠 | Oranye | File storage untuk dokumen pendaftaran mahasiswa |
    96|| **SMTP Gmail** 🟠 | Oranye | Email service untuk notifikasi pendaftaran |
    97|
    98|---
    99|
   100|## Level 2 — Container Diagram
   101|
   102|```mermaid
   103|C4Container
   104|    %%{init: {"theme": "default", "themeVariables": {"c4Actor":"#1565C0","c4ActorBorder":"#0D47A1","c4Container":"#2E7D32","c4ContainerBorder":"#1B5E20","c4System":"#2E7D32","c4SystemBorder":"#1B5E20","c4SystemBoundary":"#F1F8E9","c4Person":"#1565C0","c4PersonBorder":"#0D47A1","c4External":"#E65100","c4ExternalBorder":"#BF360C","c4Db":"#6A1B9A","c4DbBorder":"#4A148C","c4Rel":"#455A64","c4Arrow":"#455A64","c4Boundary":"#F1F8E9","c4BoundaryLabel":"#1B5E20"}}}%%
   121|    title Container Diagram - API Seminar TIF (Dual-Process Architecture)
   122|
   123|    Person(mahasiswa, "👨‍🎓 Mahasiswa")
   124|    Person(dosen, "👨‍🏫 Dosen")
   125|    Person(koordinator, "👩‍💼 Koordinator")
   126|
   127|    System_Boundary(system, "API Seminar TIF") {
   128|        Container(apiServer, "🌐 HTTP API Server", "Bun + OpenAPIHono + TypeScript", "REST API, routing, middleware (auth/rate-limit/log), OpenAPI docs, error handling, SSE streams")
   129|        Container(worker, "⚙️ Background Worker", "Bun + Hono Worker + TypeScript", "Proses job queue satu-per-satu: log creation, email sending, AI scheduling, Google Calendar invitations, constraint chat parsing")
   130|        Container(modules, "📦 Feature Modules", "Handlers + Services + Repositories", "22 modul: jadwal, pendaftaran, penilaian, dosen, mahasiswa, ruangan, dokumen, constraint, dashboard, worker-job, dll.")
   131|        ContainerDb(prismaClient, "🔷 Prisma Client", "Prisma 7 ORM + adapter-pg", "Data access abstraction — digunakan oleh repositories")
   132|        Container(aiEngine, "🧠 AI Prompt Engine", "Markdown prompts + Zod schemas", "Persona, rules, task prompts, output schema validation untuk AI scheduling & constraint parsing")
   133|    }
   134|
   135|    ContainerDb(postgres, "🗄️ PostgreSQL", "Relational DB", "Data dosen, mahasiswa, jadwal, penilaian, pendaftaran, dokumen, constraint, draft, audit log")
   136|    ContainerDb(redis, "⚡ Redis", "Key-value store", "3 fungsi: (1) Cache query/dashboard (2) Rate-limit support/fallback (3) Worker job queue (BRPOP)")
   137|
   138|    System_Ext(keycloak, "🔐 Keycloak", "SSO Auth Provider")
   139|    System_Ext(openrouter, "🤖 OpenRouter API", "LLM Gateway — model fallback untuk AI scheduling & constraint chat")
   140|    System_Ext(gcal, "📅 Google Calendar API", "Event creation & undangan jadwal")
   141|    System_Ext(gdrive, "📁 Google Drive API", "File storage dokumen pendaftaran")
   142|    System_Ext(smtp, "📧 Gmail SMTP", "Email/notifikasi pendaftaran & jadwal")
   143|
   144|    Rel(mahasiswa, apiServer, "HTTP requests", "HTTPS + JWT Bearer")
   145|    Rel(dosen, apiServer, "HTTP requests", "HTTPS + JWT Bearer")
   146|    Rel(koordinator, apiServer, "HTTP requests", "HTTPS + JWT Bearer")
   147|
   148|    Rel(apiServer, keycloak, "Validasi JWT token", "OIDC")
   149|    Rel(apiServer, modules, "Routes requests to feature handlers")
   150|    Rel(apiServer, redis, "Cache read/write + Enqueue jobs (LPUSH)", "ioredis")
   151|
   152|    Rel(worker, redis, "Dequeue jobs (BRPOP) + Update status", "ioredis")
   153|    Rel(worker, modules, "Calls service methods untuk process job")
   154|    Rel(worker, postgres, "Direct DB queries via Prisma", "Prisma adapter-pg")
   155|
   156|    Rel(modules, prismaClient, "Queries/mutations")
   157|    Rel(prismaClient, postgres, "SQL", "Prisma adapter-pg")
   158|    Rel(modules, aiEngine, "Reads prompt files & Zod schemas")
   159|    Rel(modules, openrouter, "Chat completion (scheduling + constraint chat)", "HTTP API")
   160|    Rel(modules, gcal, "Create/update/delete calendar events", "Google Calendar API v3")
   161|    Rel(modules, gdrive, "Upload/delete files", "Google Drive API v3")
   162|    Rel(modules, smtp, "Send email", "Nodemailer SMTP")
   163|
   164|    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
   165|```
   166|
   167|### Diagram Alur Job Queue (Penjelasan Tambahan Level 2)
   168|
   169|```mermaid
   170|sequenceDiagram
   171|    %%{init: {"theme": "default", "themeVariables": {"c4Actor":"#1565C0","c4ActorBorder":"#0D47A1","c4Container":"#2E7D32","c4ContainerBorder":"#1B5E20","c4System":"#2E7D32","c4SystemBorder":"#1B5E20","c4SystemBoundary":"#F1F8E9","c4Person":"#1565C0","c4PersonBorder":"#0D47A1","c4External":"#E65100","c4ExternalBorder":"#BF360C","c4Db":"#6A1B9A","c4DbBorder":"#4A148C","c4Rel":"#455A64","c4Arrow":"#455A64","c4Boundary":"#F1F8E9","c4BoundaryLabel":"#1B5E20"}}}%%
   190|    participant Client as 👤 Client
   191|    participant API as 🌐 API Server
   192|    participant Redis as ⚡ Redis
   193|    participant Worker as ⚙️ Background Worker
   194|    participant Ext as 🌍 External Services
   195|
   196|    Client->>API: HTTP Request (POST jadwal / POST constraint chat)
   197|    API->>API: Validasi input & proses bisnis
   198|    API->>Redis: LPUSH job ke queue
   199|    API-->>Client: 202 Accepted {job_id, status: queued}
   200|
   201|    Client->>API: GET /worker/jobs/:job_id (SSE)
   202|    API->>Redis: GET job status
   203|    API-->>Client: SSE stream (heartbeat, status updates)
   204|
   205|    Worker->>Redis: BRPOP ambil job dari queue
   206|    Worker->>Worker: processJob() — execute berdasarkan job type
   207|    Worker->>Ext: Call external API (OpenRouter/GCal/SMTP)
   208|    Worker->>Redis: markCompleted() / appendProgress()
   209|    Worker->>Worker: Update DB via Prisma
   210|
   211|    API->>Redis: GET job status (updated)
   212|    API-->>Client: SSE: job:completed / job:failed
   213|```
   214|
   215|### Keterangan Level 2
   216|
   217|| Container | Warna | Deskripsi |
   218||-----------|-------|-----------|
   219|| **HTTP API Server** 🟦 | Biru Muda | Menerima semua HTTP requests, menjalankan middleware, routing ke handlers, mengembalikan response |
   220|| **Background Worker** 🩵 | Tosca | Proses **terpisah** yang berjalan independen dari API server. Mengambil job dari Redis queue (BRPOP), memprosesnya satu per satu |
   221|| **Feature Modules** 📦 | Kuning | 22 modul bisnis dengan struktur konsisten: Route → Handler → Service → Repository |
   222|| **Prisma Client** 🔷 | Biru Tua | ORM layer yang digunakan oleh repositories untuk query database |
   223|| **AI Prompt Engine** 🧠 | Hijau | Koleksi prompt templates (Markdown) dan Zod schemas untuk validasi output AI |
   224|| **Redis** ⚡ | Ungu | 3 fungsi utama: cache, rate limiting, **dan job queue** (list BRPOP/LPUSH) |
   225|
   226|---
   227|
   228|## Level 3 — Component Diagram: HTTP API Server
   229|
   230|```mermaid
   231|C4Component
   232|    %%{init: {"theme": "default", "themeVariables": {"c4Actor":"#1565C0","c4ActorBorder":"#0D47A1","c4Container":"#2E7D32","c4ContainerBorder":"#1B5E20","c4System":"#2E7D32","c4SystemBorder":"#1B5E20","c4SystemBoundary":"#F1F8E9","c4Person":"#1565C0","c4PersonBorder":"#0D47A1","c4External":"#E65100","c4ExternalBorder":"#BF360C","c4Db":"#6A1B9A","c4DbBorder":"#4A148C","c4Rel":"#455A64","c4Arrow":"#455A64","c4Boundary":"#F1F8E9","c4BoundaryLabel":"#1B5E20"}}}%%
   249|    title Component Diagram - HTTP API Server
   250|
   251|    Container_Boundary(api, "HTTP API Server (Bun + OpenAPIHono)") {
   252|        Component(entry, "📝 src/index.ts", "OpenAPIHono App", "Creates app, configures CORS, OpenAPI docs, global middleware, error handlers, mounts /api router")
   253|        Component(apiRouter, "🔀 src/api.ts", "API Router", "Aggregates semua feature routes under /api")
   254|        Component(authMw, "🔑 AuthMiddleware", "Middleware", "JWT bearer token extraction + role-based access (mahasiswa/dosen/koordinator)")
   255|        Component(rateMw, "⏱️ RateLimitMiddleware", "Middleware", "Tiered rate limiting: global(300/min), read(120/min), write(30/min), auth(10/15min), AI(10/10min)")
   256|        Component(logMw, "📋 LogMiddleware", "Middleware", "Structured request logging (method, path, status, duration)")
   257|        Component(handlers, "🎯 Handlers", "Controller layer", "Maps HTTP request/response ke service calls; extract context & params")
   258|        Component(services, "💼 Services", "Business logic layer", "Validasi, rules, orchestration, cache interaction, external API calls")
   259|        Component(repositories, "🔍 Repositories", "Data access layer", "Prisma queries — findAll, findById, create, update, destroy")
   260|        Component(validators, "✅ Validators (Zod)", "Zod schemas", "Validates params, query, body, form data sebelum handler")
   261|        Component(utils, "🔧 Utils / Helpers", "Shared utilities", "APIError, logger, cache keys/invalidation, tahun ajaran, jadwal/dosen/mahasiswa helpers, crypto")
   262|        Component(sseStream, "📡 SSE Streaming", "Hono streamSSE", "Server-Sent Events untuk real-time job status & progress updates")
   263|    }
   264|
   265|    ContainerDb(postgres, "🗄️ PostgreSQL", "Database")
   266|    ContainerDb(redis, "⚡ Redis", "Cache + Job Queue")
   267|    System_Ext(keycloak, "🔐 Keycloak", "SSO Auth")
   268|
   269|    Rel(entry, apiRouter, "Mounts /api router")
   270|    Rel(entry, authMw, "Route-level auth via authMiddleware()")
   271|    Rel(entry, rateMw, "Applies /api/* global rate limiter")
   272|    Rel(entry, logMw, "Structured request logging")
   273|    Rel(apiRouter, handlers, "Routes HTTP to handlers")
   274|    Rel(apiRouter, validators, "Attaches zValidator for request validation")
   275|    Rel(handlers, services, "Calls service methods")
   276|    Rel(handlers, sseStream, "Streams SSE for long-running jobs")
   277|    Rel(services, repositories, "Uses repositories for data access")
   278|    Rel(services, utils, "Uses helpers/errors/logging/cache utils")
   279|    Rel(repositories, postgres, "Prisma queries", "Prisma adapter-pg")
   280|    Rel(services, redis, "Cache: remember/get/set/invalidate\nQueue: WorkerJobService.enqueue()", "ioredis")
   281|    Rel(authMw, keycloak, "Validasi JWT token signature", "OIDC")
   282|    Rel(sseStream, redis, "Polls job status for SSE stream", "ioredis")
   283|
   284|    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
   285|```
   286|
   287|---
   288|
   289|## Level 3 — Component Diagram: Background Worker
   290|
   291|```mermaid
   292|C4Component
   293|    %%{init: {"theme": "default", "themeVariables": {"c4Actor":"#1565C0","c4ActorBorder":"#0D47A1","c4Container":"#2E7D32","c4ContainerBorder":"#1B5E20","c4System":"#2E7D32","c4SystemBorder":"#1B5E20","c4SystemBoundary":"#F1F8E9","c4Person":"#1565C0","c4PersonBorder":"#0D47A1","c4External":"#E65100","c4ExternalBorder":"#BF360C","c4Db":"#6A1B9A","c4DbBorder":"#4A148C","c4Rel":"#455A64","c4Arrow":"#455A64","c4Boundary":"#F1F8E9","c4BoundaryLabel":"#1B5E20"}}}%%
   310|    title Component Diagram - Background Worker
   311|
   312|    Container_Boundary(worker, "⚙️ Background Worker (Bun)") {
   313|        Component(workerEntry, "🚀 src/worker.ts", "Worker Entry Point", "Initialize Prisma + Redis, poll queue (BRPOP), dispatch jobs ke processor")
   314|        Component(jobProcessor, "🔀 Job Processor", "Switch by WorkerJobType", "Route job ke service handler yang sesuai berdasarkan tipe job")
   315|        Component(workerJobService, "📊 WorkerJobService", "Job Management", "Enqueue, dequeue, markRunning/Completed/Failed, appendProgress, get status")
   316|        Component(progressTracker, "📈 Progress Tracker", "Event Emitter", "Emit real-time progress events yang disimpan ke Redis untuk SSE streaming")
   317|    }
   318|
   319|    Container_Boundary(jobTypes, "📋 Job Type Processors") {
   320|        Component(logJob, "📝 log.create", "Log Processor", "Membuat audit log entries di database")
   321|        Component(emailPendaftaran, "📧 pendaftaran.email.send", "Email Pendaftaran", "Kirim email notifikasi pendaftaran (created/updated/validated)")
   322|        Component(emailJadwal, "📅 jadwal.email.send", "Email Jadwal", "Kirim email notifikasi jadwal + sync Google Calendar invitation")
   323|        Component(aiDraft, "🤖 jadwal-draft.generate", "AI Draft Scheduler", "Generate batch draft jadwal via OpenRouter AI + validasi constraint")
   324|        Component(constraintChat, "💬 constraint-dosen.chat", "Constraint Chat", "Parse pesan natural language dosen → array constraint via OpenRouter AI")
   325|        Component(constraintChatUpdate, "🔄 constraint-dosen.chat-update", "Constraint Chat Update", "Parse pesan natural language → update existing constraint via OpenRouter AI")
   326|    }
   327|
   328|    ContainerDb(postgres, "🗄️ PostgreSQL", "Database")
   329|    ContainerDb(redis, "⚡ Redis", "Job Queue + Status Store")
   330|    System_Ext(openrouter, "🤖 OpenRouter", "LLM Gateway")
   331|    System_Ext(gcal, "📅 Google Calendar API", "Calendar invitations")
   332|    System_Ext(smtp, "📧 Gmail SMTP", "Email sending")
   333|
   334|    Rel(workerEntry, redis, "BRPOP job queue + update status", "ioredis")
   335|    Rel(workerEntry, jobProcessor, "Dispatch job by type")
   336|    Rel(jobProcessor, logJob, "Route LOG_CREATE")
   337|    Rel(jobProcessor, emailPendaftaran, "Route PENDAFTARAN_EMAIL_SEND")
   338|    Rel(jobProcessor, emailJadwal, "Route JADWAL_EMAIL_SEND")
   339|    Rel(jobProcessor, aiDraft, "Route JADWAL_DRAFT_GENERATE")
   340|    Rel(jobProcessor, constraintChat, "Route CONSTRAINT_DOSEN_CHAT")
   341|    Rel(jobProcessor, constraintChatUpdate, "Route CONSTRAINT_DOSEN_CHAT_UPDATE")
   342|
   343|    Rel(logJob, postgres, "Insert log entry")
   344|    Rel(emailPendaftaran, smtp, "Send email via Nodemailer")
   345|    Rel(emailJadwal, smtp, "Send jadwal email")
   346|    Rel(emailJadwal, gcal, "Create/update calendar event & invitation")
   347|    Rel(aiDraft, openrouter, "Generate draft jadwal via LLM")
   348|    Rel(aiDraft, postgres, "Persist draft jadwal")
   349|    Rel(aiDraft, redis, "Cache AI context (constraint + rules)")
   350|    Rel(constraintChat, openrouter, "Parse NL → structured constraints")
   351|    Rel(constraintChatUpdate, openrouter, "Parse NL → update constraint")
   352|    Rel(constraintChatUpdate, postgres, "Update constraint record")
   353|
   354|    Rel(workerJobService, redis, "Read/write job status + progress events")
   355|
   356|    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
   357|```
   358|
   359|---
   360|
   361|## Level 3 — Component Diagram: Feature Modules
   362|
   363|```mermaid
   364|C4Component
   365|    %%{init: {"theme": "default", "themeVariables": {"c4Actor":"#1565C0","c4ActorBorder":"#0D47A1","c4Container":"#2E7D32","c4ContainerBorder":"#1B5E20","c4System":"#2E7D32","c4SystemBorder":"#1B5E20","c4SystemBoundary":"#F1F8E9","c4Person":"#1565C0","c4PersonBorder":"#0D47A1","c4External":"#E65100","c4ExternalBorder":"#BF360C","c4Db":"#6A1B9A","c4DbBorder":"#4A148C","c4Rel":"#455A64","c4Arrow":"#455A64","c4Boundary":"#F1F8E9","c4BoundaryLabel":"#1B5E20"}}}%%
   382|    title Component Diagram - Feature Modules (Updated)
   383|
   384|    Person(mahasiswa, "👨‍🎓 Mahasiswa")
   385|    Person(dosen, "👨‍🏫 Dosen")
   386|    Person(koordinator, "👩‍💼 Koordinator")
   387|
   388|    Container_Boundary(features, "📦 Feature Modules (22 modules)") {
   389|
   390|        Component(jadwal, "📅 Jadwal Module", "Route + V + H + S + R", "CRUD jadwal seminar + conflict detection + Google Calendar invitation enqueue")
   391|        Component(jadwalDraft, "🤖 Jadwal Draft AI Module", "Route + V + H + S + R", "Generate batch draft jadwal via AI, approve/reject workflow, validasi constraint otomatis")
   392|        Component(pendaftaran, "📋 Pendaftaran Module", "Route + V + H + S + R", "Pengajuan & pengelolaan pendaftaran seminar, EAV data_pendaftaran, status tracking")
   393|        Component(penilaian, "📊 Penilaian Module", "Route + V + H + S + R", "Input & manajemen penilaian seminar, detail komponen nilai per role")
   394|        Component(dosenMod, "👨‍🏫 Dosen Module", "Route + V + H + S + R", "CRUD data dosen (NIP, nama, email)")
   395|        Component(mahasiswaMod, "🎓 Mahasiswa Module", "Route + V + H + S + R", "CRUD data mahasiswa (NIM, nama, status aktif)")
   396|        Component(ruangan, "🏢 Ruangan Module", "Route + V + H + S + R + Helper", "Data ruangan + deteksi konflik jadwal per ruangan")
   397|        Component(constraint, "💬 Constraint Dosen Module", "Route + V + H + S + R", "CRUD constraint dosen + AI chat (NL parsing via OpenRouter) + SSE streaming progress")
   398|        Component(koordinatorDashboard, "📈 Koordinator Dashboard Module", "Route + H + S", "Statistik dashboard, semester stats, recent activity, lecturer workload monitoring")
   399|        Component(masterData, "⚙️ Master Data Modules", "Jenis Seminar + Tahun Ajaran + Bobot Penilai + Bidang Keahlian + Komponen Penilaian", "Konfigurasi master data sistem")
   400|        Component(dokumenTemplate, "📄 Dokumen Template Module", "Route + V + H + S + R", "Template dokumen pendaftaran (FILE_UPLOAD/TEXT/URL/BOOLEAN/DATE/SELECT)")
   401|        Component(requirementDokumen, "📎 Requirement Dokumen Module", "Route + V + H + S + R", "Requirement dokumen per jenis seminar + tahun ajaran")
   402|        Component(upload, "📤 Upload Module", "Route + H + S", "Upload/delete dokumen pendaftaran via Google Drive API")
   403|        Component(log, "📋 Audit Log Module", "Route + H + S + R", "Polymorphic audit log (entity_type + entity_id) untuk semua perubahan data")
   404|        Component(workerJob, "📡 Worker Job Module", "Route + H + S + SSE", "Monitor job status via polling + SSE streaming real-time progress")
   405|    }
   406|
   407|    ContainerDb(postgres, "🗄️ PostgreSQL", "Relational DB")
   408|    ContainerDb(redis, "⚡ Redis", "Cache + Job Queue")
   409|    System_Ext(openrouter, "🤖 OpenRouter", "LLM AI service")
   410|    System_Ext(gdrive, "📁 Google Drive", "File storage")
   411|    System_Ext(gcal, "📅 Google Calendar", "Calendar invitations")
   412|    System_Ext(smtp, "📧 Gmail SMTP", "Email service")
   413|
   414|    Rel(mahasiswa, pendaftaran, "Mengajukan pendaftaran")
   415|    Rel(mahasiswa, jadwal, "Melihat jadwal seminar")
   416|    Rel(mahasiswa, upload, "Upload dokumen pendaftaran")
   417|    Rel(dosen, jadwal, "Melihat jadwal seminar")
   418|    Rel(dosen, penilaian, "Input penilaian nilai")
   419|    Rel(dosen, constraint, "CRUD constraint + chat AI (NL)")
   420|    Rel(koordinator, jadwal, "Mengelola jadwal seminar")
   421|    Rel(koordinator, jadwalDraft, "Generate/approve/reject draft AI")
   422|    Rel(koordinator, pendaftaran, "Verifikasi pendaftaran")
   423|    Rel(koordinator, koordinatorDashboard, "Monitoring statistik & workload")
   424|    Rel(koordinator, masterData, "Mengelola master data")
   425|    Rel(koordinator, dokumenTemplate, "Mengelola template dokumen")
   426|    Rel(koordinator, ruangan, "Mengelola ruangan")
   427|    Rel(koordinator, log, "Melihat audit log")
   428|
   429|    Rel(jadwal, postgres, "Read/write jadwal")
   430|    Rel(jadwal, redis, "Enqueue Google Calendar job via WorkerJobService")
   431|    Rel(jadwalDraft, openrouter, "Generate draft jadwal via LLM")
   432|    Rel(jadwalDraft, redis, "Cache AI scheduling context")
   433|    Rel(jadwalDraft, postgres, "Persist draft jadwal")
   434|    Rel(pendaftaran, postgres, "Read/write pendaftaran + EAV data")
   435|    Rel(pendaftaran, smtp, "Enqueue email notifikasi via WorkerJobService")
   436|    Rel(penilaian, postgres, "Read/write penilaian + detail penilaian")
   437|    Rel(upload, gdrive, "Upload/delete files")
   438|    Rel(upload, postgres, "Simpan metadata file")
   439|    Rel(constraint, postgres, "Read/write constraint dosen")
   440|    Rel(constraint, openrouter, "Parse NL → structured constraints (via WorkerJobService)")
   441|    Rel(constraint, redis, "Enqueue constraint chat job + SSE progress")
   442|    Rel(koordinatorDashboard, postgres, "Query stats, workload, activity")
   443|    Rel(koordinatorDashboard, redis, "Cache dashboard stats")
   444|    Rel(log, postgres, "Persist audit log")
   445|    Rel(workerJob, redis, "Poll job status + SSE stream progress events")
   446|
   447|    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
   448|```
   449|
   450|---
   451|
   452|## Level 3 — Component Diagram: Integrasi Google Calendar
   453|
   454|```mermaid
   455|C4Component
   456|    %%{init: {"theme": "default", "themeVariables": {"c4Actor":"#1565C0","c4ActorBorder":"#0D47A1","c4Container":"#2E7D32","c4ContainerBorder":"#1B5E20","c4System":"#2E7D32","c4SystemBorder":"#1B5E20","c4SystemBoundary":"#F1F8E9","c4Person":"#1565C0","c4PersonBorder":"#0D47A1","c4External":"#E65100","c4ExternalBorder":"#BF360C","c4Db":"#6A1B9A","c4DbBorder":"#4A148C","c4Rel":"#455A64","c4Arrow":"#455A64","c4Boundary":"#F1F8E9","c4BoundaryLabel":"#1B5E20"}}}%%
   473|    title Component Diagram - Google Calendar Integration Flow
   474|
   475|    Container_Boundary(jadwalSystem, "📅 Jadwal Module") {
   476|        Component(jadwalService, "💼 JadwalService", "Business Logic", "CRUD jadwal + enqueue Google Calendar invitation saat create/update")
   477|        Component(workerJobSvc, "📊 WorkerJobService", "Job Queue", "Enqueue job tipe JADWAL_EMAIL_SEND ke Redis")
   478|    }
   479|
   480|    Container_Boundary(workerSystem, "⚙️ Background Worker") {
   481|        Component(emailJadwalJob, "📧 JADWAL_EMAIL_SEND Processor", "Job Handler", "Ambil data jadwal, kirim email, sync Google Calendar event")
   482|        Component(gcalInfra, "📅 GoogleCalendarService", "Infrastructure", "Create/update/delete Google Calendar events, kirim undangan ke dosen via email")
   483|    }
   484|
   485|    System_Ext(gcal, "📅 Google Calendar API", "Event & invitation management")
   486|    System_Ext(smtp, "📧 Gmail SMTP", "Email notification")
   487|    ContainerDb(postgres, "🗄️ PostgreSQL", "Database")
   488|    ContainerDb(redis, "⚡ Redis", "Job Queue")
   489|
   490|    Rel(jadwalService, workerJobSvc, "enqueueGoogleCalendarInvitation(jadwalId, action)")
   491|    Rel(workerJobSvc, redis, "LPUSH job ke queue", "ioredis")
   492|    Rel(emailJadwalJob, redis, "BRPOP dequeue job", "ioredis")
   493|    Rel(emailJadwalJob, gcalInfra, "syncJadwalInvitation(jadwal, action)")
   494|    Rel(gcalInfra, gcal, "events.insert() / events.update() / events.delete()", "Google Calendar API v3")
   495|    Rel(gcalInfra, smtp, "Kirim email undangan ke dosen")
   496|    Rel(emailJadwalJob, postgres, "Read jadwal + penilaian data")
   497|
   498|    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
   499|```
   500|
   501|