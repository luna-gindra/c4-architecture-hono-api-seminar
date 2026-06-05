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

Semua diagram ada di [`architecture-c4.md`](architecture-c4.md) menggunakan sintaks **Mermaid C4**.

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
