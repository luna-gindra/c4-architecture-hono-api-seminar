# C4 Level 4 — Code Diagrams

> **Hono API Seminar — Tugas Akhir: Pengembangan Sistem Penjadwalan dan Penilaian Seminar**
> UIN Suska Riau — NIM: 12250111323

---

## Kode Warna Universal

| Warna | Representasi | Hex |
|-------|-------------|-----|
| 🔵 Biru Tua | Presentation / Route Layer | `#1565C0` |
| 🟣 Ungu | Handler / Controller Layer | `#7B1FA2` |
| 🔴 Merah Muda | Middleware / Security | `#E53935` |
| 🟢 Hijau | Service / Business Logic | `#2E7D32` |
| 🟡 Kuning | Repository / Data Access | `#F9A825` |
| 🩵 Tosca | Infrastructure / External | `#00897B` |
| 🔘 Abu-abu Gelap | Database / Storage | `#37474F` |
| 🟠 Oranye | AI / Prompt Engine | `#E65100` |
| 🟢 Hijau Muda | Background Worker | `#43A047` |

---

## 4.1 Layered Architecture Overview

Sistem ini menggunakan **Layered Architecture** dengan pola **Route → Handler → Service → Repository → Database** (Prisma ORM). Semua layer berkomunikasi satu arah (unidirectional) dari atas ke bawah.

```mermaid
%%{init: {"theme": "default"}}%%
graph TB
    subgraph PRESENTATION ["🎯 Presentation Layer"]
        R["📝 Route<br/>Hono Routes + Zod Validation"]
    end

    subgraph HANDLER_LAYER ["🟣 Handler Layer"]
        H["⚡ Handler<br/>Request Extraction + Response Formatting"]
    end

    subgraph MIDDLEWARE_LAYER ["🔴 Middleware Layer"]
        AUTH["🔑 AuthMiddleware<br/>JWT Bearer Token"]
        RL["⏱️ RateLimitMiddleware<br/>IP-based + Redis"]
        LOG["📋 LogMiddleware<br/>Request/Response Logging"]
    end

    subgraph SERVICE_LAYER ["🟢 Service Layer"]
        S["💼 Service<br/>Business Logic + Validation"]
    end

    subgraph REPO_LAYER ["🟡 Repository Layer"]
        REPO["🔍 Repository<br/>Prisma Query Builder"]
    end

    subgraph INFRA_LAYER ["🩵 Infrastructure Layer"]
        DB[("🗄️ Prisma Client")]
        REDIS[("⚡ Redis")]
        SMTP["📧 Mailer Service"]
        OR["🤖 OpenRouter AI"]
        GCAL["📅 Google Calendar"]
        GDRIVE["📁 Google Drive"]
    end

    R -->|"middleware chain"| AUTH
    R -->|"rate limit"| RL
    R -->|"logging"| LOG
    R -->|"dispatch"| H
    H -->|"calls"| S
    S -->|"queries"| REPO
    S -->|"caches"| REDIS
    S -->|"sends"| SMTP
    S -->|"AI inference"| OR
    S -->|"sync calendar"| GCAL
    S -->|"file upload"| GDRIVE
    REPO -->|"SQL"| DB

    style R fill:#1565C0,color:#fff,stroke:#0D47A1,stroke-width:2px
    style H fill:#7B1FA2,color:#fff,stroke:#4A148C,stroke-width:2px
    style AUTH fill:#E53935,color:#fff,stroke:#B71C1C,stroke-width:2px
    style RL fill:#E53935,color:#fff,stroke:#B71C1C,stroke-width:2px
    style LOG fill:#E53935,color:#fff,stroke:#B71C1C,stroke-width:2px
    style S fill:#2E7D32,color:#fff,stroke:#1B5E20,stroke-width:2px
    style REPO fill:#F9A825,color:#000,stroke:#F57F17,stroke-width:2px
    style DB fill:#37474F,color:#fff,stroke:#263238,stroke-width:2px
    style REDIS fill:#D32F2F,color:#fff,stroke:#B71C1C,stroke-width:2px
    style SMTP fill:#00897B,color:#fff,stroke:#00695C,stroke-width:2px
    style OR fill:#E65100,color:#fff,stroke:#BF360C,stroke-width:2px
    style GCAL fill:#00897B,color:#fff,stroke:#00695C,stroke-width:2px
    style GDRIVE fill:#00897B,color:#fff,stroke:#00695C,stroke-width:2px

    classDef presentation fill:#1565C0,color:#fff
    classDef handler fill:#7B1FA2,color:#fff
    classDef middleware fill:#E53935,color:#fff
    classDef service fill:#2E7D32,color:#fff
    classDef repository fill:#F9A825,color:#000
    classDef infrastructure fill:#00897B,color:#fff
    classDef database fill:#37474F,color:#fff
```

---

## 4.2 Core Package — Dependency Injection Container

