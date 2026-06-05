# C4 Architecture — Sistem Penjadwalan dan Penilaian Seminar

Arsitektur sistem berbasis [C4 Model](https://c4model.com/) untuk tugas akhir **"Pengembangan Sistem Penjadwalan dan Penilaian Seminar"** — UIN Suska Riau.

## Stack

- **Backend:** Bun + Hono v4 (OpenAPIHono) + RegExpRouter
- **Database:** PostgreSQL 15 + Prisma 7 (adapter-pg + Accelerate)
- **Cache & Queue:** Redis 7 (ioredis) — cache, rate limit, BRPOP job queue
- **Auth:** Keycloak (OIDC) via JWT Bearer token
- **AI:** OpenRouter (gpt-4o-mini) — constraint chat, jadwal draft generation
- **Email:** Nodemailer SMTP (465)
- **Calendar:** Google Calendar API (Node.js googleapis)
- **Storage:** Google Drive API (Node.js googleapis)

## C4 Diagrams

- **Level 1–3:** [`architecture-c4.md`](architecture-c4.md) — Context, Container, Component diagrams (Mermaid C4 syntax)
- **Level 4:** [`architecture-c4-level4.md`](architecture-c4-level4.md) — Code diagrams: class diagrams, sequence diagrams, ER diagrams, file structure

### Level 1 — Context Diagram
Sistem utama, aktor (Mahasiswa, Dosen, Koordinator), dan external systems (Keycloak, PostgreSQL, Redis, OpenRouter, Google Drive, Google Calendar, SMTP).

### Level 2 — Container Diagram
HTTP API Server, Background Worker (dual-process), PostgreSQL, Redis, Keycloak, OpenRouter, Google Drive, Google Calendar, SMTP.

### Level 3 — Component Diagrams

| Diagram | Components |
|---------|-----------|
| **HTTP API Server** | Request Handler, JWT Auth Middleware, Rate Limiter (Redis), Response Formatter, OpenAPI Spec Generator |
| **Background Worker** | Job Queue Manager, BRPOP Consumer, Job Dispatcher, SSE Broadcaster, Redis Connection Pool |
| **Feature Modules** | 20+ module groups (Dokumen Template, Upload, Jadwal, Pendaftaran, Penilaian, Bidang Keahlian, Koordinator, Dosen, etc.) |
| **Google Calendar Integration Flow** | Jadwal Publish → Create Calendar Event → Send Email Invitation (full sequence) |

### Bonus Diagrams
- **Job Queue Sequence Diagram** — Alur job dari HTTP request → Redis Queue → Worker dispatch → Job execution → SSE notification
- **Google Calendar Integration Flow** — Sequence dari jadwal publish sampai email invitation terkirim

## Untuk Render

- **VS Code:** Install extension [Mermaid Preview](https://marketplace.visualstudio.com/items?itemName=bierner.markdown-mermaid)
- **GitHub:** Diagram langsung render di markdown viewer
- **Online:** [Mermaid Live Editor](https://mermaid.live/)

### Level 4 — Code Diagrams (`architecture-c4-level4.md`)

| Diagram | Description |
|---------|-------------|
| **4.1 Layered Architecture** | Route → Handler → Service → Repository → Database |
| **4.2 DI Container** | Singleton container, service tokens, bootstrap |
| **4.3 Request Lifecycle** | Middleware chain: CORS → Logger → RateLimit → Auth → Handler |
| **4.4 Feature Module Pattern** | UML class: Route → Handler → Service → Repository |
| **4.5 Infrastructure Layer** | Singleton services: DB, Redis, Mail, OpenRouter, Calendar, Drive |
| **4.6 Worker Job Queue** | Sequence: HTTP → Redis BRPOP → Worker → Service → External API |
| **4.7 Job Types Dispatch** | 6 job types with status flow and progress tracking |
| **4.8 Domain Models ER** | 16 Prisma models with relationships |
| **4.9 AI Constraint Chat** | Code flow: Handler → Service → OpenRouter → SSE stream |
| **4.10 AI Scheduling** | Jadwal draft generation with typed rules + Zod schemas |
| **4.11 File Structure** | Full source code organization tree |
| **4.12 Middleware Chain** | UML: CORS, Logger, RateLimit, JWT, Role guard |
| **4.13 Cache Strategy** | Sequence: Cache hit/miss pattern + invalidation |
