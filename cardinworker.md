# 🏗️ CARDIN TAXI WORKER — TIZIM ARXITEKTURASI
## System Architecture & Infrastructure Design

**Versiya**: 2.0 | **Sana**: 2026-02-18 | **Muallif**: Senior Architect

---

## 1. LOYIHANING UMUMIY KO'RINISHI (System Overview)

### 1.1. Maqsad va Doira

Cardin Taxi Worker — haydovchilarni **ro'yxatdan o'tkazish → tekshirish → tasdiqlash → kanalga yo'naltirish** to'liq hayot siklini boshqaruvchi platforma.

### 1.2. Stakeholderlar

| Rol | Tavsifi | Tizimga kirish usuli |
|-----|---------|---------------------|
| **Haydovchi** | Yangi ro'yxatdan o'tuvchi taksi haydovchisi | Telegram Bot → Web App |
| **Operator** | Arizalarni ko'rib chiquvchi xodim | Admin Panel (Web) |
| **Administrator** | Tizim boshqaruvchisi | Admin Panel (Web) |
| **Tizim** | Avtomatik jarayonlar (SMS, Notification) | Backend Services |

---

## 2. HIGH-LEVEL ARCHITECTURE

### 2.1. Tizim Kontekst Diagrammasi (C4 — Level 0)

```mermaid
flowchart TB
    subgraph External["🌍 TASHQI TIZIMLAR"]
        TelegramAPI["📱 Telegram Bot API"]
        SMSGateway["📨 SMS Gateway\n(Eskiz.uz)"]
        FileStorage["☁️ Object Storage\n(S3 / MinIO)"]
    end

    subgraph System["🏢 CARDIN TAXI WORKER"]
        CTW["Cardin Taxi Worker\nPlatformasi"]
    end

    subgraph Users["👥 FOYDALANUVCHILAR"]
        Driver["🚗 Haydovchi"]
        Operator["👨‍💼 Operator"]
        Admin["🔑 Administrator"]
    end

    Driver -->|"Telegram orqali\nro'yxatdan o'tish"| CTW
    Operator -->|"Arizalarni\nboshqarish"| CTW
    Admin -->|"Tizimni\nsozlash"| CTW

    CTW <-->|"Bot xabarlar\nWeb App"| TelegramAPI
    CTW -->|"SMS kodlar\nyuborish"| SMSGateway
    CTW <-->|"Hujjatlar\nyuklash/olish"| FileStorage

    style System fill:#1a237e,color:#fff
    style External fill:#e8eaf6
    style Users fill:#e3f2fd
```

### 2.2. Container Diagrammasi (C4 — Level 1)

```mermaid
flowchart TB
    subgraph ClientLayer["📱 CLIENT LAYER"]
        TGBot["🤖 Telegram Bot\n«Container»\nNode.js + Telegraf"]
        WebApp["🌐 Web Application\n«Container»\nReact 18 + Tailwind"]
        AdminPanel["🖥️ Admin Panel\n«Container»\nReact 18 + Tailwind"]
    end

    subgraph GatewayLayer["🔐 GATEWAY LAYER"]
        Nginx["⚡ Nginx\n«Reverse Proxy»\nSSL Termination\nRate Limiting\nLoad Balancing"]
    end

    subgraph ApplicationLayer["⚙️ APPLICATION LAYER"]
        AuthModule["🔑 Auth Module\nJWT + BCrypt\nSMS Verification"]
        UserModule["👤 User Module\nProfile CRUD\nVehicle Management"]
        AdminModule["👔 Admin Module\nApplications\nRegions & Operators"]
        NotifyModule["📤 Notification Module\nTelegram Messages\nInline Keyboards"]
        FileModule["📁 File Module\nUpload / Download\nImage Processing"]
    end

    subgraph DataLayer["💾 DATA LAYER"]
        PostgreSQL[("🐘 PostgreSQL 15\n«Primary Database»\nUsers, Vehicles,\nRegions, Operators")]
        Redis[("⚡ Redis 7\n«Cache & Sessions»\nSMS Codes, JWT,\nRate Limiting")]
        S3[("☁️ S3 / MinIO\n«Object Store»\nTexpassport rasmlar")]
    end

    TGBot -->|"HTTPS"| Nginx
    WebApp -->|"HTTPS"| Nginx
    AdminPanel -->|"HTTPS"| Nginx

    Nginx --> AuthModule
    Nginx --> UserModule
    Nginx --> AdminModule

    AuthModule --> PostgreSQL
    AuthModule --> Redis
    UserModule --> PostgreSQL
    UserModule --> FileModule
    AdminModule --> PostgreSQL
    AdminModule --> NotifyModule
    NotifyModule --> TGBot
    FileModule --> S3

    style ClientLayer fill:#e3f2fd
    style GatewayLayer fill:#fff3e0
    style ApplicationLayer fill:#e8f5e9
    style DataLayer fill:#fce4ec
```