```mermaid
%%{init: {"theme": "default"}}%%
classDiagram
    class Container {
        -services: Map~string, ServiceDefinition~
        -resolving: Set~string~
        -instance: Container
        +getInstance(): Container
        +registerSingleton(token, factory)
        +registerTransient(token, factory)
        +registerInstance(token, instance)
        +registerClass(token, ctor, singleton)
        +resolve~T~(token): T
        +getRegisteredServices(): string[]
        +clear(): void
    }

    class ServiceDefinition {
        <<interface>>
        +factory: Factory
        +singleton: boolean
        +instance?: any
    }

    class Bootstrap {
        <<module>>
        +bootstrap(): Promise~void~
        +shutdown(): Promise~void~
        +getConfig(): Config
        +getDatabase(): PrismaClient
        +getMailer(): Transporter
        +getRedis(): RedisService
        +getLogger(): Logger
    }

    class ServiceTokens {
        <<enum>>
        CONFIG = "config"
        LOGGER = "logger"
        DATABASE = "database"
        MAILER = "mailer"
        REDIS = "redis"
    }

    Container "1" *-- "*" ServiceDefinition : contains
    Bootstrap ..> Container : registers singletons
    Bootstrap ..> ServiceTokens : resolves tokens
    ServiceTokens ..> Container : token keys

    style Container fill:#2E7D32,color:#fff,stroke:#1B5E20,stroke-width:2px
    style ServiceDefinition fill:#4CAF50,color:#fff,stroke:#388E3C,stroke-width:2px
    style Bootstrap fill:#00897B,color:#fff,stroke:#00695C,stroke-width:2px
    style ServiceTokens fill:#F9A825,color:#000,stroke:#F57F17,stroke-width:2px
```

---

## 4.3 HTTP API Server — Request Lifecycle (Class Diagram)

```mermaid
%%{init: {"theme": "default"}}%%
classDiagram
    class OpenAPIHono {
        +router: RegExpRouter
        +doc(path, info)
        +route(path, subApp)
        +use(path, middleware)
        +get(path, handler)
        +post(path, handler)
        +put(path, handler)
        +delete(path, handler)
    }

    class AuthMiddleware {
        <<static>>
        +JWTBearerTokenExtraction(c, next)
        +requireRole(...roles): Function
    }

    class RateLimitMiddleware {
        <<static>>
        +global(): Handler
        +write(): Handler
    }

    class LogMiddleware {
        <<static>>
        +structuredLogger(c, next)
        +extractNetworkInformation(c, next)
    }

    class AuthHelper {
        <<static>>
        +decodeJwtPayload(token): Record
        -padBase64(base64): string
    }

    class APIError {
        +message: string
        +status: number
        +code?: string
    }

    class GlobalHandler {
        <<static>>
        +notFound(c)
        +error(c, err)
    }

    OpenAPIHono --> AuthMiddleware : use(*)
    OpenAPIHono --> RateLimitMiddleware : use(/api/*)
    OpenAPIHono --> LogMiddleware : use(*)
    OpenAPIHono --> GlobalHandler : notFound + onError
    AuthMiddleware --> AuthHelper : decodeJwtPayload
    AuthMiddleware ..> APIError : throw on invalid token

    style OpenAPIHono fill:#1565C0,color:#fff,stroke:#0D47A1,stroke-width:2px
    style AuthMiddleware fill:#E53935,color:#fff,stroke:#B71C1C,stroke-width:2px
    style RateLimitMiddleware fill:#E53935,color:#fff,stroke:#B71C1C,stroke-width:2px
    style LogMiddleware fill:#E53935,color:#fff,stroke:#B71C1C,stroke-width:2px
    style AuthHelper fill:#FF7043,color:#fff,stroke:#E64A19,stroke-width:2px
    style APIError fill:#C62828,color:#fff,stroke:#8E0000,stroke-width:2px
    style GlobalHandler fill:#5D4037,color:#fff,stroke:#3E2723,stroke-width:2px
```

---

## 4.4 Generic Feature Module — Layered Pattern (UML)

> Setiap modul (jadwal, dosen, mahasiswa, penilaian, dll) mengikuti pola yang sama.

