# 📋 Simple Assessment & Task Breakdown: Integrasi Telegram Bot Hermes Gateway (1 Pintu)

> **Dokumen:** Assessment Ringkas & Panduan Eksekusi Tugas  
> **Versi:** 1.0.0 (Simplified & Action-Oriented)  
> **Target:** Developer / Engineer Pelaksana  
> **Status:** Siap Eksekusi  

---

## 🎯 1. Ringkasan Proyek & Tujuan

Mengubah Telegram Bot yang sebelumnya terhubung langsung via Hermes CLI (yang menyebabkan **chat publik tanpa izin** dan **memori saling tumpang tindih**) menjadi:
1. **Akses Terbatas (Whitelist)**: Hanya nomor HP yang terdaftar di database karyawan yang bisa memakai bot.
2. **Standalone UX di Telegram**: Registrasi langsung via tombol *Share Contact* native Telegram tanpa wajib login ke web portal.
3. **Arsitektur 1 Pintu**: Telegram Bot masuk melalui Backend Portal (`factory-portal-service`), sehingga *Trusted Context* (Redis) aktif dan AI Agent bisa commit ke GitLab.
4. **Isolasi Memori 100%**: Hermes menerima `session_id` unik untuk tiap user & proyek sehingga memori tidak akan pernah tertukar/tertimpa.

---

## 🏗️ 2. Arsitektur Sistem (High-Level Design)

Berdasarkan blueprint arsitektur yang dirancang:

```mermaid
flowchart TD
    subgraph TelegramBox["📱 Telegram"]
        TeleBot["Tele Bot App"]
        TeleMW["Telegram Middleware"]
        TeleBot <--> TeleMW
    end

    subgraph WebPortalBox["🌐 Web Portal"]
        WebFE["Frontend<br/>(factory-portal-web)"]
        WebBE["Backend<br/>(factory-portal-service)"]
        WebFE <--> WebBE
    end

    subgraph HermesGatewayBox["🚪 Hermes Gateway Layer"]
        HMW["Middleware Hermes<br/>(factory-agent-adapter)"]
        HGW["Hermes Gateway<br/>(:8643 REST API)"]
        HMW <--> HGW
    end

    subgraph HermesCoreBox["🤖 Hermes"]
        HermesCore["Hermes (AI Agent Engine)"]
    end

    subgraph StorageBox["📦 Artifact Storage"]
        MinIO["Storage (MinIO)"]
    end

    %% Alur Komunikasi
    TeleMW <-->|1. Auth & Return User Key| WebBE
    TeleMW -->|2. Transaksi Chat + User Key| HMW
    WebBE <-->|Trusted Context & Runs| HMW
    HGW <--> HermesCore
    HermesCore <--> MinIO
    WebBE -.-> MinIO
```

### 🧩 Deskripsi Komponen & Peran:

1. **Telegram Layer (`Tele Bot App` & `Telegram Middleware`)**:
   - **`Tele Bot App`**: Menangani interaksi pengguna di Telegram (command `/start`, `/projects`, `/project-active-<id>`, chat teks, dan tombol native *Share Contact*).
   - **`Telegram Middleware`**: 
     - Melakukan proses autentikasi awal ke **`factory-portal-service` (Backend)** untuk validasi whitelist user dan mendapatkan **User Key** (JWT Token & Context ID).
     - Mengirimkan pesan/prompt pengguna melalui salah satu dari 2 pintu integrasi:
       - **Pintu Backend Portal (`factory-portal-service`)**: Menggunakan endpoint `/agent-conversations` (Trusted Context aman dengan RBAC terpusat).
       - **Pintu Langsung ke Adapter (`factory-agent-adapter`)**: Menggunakan endpoint `/channel/sessions` dan `/channel/sessions/{id}/messages`.

2. **Web Portal (`factory-portal-web` [FE] & `factory-portal-service` [BE])**:
   - **`Frontend (factory-portal-web)`**: UI Web Portal untuk manajemen proyek, user, dan pemantauan AI.
   - **`Backend (factory-portal-service)`**: 
     - Pintu autentikasi utama, validasi whitelist nomor HP, mengembalikan **User Key**, RBAC proyek, manajemen JWT, serta mengelola penulisan izin ke cache context (`AgentContextStore`).
     - Menyediakan API percakapan agen via **`/agent-conversations`** (`POST /agent-conversations`, `POST /{portalConversationId}/messages`, `GET /{portalConversationId}/stream`).

3. **Hermes Gateway Layer (`factory-agent-adapter` & `Hermes Gateway`)**:
   - **`Middleware Hermes (factory-agent-adapter)`**: 
     - Menerima request dari Backend Portal maupun bot, mengelola isolasi sesi (`/channel/sessions`), device lock (`X-Device-Id`), dan idempotency key.
     - **Menerjemahkan request pesan menjadi pemanggilan Hermes Gateway `POST /v1/runs`** dan merelay event SSE `GET /v1/runs/{id}/events` ke client.
   - **`Hermes Gateway`**: REST API Gateway resmi (`:8643`) yang menjalankan eksekusi AI Agent via **`POST /v1/runs`**, `POST /v1/runs/{id}/stop`, dan `POST /v1/runs/{id}/approval`.