### 2.3. Component Diagrammasi (C4 — Level 2)

```mermaid
flowchart TB
    subgraph API["⚙️ BACKEND API SERVER (Express.js)"]
        direction TB

        subgraph Middleware["🛡️ MIDDLEWARE STACK"]
            Helmet["helmet()\nSecurity Headers"]
            Cors["cors()\nOrigin Control"]
            RateLimiter["rate-limiter\n100 req/min"]
            BodyParser["body-parser\nJSON, Multipart"]
            JWTMiddleware["jwt-verify\nToken Validation"]
        end

        subgraph Routes["📡 ROUTE HANDLERS"]
            AuthRoutes["/api/auth/*\nregister, login,\nverify, refresh"]
            UserRoutes["/api/user/*\nprofile, vehicle,\nstatus"]
            AdminRoutes["/api/admin/*\napplications,\nregions, operators"]
            PublicRoutes["/api/public/*\nregions list,\nhealth check"]
        end

        subgraph Services["🔧 BUSINESS LOGIC"]
            AuthService["AuthService\nhashPassword()\nverifyToken()\ngenerateJWT()"]
            SMSService["SMSService\nsendCode()\nverifyCode()\nrateLimitCheck()"]
            UserService["UserService\ncreateUser()\nupdateProfile()\ngetStatus()"]
            VehicleService["VehicleService\naddVehicle()\nuploadPassport()"]
            AdminService["AdminService\napprove()\nreject()\ngetStats()"]
            NotificationService["NotificationService\nsendApproval()\nsendRejection()\nsendWelcome()"]
        end

        subgraph DataAccess["💾 DATA ACCESS"]
            UserRepo["UserRepository"]
            VehicleRepo["VehicleRepository"]
            RegionRepo["RegionRepository"]
            OperatorRepo["OperatorRepository"]
            LogRepo["ActivityLogRepository"]
        end
    end

    Middleware --> Routes
    Routes --> Services
    Services --> DataAccess

    style Middleware fill:#fff9c4
    style Routes fill:#c8e6c9
    style Services fill:#bbdefb
    style DataAccess fill:#ffccbc
```

---

## 3. NETWORK ARCHITECTURE

### 3.1. Tarmoq Topologiyasi

```mermaid
flowchart TB
    Internet["🌐 INTERNET"]

    subgraph DMZ["🛡️ DMZ ZONE"]
        DNS["DNS\nworker.cardintaxi.uz"]
        CDN["CDN\nStatic Assets"]
        LB["⚡ Nginx\nLoad Balancer\n:80 → :443"]
    end

    subgraph PrivateNet["🔒 PRIVATE NETWORK (172.20.0.0/16)"]
        subgraph AppSubnet["📦 App Subnet (172.20.1.0/24)"]
            FE["Frontend\n172.20.1.10:3000"]
            BE["Backend API\n172.20.1.20:5000"]
            Bot["Telegram Bot\n172.20.1.30:8443"]
        end

        subgraph DataSubnet["💾 Data Subnet (172.20.2.0/24)"]
            PG["PostgreSQL\n172.20.2.10:5432"]
            RD["Redis\n172.20.2.20:6379"]
        end

        subgraph StorageSubnet["☁️ Storage Subnet (172.20.3.0/24)"]
            MinIO["MinIO / S3\n172.20.3.10:9000"]
        end
    end

    Internet --> DNS
    DNS --> CDN
    CDN --> LB
    LB -->|":3000"| FE
    LB -->|":5000/api"| BE
    Internet -->|"Webhook"| Bot

    BE --> PG
    BE --> RD
    BE --> MinIO
    Bot --> BE

    style DMZ fill:#fff3e0
    style PrivateNet fill:#e8eaf6
    style AppSubnet fill:#e3f2fd
    style DataSubnet fill:#fce4ec
    style StorageSubnet fill:#e8f5e9
```

### 3.2. Port va Protokollar Matritsasi