```mermaid
%%{init: {"theme": "default"}}%%
classDiagram
    class BaseRoute {
        <<abstract>>
        +hono: Hono
    }

    class JadwalRoute {
        +get(dosen/jadwal-saya, handler)
        +get(koordinator/jadwal/:id, handler)
        +post(koordinator/jadwal, handler)
        +put(koordinator/jadwal/:id, handler)
        +delete(koordinator/jadwal/:id, handler)
    }

    class BaseHandler {
        <<abstract>>
    }

    class JadwalHandler {
        <<static>>
        +getJadwalDosenSaya(c)
        +getStatistikDosenSaya(c)
        +getAll(c)
        +get(c)
        +post(c)
        +put(c)
        +delete(c)
        -getUserPayload(c)
        -getEmail(c)
    }

    class BaseService {
        <<abstract>>
        #logger: Logger
        #logInfo(msg, meta)
        #logError(msg, meta)
        #logDebug(msg, meta)
        #logWarn(msg, meta)
    }

    class JadwalService {
        <<static>>
        +getAll(params)
        +get(id)
        +post(data, context)
        +put(id, data, context)
        +delete(id, context)
        +getJadwalDosenSaya(email, query)
        +getJadwalMahasiswaSaya(email)
        +generateId(kodeJenis)
    }

    class BaseRepository {
        <<abstract>>
        #db: PrismaClient
        #getClient(): PrismaClient
    }

    class JadwalRepository {
        <<static>>
        +findAll(filters)
        +findById(id)
        +findLastIdByPrefix(prefix)
        +create(data)
        +update(id, data)
        +delete(id)
    }

    class ZodValidator {
        +postJadwalSchema
        +putJadwalSchema
        +getJadwalQuerySchema
        +jadwalIdParamSchema
    }

    BaseRoute <|-- JadwalRoute
    BaseHandler <|-- JadwalHandler
    BaseService <|-- JadwalService
    BaseRepository <|-- JadwalRepository
    JadwalRoute --> AuthMiddleware : middleware
    JadwalRoute --> RateLimitMiddleware : write limit
    JadwalRoute --> ZodValidator : zValidator
    JadwalRoute --> JadwalHandler : delegates
    JadwalHandler --> JadwalService : calls
    JadwalHandler --> LogService : audit logging
    JadwalService --> JadwalRepository : queries
    JadwalService --> RedisService : caching
    JadwalService --> GoogleCalendarService : sync event
    JadwalService --> WorkerJobService : enqueue email
    JadwalRepository --> PrismaClient : database

    style BaseRoute fill:#1565C0,color:#fff,stroke:#0D47A1,stroke-width:2px
    style JadwalRoute fill:#1E88E5,color:#fff,stroke:#1565C0,stroke-width:2px
    style BaseHandler fill:#7B1FA2,color:#fff,stroke:#4A148C,stroke-width:2px
    style JadwalHandler fill:#9C27B0,color:#fff,stroke:#6A1B9A,stroke-width:2px
    style BaseService fill:#2E7D32,color:#fff,stroke:#1B5E20,stroke-width:2px
    style JadwalService fill:#43A047,color:#fff,stroke:#2E7D32,stroke-width:2px
    style BaseRepository fill:#F9A825,color:#000,stroke:#F57F17,stroke-width:2px
    style JadwalRepository fill:#FFB300,color:#000,stroke:#FF8F00,stroke-width:2px
    style ZodValidator fill:#FF7043,color:#fff,stroke:#E64A19,stroke-width:2px
    style AuthMiddleware fill:#E53935,color:#fff,stroke:#B71C1C,stroke-width:1px
    style RateLimitMiddleware fill:#E53935,color:#fff,stroke:#B71C1C,stroke-width:1px
    style LogService fill:#546E7A,color:#fff,stroke:#37474F,stroke-width:1px
    style RedisService fill:#D32F2F,color:#fff,stroke:#B71C1C,stroke-width:1px
    style GoogleCalendarService fill:#00897B,color:#fff,stroke:#00695C,stroke-width:1px
    style WorkerJobService fill:#00897B,color:#fff,stroke:#00695C,stroke-width:1px
    style PrismaClient fill:#37474F,color:#fff,stroke:#263238,stroke-width:1px
```

---

## 4.5 Infrastructure Layer — Singleton Services

```mermaid
%%{init: {"theme": "default"}}%%
classDiagram
    class DatabaseService {
        <<Prisma Infrastructure>>
        -client: PrismaClient
        +getPrisma(): PrismaClient
        +disconnect(): Promise~void~
        note: Uses PrismaPg adapter<br/>+ Accelerate extension
    }

    class RedisService {
        <<Redis Infrastructure>>
        -client: RedisClient
        -isConnected: boolean
        -unavailable: boolean
        +connect(): Promise~void~
        +disconnect(): Promise~void~
        +getJson~T~(key): T | null
        +setJson(key, value, ttl?)
        +deleteKey(key): boolean
        +getRawClient(): RedisClient
        +isHealthy(): boolean
        +namespacedKey(key): string
        note: ioredis singleton<br/>+ JSON cache layer<br/>+ graceful degradation
    }

    class MailService {
        <<Mail Infrastructure>>
        -transporter: Transporter
        +sendMail(options): Promise
        +close(): Promise~void~
        note: Nodemailer SMTP (port 465)<br/>+ dev email sink override
    }

    class OpenRouterService {
        <<OpenRouter Infrastructure>>
        -config: OpenRouterConfig
        +initialize(): Promise~void~
        +chatCompletion(options): Promise~Response~
        note: gpt-4o-mini model<br/>+ retry with backoff<br/>+ structured output (Zod)
    }

    class GoogleCalendarService {
        <<Google Calendar Infrastructure>>
        -calendar: calendar_v3.Calendar
        +getInstance(): GoogleCalendarService
        +syncJadwalInvitation(jadwal, action): Promise~Result~
        -getCalendarClient(): Calendar
        -buildAttendees(jadwal): Attendee[]
        note: Google APIs JWT auth<br/>+ calendar.events.insert
    }

    class GoogleDriveService {
        <<Google Drive Infrastructure>>
        -drive: drive_v3.Drive
        +getInstance(): GoogleDriveService
        +uploadFile(name, mimeType, folderId, content)
        +createFolder(name, parentId)
        note: Google APIs JWT auth<br/>+ folder-based file storage
    }

    class IoredisRateLimitStore {
        <<Rate Limit Store>>
        +prefix: string
        +windowMs: number
        -fallback: MemoryStore
        +init(options)
        +increment(key): ClientRateLimitInfo
        +decrement(key)
        +resetKey(key)
        note: Custom Store impl<br/>hono-rate-limiter<br/>INCR + EXPIRE pattern
    }

    RedisService <.. IoredisRateLimitStore : uses
    GoogleCalendarService ..> RedisService : may cache

    style DatabaseService fill:#37474F,color:#fff,stroke:#263238,stroke-width:2px
    style RedisService fill:#D32F2F,color:#fff,stroke:#B71C1C,stroke-width:2px
    style MailService fill:#00897B,color:#fff,stroke:#00695C,stroke-width:2px
    style OpenRouterService fill:#E65100,color:#fff,stroke:#BF360C,stroke-width:2px
    style GoogleCalendarService fill:#1565C0,color:#fff,stroke:#0D47A1,stroke-width:2px
    style GoogleDriveService fill:#1565C0,color:#fff,stroke:#0D47A1,stroke-width:2px
    style IoredisRateLimitStore fill:#FF7043,color:#fff,stroke:#E64A19,stroke-width:2px
```

