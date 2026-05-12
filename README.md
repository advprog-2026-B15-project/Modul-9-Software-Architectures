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

*[Tambahkan diagram]*

```mermaid
%% TEMPLATE Salin dan modifikasi Container Diagram dari Commit 1
%% Tambahkan komponen mitigasi seperti:
%% - API Gateway / Load Balancer
%% - CDN
%% - Monitoring (Prometheus, Grafana)
%% - Circuit Breaker
%% - Database clustering / read replica
%% - dll.

C4Container
    title Future Architecture Diagram (Post Risk Storming)
    %% [Isi dengan diagram hasil mitigasi risiko]
```

### Arsitektur Masa Depan BidMart

*[Tambahkan deskripsi singkat perubahan arsitektur yang dilakukan]*

**Perubahan dari Current ke Future Architecture:**

| Area | Current | Future | Alasan |
|---|---|---|---|
| *[Isi setelah risk storming]* | | | |

## Risk Storming Explanation

### 1. Identification

*[Jelaskan risiko apa saja yang diidentifikasi tiap anggota secara individual, area mana saja yang dinilai berisiko, dan skor risiko masing-masing menggunakan Risk Matrix (Impact × Likelihood = Score).]*

| Area Risiko | Deskripsi Risiko | Impact (1-3) | Likelihood (1-3) | Skor | Level |
|---|---|---|---|---|---|
| *[Tambahkan temuan masing-masing anggota]* | | | | | |

### 2. Consensus

*[Jelaskan bagaimana proses diskusi berjalan, perbedaan pendapat yang muncul antar anggota, dan bagaimana tim mencapai kesepakatan akhir mengenai level risiko tiap area.]*

### 3. Mitigation

*[Jelaskan solusi mitigasi yang dipilih untuk setiap risiko yang disepakati, mengapa solusi tersebut dipilih, dan hubungkan dengan perubahan arsitektur di Commit 2.]*

| Risiko | Solusi Mitigasi | Perubahan Arsitektur |
|---|---|---|
| *[Isi setelah diskusi]* | | |

## Individual Works

### Individual Work [Nama] ([NPM])

*[Menambahkan **Component Diagram** dan **Code Diagram** dari container yang dikerjakan.]*