| Xizmat | Port | Protokol | Kirish | Tavsif |
|--------|------|----------|--------|--------|
| Nginx | 80, 443 | HTTP/HTTPS | Public | Reverse proxy, SSL termination |
| Frontend | 3000 | HTTP | Internal | React Dev/Prod server |
| Backend API | 5000 | HTTP | Internal | Express.js REST API |
| Telegram Bot | 8443 | HTTPS | Public | Webhook endpoint |
| PostgreSQL | 5432 | TCP | Internal | Primary database |
| Redis | 6379 | TCP | Internal | Cache & sessions |
| MinIO | 9000, 9001 | HTTP | Internal | Object storage + Console |

---

## 4. DEPLOYMENT ARCHITECTURE

### 4.1. Docker Compose Stack

```mermaid
flowchart TB
    subgraph DockerHost["🐳 DOCKER HOST (Ubuntu 22.04 LTS)"]
        subgraph Compose["docker-compose.yml"]
            subgraph FrontendC["📱 frontend"]
                React["node:18-alpine\nReact Build\nServed by Nginx"]
            end

            subgraph BackendC["⚙️ backend"]
                Node["node:18-alpine\nExpress.js\nPM2 Process Manager"]
            end

            subgraph BotC["🤖 bot"]
                TGBot["node:18-alpine\nTelegraf Bot\nWebhook Mode"]
            end

            subgraph NginxC["⚡ nginx"]
                NginxImg["nginx:alpine\nReverse Proxy\nSSL + Gzip"]
            end

            subgraph PGContainer["🐘 postgres"]
                PG["postgres:15-alpine\nMax Connections: 100\nShared Buffers: 256MB"]
            end

            subgraph RedisC["⚡ redis"]
                RD["redis:7-alpine\nMaxmemory: 256MB\nEviction: allkeys-lru"]
            end

            subgraph MinIOC["☁️ minio"]
                MinIO["minio:latest\nBucket: cardin-files\nConsole: :9001"]
            end
        end

        subgraph Volumes["📁 PERSISTENT VOLUMES"]
            V1["pg_data\n→ /var/lib/postgresql/data"]
            V2["redis_data\n→ /data"]
            V3["minio_data\n→ /data"]
            V4["nginx_certs\n→ /etc/letsencrypt"]
            V5["app_logs\n→ /var/log/app"]
        end
    end

    NginxC --> FrontendC
    NginxC --> BackendC
    BackendC --> PGContainer
    BackendC --> RedisC
    BackendC --> MinIOC
    BotC --> BackendC

    PGContainer --> V1
    RedisC --> V2
    MinIOC --> V3
    NginxC --> V4

    style DockerHost fill:#e3f2fd
    style Compose fill:#f3e5f5
    style Volumes fill:#fff9c4
```

### 4.2. Health Check & Recovery

```mermaid
flowchart TD
    Timer["⏱️ Health Check Timer\nHar 30 soniyada"] --> Check{"Service\nJavob beryaptimi?"}

    Check -->|"✅ Healthy"| Normal["Normal ishlash"]
    Check -->|"❌ Unhealthy"| Count{"Consecutive\nfails >= 3?"}

    Count -->|"Yo'q"| Retry["Qayta tekshirish"]
    Count -->|"Ha"| Restart["🔄 Container restart"]

    Restart --> Notify["📨 Alert yuborish\n(Telegram/Email)"]
    Notify --> WaitStart["⏳ Startup kutish\n(30s grace period)"]
    WaitStart --> Check

    Retry --> Timer

    subgraph HealthEndpoints["🏥 HEALTH ENDPOINTS"]
        H1["GET /health\n→ 200 OK + uptime"]
        H2["GET /health/db\n→ PostgreSQL ping"]
        H3["GET /health/redis\n→ Redis ping"]
        H4["GET /health/storage\n→ S3 bucket check"]
    end
```

---

## 5. DATA FLOW ARCHITECTURE

### 5.1. Umumiy Ma'lumotlar Oqimi

```mermaid
flowchart LR
    subgraph Ingress["📥 DATA INGRESS"]
        UserInput["Form Data\nTelefon, Parol,\nShaxsiy ma'lumotlar"]
        FileInput["Binary Data\nTexpassport rasmlar"]
        TelegramInput["Telegram Data\ntelegram_id, username"]
    end

    subgraph Processing["⚙️ PROCESSING PIPELINE"]
        Validate["1️⃣ Validatsiya\nInput sanitize\nFormat check"]
        Transform["2️⃣ Transformatsiya\nPaol hash\nRasm compress"]
        Persist["3️⃣ Saqlash\nDB write\nFile upload"]
        Index["4️⃣ Indekslash\nCache warm\nSearch index"]
    end

    subgraph Egress["📤 DATA EGRESS"]
        APIResp["JSON Response\nProfile, Status"]
        Notification["Telegram Xabar\nApprove/Reject"]
        FileServe["Signed URL\nHujjat ko'rish"]
    end

    Ingress --> Processing
    Processing --> Egress
```