---

## 4.6 Worker Job Processing — Queue Architecture (Code)

```mermaid
%%{init: {"theme": "default"}}%%
sequenceDiagram
    participant Client as 👤 Client
    participant HTTP as 🌐 HTTP API Server<br/>(index.ts)
    participant Handler as 🎯 Handler
    participant WJS as 📊 WorkerJobService<br/>(worker-job.service.ts)
    participant Redis as ⚡ Redis Queue<br/>(BRPOP)
    participant Worker as ⚙️ Worker Process<br/>(worker.ts)
    participant Dispatcher as 🔀 processJob()
    participant Service as 💼 Domain Service<br/>(e.g. JadwalDraftService)
    participant ExtAPI as 🌍 External API<br/>(OpenRouter/Google)

    Client->>HTTP: POST /api/koordinator/jadwal-draft/generate
    HTTP->>Handler: ConstraintDosenHandler.chat(c)
    Handler->>WJS: enqueue(CONSTRAINT_DOSEN_CHAT, payload)
    WJS->>Redis: LPUSH "worker:jobs:queue" (serialized job)
    WJS-->>Handler: { id, status: "queued" }
    Handler-->>Client: SSE stream (event: "connected")

    loop SSE Polling (1s interval)
        Worker->>Redis: BRPOP "worker:jobs:queue" (blocking)
        Redis-->>Worker: job data
        Worker->>Dispatcher: processJob(job)
        Dispatcher->>WJS: markRunning(job.id)
        Dispatcher->>WJS: appendProgress(job.id, event, payload)
        WJS->>Redis: SET "worker:jobs:item:{id}" (updated)
        Dispatcher->>Service: chat(email, message, progressEmitter)
        Service->>ExtAPI: OpenRouter chatCompletion()
        ExtAPI-->>Service: AI response
        Service-->>Dispatcher: result
        Dispatcher->>WJS: markCompleted(job.id, result)
        WJS->>Redis: SET "worker:jobs:item:{id}" (completed)
    end

    loop SSE Polling
        Client->>HTTP: GET /api/worker-jobs/:job_id/sse
        HTTP->>WJS: get(job.id)
        WJS-->>Client: SSE event (job:status / job:progress / job:completed)
    end
```

---

## 4.7 Worker Job Types — Dispatch Table

```mermaid
%%{init: {"theme": "default"}}%%
classDiagram
    class WorkerJobType {
        <<enum>>
        LOG_CREATE = "log.create"
        PENDAFTARAN_EMAIL_SEND = "pendaftaran.email.send"
        JADWAL_EMAIL_SEND = "jadwal.email.send"
        JADWAL_DRAFT_GENERATE = "jadwal-draft.generate"
        CONSTRAINT_DOSEN_CHAT = "constraint-dosen.chat"
        CONSTRAINT_DOSEN_CHAT_UPDATE = "constraint-dosen.chat-update"
    }

    class WorkerJobStatus {
        <<enum>>
        QUEUED = "queued"
        RUNNING = "running"
        COMPLETED = "completed"
        FAILED = "failed"
    }

    class WorkerJob {
        <<type>>
        +id: string
        +type: WorkerJobType
        +status: WorkerJobStatus
        +payload: WorkerJobPayload
        +attempts: number
        +max_attempts: number
        +progress: WorkerJobProgress[]
        +created_at: string
        +started_at?: string
        +completed_at?: string
        +error?: string
        +response?: any
    }

    class WorkerJobProgress {
        <<type>>
        +sequence: number
        +event: string
        +payload: Record
        +timestamp: string
    }

    class processJob {
        <<worker.ts>>
        +processJob(job): Promise~void~
        +normalizeJadwalDraftPayload(p)
    }

    WorkerJob "1" *-- "*" WorkerJobProgress : contains
    WorkerJob ..> WorkerJobType : has type
    WorkerJob ..> WorkerJobStatus : has status
    processJob ..> WorkerJobType : switch on type
    processJob ..> WorkerJobStatus : sets status

    style WorkerJobType fill:#E65100,color:#fff,stroke:#BF360C,stroke-width:2px
    style WorkerJobStatus fill:#F9A825,color:#000,stroke:#F57F17,stroke-width:2px
    style WorkerJob fill:#00897B,color:#fff,stroke:#00695C,stroke-width:2px
    style WorkerJobProgress fill:#4DB6AC,color:#000,stroke:#00897B,stroke-width:2px
    style processJob fill:#43A047,color:#fff,stroke:#2E7D32,stroke-width:2px
```

