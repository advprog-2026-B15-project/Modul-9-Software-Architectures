# Tutorial 9B: Visualizing Architecture & Architectural Risk

## Daftar Isi

- [Current Architecture (Commit 1)](#current-architecture)
  - [System Context Diagram](#system-context-diagram)
  - [Container Diagram](#container-diagram)
  - [Deployment Diagram](#deployment-diagram)
- [Future Architecture (Commit 2)](#future-architecture)
- [Risk Storming Explanation (Commit 3)](#risk-storming-explanation)
- [Individual Works (Commit 4)](#individual-works)

## Current Architecture

BidMart adalah platform lelang online berbasis *microservices* yang memungkinkan pengguna untuk menjual dan membeli barang melalui mekanisme lelang. Sistem terdiri dari 5 *microservice* utama yang saling berkomunikasi secara *synchronous* (REST) maupun *asynchronous* (RabbitMQ/AMQP).

### System Context Diagram

Diagram ini menunjukkan gambaran paling abstrak dari sistem BidMart, bagaimana sistem berinteraksi dengan pengguna dan sistem eksternal, tanpa menunjukkan detail teknis internal.

```mermaid
C4Context
    title System Context Diagram

    Person(seller, "Seller", "Mengelola listing barang untuk dilelang")
    Person(buyer, "Buyer / Bidder", "Melakukan penawaran (bid) pada lelang")
    Person(admin, "Admin", "Manajemen user dan konfigurasi platform")

    System(bidmart, "BidMart Platform", "Platform lelang online (Katalog, Dompet, Lelang, Pesanan)")

    %% Relasi Aktor ke Sistem
    Rel(seller, bidmart, "Membuat listing & mengelola pengiriman", "HTTPS")
    Rel(buyer, bidmart, "Melihat katalog, melakukan bid & top-up", "HTTPS")
    Rel(admin, bidmart, "Mengelola user dan konfigurasi sistem", "HTTPS")
```

### Container Diagram

Diagram ini menunjukkan seluruh *container* (unit yang dapat dijalankan secara independen) yang menyusun sistem BidMart dan bagaimana mereka berkomunikasi satu sama lain. Pengguna berinteraksi melalui *Frontend Web Application*.

```mermaid
C4Container
    title Container Diagram

    Person(seller, "Seller", "Penjual barang")
    Person(buyer, "Buyer", "Pembeli / Bidder")
    Person(admin, "Admin", "Pengelola operasional platform")

    System_Boundary(bidmart, "BidMart Platform") {
        
        Container(frontend, "Web Application", "Next.js, React, Tailwind", "Antarmuka utama bagi pengguna")

        Container(auth_api, "Auth Service", "Spring Boot 4.0.3", "Registrasi, login, JWT, 2FA")
        ContainerDb(auth_db, "Auth DB", "PostgreSQL 16", "Data user, role & token")

        Container(catalog_api, "Catalog Service", "Spring Boot 3.5.11", "Listing & sinkronisasi harga")
        ContainerDb(catalog_db, "Catalog DB", "PostgreSQL 15", "Data listing & kategori")

        Container(auction_api, "Auction Service", "Spring Boot 3.5.11", "Core lelang & anti-sniping")
        ContainerDb(auction_db, "Auction DB", "Neon DB Serverless", "Data lelang & bid")
        Container(redis, "Distributed Lock", "Upstash Redis", "Locking concurrent bid")

        Container(wallet_api, "Wallet Service", "Spring Boot 3.3.2", "Top-up & hold saldo")
        ContainerDb(wallet_db, "Wallet DB", "PostgreSQL", "Data saldo & transaksi")

        Container(booking_api, "Booking Service", "Spring Boot 3.5.x", "Pesanan, notifikasi & sengketa")
        ContainerDb(booking_db, "Booking DB", "PostgreSQL 16", "Data pesanan & shipment")

        Container(mq, "Message Broker", "CloudAMQP RabbitMQ", "Event-driven broker")
    }

    %% User to Frontend
    Rel(seller, frontend, "Membuat listing & update pengiriman", "HTTPS")
    Rel(buyer, frontend, "Mencari barang, bid, top-up & lacak pesanan", "HTTPS")
    Rel(admin, frontend, "Kelola user, moderasi listing & tangani sengketa", "HTTPS")

    %% Frontend to Backend APIs
    Rel(frontend, auth_api, "Request otentikasi & kelola role/user", "REST/JSON")
    Rel(frontend, catalog_api, "Request katalog & moderasi listing", "REST/JSON")
    Rel(frontend, auction_api, "Request operasi & aktivitas lelang", "REST/JSON")
    Rel(frontend, wallet_api, "Request manajemen saldo dompet", "REST/JSON")
    Rel(frontend, booking_api, "Request status pesanan & sengketa", "REST/JSON")

    %% Backend to DBs
    Rel(auth_api, auth_db, "R/W user data", "SQL/TCP")
    Rel(catalog_api, catalog_db, "R/W listing data", "SQL/TCP")
    Rel(auction_api, auction_db, "R/W auction data", "SQL/TCP")
    Rel(auction_api, redis, "Acquire/release lock", "RESP")
    Rel(wallet_api, wallet_db, "R/W wallet data", "SQL/TCP")
    Rel(booking_api, booking_db, "R/W booking data", "SQL/TCP")

    %% Inter-service Sync
    Rel(auction_api, wallet_api, "Hold/release saldo", "REST [POST]")
    Rel(auction_api, catalog_api, "Validasi listing", "REST [GET]")

    %% Message Broker Comm
    Rel(auth_api, mq, "Publish auth events", "AMQP")
    Rel(auction_api, mq, "Publish auction events", "AMQP")
    Rel(wallet_api, mq, "Publish wallet events", "AMQP")
    Rel(catalog_api, mq, "Consume bid events", "AMQP")
    Rel(booking_api, mq, "Consume closure & bid events", "AMQP")
```
*Catatan: Garis panah dengan label [REST] atau [HTTPS] menandakan komunikasi synchronous, sedangkan [AMQP] menandakan komunikasi asynchronous.*

### Deployment Diagram

Diagram ini menunjukkan di mana dan bagaimana setiap *container* di-*deploy* pada infrastruktur produksi. *Frontend* berjalan secara *serverless* di Vercel, *service* aplikasi berjalan sebagai Docker container di AWS EC2, dan infrastruktur pendukung menggunakan *managed cloud services*.

```mermaid
graph TB
    subgraph Users["👤 Users"]
        Browser["Web Browser"]
    end

    subgraph VERCEL["☁️ Vercel (Edge Network)"]
        FRONTEND["BidMart Web App\n(Next.js / React)"]
    end

    subgraph AWS["☁️ Amazon Web Services"]
        
        %% Dipecah menjadi 5 EC2 berbeda
        subgraph EC2_AUTH["🖥️ AWS EC2 (Auth)"]
            AUTH["🐳 Auth Service\n(:8081)"]
            AUTH_DB[("🐳 Auth DB\n(PG 16)")]
        end
        
        subgraph EC2_CATALOG["🖥️ AWS EC2 (Catalog)"]
            CATALOG["🐳 Catalog Service\n(:8080)"]
            CATALOG_DB[("🐳 Catalog DB\n(PG 15)")]
        end
        
        subgraph EC2_AUCTION["🖥️ AWS EC2 (Auction)"]
            AUCTION["🐳 Auction Service\n(:8083)"]
        end
        
        subgraph EC2_WALLET["🖥️ AWS EC2 (Wallet)"]
            WALLET["🐳 Wallet Service\n(:8080)"]
            WALLET_DB[("🐳 Wallet DB\n(PG)")]
        end
        
        subgraph EC2_BOOKING["🖥️ AWS EC2 (Booking)"]
            BOOKING["🐳 Booking Service\n(:8085)"]
            BOOKING_DB[("🐳 Booking DB\n(PG 16)")]
        end
    end

    subgraph MANAGED["☁️ Managed Cloud Services"]
        direction LR
        AUCTION_DB[("Neon DB\n(PostgreSQL)")]
        REDIS[("Upstash Redis\n(Dist. Lock)")]
        MQ["CloudAMQP\n(RabbitMQ)"]
    end

    %% Client Connections
    Browser -->|"HTTPS"| FRONTEND
    
    %% Hubungan Frontend ke masing-masing API (IP Publik Berbeda-beda)
    FRONTEND -.->|"REST"| AUTH
    FRONTEND -.->|"REST"| CATALOG
    FRONTEND -.->|"REST"| AUCTION
    FRONTEND -.->|"REST"| WALLET
    FRONTEND -.->|"REST"| BOOKING

    %% Inter-service Sync (Menimbulkan Risiko Latency Jaringan Antar-EC2)
    AUCTION -.->|"REST [POST]"| WALLET
    AUCTION -.->|"REST [GET]"| CATALOG

    %% Local DB Connections (Berada di mesin yang sama dengan servicenya)
    AUTH -.->|"JDBC"| AUTH_DB
    CATALOG -.->|"JDBC"| CATALOG_DB
    WALLET -.->|"JDBC"| WALLET_DB
    BOOKING -.->|"JDBC"| BOOKING_DB

    %% Managed Cloud Connections
    AUCTION -.->|"JDBC"| AUCTION_DB
    AUCTION -.->|"RESP/TLS"| REDIS
    AUTH -.->|"AMQPS (Publish)"| MQ
    AUCTION -.->|"AMQPS (Publish)"| MQ
    WALLET -.->|"AMQPS (Publish)"| MQ
    CATALOG -.->|"AMQPS (Consume)"| MQ
    BOOKING -.->|"AMQPS (Consume)"| MQ
```

## Future Architecture

Arsitektur masa depan BidMart ditentukan berdasarkan hasil *risk storming* tim. Tiga perubahan utama yang dilakukan adalah: menambahkan **API Gateway** sebagai *single entry point* agar frontend tidak langsung bergantung ke setiap service, menambahkan **Circuit Breaker** pada jalur sinkron kritis (Auction → Wallet dan Auction → Catalog) untuk mencegah *cascade failure*, dan menambahkan **Monitoring Stack** (Prometheus + Grafana) agar masalah dapat terdeteksi sebelum berdampak ke pengguna.

```mermaid
C4Container
    title Future Architecture Diagram (Post Risk Storming)

    Person(seller, "Seller", "Penjual barang")
    Person(buyer, "Buyer", "Pembeli / Bidder")
    Person(admin, "Admin", "Pengelola operasional platform")

    System_Boundary(bidmart, "BidMart Platform") {

        Container(frontend, "Web Application", "Next.js, React, Tailwind", "Antarmuka utama bagi pengguna")

        Container(gateway, "API Gateway", "Nginx / Kong", "Single entry point: routing, rate limiting, dan JWT validation")

        Container(auth_api, "Auth Service", "Spring Boot 4.0.3", "Registrasi, login, JWT, 2FA")
        ContainerDb(auth_db, "Auth DB", "PostgreSQL 16", "Data user, role & token")

        Container(catalog_api, "Catalog Service", "Spring Boot 3.5.11", "Listing & sinkronisasi harga")
        ContainerDb(catalog_db, "Catalog DB", "PostgreSQL 15", "Data listing & kategori")

        Container(auction_api, "Auction Service", "Spring Boot 3.5.11", "Core lelang & anti-sniping")
        ContainerDb(auction_db, "Auction DB", "Neon DB Serverless", "Data lelang & bid")
        Container(redis, "Distributed Lock", "Upstash Redis", "Locking concurrent bid")
        Container(cb, "Circuit Breaker", "Resilience4j", "Fallback untuk call sinkron Auction → Wallet & Catalog")

        Container(wallet_api, "Wallet Service", "Spring Boot 3.3.2", "Top-up & hold saldo")
        ContainerDb(wallet_db, "Wallet DB", "PostgreSQL", "Data saldo & transaksi")

        Container(booking_api, "Booking Service", "Spring Boot 3.5.x", "Pesanan, notifikasi & sengketa")
        ContainerDb(booking_db, "Booking DB", "PostgreSQL 16", "Data pesanan & shipment")

        Container(mq, "Message Broker", "CloudAMQP RabbitMQ", "Event-driven broker")

        Container(monitoring, "Monitoring Stack", "Prometheus + Grafana", "Scrape metrics tiap service, alert & dashboard")
    }

    Rel(seller, frontend, "Membuat listing & update pengiriman", "HTTPS")
    Rel(buyer, frontend, "Mencari barang, bid, top-up & lacak pesanan", "HTTPS")
    Rel(admin, frontend, "Kelola user, moderasi listing & tangani sengketa", "HTTPS")

    Rel(frontend, gateway, "Semua request API", "REST/HTTPS")

    Rel(gateway, auth_api, "Route auth requests", "REST/JSON")
    Rel(gateway, catalog_api, "Route catalog requests", "REST/JSON")
    Rel(gateway, auction_api, "Route auction requests", "REST/JSON")
    Rel(gateway, wallet_api, "Route wallet requests", "REST/JSON")
    Rel(gateway, booking_api, "Route booking requests", "REST/JSON")

    Rel(auth_api, auth_db, "R/W user data", "SQL/TCP")
    Rel(catalog_api, catalog_db, "R/W listing data", "SQL/TCP")
    Rel(auction_api, auction_db, "R/W auction data", "SQL/TCP")
    Rel(auction_api, redis, "Acquire/release lock", "RESP")
    Rel(wallet_api, wallet_db, "R/W wallet data", "SQL/TCP")
    Rel(booking_api, booking_db, "R/W booking data", "SQL/TCP")

    Rel(auction_api, cb, "Call via circuit breaker", "Internal")
    Rel(cb, wallet_api, "Hold/release saldo (dengan fallback)", "REST [POST]")
    Rel(cb, catalog_api, "Validasi listing (dengan fallback)", "REST [GET]")

    Rel(auth_api, mq, "Publish auth events", "AMQP")
    Rel(auction_api, mq, "Publish auction events", "AMQP")
    Rel(wallet_api, mq, "Publish wallet events", "AMQP")
    Rel(catalog_api, mq, "Consume bid events", "AMQP")
    Rel(booking_api, mq, "Consume closure & bid events", "AMQP")

    Rel(monitoring, auth_api, "Scrape /actuator/prometheus", "HTTP")
    Rel(monitoring, catalog_api, "Scrape /actuator/prometheus", "HTTP")
    Rel(monitoring, auction_api, "Scrape /actuator/prometheus", "HTTP")
    Rel(monitoring, wallet_api, "Scrape /actuator/prometheus", "HTTP")
    Rel(monitoring, booking_api, "Scrape /actuator/prometheus", "HTTP")
```

### Perubahan dari Current ke Future Architecture

| Area | Current | Future | Alasan |
|---|---|---|---|
| Entry point frontend | Frontend memanggil 5 service langsung (5 IP/domain berbeda) | Semua request melalui satu **API Gateway** | Menghilangkan coupling frontend ke IP tiap service; satu titik untuk rate limiting & JWT validation |
| Resiliensi Auction → Wallet/Catalog | Panggilan REST langsung tanpa fallback; jika Wallet/Catalog down, Auction ikut gagal | **Circuit Breaker** (Resilience4j) membungkus call sinkron dan menyediakan fallback response | Mencegah *cascade failure*: Auction tetap bisa merespons meski dependensi sementara hilang |
| Observabilitas | Tidak ada monitoring terpusat; masalah hanya diketahui setelah user laporan | **Prometheus + Grafana** scrape `/actuator/prometheus` semua service; alert otomatis saat error rate / latency naik | Mendeteksi anomali lebih awal sebelum berdampak ke pengguna |

## Risk Storming Explanation

*Risk storming* dilakukan dengan cara masing-masing anggota tim mereview diagram arsitektur *current* secara mandiri, memberi label risiko pada setiap area, lalu mendiskusikan hasilnya bersama untuk mencapai konsensus. Diagram berikut menunjukkan area-area yang diidentifikasi berisiko beserta level risikonya (🔴 Tinggi, 🟠 Sedang, 🟢 Rendah).

```mermaid
graph TB
    classDef high fill:#f28b82,stroke:#c62828,color:#000
    classDef medium fill:#ffb74d,stroke:#e65100,color:#000
    classDef low fill:#81c995,stroke:#2e7d32,color:#000
    classDef normal fill:#aecbfa,stroke:#1a73e8,color:#000

    Browser["Web Browser"]

    subgraph VERCEL["Vercel"]
        FRONTEND["Web App\n(Next.js)"]:::normal
    end

    subgraph ENTRY["Entry Point"]
        NO_GW["⚠️ Tidak ada API Gateway\nFrontend → 5 service langsung"]:::high
    end

    subgraph AWS["AWS EC2 (per service)"]
        AUTH["Auth Service\n(:8081)"]:::low
        CATALOG["Catalog Service\n(:8080)"]:::medium
        AUCTION["Auction Service\n(:8083)"]:::high
        WALLET["Wallet Service\n(:8080)"]:::medium
        BOOKING["Booking Service\n(:8085)"]:::low
    end

    subgraph MANAGED["Managed Cloud"]
        AUCTION_DB["Neon DB\n(cold start risk)"]:::low
        REDIS["Upstash Redis"]:::low
        MQ["CloudAMQP\n(single broker)"]:::medium
    end

    subgraph OBSERVABILITY["Observabilitas"]
        NO_MON["⚠️ Tidak ada monitoring\nterpusat"]:::high
    end

    Browser --> FRONTEND
    FRONTEND --> NO_GW
    NO_GW -.->|"5 direct calls"| AUTH & CATALOG & AUCTION & WALLET & BOOKING
    AUCTION -.->|"🔴 sync REST\n(cascade risk)"| WALLET
    AUCTION -.->|"🔴 sync REST\n(cascade risk)"| CATALOG
    AUCTION -.-> AUCTION_DB & REDIS
    AUTH & AUCTION & WALLET -.->|"AMQP"| MQ
    MQ -.-> CATALOG & BOOKING
```

### 1. Identification

Tiap anggota tim mereview diagram *current* secara mandiri dan menandai area yang dianggap berisiko. Berikut hasil identifikasi yang digabungkan:

| Area Risiko | Deskripsi Risiko | Impact (1-3) | Likelihood (1-3) | Skor | Level |
|---|---|---|---|---|---|
| Frontend → 5 service langsung (tanpa API Gateway) | Tidak ada rate limiting atau auth centralization; jika IP salah satu service berubah, frontend wajib diubah | 2 | 3 | 6 | 🔴 Tinggi |
| Auction → Wallet REST sinkron | Jika Wallet down atau lambat, proses bid gagal/timeout; tidak ada fallback | 3 | 2 | 6 | 🔴 Tinggi |
| Auction → Catalog REST sinkron | Jika Catalog down saat validasi listing, Auction tidak bisa memproses bid apapun | 3 | 2 | 6 | 🔴 Tinggi |
| Tidak ada monitoring terpusat | Error/degradasi hanya diketahui setelah user komplain; waktu deteksi & penanganan (MTTR) tinggi | 2 | 3 | 6 | 🔴 Tinggi |
| Single EC2 per service (tanpa HA) | Jika satu EC2 down (hardware failure, patching), service tersebut mati total | 3 | 1 | 3 | 🟠 Sedang |
| CloudAMQP sebagai single message broker | Jika CloudAMQP mengalami outage, semua alur event (auth, auction, wallet, booking) berhenti | 2 | 2 | 4 | 🟠 Sedang |
| Neon DB Serverless untuk Auction | Cold start Neon DB menambah latensi pada query pertama setelah periode idle | 1 | 2 | 2 | 🟢 Rendah |

### 2. Consensus

Setelah identifikasi individual, tim berdiskusi untuk menyelaraskan skor dan menentukan prioritas. Beberapa perbedaan pendapat yang muncul:

- **"Frontend → 5 service langsung"** — ada anggota yang menilai *Likelihood* = 2 karena URL service jarang berubah di production. Setelah diskusi, disepakati *Likelihood* = 3 karena tanpa API Gateway tidak ada rate limiting sehingga endpoint service terbuka langsung ke internet, dan dalam skenario scaling atau re-deploy region, IP EC2 bisa berganti.
- **"Single EC2 per service"** — sebagian anggota menilai *Likelihood* = 2. Setelah mempertimbangkan SLA AWS EC2 yang tinggi dan kemampuan restart cepat, disepakati *Likelihood* = 1, namun *Impact* tetap 3 karena jika terjadi maka service benar-benar tidak tersedia.
- **"CloudAMQP single broker"** — awalnya dinilai rendah (skor 2). Setelah diskusi, disepakati sedang (skor 4) karena jika broker mati, seluruh alur async (notifikasi, sinkronisasi harga, penutupan lelang) berhenti sekaligus.

Kesepakatan akhir: fokus mitigasi pada **4 risiko skor tertinggi** (skor 6) karena keempatnya paling actionable dalam scope semester ini tanpa perlu perubahan infrastruktur besar.

### 3. Mitigation

| Risiko | Solusi Mitigasi | Perubahan Arsitektur |
|---|---|---|
| Frontend → 5 service langsung | Tambahkan **API Gateway** (Nginx/Kong) sebagai *reverse proxy* tunggal; frontend hanya butuh satu base URL; gateway yang handle routing, rate limiting, dan JWT validation | Ditambahkan container `API Gateway` antara Frontend dan kelima Backend Service |
| Auction → Wallet REST sinkron (cascade failure) | Bungkus call dengan **Circuit Breaker** (Resilience4j `@CircuitBreaker`); definisikan fallback agar Auction tetap bisa merespons saat Wallet tidak tersedia sementara | Ditambahkan container `Circuit Breaker` antara Auction dan Wallet |
| Auction → Catalog REST sinkron (cascade failure) | Sama seperti di atas — Circuit Breaker membungkus call Auction → Catalog dengan fallback | Circuit Breaker yang sama juga meng-cover call ke Catalog |
| Tidak ada monitoring terpusat | Deploy **Prometheus** untuk scrape `/actuator/prometheus` tiap service + **Grafana** untuk dashboard dan alert rule (error rate > 1% atau p95 latency > 500ms) | Ditambahkan container `Monitoring Stack (Prometheus + Grafana)` dengan relasi scrape ke semua service |

## Individual Works

### Individual Work M Naufal Zhafran Rabiul Batara (2406361694)

Container yang dikerjakan: **Booking Service** (Spring Boot 3.5.x) — menangani pembuatan pesanan (*order*) dari hasil lelang, manajemen notifikasi in-app secara real-time, dan pengiriman (*shipment*). Service ini adalah *consumer* utama event dari RabbitMQ (WinnerDetermined, BidPlaced, BalanceConverted, BalanceReleased) dan menggunakan SSE (*Server-Sent Events*) untuk push notifikasi ke browser secara real-time.

#### Component Diagram — Booking Service

```mermaid
C4Component
    title Component Diagram — Booking Service (Order & Notification)

    Person(user, "Buyer / Seller", "Pengguna platform")

    Container_Boundary(booking_api, "Booking Service") {
        Component(booking_ctrl, "BookingController", "Spring @RestController\n/api/bookings/**", "Endpoint REST untuk melihat pesanan, update status shipment, dan konfirmasi penerimaan barang")
        Component(notif_ctrl, "NotificationController", "Spring @RestController\n/api/notifications/**", "Endpoint REST untuk melihat notifikasi, subscribe SSE stream, dan kelola preferensi notifikasi")

        Component(booking_svc, "BookingService", "Spring @Service", "Membuat booking dari event lelang, transisi status booking, update shipment, dan konfirmasi delivery")
        Component(notif_svc, "NotificationService", "Spring @Service", "Membuat notifikasi WIN/LOSE/NEW_BID/OUTBID/PAYMENT, cek preferensi user, simpan dan push notifikasi")
        Component(realtime_svc, "RealtimeEventService", "Spring @Service (SSE)", "Mengelola SSE emitter per user; push notification, booking status change, dan auction update ke browser secara real-time")
        Component(audit_svc, "BookingStatusAuditLogService", "Spring @Service", "Mencatat setiap transisi status booking ke audit log")
        Component(reliable_proc, "ReliableEventProcessor", "Spring @Component", "Memastikan idempotency: cek ProcessedEvent sebelum eksekusi handler, tangani dead-letter jika gagal")

        Component(event_consumer, "BookingEventConsumer", "Spring @Component\n(manual dispatch)", "Menerima dan memvalidasi event dari RabbitMQ: WinnerDetermined, AuctionClosed, BidPlaced, BalanceConverted, BalanceReleased")

        Component(booking_repo, "BookingRepository", "Spring Data JPA", "R/W data Booking & BookingItem ke DB")
        Component(notif_repo, "NotificationRepository", "Spring Data JPA", "R/W data Notification & NotificationPreference ke DB")
        Component(processed_repo, "ProcessedEventRepository", "Spring Data JPA", "R/W ProcessedEvent untuk deduplication event")
    }

    ContainerDb(booking_db, "Booking DB", "PostgreSQL 16", "Tabel: bookings, booking_items, shipments, notifications, notification_preferences, booking_status_audit_logs, processed_events")
    Container(mq, "Message Broker", "CloudAMQP RabbitMQ", "Event-driven broker")
    Container(gateway, "API Gateway", "Nginx / Kong", "Entry point semua request")

    Rel(user, gateway, "Request booking & notifikasi", "HTTPS")
    Rel(gateway, booking_ctrl, "Route /api/bookings/**", "REST/JSON")
    Rel(gateway, notif_ctrl, "Route /api/notifications/**", "REST/JSON + SSE")

    Rel(booking_ctrl, booking_svc, "Delegasi business logic", "Method call")
    Rel(notif_ctrl, notif_svc, "Delegasi business logic", "Method call")
    Rel(notif_ctrl, realtime_svc, "subscribe(userId) → SseEmitter", "Method call")

    Rel(booking_svc, notif_svc, "Trigger win/lose/payment notifications", "Method call")
    Rel(booking_svc, audit_svc, "recordStatusChange()", "Method call")
    Rel(booking_svc, realtime_svc, "publishBookingStatusChange()", "Method call")
    Rel(notif_svc, realtime_svc, "publishNotification()", "Method call")

    Rel(mq, event_consumer, "Deliver events (WinnerDetermined, BidPlaced, dll.)", "AMQP")
    Rel(event_consumer, reliable_proc, "process(event, type, handler)", "Method call")
    Rel(reliable_proc, processed_repo, "hasProcessed() / markProcessed()", "JPA")
    Rel(reliable_proc, booking_svc, "createBookingFromWinnerEvent()", "Method call")
    Rel(reliable_proc, notif_svc, "createWinLoseNotifications() / createBidPlacedNotifications()", "Method call")

    Rel(booking_svc, booking_repo, "R/W booking data", "JPA")
    Rel(notif_svc, notif_repo, "R/W notification data", "JPA")
    Rel(booking_repo, booking_db, "SQL", "JDBC/TCP")
    Rel(notif_repo, booking_db, "SQL", "JDBC/TCP")
    Rel(processed_repo, booking_db, "SQL", "JDBC/TCP")
```

#### Code Diagram — NotificationService

Diagram ini menunjukkan struktur kelas internal komponen **NotificationService** beserta kelas-kelas yang berkolaborasi langsung dengannya.

```mermaid
classDiagram
    class NotificationController {
        -NotificationService notificationService
        -RealtimeEventService realtimeEventService
        +getMyNotifications(userId: String) List~NotificationResponse~
        +streamMyRealtimeEvents(userId: String) SseEmitter
        +getMyNotificationPreference(userId: String) NotificationPreferenceResponse
        +updateMyNotificationPreference(userId: String, request: UpdateNotificationPreferenceRequest) NotificationPreferenceResponse
        +markNotificationAsRead(id: Long, userId: String, request: MarkNotificationReadRequest) NotificationResponse
    }

    class NotificationService {
        -NotificationRepository notificationRepository
        -NotificationPreferenceRepository notificationPreferenceRepository
        -RealtimeEventService realtimeEventService
        +getMyNotifications(userId: String) List~Notification~
        +getMyNotificationPreference(userId: String) NotificationPreference
        +upsertNotificationPreference(userId: String, emailEnabled: Boolean, inAppEnabled: Boolean) NotificationPreference
        +markNotificationAsRead(id: Long, userId: String) Notification
        +createWinLoseNotifications(winnerUserId: String, loserUserIds: List, auctionId: String, finalPrice: Long) void
        +createBidPlacedNotifications(sellerUserId: String, bidderUserId: String, prevBidderUserId: String, auctionId: String, bidAmount: Long, itemName: String) void
        +createBalanceConvertedNotification(userId: String, auctionId: String, amount: Long) void
        +createBalanceReleasedNotification(userId: String, auctionId: String, amount: Long) void
        -addIfInAppEnabled(notifications: List, userId: String, type: NotificationType, title: String, message: String, auctionId: String) void
        -isInAppEnabled(userId: String) boolean
        -saveNotifications(notifications: List~Notification~) void
    }

    class RealtimeEventService {
        -emittersByUserId: Map~String, List~SseEmitter~~
        +subscribe(userId: String) SseEmitter
        +publishNotification(notification: Notification) void
        +publishAuctionUpdate(userId: String, update: RealtimeAuctionUpdateResponse) void
        +publishBookingStatusChange(booking: Booking, fromStatus: BookingStatus, toStatus: BookingStatus, ...) void
        -send(userId: String, eventName: String, data: Object) void
        -removeEmitter(userId: String, emitter: SseEmitter) void
    }

    class NotificationRepository {
        <<interface>>
        +findByUserIdOrderByCreatedAtDesc(userId: String) List~Notification~
        +findByIdAndUserId(id: Long, userId: String) Optional~Notification~
    }

    class NotificationPreferenceRepository {
        <<interface>>
        +findByUserId(userId: String) Optional~NotificationPreference~
    }

    class Notification {
        -Long id
        -String userId
        -NotificationType type
        -String title
        -String message
        -Boolean isRead
        -OffsetDateTime readAt
        -String relatedAuctionId
        -Booking relatedBooking
        -OffsetDateTime createdAt
        -OffsetDateTime updatedAt
    }

    class NotificationPreference {
        -Long id
        -String userId
        -Boolean emailEnabled
        -Boolean inAppEnabled
    }

    class NotificationType {
        <<enumeration>>
        WIN
        LOSE
        NEW_BID
        OUTBID
        PAYMENT_CONFIRMED
        BALANCE_RELEASED
        INFO
    }

    NotificationController --> NotificationService : uses
    NotificationController --> RealtimeEventService : subscribe()
    NotificationService --> NotificationRepository : findBy / save
    NotificationService --> NotificationPreferenceRepository : findByUserId / save
    NotificationService --> RealtimeEventService : publishNotification()
    NotificationRepository --> Notification : manages
    NotificationPreferenceRepository --> NotificationPreference : manages
    Notification --> NotificationType : type
```

### Individual Work Keisha Vania Laurent (2406437331)

Container yang dikerjakan: **Auction Service** (Spring Boot 3.5.11) — menangani proses inti lelang, pengelolaan penawaran harga (bid) dengan concurrency control (*anti-sniping* & distributed lock), dan penjadwalan otomatis penutupan lelang. Service ini adalah *producer* event utama ke RabbitMQ (BidPlaced, WinnerDetermined, AuctionClosed) dan berkomunikasi dengan Wallet Service untuk menahan (hold) saldo.

#### Component Diagram — Auction Service

```mermaid
C4Component
    title Component Diagram — Auction Service

    Person(seller, "Seller", "Pembuat lelang")
    Person(buyer, "Buyer", "Peserta lelang")

    Container_Boundary(auction_api, "Auction Service") {
        Component(auction_ctrl, "AuctionController", "Spring @RestController\n/api/auctions/**", "Endpoint REST untuk pembuatan lelang, aktivasi, dan penawaran (bid).")
        
        Component(auction_svc, "AuctionService", "Spring @Service", "Menangani logika inti lelang, validasi anti-sniping, dan memproses penempatan bid.")
        
        Component(closure_scheduler, "AuctionClosingScheduler", "Spring @Scheduled", "Job periodik (cron) untuk mendeteksi lelang yang telah melewati batas waktu (end_time).")
        
        Component(closure_svc, "AuctionClosureService", "Spring @Service", "Memproses penutupan lelang secara aman menggunakan distributed lock (Redisson).")
        
        Component(closure_db_svc, "AuctionClosureDatabaseService", "Spring @Service", "Mendelegasikan update status akhir lelang ke database secara transaksional (@Transactional).")
        
        Component(lock_template, "DistributedLockTemplate", "Redisson Component", "Mengelola distributed locking via Redis untuk mencegah race condition saat bid/closure.")
        
        Component(wallet_adapter, "WalletRestAdapter", "Spring @Component", "Berkomunikasi dengan eksternal Wallet Service untuk request hold saldo via RestClient.")
        
        Component(event_publisher, "RabbitMqAuctionEventPublisher", "Spring @Component", "Mempublikasikan event (BidPlaced, WinnerDetermined, dll) ke Message Broker.")
        
        Component(auction_repo, "AuctionRepository", "Spring Data JPA", "R/W data lelang ke DB.")
        Component(bid_repo, "BidRepository", "Spring Data JPA", "R/W riwayat bid ke DB.")
    }

    ContainerDb(auction_db, "Auction DB", "Neon DB Serverless", "Tabel: auctions, bids")
    Container(redis, "Distributed Lock", "Upstash Redis", "Distributed lock storage")
    Container(mq, "Message Broker", "CloudAMQP RabbitMQ", "Event-driven broker")
    Container(gateway, "API Gateway", "Nginx / Kong", "Entry point request")
    Container(wallet_api, "Wallet Service", "Spring Boot", "Layanan dompet eksternal")

    Rel(seller, gateway, "Membuat & aktivasi lelang", "HTTPS")
    Rel(buyer, gateway, "Melakukan bid", "HTTPS")
    Rel(gateway, auction_ctrl, "Route /api/auctions/**", "REST/JSON")

    Rel(auction_ctrl, auction_svc, "Delegasi business logic", "Method call")
    Rel(auction_svc, lock_template, "Acquire lock sebelum bid", "Method call")
    Rel(auction_svc, wallet_adapter, "Request hold saldo", "Method call")
    Rel(auction_svc, event_publisher, "Publish BidPlaced", "Method call")
    Rel(auction_svc, auction_repo, "R/W data lelang", "JPA")
    Rel(auction_svc, bid_repo, "R/W riwayat bid", "JPA")
    
    Rel(closure_scheduler, closure_svc, "Trigger evaluasi lelang", "Method call")
    Rel(closure_svc, lock_template, "Acquire lock sebelum close", "Method call")
    Rel(closure_svc, closure_db_svc, "Delegasi DB Tx", "Method call")
    Rel(closure_db_svc, event_publisher, "Publish WinnerDetermined", "Method call")
    Rel(closure_db_svc, auction_repo, "Update status lelang", "JPA")
    
    Rel(auction_repo, auction_db, "SQL", "JDBC/TCP")
    Rel(bid_repo, auction_db, "SQL", "JDBC/TCP")
    Rel(lock_template, redis, "Acquire/Release lock", "RESP/TLS")
    Rel(wallet_adapter, wallet_api, "HTTP POST /internal/wallet/hold", "REST")
    Rel(event_publisher, mq, "Publish events", "AMQP")
```

#### Code Diagram 1 — AuctionService

```mermaid
classDiagram
    class AuctionController {
        -AuctionService auctionService
        +create(req: CreateAuctionRequest, sellerId: String) AuctionResponse
        +activate(id: String, sellerId: String) AuctionResponse
        +placeBid(id: String, req: PlaceBidRequest, bidderId: String) BidResponse
    }

    class AuctionService {
        -AuctionRepository auctionRepository
        -BidRepository bidRepository
        -List~BidValidationStrategy~ validationStrategies
        -HoldBalancePort holdBalancePort
        -AuctionEventPort auctionEventPort
        -DistributedLockTemplate lockTemplate
        +create(req: CreateAuctionRequest, sellerId: String) Auction
        +activate(auctionId: String, sellerId: String) Auction
        +placeBid(auctionId: String, bidderId: String, amount: Long) Bid
    }

    class HoldBalancePort {
        <<interface>>
        +holdBalance(userId: String, auctionId: String, amount: Long) void
    }

    class AuctionEventPort {
        <<interface>>
        +publishBidPlaced(event: BidPlacedEvent) void
        +publishWinnerDetermined(...) void
    }

    class DistributedLockTemplate {
        +executeWithLock(lockKey: String, waitTime: long, leaseTime: long, unit: TimeUnit, supplier: Supplier~T~) T
    }

    class BidValidationStrategy {
        <<interface>>
        +validate(auction: Auction, amount: Long) void
    }

    AuctionController --> AuctionService : uses
    AuctionService --> HoldBalancePort : uses
    AuctionService --> AuctionEventPort : uses
    AuctionService --> DistributedLockTemplate : uses
    AuctionService --> BidValidationStrategy : uses
```

#### Code Diagram 2 — AuctionClosureService

```mermaid
classDiagram
    class AuctionClosingScheduler {
        -AuctionRepository auctionRepository
        -AuctionClosureService closureService
        +scheduleAuctionClosure() void
    }

    class AuctionClosureService {
        -DistributedLockTemplate lockTemplate
        -AuctionClosureDatabaseService auctionClosureDatabaseService
        +processAuctionClosure(auction: Auction, closedAt: OffsetDateTime) void
    }

    class AuctionClosureDatabaseService {
        -AuctionRepository auctionRepository
        -BidRepository bidRepository
        -AuctionEventPort auctionEventPort
        +closeAuction(auctionId: String, closedAt: OffsetDateTime) void
        -publishWinnerEvent(...) void
        -publishClosedEvent(...) void
    }

    AuctionClosingScheduler --> AuctionClosureService : triggers
    AuctionClosureService --> DistributedLockTemplate : uses
    AuctionClosureService --> AuctionClosureDatabaseService : delegates Tx
```

#### Code Diagram 3 — Domain Entities

```mermaid
classDiagram
    class Auction {
        -String id
        -String title
        -Long startingPrice
        -Long reservePrice
        -Long minimumIncrement
        -Long currentPrice
        -AuctionStatus status
        -OffsetDateTime endTime
        -String listingId
        -String sellerId
        +prePersist() void
        +preUpdate() void
    }

    class Bid {
        -String id
        -Auction auction
        -String bidderId
        -Long amount
        -OffsetDateTime createdAt
        +prePersist() void
    }

    class AuctionStatus {
        <<enumeration>>
        DRAFT
        ACTIVE
        EXTENDED
        CLOSED
        WON
        UNSOLD
    }

    Auction --> AuctionStatus : defines status
    Bid --> Auction : references
```

#### Bonus: Additional Component Diagram — Web Application (Frontend)

```mermaid
C4Component
    title Component Diagram — Web Application (Frontend)

    Person(buyer, "Buyer / Bidder", "Pengguna aplikasi")
    Person(seller, "Seller", "Pengguna aplikasi")

    Container_Boundary(frontend_app, "Web Application") {
        Component(app_router, "App Router Layer", "Next.js Pages (src/app)", "Menangani routing halaman (contoh: /auctions/[id]) dan integrasi tata letak UI.")
        Component(ui_components, "UI Components", "React Components (src/components)", "Komponen presentasional yang dapat digunakan kembali (ProductCard, BiddingForm, NavBar).")
        
        Component(module_auth, "Auth Module", "Service Logic (src/modules/auth)", "Menangani logika pemanggilan API otentikasi (login, register, session).")
        Component(module_catalog, "Catalog Module", "Service Logic (src/modules/catalog)", "Menangani logika pemanggilan API katalog barang dan pencarian.")
        Component(module_auction, "Auction Module", "Service Logic (src/modules/auction)", "Menangani pemanggilan API (fetch data) dan manajemen state khusus domain lelang.")
        Component(module_wallet, "Wallet Module", "Service Logic (src/modules/wallet)", "Menangani logika pemanggilan API dompet (cek saldo, top-up).")
        Component(module_booking, "Booking Module", "Service Logic (src/modules/booking)", "Menangani logika pemanggilan API status pesanan dan pengiriman.")
        
        Component(config_lib, "Config & Utilities", "TypeScript (src/lib)", "Menyimpan konfigurasi URL dan instance pemanggil HTTP asinkron (Axios/Fetch).")
    }

    Container(auth_api, "Auth Service", "Spring Boot", "Layanan backend otentikasi")
    Container(catalog_api, "Catalog Service", "Spring Boot", "Layanan backend katalog")
    Container(auction_api, "Auction Service", "Spring Boot", "Layanan backend lelang")
    Container(wallet_api, "Wallet Service", "Spring Boot", "Layanan backend dompet")
    Container(booking_api, "Booking Service", "Spring Boot", "Layanan backend pesanan")

    Rel(buyer, app_router, "Akses halaman, klik bid, cek saldo", "HTTPS")
    Rel(seller, app_router, "Kelola lelang dan pengiriman", "HTTPS")
    
    Rel(app_router, ui_components, "Merender UI", "React Props")
    Rel(app_router, module_auth, "Delegasi operasi auth", "Method call")
    Rel(app_router, module_catalog, "Delegasi ambil produk", "Method call")
    Rel(app_router, module_auction, "Delegasi getAuction / placeBid", "Method call")
    Rel(app_router, module_wallet, "Delegasi getBalance", "Method call")
    Rel(app_router, module_booking, "Delegasi lacak pesanan", "Method call")
    
    Rel(module_auth, config_lib, "Ambil config URL", "Method call")
    Rel(module_catalog, config_lib, "Ambil config URL", "Method call")
    Rel(module_auction, config_lib, "Ambil config URL", "Method call")
    Rel(module_wallet, config_lib, "Ambil config URL", "Method call")
    Rel(module_booking, config_lib, "Ambil config URL", "Method call")
    
    Rel(module_auth, auth_api, "REST API Auth", "JSON/HTTPS")
    Rel(module_catalog, catalog_api, "REST API Katalog", "JSON/HTTPS")
    Rel(module_auction, auction_api, "REST API Lelang", "JSON/HTTPS")
    Rel(module_wallet, wallet_api, "REST API Dompet", "JSON/HTTPS")
    Rel(module_booking, booking_api, "REST API Pesanan", "JSON/HTTPS")
```

#### Bonus: Additional Code Diagram — Auction Module

```mermaid
classDiagram
    class AuctionPage {
        <<Next.js Page>>
        -String auctionId
        -AuctionState state
        +render() JSX.Element
        +handleBidSubmit(amount: number) void
    }

    class BiddingFormUI {
        <<React Component>>
        -number currentPrice
        -function onSubmit
        +render() JSX.Element
    }

    class AuctionModule {
        <<Service/Hook>>
        -String baseUrl
        +getAuctionDetails(id: String) Promise~AuctionData~
        +placeBid(id: String, amount: number) Promise~BidResponse~
        +getBidHistory(id: String) Promise~BidHistory[]~
    }

    class ApiConfig {
        <<Utility>>
        +String AUCTION_API_URL
        +fetchWithAuth(url: String, options: Object) Promise~Response~
    }

    AuctionPage --> BiddingFormUI : uses
    AuctionPage --> AuctionModule : calls
    AuctionModule --> ApiConfig : depends on
```

### Individual Work Sean Marcello Maheron (2406401792)

Container yang dikerjakan: **Catalog Service** (Spring Boot 3.5.11) — menangani seluruh siklus hidup *listing* barang yang akan dilelang, mulai dari pembuatan dan moderasi listing oleh Admin, pengelolaan kategori, hingga sinkronisasi harga saat lelang berlangsung. Service ini adalah *consumer* event `BidPlaced` dan `AuctionClosed` dari RabbitMQ untuk menjaga data harga listing tetap sinkron, dan menyediakan API bagi frontend untuk browsing katalog serta bagi Auction Service untuk validasi listing.

---

#### Component Diagram — Catalog Service

Diagram ini menunjukkan komponen-komponen internal di dalam **Catalog Service** beserta cara mereka berkolaborasi untuk melayani request REST dari frontend/gateway dan memproses event dari message broker.

```mermaid
C4Component
    title Component Diagram — Catalog Service

    Person(seller, "Seller", "Penjual barang")
    Person(buyer, "Buyer", "Pencari barang")
    Person(admin, "Admin", "Moderator listing")

    Container_Boundary(catalog_api, "Catalog Service") {

        Component(listing_ctrl, "ListingController", "Spring @RestController\n/api/listings/**", "Endpoint REST untuk CRUD listing: seller membuat listing, admin moderasi (approve/reject), buyer browsing & search.")

        Component(category_ctrl, "CategoryController", "Spring @RestController\n/api/categories/**", "Endpoint REST untuk pengelolaan kategori barang (hanya Admin).")

        Component(internal_ctrl, "CatalogInternalController", "Spring @RestController\n/internal/catalog/**", "Endpoint internal (non-publik) untuk Auction Service: validasi listing aktif sebelum membuka lelang.")

        Component(listing_svc, "ListingService", "Spring @Service", "Business logic inti: membuat listing baru, mengubah status (DRAFT→PENDING→ACTIVE→SOLD), sinkronisasi current_price dari event lelang, dan query pencarian.")

        Component(category_svc, "CategoryService", "Spring @Service", "Mengelola hierarki kategori (parent-child), validasi kategori aktif, dan cascade update.")

        Component(image_svc, "ListingImageService", "Spring @Service", "Mengelola upload, penyimpanan, dan penghapusan gambar listing ke object storage.")

        Component(search_svc, "ListingSearchService", "Spring @Service", "Membangun query pencarian dinamis berdasarkan filter (keyword, kategori, harga, status) menggunakan JPA Specification.")

        Component(event_consumer, "CatalogEventConsumer", "Spring @Component\n(RabbitMQ Listener)", "Menerima event BidPlaced dan AuctionClosed dari broker, mendelegasikan ke ReliableEventProcessor untuk pemrosesan idempoten.")

        Component(reliable_proc, "ReliableEventProcessor", "Spring @Component", "Guard idempotency: memeriksa ProcessedEvent sebelum menjalankan handler. Jika event sudah diproses, pesan di-ACK dan dilewati.")

        Component(listing_repo, "ListingRepository", "Spring Data JPA", "R/W entitas Listing & ListingImage ke Catalog DB.")
        Component(category_repo, "CategoryRepository", "Spring Data JPA", "R/W entitas Category ke Catalog DB.")
        Component(processed_repo, "ProcessedEventRepository", "Spring Data JPA", "R/W tabel processed_events untuk deduplication event dari broker.")
    }

    ContainerDb(catalog_db, "Catalog DB", "PostgreSQL 15", "Tabel: listings, listing_images, categories, processed_events")
    Container(mq, "Message Broker", "CloudAMQP RabbitMQ", "Event-driven broker")
    Container(gateway, "API Gateway", "Nginx / Kong", "Single entry point semua request publik")
    Container(auction_api, "Auction Service", "Spring Boot", "Memanggil /internal/catalog untuk validasi listing")
    Container(storage, "Object Storage", "AWS S3 / Cloudflare R2", "Menyimpan gambar listing")

    %% User → Gateway → Controller
    Rel(seller, gateway, "Buat & kelola listing", "HTTPS")
    Rel(buyer, gateway, "Browse & search listing", "HTTPS")
    Rel(admin, gateway, "Moderasi & kelola kategori", "HTTPS")
    Rel(gateway, listing_ctrl, "Route /api/listings/**", "REST/JSON")
    Rel(gateway, category_ctrl, "Route /api/categories/**", "REST/JSON")

    %% Internal call dari Auction Service (bypass gateway)
    Rel(auction_api, internal_ctrl, "GET /internal/catalog/listings/{id}/validate", "REST/JSON")

    %% Controller → Service
    Rel(listing_ctrl, listing_svc, "Delegasi business logic", "Method call")
    Rel(listing_ctrl, search_svc, "Delegasi query pencarian", "Method call")
    Rel(listing_ctrl, image_svc, "Upload / hapus gambar", "Method call")
    Rel(category_ctrl, category_svc, "Delegasi business logic", "Method call")
    Rel(internal_ctrl, listing_svc, "validateListingForAuction(id)", "Method call")

    %% Event Consumer
    Rel(mq, event_consumer, "Deliver BidPlaced & AuctionClosed", "AMQP")
    Rel(event_consumer, reliable_proc, "process(event, type, handler)", "Method call")
    Rel(reliable_proc, processed_repo, "hasProcessed() / markProcessed()", "JPA")
    Rel(reliable_proc, listing_svc, "syncCurrentPrice() / markAsSold()", "Method call")

    %% Service → Repository → DB
    Rel(listing_svc, listing_repo, "R/W data listing", "JPA")
    Rel(category_svc, category_repo, "R/W data kategori", "JPA")
    Rel(image_svc, listing_repo, "R/W ListingImage", "JPA")
    Rel(image_svc, storage, "Upload / delete object", "HTTPS / SDK")
    Rel(listing_repo, catalog_db, "SQL", "JDBC/TCP")
    Rel(category_repo, catalog_db, "SQL", "JDBC/TCP")
    Rel(processed_repo, catalog_db, "SQL", "JDBC/TCP")
```

---

#### Code Diagram 1 — ListingService (Inti Business Logic)

Diagram ini menunjukkan struktur kelas **ListingService** beserta semua dependensi dan kelas kolaboratornya. `ListingService` adalah pusat logika bisnis untuk siklus hidup listing dan sinkronisasi harga dari event lelang.

```mermaid
classDiagram
    class ListingController {
        -ListingService listingService
        -ListingSearchService searchService
        -ListingImageService imageService
        +createListing(req: CreateListingRequest, sellerId: String) ListingResponse
        +getListingById(id: String) ListingResponse
        +searchListings(filter: ListingFilterRequest, pageable: Pageable) Page~ListingResponse~
        +updateListing(id: String, req: UpdateListingRequest, sellerId: String) ListingResponse
        +approveListing(id: String) ListingResponse
        +rejectListing(id: String, reason: String) ListingResponse
        +deleteListing(id: String, sellerId: String) void
    }

    class ListingService {
        -ListingRepository listingRepository
        -CategoryRepository categoryRepository
        -ListingMapper listingMapper
        +createListing(req: CreateListingRequest, sellerId: String) Listing
        +getListingById(id: String) Listing
        +updateListing(id: String, req: UpdateListingRequest, sellerId: String) Listing
        +approveListing(id: String) Listing
        +rejectListing(id: String, reason: String) Listing
        +deleteListing(id: String, sellerId: String) void
        +validateListingForAuction(listingId: String) ListingValidationResult
        +syncCurrentPrice(listingId: String, newPrice: Long) void
        +markAsSold(listingId: String, finalPrice: Long) void
        -assertOwnership(listing: Listing, sellerId: String) void
        -assertStatusTransitionAllowed(from: ListingStatus, to: ListingStatus) void
    }

    class ListingSearchService {
        -ListingRepository listingRepository
        +search(filter: ListingFilterRequest, pageable: Pageable) Page~Listing~
        -buildSpecification(filter: ListingFilterRequest) Specification~Listing~
    }

    class ListingMapper {
        +toResponse(listing: Listing) ListingResponse
        +toEntity(req: CreateListingRequest) Listing
    }

    class ListingRepository {
        <<interface>>
        +findByIdAndSellerId(id: String, sellerId: String) Optional~Listing~
        +findAll(spec: Specification~Listing~, pageable: Pageable) Page~Listing~
        +existsByIdAndStatus(id: String, status: ListingStatus) boolean
    }

    class CategoryRepository {
        <<interface>>
        +findByIdAndActiveTrue(id: String) Optional~Category~
    }

    class ListingValidationResult {
        -boolean valid
        -String reason
        -Long startingPrice
        +isValid() boolean
        +static ok(startingPrice: Long) ListingValidationResult
        +static fail(reason: String) ListingValidationResult
    }

    ListingController --> ListingService : uses
    ListingController --> ListingSearchService : uses
    ListingService --> ListingRepository : R/W
    ListingService --> CategoryRepository : validates category
    ListingService --> ListingMapper : transforms
    ListingService --> ListingValidationResult : returns
    ListingSearchService --> ListingRepository : queries
```

---

#### Code Diagram 2 — Domain Entities

Diagram ini menunjukkan seluruh entitas domain dalam Catalog Service beserta relasi dan enumerasi status yang mengatur siklus hidup sebuah listing.

```mermaid
classDiagram
    class Listing {
        -String id
        -String sellerId
        -String title
        -String description
        -Long startingPrice
        -Long currentPrice
        -ListingStatus status
        -String rejectionReason
        -Category category
        -List~ListingImage~ images
        -OffsetDateTime createdAt
        -OffsetDateTime updatedAt
        +prePersist() void
        +preUpdate() void
    }

    class ListingImage {
        -Long id
        -Listing listing
        -String imageUrl
        -String storageKey
        -Boolean isPrimary
        -OffsetDateTime uploadedAt
    }

    class Category {
        -String id
        -String name
        -String slug
        -Category parent
        -List~Category~ children
        -Boolean active
        -OffsetDateTime createdAt
        +prePersist() void
    }

    class ProcessedEvent {
        -String eventId
        -String eventType
        -OffsetDateTime processedAt
    }

    class ListingStatus {
        <<enumeration>>
        DRAFT
        PENDING_REVIEW
        ACTIVE
        REJECTED
        SOLD
        ARCHIVED
    }

    Listing --> ListingStatus : status
    Listing --> Category : belongs to
    Listing "1" --> "*" ListingImage : has
    Category --> Category : parent (self-ref)
```

---

#### Code Diagram 3 — Idempotent Event Consumer (BidPlaced & AuctionClosed)

Diagram ini menunjukkan alur pemrosesan event dari RabbitMQ di dalam Catalog Service. Pola **ReliableEventProcessor** memastikan bahwa setiap event hanya diproses tepat satu kali meskipun broker mengirim ulang pesan yang sama (*at-least-once delivery*).

```mermaid
classDiagram
    class CatalogEventConsumer {
        -ReliableEventProcessor reliableEventProcessor
        +onBidPlaced(payload: Map~String,Object~) void
        +onAuctionClosed(payload: Map~String,Object~) void
        -parseBidPlacedEvent(payload: Map~String,Object~) BidPlacedEvent
        -parseAuctionClosedEvent(payload: Map~String,Object~) AuctionClosedEvent
    }

    class ReliableEventProcessor {
        -ProcessedEventRepository processedEventRepository
        +process(eventId: String, eventType: String, handler: Runnable) void
        -hasBeenProcessed(eventId: String) boolean
        -markAsProcessed(eventId: String, eventType: String) void
    }

    class ProcessedEventRepository {
        <<interface>>
        +existsByEventId(eventId: String) boolean
        +save(event: ProcessedEvent) ProcessedEvent
    }

    class ListingService {
        +syncCurrentPrice(listingId: String, newPrice: Long) void
        +markAsSold(listingId: String, finalPrice: Long) void
    }

    class BidPlacedEvent {
        -String eventId
        -String eventType
        -String auctionId
        -String listingId
        -String bidderId
        -Long amount
        -OffsetDateTime occurredAt
    }

    class AuctionClosedEvent {
        -String eventId
        -String eventType
        -String auctionId
        -String listingId
        -String winningBidderId
        -Long finalPrice
        -OffsetDateTime occurredAt
    }

    note for ReliableEventProcessor "Algorithm:\n1. Check processed_events table\n2. If found → skip (duplicate)\n3. If not found → run handler\n4. Insert to processed_events\n5. Commit"

    CatalogEventConsumer --> ReliableEventProcessor : delegates to
    CatalogEventConsumer --> BidPlacedEvent : parses
    CatalogEventConsumer --> AuctionClosedEvent : parses
    ReliableEventProcessor --> ProcessedEventRepository : checks & marks
    ReliableEventProcessor --> ListingService : calls syncCurrentPrice()
    ReliableEventProcessor --> ListingService : calls markAsSold()
```

---

#### Code Diagram 4 — Catalog Module (Frontend — Next.js)

Diagram ini menunjukkan struktur modul **Catalog** pada sisi frontend. Modul ini bertanggung jawab atas semua interaksi UI yang berkaitan dengan browsing katalog, detail listing, pencarian & filter, serta pengelolaan listing oleh Seller.

```mermaid
classDiagram
    class CatalogPage {
        <<Next.js Page — /catalog>>
        -ListingFilterState filterState
        -Page~ListingCard~ listingPage
        +render() JSX.Element
        +handleFilterChange(filter: ListingFilterRequest) void
        +handlePageChange(page: number) void
    }

    class ListingDetailPage {
        <<Next.js Page — /catalog/[id]>>
        -ListingDetail listing
        -boolean isOwner
        +render() JSX.Element
        +handleEditRedirect() void
    }

    class SellerListingPage {
        <<Next.js Page — /seller/listings>>
        -ListingDetail[] myListings
        +render() JSX.Element
        +handleCreateNew() void
        +handleDelete(id: String) void
    }

    class ListingCard {
        <<React Component>>
        -ListingCardProps props
        +render() JSX.Element
    }

    class ListingFilterPanel {
        <<React Component>>
        -CategoryOption[] categories
        -ListingFilterState state
        +onFilterChange: Function
        +render() JSX.Element
    }

    class CreateListingForm {
        <<React Component>>
        -CategoryOption[] categories
        -FormState state
        +handleSubmit() void
        +handleImageUpload(files: File[]) void
        +render() JSX.Element
    }

    class CatalogApiClient {
        <<Service>>
        -String BASE_URL
        +getListings(filter: ListingFilterRequest, page: number, size: number) Promise~PageResponse~ListingCard~~
        +getListingById(id: String) Promise~ListingDetail~
        +createListing(req: CreateListingRequest) Promise~ListingDetail~
        +updateListing(id: String, req: UpdateListingRequest) Promise~ListingDetail~
        +deleteListing(id: String) Promise~void~
        +getCategories() Promise~CategoryOption[]~
        +uploadImages(listingId: String, files: File[]) Promise~void~
    }

    class ListingFilterRequest {
        +String keyword
        +String categoryId
        +Long minPrice
        +Long maxPrice
        +ListingStatus status
    }

    class ListingDetail {
        +String id
        +String title
        +String description
        +Long startingPrice
        +Long currentPrice
        +ListingStatus status
        +CategoryOption category
        +String[] imageUrls
        +String sellerId
    }

    CatalogPage --> ListingCard : renders many
    CatalogPage --> ListingFilterPanel : uses
    CatalogPage --> CatalogApiClient : calls getListings()
    ListingDetailPage --> CatalogApiClient : calls getListingById()
    SellerListingPage --> CatalogApiClient : calls createListing() / deleteListing()
    SellerListingPage --> CreateListingForm : renders
    CreateListingForm --> CatalogApiClient : calls createListing() / uploadImages()
    CatalogApiClient --> ListingFilterRequest : accepts
    CatalogApiClient --> ListingDetail : returns
```