### 5.2. Request-Response Sikli

```mermaid
sequenceDiagram
    participant C as 📱 Client
    participant N as ⚡ Nginx
    participant M as 🛡️ Middleware
    participant R as 📡 Router
    participant S as 🔧 Service
    participant D as 💾 Database
    participant Ca as ⚡ Cache

    C->>N: HTTPS Request
    N->>N: SSL Termination
    N->>N: Rate Limit Check
    N->>M: Forward Request

    M->>M: helmet(), cors()
    M->>M: body-parser
    M->>M: JWT verify

    alt Token Valid
        M->>R: Route Match
        R->>S: Business Logic

        S->>Ca: Cache Check
        alt Cache Hit
            Ca-->>S: Cached Data
        else Cache Miss
            S->>D: SQL Query
            D-->>S: Result Set
            S->>Ca: Cache Write (TTL)
        end

        S-->>R: Service Response
        R-->>N: JSON Response
        N-->>C: 200 OK
    else Token Invalid
        M-->>N: Auth Error
        N-->>C: 401 Unauthorized
    end
```

---

## 6. SCALABILITY STRATEGY

### 6.1. Kuzatuv va Scaling Qarorlari

```mermaid
flowchart TB
    subgraph Metrics["📊 MONITORING METRICS"]
        CPU["CPU Usage > 80%"]
        Mem["Memory Usage > 85%"]
        Req["Request Latency > 500ms"]
        Queue["Queue Depth > 100"]
        DB["DB Connections > 80%"]
    end

    subgraph Decisions["🤔 SCALING DECISIONS"]
        HScale["↔️ Horizontal Scale\nAdd more containers"]
        VScale["↕️ Vertical Scale\nIncrease resources"]
        CacheOpt["⚡ Cache Optimization\nRedis tuning"]
        DBOpt["🐘 DB Optimization\nQuery tuning, indexes"]
        QueueScale["📋 Queue Scale\nAdd workers"]
    end

    CPU --> HScale
    CPU --> VScale
    Mem --> VScale
    Req --> CacheOpt
    Req --> DBOpt
    Queue --> QueueScale
    DB --> DBOpt
```

### 6.2. Capacity Planning

| Metrika | Boshlang'ich | O'rta Yuk | Yuqori Yuk |
|---------|-------------|-----------|------------|
| Foydalanuvchilar | 0-500 | 500-2000 | 2000-10000 |
| API req/s | 10-50 | 50-200 | 200-1000 |
| DB connections | 10-20 | 20-50 | 50-100 |
| Redis memory | 64MB | 128MB | 512MB |
| File storage | 5GB | 25GB | 100GB |
| Backend instances | 1 | 2 | 4 |

---

## 7. ENVIRONMENT CONFIGURATION

### 7.1. Environment Matritsasi

```mermaid
flowchart LR
    subgraph Environments["🌍 MUHITLAR"]
        Dev["🛠️ DEVELOPMENT\nlocalhost\nDocker Compose\nDebug Mode ON"]
        Staging["🧪 STAGING\nstaging.cardintaxi.uz\nDocker Compose\nProduction-like"]
        Prod["🚀 PRODUCTION\nworker.cardintaxi.uz\nDocker Compose\nFull Security"]
    end

    Dev -->|"PR Merged"| Staging
    Staging -->|"QA Passed"| Prod
```

### 7.2. Environment Variables (.env)

| O'zgaruvchi | Tavsif | Misol |
|-------------|--------|-------|
| `NODE_ENV` | Muhit turi | production |
| `PORT` | API port | 5000 |
| `DATABASE_URL` | PostgreSQL connection | postgresql://user:pass@db:5432/cardin |
| `REDIS_URL` | Redis connection | redis://redis:6379 |
| `JWT_SECRET` | JWT imzolash kaliti | [random 64 char] |
| `JWT_REFRESH_SECRET` | Refresh token kaliti | [random 64 char] |
| `JWT_ACCESS_TTL` | Access token muddati | 3600 (1 soat) |
| `JWT_REFRESH_TTL` | Refresh token muddati | 2592000 (30 kun) |
| `TELEGRAM_BOT_TOKEN` | Bot API token | 123456:ABC-DEF... |
| `TELEGRAM_WEBHOOK_URL` | Webhook URL | https://api.../webhook |
| `SMS_API_URL` | SMS Gateway URL | https://notify.eskiz.uz |
| `SMS_API_TOKEN` | SMS API token | [eskiz token] |
| `S3_ENDPOINT` | Object storage endpoint | http://minio:9000 |
| `S3_BUCKET` | Bucket nomi | cardin-files |
| `S3_ACCESS_KEY` | S3 access key | [key] |
| `S3_SECRET_KEY` | S3 secret key | [secret] |
| `BCRYPT_ROUNDS` | Hashing rounds | 12 |
| `RATE_LIMIT_WINDOW` | Rate limit oynasi (ms) | 60000 |
| `RATE_LIMIT_MAX` | Max requestlar | 100 |