---

## 4.8 Domain Models — Prisma ER Diagram (Code)

```mermaid
%%{init: {"theme": "default"}}%%
erDiagram
    dosen {
        varchar(18) nip PK
        varchar(255) nama
        varchar(255) email UK
        varchar(14) no_hp UK
    }

    mahasiswa {
        varchar(11) nim PK
        varchar(255) nama
        varchar(36) email UK
        boolean aktif
        varchar(14) no_hp UK
    }

    ruangan {
        varchar(10) kode PK
        varchar(50) nama
        boolean status
        int urutan
    }

    jenis_seminar {
        varchar(20) kode UK
        varchar(100) nama
        text deskripsi
        boolean is_aktif
        int wajib_pembimbing
        int wajib_penguji
        boolean ada_ketua_sidang
    }

    jadwal {
        varchar(14) id PK
        timestamptz tanggal
        timestamptz waktu_mulai
        timestamptz waktu_selesai
        varchar(11) nim FK
        varchar(10) kode_ruangan FK
        varchar(20) id_jenis_seminar FK
        varchar(5) kode_tahun_ajaran
    }

    jadwal_draft {
        cuid id PK
        varchar(20) batch_id
        varchar(11) nim
        timestamptz tanggal
        timestamptz waktu_mulai
        timestamptz waktu_selesai
        varchar(10) kode_ruangan
        json list_dosen
        json llm_reasoning
        float confidence
        enum status DRAFT|APPROVED|REJECTED
    }

    penilaian {
        cuid id PK
        varchar(14) id_jadwal FK
        varchar(18) nip FK
        enum role PenilaiRole
    }

    detail_penilaian {
        cuid id PK
        cuid id_penilaian FK
        varchar(7) id_komponen FK
        float nilai
        text catatan
    }

    komponen_penilaian {
        varchar(7) id PK
        varchar(50) nama
        int persentase
        boolean is_aktif
        enum role PenilaiRole
    }

    pendaftaran {
        cuid id PK
        varchar(11) nim FK
        varchar(5) kode_tahun_ajaran
        varchar(50) id_pengajuan_fst UK
        varchar(20) id_jenis_seminar FK
        enum status_berkas PENDING|REVISI|APPROVED|REJECTED|UPLOAD_ULANG
        enum status_jadwal BELUM_JADWAL|SUDAH_JADWAL
        varchar(18) nip_pembimbing_1
        varchar(18) nip_pembimbing_2
        varchar(18) nip_penguji_1
        varchar(18) nip_penguji_2
        varchar(18) nip_ketua_sidang
    }

    data_pendaftaran {
        cuid id PK
        cuid id_pendaftaran FK
        cuid id_dokumen_template FK
        text nilai_text
        text nilai_file_url
        boolean nilai_boolean
        timestamptz nilai_date
        json nilai_json
    }

    dokumen_template {
        varchar(150) nama
        varchar(50) kode UK
        text deskripsi
        enum tipe_input FILE_UPLOAD|TEXT|URL|BOOLEAN|DATE|SELECT|MULTI_SELECT
        json opsi
        varchar(50) format_file
        int max_size_mb
        boolean is_special
    }

    requirement_dokumen {
        cuid id PK
        varchar(20) id_jenis_seminar FK
        cuid id_dokumen_template FK
        int urutan
        boolean is_wajib
        text keterangan_tambahan
    }

    bidang_keahlian {
        cuid id PK
        varchar(100) nama UK
    }

    keahlian_dosen {
        cuid id PK
        varchar(18) nip FK
        cuid id_bidang_keahlian FK
    }

    constraint_dosen {
        cuid id PK
        varchar(18) nip FK
        enum type AVAILABLE_TIME|UNAVAILABLE_TIME|PREFERENCE|LOCATION
        int hari
        timestamptz waktu_mulai
        timestamptz waktu_selesai
        text keterangan
        int priority
        boolean is_active
        json raw_data
    }

    log {
        cuid id PK
        timestamptz timestamp
        enum action CREATE|UPDATE|DELETE|GANTI_JADWAL|GANTI_DOSEN
        enum actor_type KOORDINATOR|DOSEN|MAHASISWA
        varchar actor_id
        enum entity_type JADWAL|PENILAIAN|PENDAFTARAN|...
        varchar entity_id
        json context
        json old_values
        json new_values
    }

    bobot_penilai {
        cuid id PK
        varchar(20) id_jenis_seminar FK
        enum role PenilaiRole
        int persentase
    }

    dosen ||--o{ keahlian_dosen : "has expertise"
    bidang_keahlian ||--o{ keahlian_dosen : "maps to"
    dosen ||--o{ penilaian : "grades"
    dosen ||--o{ constraint_dosen : "has constraints"
    mahasiswa ||--o{ jadwal : "has schedule"
    mahasiswa ||--o{ pendaftaran : "registers"
    ruangan ||--o{ jadwal : "holds"
    jenis_seminar ||--o{ jadwal : "defines type"
    jenis_seminar ||--o{ jadwal_draft : "defines type"
    jenis_seminar ||--o{ pendaftaran : "defines type"
    jenis_seminar ||--o{ requirement_dokumen : "requires docs"
    jenis_seminar ||--o{ bobot_penilai : "weight config"
    jadwal ||--o{ penilaian : "has assessments"
    penilaian ||--o{ detail_penilaian : "composed of"
    komponen_penilaian ||--o{ detail_penilaian : "evaluates"
    pendaftaran ||--o{ data_pendaftaran : "stores EAV data"
    dokumen_template ||--o{ data_pendaftaran : "defines field"
    dokumen_template ||--o{ requirement_dokumen : "linked to"
```