4. **Hermes (Core Engine)**:
   - Engine AI Agent murni yang menjalankan eksekusi tugas, skills, dan reasoning tanpa terhubung langsung ke chat publik Telegram.

5. **Artifact Storage (`MinIO`)**:
   - Object Storage S3-compatible tersentralisasi untuk menyimpan berkas keluaran AI (dokumen hasil generate, artifact, file analisa, dan log) yang dapat dibaca/ditulis oleh Hermes dan Web Portal.

---

## 🛠️ 3. Tech Stack yang Dibutuhkan

| Layer | Teknologi / Library | Alasan & Fungsi |
|---|---|---|
| **Bot / Ingress** | **Python 3.11+** + `python-telegram-bot` (v21 Async) | Menangani webhook Telegram, tombol native *Share Contact*, dan inline keyboard. |
| **HTTP Client** | **`httpx`** (Async) | Memanggil backend portal dan Hermes secara non-blocking (tidak membuat bot macet saat LLM mikir). |
| **Gateway / Core**| **Java 17+** (Quarkus / Spring Boot) di `factory-portal-service` | Pintu utama: validasi RBAC proyek, pemetaan user, dan penulisan izin ke Redis. |
| **State Cache** | **Redis 7.x** | Menyimpan *Trusted Context* (2 jam) dan mapping sesi user aktif. |
| **Identity DB** | **PostgreSQL 15+** + Flyway Migration | Menyimpan tabel whitelist karyawan & binding Telegram ID. |
| **AI Engine & Gateway** | **Hermes Gateway REST API** (`:8643`) | Engine LLM murni via endpoint `POST /v1/runs` + Middleware Hermes. |
| **Artifact Storage** | **MinIO** (S3 Compatible) | Menyimpan dokumen, artefak, dan file hasil generate AI. |
| **Infrastruktur** | **Docker** + **Kubernetes** + NGINX Ingress | Deployment container, manajemen Secret token, dan reverse proxy HTTPS. |

---

## 📝 4. Breakdown Task yang Harus Dilakukan

### Task 1: Setup Telegram Middleware, Koneksi Bot, & Project Selection Flow
* **Objective**: Menginisialisasi service Telegram Middleware yang terhubung ke platform Telegram, mengintegrasikan alur autentikasi user untuk mendapatkan **User Key** dari Backend (`factory-portal-service`), serta mengimplementasikan fitur load dan pemilihan active project (`/projects` & `/project-active-<id>`).
* **Estimasi Durasi**: `1 - 2 Hari`
* **Yang Harus Dikerjakan**:
  1. **Koneksi Bot & Telegram Middleware**:
     - Setup service Telegram Middleware menggunakan `python-telegram-bot` (Async).
     - Sambungkan bot token resmi dan pastikan bot dapat menerima event/pesan dari Telegram.
  2. **Autentikasi & Pengambilan User Key**:
     - Handler verifikasi user (via `/start` & Share Contact).
     - Panggil endpoint autentikasi Backend (`factory-portal-service`) untuk validasi whitelist nomor HP / Telegram ID.
     - Simpan **User Key** yang dikembalikan oleh Backend sebagai kredensial valid untuk transaksi berikutnya.
  3. **Load Daftar Proyek (`/projects`)**:
     - Buat command handler `/projects`.
     - Request daftar proyek yang diizinkan untuk user tersebut ke Backend (`factory-portal-service`) menggunakan User Key.
     - Tampilkan list proyek berpenomoran (misalnya nomor 1 s.d. 10) beserta instruksi pemilihan.
  4. **Pilih Proyek Aktif (`/project-active-<nomor>` / Command Selector)**:
     - Buat handler pemilihan proyek (contoh: user mengetik `/project-active-1` atau menekan tombol proyek 1).
     - Set konteks `active_project_id` di cache/session state pengguna.
     - Kirim konfirmasi ke user bahwa proyek aktif telah berhasil diset dan siap untuk bertransaksi dengan AI Agent.
* **Potential Obstacle (Kendala)**:
  - *Kendala*: User mencoba menjalankan `/projects` atau `/project-active-<id>` sebelum melakukan autentikasi / belum memiliki User Key yang valid.
  - *Solusi*: Berikan middleware validation guard. Jika `User Key` belum ada di session cache, arahkan user secara otomatis untuk `/start` dan membagikan kontak terlebih dahulu.
* **Definition of Done (DoD)**:
  - Service Telegram Middleware aktif dan tersambung ke Telegram.
  - User yang terverifikasi berhasil mendapatkan User Key dari Backend (`factory-portal-service`).
  - Command `/projects` sukses menampilkan daftar proyek dari Backend, dan command `/project-active-<id>` berhasil menyetel proyek aktif pengguna.