---

## 8. LOYIHA PAPKA STRUKTURASI

```
cardin-taxi-worker/
├── 📁 frontend/                 # React Web App
│   ├── 📁 public/
│   ├── 📁 src/
│   │   ├── 📁 components/       # UI komponentlar
│   │   │   ├── 📁 auth/         # Login, Register
│   │   │   ├── 📁 profile/      # Profil sahifalari
│   │   │   ├── 📁 common/       # Umumiy komponentlar
│   │   │   └── 📁 layout/       # Layout, Header, Footer
│   │   ├── 📁 pages/            # Sahifalar
│   │   ├── 📁 hooks/            # Custom React hooks
│   │   ├── 📁 services/         # API calls (Axios)
│   │   ├── 📁 store/            # State management
│   │   ├── 📁 utils/            # Yordamchi funksiyalar
│   │   ├── 📁 i18n/             # Til fayllari (uz, ru)
│   │   ├── 📁 styles/           # CSS / Tailwind
│   │   └── App.jsx
│   ├── tailwind.config.js
│   ├── Dockerfile
│   └── package.json
│
├── 📁 backend/                  # Express.js API
│   ├── 📁 src/
│   │   ├── 📁 config/           # DB, Redis, S3 config
│   │   ├── 📁 middleware/       # Auth, CORS, Rate limit
│   │   ├── 📁 routes/           # API route definitions
│   │   ├── 📁 controllers/      # Request handlers
│   │   ├── 📁 services/         # Business logic
│   │   ├── 📁 repositories/     # Data access layer
│   │   ├── 📁 models/           # DB models (Knex/Sequelize)
│   │   ├── 📁 validators/       # Input validation schemas
│   │   ├── 📁 utils/            # Helpers, constants
│   │   └── app.js               # Express app setup
│   ├── 📁 migrations/           # DB migrations
│   ├── 📁 seeds/                # DB seed data
│   ├── 📁 tests/                # Test fayllar
│   ├── Dockerfile
│   └── package.json
│
├── 📁 bot/                      # Telegram Bot
│   ├── 📁 src/
│   │   ├── 📁 commands/         # Bot komandalar
│   │   ├── 📁 handlers/         # Xabar handlerlari
│   │   ├── 📁 keyboards/        # Inline/Reply keyboards
│   │   ├── 📁 services/         # API integration
│   │   └── bot.js               # Bot initialization
│   ├── Dockerfile
│   └── package.json
│
├── 📁 admin/                    # Admin Panel (React)
│   ├── 📁 src/
│   │   ├── 📁 components/       # Admin UI komponentlar
│   │   ├── 📁 pages/            # Dashboard, Applications
│   │   ├── 📁 services/         # Admin API calls
│   │   └── App.jsx
│   ├── Dockerfile
│   └── package.json
│
├── 📁 nginx/                    # Nginx konfiguratsiya
│   ├── nginx.conf
│   └── ssl/
│
├── 📁 docs/                     # Hujjatlar
│   ├── 01_TIZIM_ARXITEKTURASI.md
│   ├── 02_ALGORITMLAR.md
│   ├── 03_DATABASE.md
│   ├── 04_USER_FLOWS.md
│   ├── 05_API_DOCS.md
│   ├── 06_XAVFSIZLIK.md
│   └── 07_DEVOPS.md
│
├── docker-compose.yml
├── docker-compose.dev.yml
├── .env.example
├── .gitignore
└── README.md
```

---

> [!IMPORTANT]
> Bu hujjat tizimning arxitekturaviy asosini belgilaydi. Barcha keyingi hujjatlar (algoritmlar, database, API) shu arxitekturaga asoslanadi.

---

**Keyingi hujjat**: [02_ALGORITMLAR.md](file:///c:/Users/ZenBook/OneDrive/Рабочий%20стол/Cardin%20Taxi%20Worker/02_ALGORITMLAR.md)