---

## 4.9 AI Constraint Chat — Code Flow (Class Diagram)

```mermaid
%%{init: {"theme": "default"}}%%
classDiagram
    class ConstraintDosenHandler {
        <<static>>
        +chat(c)
        +chatUpdate(c)
        +getAll(c)
        +get(c)
        +create(c)
        +update(c)
        +delete(c)
    }

    class ConstraintDosenService {
        <<static>>
        +chat(email, message, emitter)
        +chatUpdate(email, id, message, emitter)
        +getAll(email)
        +get(email, id)
        +create(email, data)
        +update(email, id, data)
        +delete(email, id)
        -getNipFromEmail(email)
    }

    class ConstraintDosenRepository {
        <<static>>
        +findByNip(nip)
        +findById(id)
        +create(data)
        +update(id, data)
        +delete(id)
    }

    class OpenRouterService {
        <<singleton>>
        +chatCompletion(options)
    }

    class ParseConstraintPrompt {
        <<prompt>>
        +PARSE_CONSTRAINT_PROMPT: string
    }

    class ParsedConstraint {
        <<Zod Schema>>
        +ParseConstraintOutputSchema
        +ParsedConstraint
    }

    class WorkerJobService {
        <<static>>
        +enqueue(type, payload, options)
        +markRunning(id)
        +markCompleted(id, result)
        +appendProgress(id, event, payload)
    }

    class StreamWorkerJob {
        <<SSE>>
        +streamWorkerJob(c, options)
    }

    ConstraintDosenHandler --> WorkerJobService : enqueue chat job
    ConstraintDosenHandler --> StreamWorkerJob : SSE stream
    WorkerJobService ..> ConstraintDosenService : dispatches
    ConstraintDosenService --> ConstraintDosenRepository : CRUD
    ConstraintDosenService --> OpenRouterService : AI chat
    ConstraintDosenService ..> ParseConstraintPrompt : loads prompt
    ConstraintDosenService ..> ParsedConstraint : validates output

    style ConstraintDosenHandler fill:#7B1FA2,color:#fff,stroke:#4A148C,stroke-width:2px
    style ConstraintDosenService fill:#2E7D32,color:#fff,stroke:#1B5E20,stroke-width:2px
    style ConstraintDosenRepository fill:#F9A825,color:#000,stroke:#F57F17,stroke-width:2px
    style OpenRouterService fill:#E65100,color:#fff,stroke:#BF360C,stroke-width:2px
    style ParseConstraintPrompt fill:#FF7043,color:#fff,stroke:#E64A19,stroke-width:2px
    style ParsedConstraint fill:#AB47BC,color:#fff,stroke:#7B1FA2,stroke-width:2px
    style WorkerJobService fill:#00897B,color:#fff,stroke:#00695C,stroke-width:2px
    style StreamWorkerJob fill:#0097A7,color:#fff,stroke:#006064,stroke-width:2px
```

---

## 4.10 Jadwal Draft Generation — AI Scheduling Code Flow