---

### Task 2: Integrasi Komunikasi ke Middleware Hermes (`factory-agent-adapter`), Pembuatan Sesi Upstream, & Handling Respon AI
* **Objective**: Mengimplementasikan komunikasi dari Telegram Middleware ke `factory-agent-adapter` (Middleware Hermes) mengikuti standar kontrak API backend: inisiasi sesi via upstream (`POST /channel/sessions`), serialisasi pengiriman pesan (`POST /channel/sessions/{id}/messages`), penanganan respon generate AI (SSE Stream / Polling), serta auto-save session berbasis UUID resmi dari Hermes/Adapter.
* **Estimasi Durasi**: `2 - 3 Hari`
* **Yang Harus Dikerjakan**:
  1. **Inisiasi Sesi Upstream (`POST /channel/sessions`)**:
     - Membuka/membuat sesi percakapan ke `factory-agent-adapter` dengan payload JSON:
       ```json
       {
         "channel": "TELEGRAM",
         "agentRole": "SYSTEM_ANALYST",
         "tenantId": "<tenant_uuid>",
         "projectId": "<active_project_uuid>"
       }
       ```
     - Menyertakan headers wajib: `Authorization: Bearer <User Key>`, `X-Device-Id: tg_<user_id>`, dan `Idempotency-Key: <uuid>`.
     - **Catatan Sesi**: `session_id` (UUID) di-generate langsung secara resmi oleh upstream (`factory-agent-adapter` / Hermes), sehingga bot cukup menyimpan UUID session yang dikembalikan untuk percakapan aktif.
  2. **Serialisasi Pengiriman Pesan (`POST /channel/sessions/{id}/messages`)**:
     - Mengubah pesan teks dari user Telegram menjadi request body (multipart/form) yang memuat field `message` dan `markdownMode`.
     - Mengirim request ke `factory-agent-adapter` menggunakan `httpx` (Async Client) dengan menyertakan User Key & Device ID.
     - Mengirim status visual *typing action* (`send_chat_action("typing")`) ke Telegram agar pengguna mengetahui AI sedang memproses generate.
  3. **Handling Respon AI & Chunking (SSE Stream / Polling)**:
     - Menangkap `processId` dari respon awal `202 Accepted` dan mengonsumsi stream respon via SSE (`GET /channel/sessions/{id}/stream?processId=...`) atau polling riwayat pesan.
     - Memformat output pesan ke Markdown Telegram secara rapi.
     - Mengimplementasikan text chunking jika respon teks melebihi batas 4.096 karakter Telegram.
  4. **Auto-Save Session & Riwayat Obrolan**:
     - Riwayat pesan otomatis tersimpan di sisi adapter & Hermes berdasarkan session UUID aktif.
     - Bot menyimpan mapping user $\leftrightarrow$ active `sessionId` di cache lokal/Redis agar percakapan berikutnya otomatis melanjutkan konteks sesi yang sama tanpa perlu inisiasi ulang.
* **Potential Obstacle (Kendala)**:
  - *Kendala*: Terjadi jeda koneksi saat mengonsumsi stream respon atau request timeout saat proses generate AI membutuhkan waktu cukup lama.
  - *Solusi*: Gunakan streaming non-blocking dengan retry policy dan setting timeout HTTP Client yang memadai (60 - 120 detik).
* **Definition of Done (DoD)**:
  - Sesi obrolan berhasil dibuka di `factory-agent-adapter` dan mengembalikan session UUID resmi dari upstream.
  - Pesan user berhasil terkirim dan AI Agent merespon secara lancar ke antarmuka Telegram.
  - Sesi obrolan tersimpan otomatis dan pengguna dapat melanjutkan percakapan berkonteks.

---

## ⏱️ 5. Estimasi Total Durasi & Urutan Pengerjaan

*(Akan disesuaikan seiring penambahan task berikutnya)*

---

## 📌 6. Ringkasan Singkat yang Perlu Diingat

1. **Jangan biarkan Hermes CLI memegang Bot Telegram langsung** $\rightarrow$ Pisahkan ke layer middleware/portal.
2. **Autentikasi ke Backend terlebih dahulu** $\rightarrow$ Dapatkan `User Key` sebelum mengizinkan transaksi chat ke `factory-agent-adapter` (Middleware Hermes).
3. **Session ID dibuat oleh Upstream** $\rightarrow$ `factory-agent-adapter` / Hermes yang meng-generate UUID sesi, bot cukup menyimpan dan menggunakannya.
4. **Gunakan `request_contact=True`** $\rightarrow$ Anti-spoofing nomor HP.
5. **Gunakan `X-Device-Id: tg_<id>`** $\rightarrow$ User bisa buka Web Portal dan Telegram bersamaan tanpa saling *kick*.
6. **Simpan hasil keluaran & artefak di MinIO** $\rightarrow$ Tersentralisasi dan dapat diakses baik oleh Web Portal maupun bot.