```mermaid
%%{init: {"theme": "default"}}%%
classDiagram
    class JadwalDraftHandler {
        <<static>>
        +generate(c)
        +batchApprove(c)
        +batchReject(c)
        +batchDelete(c)
    }

    class JadwalDraftService {
        <<static>>
        +generate(data, context, emitter)
        +batchApprove(batchId)
        +batchReject(batchId)
        +batchDelete(batchId)
    }

    class ScheduleRules {
        <<prompt/context>>
        +SEMINAR_DURATION_MINUTES
        +OPERATING_HOURS
        +WORK_DAYS
        +JENIS_SEMINAR
        +KP_ROLES
        +TA_ROLES
        +CONSTRAINT_TYPES
        +ROOM_BUFFER_MINUTES
        +MAX_SEMINAR_PER_DOSEN_PER_DAY
        +getScheduleRulesAsText(): string
    }

    class CreateScheduleOutput {
        <<Zod Schema>>
        +TimeSlotSchema
        +ScheduleSuggestionSchema
        +CreateScheduleOutputSchema
    }

    class ResolveConflictOutput {
        <<Zod Schema>>
        +ConflictSolutionSchema
        +ConflictSchema
        +ResolveConflictOutputSchema
    }

    class SuggestAlternativesOutput {
        <<Zod Schema>>
        +AlternativeSchema
        +SuggestAlternativesOutputSchema
    }

    JadwalDraftHandler --> WorkerJobService : enqueue JADWAL_DRAFT_GENERATE
    WorkerJobService ..> JadwalDraftService : dispatches
    JadwalDraftService --> ScheduleRules : loads rules as context
    JadwalDraftService --> CreateScheduleOutput : validates AI output
    JadwalDraftService --> ResolveConflictOutput : validates conflicts
    JadwalDraftService --> SuggestAlternativesOutput : validates alternatives
    JadwalDraftService --> OpenRouterService : AI generation
    JadwalDraftService --> ConstraintDosenRepository : fetch constraints

    style JadwalDraftHandler fill:#7B1FA2,color:#fff,stroke:#4A148C,stroke-width:2px
    style JadwalDraftService fill:#2E7D32,color:#fff,stroke:#1B5E20,stroke-width:2px
    style ScheduleRules fill:#E65100,color:#fff,stroke:#BF360C,stroke-width:2px
    style CreateScheduleOutput fill:#AB47BC,color:#fff,stroke:#7B1FA2,stroke-width:2px
    style ResolveConflictOutput fill:#AB47BC,color:#fff,stroke:#7B1FA2,stroke-width:2px
    style SuggestAlternativesOutput fill:#AB47BC,color:#fff,stroke:#7B1FA2,stroke-width:2px
    style WorkerJobService fill:#00897B,color:#fff,stroke:#00695C,stroke-width:1px
    style OpenRouterService fill:#E65100,color:#fff,stroke:#BF360C,stroke-width:1px
    style ConstraintDosenRepository fill:#F9A825,color:#000,stroke:#F57F17,stroke-width:1px
```

---

## 4.11 File Structure — Source Code Organization

```
src/
├── api.ts                          # 🔀 API Router: registers all module routes
├── index.ts                        # 🌐 HTTP Server entry point (APP_PROCESS=server)
├── worker.ts                       # ⚙️ Worker entry point (APP_PROCESS=worker)
│
├── core/                           # 🟢 CORE — DI Container & Configuration
│   ├── bootstrap.ts                #   DI container setup + service registration
│   ├── config.ts                   #   Zod-validated env schema (374 lines)
│   ├── container.ts                #   Singleton DI container (191 lines)
│   └── index.ts                    #   Barrel exports
│
├── infrastructures/                # 🩵 INFRASTRUCTURE — External Service Clients
│   ├── db.infrastructure.ts        #   PrismaClient + PrismaPg adapter + Accelerate
│   ├── redis.infrastructure.ts     #   ioredis singleton + JSON cache layer
│   ├── mail.infrastructure.ts      #   Nodemailer SMTP + dev email sink
│   ├── openrouter.infrastructure.ts#   OpenRouter API client + retry
│   ├── google-calendar.infrastructure.ts # Google Calendar v3 API
│   └── google-drive.infrastructure.ts    # Google Drive v3 API
│
├── middlewares/                    # 🔴 MIDDLEWARE — Auth, Rate Limit, Logging
│   ├── auth.middleware.ts          #   JWT extraction + role checking
│   ├── rate-limit.middleware.ts    #   IP-based rate limiting (Redis/memory)
│   └── log.middleware.ts           #   Request/response structured logging
│
├── modules/                        # 📦 22 FEATURE MODULES
│   ├── bidang-keahlian/            #   7 files (handler, repository, route, service, type, validator, index)
│   ├── bobot-penilai/              #   7 files
│   ├── constraint-dosen/           #   7 files (+ AI chat)
│   ├── detail-penilaian/           #   7 files
│   ├── dokumen-template/           #   7 files
│   ├── dosen/                      #   6 files (no validator)
│   ├── jadwal/                     #   7 files (+ Calendar sync)
│   ├── jadwal-draft/               #   7 files (+ AI generation)
│   ├── jenis-seminar/              #   7 files
│   ├── keahlian-dosen/             #   7 files
│   ├── komponen-penilaian/         #   6 files
│   ├── log/                        #   7 files
│   ├── mahasiswa/                  #   7 files
│   ├── pendaftaran/                #   8 files (+ email.service)
│   ├── requirement-dokumen/        #   7 files
│   ├── ruangan/                    #   8 files (+ helper)
│   ├── tahun-ajaran/               #   7 files
│   ├── upload/                     #   5 files
│   └── worker-job/                 #   6 files (+ SSE)
│
├── handlers/                       # 🟣 HANDLERS — Cross-module Controllers
│   ├── dosen-seminar.handler.ts
│   ├── global.handler.ts           #   404 + global error handler
│   ├── koordinator.handler.ts
│   └── penilaian.handler.ts
│
├── routes/                         # 🔵 ROUTES — Cross-module Route Definitions
│   ├── dosen-seminar.route.ts
│   ├── global.route.ts
│   ├── koordinator.route.ts
│   └── penilaian.route.ts
│
├── services/                       # 🟢 SERVICES — Cross-module Business Logic
│   ├── base.service.ts             #   Abstract BaseService with logger
│   ├── dosen-seminar.service.ts
│   ├── global.service.ts
│   ├── koordinator.service.ts
│   └── penilaian.service.ts
│
├── repositories/                   # 🟡 REPOSITORIES — Cross-module Data Access
│   ├── base.repository.ts          #   Abstract BaseRepository with Prisma
│   └── penilaian.repository.ts
│
├── helpers/                        # 🔧 HELPERS — Utility Functions
│   ├── auth.helper.ts              #   JWT decode (base64)
│   ├── crypto.helper.ts
│   ├── dosen.helper.ts
│   ├── jadwal.helper.ts            #   ID generation + timezone conversion
│   ├── jenis-seminar.helper.ts
│   ├── log-jadwal.helper.ts
│   ├── log-nilai.helper.ts
│   ├── log.helper.ts
│   ├── mahasiswa.helper.ts
│   └── tahun-ajaran.helper.ts
│
├── prompts/                        # 🟠 AI PROMPTS — LLM Templates & Schemas
│   ├── context/
│   │   └── schedule-rules.ts       #   Typed scheduling rules for AI
│   └── output/
│       ├── constraint-schema.ts    #   Zod schema: parsed constraint
│       └── schedule-schema.ts      #   Zod schemas: schedule suggestions
│
├── types/                          # 🔷 TYPES — TypeScript Type Definitions
│   ├── container.type.ts           #   DI service type definitions
│   ├── global.type.ts
│   ├── penilaian.type.ts
│   └── tahun_ajaran.type.ts
│
├── utils/                          # 🛠️ UTILITIES — Shared Helpers
│   ├── api-error.util.ts           #   Custom APIError class
│   ├── cache-invalidation.util.ts
│   ├── cache-key.util.ts           #   Cache key hashing
│   ├── logger.util.ts              #   Structured logger
│   ├── openrouter.util.ts          #   OpenRouter request/response types
│   └── zod-error.util.ts           #   Zod validation error formatting
│
└── validators/                     # ✅ VALIDATORS — Zod Schema Validators
    ├── dosen-seminar.validator.ts
    └── penilaian.validator.ts
```

---

## 4.12 Middleware Chain — Class Diagram

```mermaid
%%{init: {"theme": "default"}}%%
classDiagram
    class HonoMiddleware {
        <<interface>>
        +middleware(c: Context, next: Next): Promise
    }

    class CorsMiddleware {
        +cors(options)
    }

    class StructuredLogger {
        +structuredLogger(c, next)
        -extractRequest(c)
    }

    class GlobalRateLimit {
        +global(): RateLimitHandler
    }

    class WriteRateLimit {
        +write(): RateLimitHandler
    }

    class IoredisRateLimitStore {
        +increment(key)
        +decrement(key)
        +resetKey(key)
    }

    class JWTExtraction {
        +JWTBearerTokenExtraction(c, next)
    }

    class RoleGuard {
        +requireRole(...roles): Function
    }

    HonoMiddleware <|.. StructuredLogger
    HonoMiddleware <|.. JWTExtraction
    HonoMiddleware <|.. RoleGuard
    HonoMiddleware <|.. GlobalRateLimit
    HonoMiddleware <|.. WriteRateLimit

    GlobalRateLimit --> IoredisRateLimitStore : Redis-backed
    WriteRateLimit --> IoredisRateLimitStore : Redis-backed
    JWTExtraction --> AuthHelper : decodeJwt
    RoleGuard --> APIError : 403 forbidden

    style HonoMiddleware fill:#E53935,color:#fff,stroke:#B71C1C,stroke-width:2px
    style CorsMiddleware fill:#FF5252,color:#fff,stroke:#D32F2F,stroke-width:2px
    style StructuredLogger fill:#FF5252,color:#fff,stroke:#D32F2F,stroke-width:2px
    style GlobalRateLimit fill:#FF7043,color:#fff,stroke:#E64A19,stroke-width:2px
    style WriteRateLimit fill:#FF7043,color:#fff,stroke:#E64A19,stroke-width:2px
    style IoredisRateLimitStore fill:#D32F2F,color:#fff,stroke:#B71C1C,stroke-width:2px
    style JWTExtraction fill:#E53935,color:#fff,stroke:#B71C1C,stroke-width:2px
    style RoleGuard fill:#C62828,color:#fff,stroke:#8E0000,stroke-width:2px
    style AuthHelper fill:#FF8A65,color:#fff,stroke:#E64A19,stroke-width:1px
    style APIError fill:#B71C1C,color:#fff,stroke:#7F0000,stroke-width:1px
```

---

## 4.13 Cache Strategy — Code Pattern

```mermaid
%%{init: {"theme": "default"}}%%
sequenceDiagram
    participant Handler as 🎯 Handler
    participant Service as 💼 Service
    participant Cache as ⚡ RedisService
    participant DB as 🗄️ PrismaClient

    Handler->>Service: getAll(params)

    alt Cache Hit ✅
        Service->>Cache: getJson(key)
        Cache-->>Service: cached data
        Service-->>Handler: { data, cached: true }
    else Cache Miss ❌
        Service->>DB: prisma.jadwal.findMany(where)
        DB-->>Service: database results
        Service->>Cache: setJson(key, data, ttl)
        Service-->>Handler: { data, cached: false }
    end

    Note over Service: 📝 On write operations:
    Service->>Cache: CacheInvalidation.invalidate(prefix)
    Note right of Cache: 🗑️ Invalidates related<br/>cache keys
```