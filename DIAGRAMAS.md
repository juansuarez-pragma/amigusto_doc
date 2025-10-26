# Diagramas de Arquitectura - Amigusto
## Visualizaciones de Flujos y Arquitectura del Sistema

> Estos diagramas utilizan Mermaid syntax, que se renderiza automáticamente en GitHub, GitLab y muchas otras plataformas.

---

## 1. ARQUITECTURA GENERAL DEL SISTEMA

```mermaid
graph TB
    subgraph "CAPA DE CLIENTE"
        iOS["📱 App iOS<br/>(Swift + SwiftUI)"]
        Android["🤖 App Android<br/>(Kotlin + Compose)"]
        Portal["💼 Portal Web B2B<br/>(Angular)"]
        Admin["⚙️ Panel Admin<br/>(Angular)"]
    end

    subgraph "CAPA DE API"
        Gateway["API Gateway<br/>(Spring Cloud)"]

        subgraph "Servicios"
            AuthAPI["Auth Service<br/>(Spring Boot)"]
            EventsAPI["Events Service<br/>(Spring Boot)"]
            ContentAPI["Content Service<br/>(Spring Boot)"]
        end
    end

    subgraph "CAPA DE DATOS"
        Postgres[(PostgreSQL 16<br/>+ PostGIS)]
        Redis[(Redis 7<br/>Cache)]
        S3[("S3/Cloudinary<br/>Storage")]
        Search[(Elasticsearch<br/>Búsqueda)]
    end

    iOS --> Gateway
    Android --> Gateway
    Portal --> Gateway
    Admin --> Gateway

    Gateway --> AuthAPI
    Gateway --> EventsAPI
    Gateway --> ContentAPI

    AuthAPI --> Postgres
    AuthAPI --> Redis
    EventsAPI --> Postgres
    EventsAPI --> Redis
    EventsAPI --> Search
    ContentAPI --> Postgres
    ContentAPI --> S3

    style iOS fill:#007AFF
    style Android fill:#3DDC84
    style Portal fill:#DD0031
    style Admin fill:#FF9800
    style Postgres fill:#336791
    style Redis fill:#DC382D
```

---

## 2. FLUJO DE CURACIÓN DE EVENTOS (LA "SALSA SECRETA")

```mermaid
sequenceDiagram
    participant P as 🏢 Promotor<br/>(Portal Angular)
    participant API as 🔧 Backend<br/>(Spring Boot)
    participant DB as 💾 PostgreSQL
    participant A as 👤 Admin<br/>(Panel Angular)
    participant iOS as 📱 Usuario iOS<br/>(Swift)
    participant Android as 🤖 Usuario Android<br/>(Kotlin)

    Note over P,Android: Flujo de Creación y Aprobación de Evento

    P->>API: 1. Crear evento (POST /api/v1/promoter/events)
    API->>DB: 2. Guardar con status="DRAFT"
    DB-->>API: Evento creado
    API-->>P: Confirmación

    P->>API: 3. Enviar a revisión (POST /events/{id}/submit-review)
    API->>DB: 4. Actualizar status="PENDING_REVIEW"
    DB-->>API: Status actualizado
    API-->>P: "Evento enviado a revisión"

    Note over A: Admin revisa el evento

    A->>API: 5. Ver cola (GET /api/v1/admin/events?status=PENDING)
    API->>DB: Query eventos pendientes
    DB-->>API: Lista de eventos
    API-->>A: Eventos para revisar

    A->>API: 6. Aprobar evento (POST /api/v1/admin/events/{id}/approve)
    API->>DB: 7. Actualizar status="APPROVED"
    API->>DB: 8. Set publishedAt=now()
    DB-->>API: Evento aprobado
    API->>P: 9. Email de notificación
    API-->>A: Confirmación

    Note over iOS,Android: Usuarios descubren el evento

    iOS->>API: 10. GET /api/v1/events?city=Bogotá&gustos=cafe
    API->>DB: 11. Query WHERE status=APPROVED
    DB-->>API: Eventos aprobados
    API-->>iOS: Lista de eventos

    Android->>API: 12. GET /api/v1/events?city=Bogotá
    API->>DB: Query eventos aprobados
    DB-->>API: Eventos
    API-->>Android: Lista de eventos

    Note over P,Android: ✅ Evento visible en apps móviles
```

---

## 3. FLUJO DE USUARIO EN LAS APPS MÓVILES

```mermaid
graph LR
    A[🚀 Abrir App<br/>iOS o Android] --> B{¿Primera vez?}

    B -->|Sí| C[📋 Onboarding:<br/>Seleccionar Gustos]
    B -->|No| E[📍 Detectar<br/>Ubicación]

    C --> D[💾 Guardar Gustos<br/>CoreData/Room]
    D --> E

    E --> F[🔍 Feed de<br/>Descubrimiento]

    F --> G{Acción}

    G -->|Click evento| H[📄 Ver Detalle]
    G -->|Scroll down| I[📥 Cargar más]
    G -->|Pull refresh| J[🔄 Refrescar]

    H --> K{Acción Detalle}

    K -->|Guardar| L[💾 Añadir a<br/>Mis Planes]
    K -->|Compartir| M[📤 Share Sheet<br/>iOS/Android]
    K -->|Ver mapa| N[🗺️ Abrir Maps]

    L --> O[✅ Guardado]
    M --> O
    N --> O

    O --> F
    I --> F
    J --> F

    style A fill:#4CAF50
    style C fill:#FF9800
    style F fill:#2196F3
    style L fill:#9C27B0
```

---

## 4. MODELO DE DATOS (ENTITY-RELATIONSHIP)

```mermaid
erDiagram
    USER ||--o{ SAVED_EVENT : guarda
    USER ||--o| PROMOTER : "puede ser"
    USER }o--o{ GUSTO : selecciona

    PROMOTER ||--o{ EVENT : crea

    EVENT }o--o{ GUSTO : "etiquetado con"
    EVENT ||--o{ SAVED_EVENT : "guardado por"

    CITY ||--o{ EVENT : "tiene"

    USER {
        uuid id PK
        varchar email UK
        varchar name
        enum role "CONSUMER|PROMOTER|ADMIN"
        varchar city
        decimal lat
        decimal lng
    }

    PROMOTER {
        uuid id PK
        uuid userId FK
        varchar organizationName
        enum status "PENDING|VERIFIED|SUSPENDED"
    }

    EVENT {
        uuid id PK
        uuid promoterId FK
        varchar title
        text description
        varchar imageUrl
        timestamp startDate
        timestamp endDate
        varchar venueName
        varchar city
        decimal lat
        decimal lng
        boolean isFree
        decimal price
        enum status "DRAFT|PENDING|APPROVED|REJECTED"
    }

    GUSTO {
        uuid id PK
        varchar name UK
        varchar slug UK
        varchar icon
        varchar color
    }

    SAVED_EVENT {
        uuid id PK
        uuid userId FK
        uuid eventId FK
        timestamp createdAt
    }

    CITY {
        uuid id PK
        varchar name UK
        varchar slug UK
        decimal lat
        decimal lng
        boolean isActive
    }
```

---

## 5. ESTADOS DE UN EVENTO (STATE MACHINE)

```mermaid
stateDiagram-v2
    [*] --> DRAFT: Promotor crea evento

    DRAFT --> PENDING_REVIEW: Enviar a revisión
    DRAFT --> DRAFT: Editar

    PENDING_REVIEW --> APPROVED: Admin aprueba
    PENDING_REVIEW --> REJECTED: Admin rechaza

    REJECTED --> DRAFT: Promotor puede editar

    APPROVED --> ENDED: Fecha pasada
    APPROVED --> CANCELLED: Promotor cancela

    ENDED --> [*]
    CANCELLED --> [*]

    note right of DRAFT
        Visible solo para promotor
    end note

    note right of PENDING_REVIEW
        Visible para admins
        NO visible en apps
    end note

    note right of APPROVED
        ✅ VISIBLE EN iOS Y ANDROID
        para usuarios finales
    end note

    note right of REJECTED
        Promotor recibe email
        con razón del rechazo
    end note
```

---

## 6. ARQUITECTURA DE AUTENTICACIÓN (SPRING SECURITY + JWT)

```mermaid
sequenceDiagram
    participant C as Cliente<br/>(iOS/Android/Web)
    participant API as Spring Boot API
    participant DB as PostgreSQL
    participant R as Redis

    Note over C,R: Flujo de Login

    C->>API: POST /api/v1/auth/login<br/>{email, password}
    API->>DB: findByEmail(email)
    DB-->>API: User entity
    API->>API: BCrypt.matches(password)

    alt Password válido
        API->>API: Generar JWT Access Token<br/>(15min)
        API->>API: Generar Refresh Token<br/>(7 días)
        API->>R: Guardar refresh token
        R-->>API: OK
        API-->>C: {accessToken, refreshToken, user}

        Note over C: Cliente guarda tokens<br/>Keychain(iOS)/DataStore(Android)
    else Password inválido
        API-->>C: 401 Unauthorized
    end

    Note over C,R: Uso del Access Token

    C->>API: GET /api/v1/events<br/>Header: Authorization Bearer <token>
    API->>API: JwtTokenProvider.validateToken()

    alt Token válido
        API->>DB: Ejecutar query
        DB-->>API: Datos
        API-->>C: 200 OK {data}
    else Token expirado
        API-->>C: 401 Token Expired
        Note over C: Cliente usa Refresh Token
    end

    Note over C,R: Renovación de Token

    C->>API: POST /api/v1/auth/refresh<br/>{refreshToken}
    API->>R: Verificar refresh token
    R-->>API: Token válido
    API->>API: Generar nuevo Access Token
    API-->>C: {accessToken}
```

---

## 7. FLUJO DE BÚSQUEDA GEOESPACIAL (POSTGIS)

```mermaid
flowchart TD
    A[🔍 Usuario busca eventos] --> B[Obtener ubicación<br/>CoreLocation/FusedLocation]

    B --> C[Obtener gustos seleccionados]

    C --> D{¿Tiene lat/lng?}

    D -->|Sí| E[Query con PostGIS<br/>ST_DWithin function]
    D -->|No| F[Query simple por ciudad]

    E --> G[Filtrar por:<br/>- status = APPROVED<br/>- ciudad = user.city<br/>- distancia <= radio km<br/>- gustos IN user.gustos]

    F --> H[Filtrar por:<br/>- status = APPROVED<br/>- ciudad = user.city<br/>- gustos IN user.gustos]

    G --> I[Ordenar por:<br/>1. startDate ASC<br/>2. distancia ASC]

    H --> J[Ordenar por:<br/>startDate ASC]

    I --> K[Paginación Spring Data<br/>Pageable interface]
    J --> K

    K --> L[Incluir relaciones JPA:<br/>@ManyToMany gustos<br/>@ManyToOne promoter]

    L --> M[ResponseEntity con:<br/>Page EventResponse]

    M --> N[📱 Renderizar en<br/>SwiftUI/Compose]

    style A fill:#4CAF50
    style E fill:#FF9800
    style N fill:#2196F3
```

---

## 8. PIPELINE DE CI/CD

```mermaid
flowchart LR
    A[🔨 Git Push] --> B{¿Qué branch?}

    B -->|feature/*| C[GitHub Actions:<br/>Lint + Test]
    B -->|develop| D[Deploy to Staging]
    B -->|main| E[Deploy to Production]

    C --> F{¿Tests pasan?}

    F -->|✅ Sí| G[Merge to develop]
    F -->|❌ No| H[❌ PR bloqueado]

    G --> D

    D --> I[Build Spring Boot JAR<br/>./mvnw package]
    D --> J[Build Angular<br/>ng build]
    D --> K[Test iOS<br/>xcodebuild test]
    D --> L[Test Android<br/>./gradlew test]

    I --> M[🧪 Staging Ready]
    J --> M
    K --> M
    L --> M

    M --> N{¿Aprobado QA?}

    N -->|Sí| O[Create Release]
    N -->|No| P[🐛 Fix bugs]

    P --> A

    O --> E

    E --> Q[Deploy Spring Boot<br/>AWS ECS/GCP]
    E --> R[Deploy Angular<br/>Vercel/Netlify]
    E --> S[Build iOS<br/>Fastlane + TestFlight]
    E --> T[Build Android<br/>./gradlew bundle]

    Q --> U[🚀 Production Live]
    R --> U
    S --> V[📦 App Store]
    T --> W[📦 Google Play]

    V --> X[🎉 Apps Live]
    W --> X

    style A fill:#4CAF50
    style H fill:#F44336
    style U fill:#2196F3
    style X fill:#9C27B0
```

---

## 9. ARQUITECTURA MULTI-REPOSITORIO

```mermaid
graph TB
    subgraph "Repositorios Git"
        R1["📦 amigusto-backend<br/>(Java Spring Boot)"]
        R2["📦 amigusto-ios<br/>(Swift + SwiftUI)"]
        R3["📦 amigusto-android<br/>(Kotlin + Compose)"]
        R4["📦 amigusto-web<br/>(Angular)"]
    end

    subgraph "Build & Deploy"
        B1["Maven/Gradle<br/>Build JAR"]
        B2["Xcode Build<br/>IPA"]
        B3["Gradle Build<br/>APK/AAB"]
        B4["Angular CLI<br/>Build Dist"]
    end

    subgraph "Hosting"
        H1["AWS ECS<br/>Spring Boot"]
        H2["TestFlight<br/>→ App Store"]
        H3["Google Play<br/>Console"]
        H4["Vercel<br/>Angular Apps"]
    end

    R1 --> B1 --> H1
    R2 --> B2 --> H2
    R3 --> B3 --> H3
    R4 --> B4 --> H4

    style R1 fill:#68A063
    style R2 fill:#FA7343
    style R3 fill:#3DDC84
    style R4 fill:#DD0031
```

---

## 10. ESCALAMIENTO PROGRESIVO

```mermaid
graph LR
    subgraph "Fase 1: MVP<br/>(0-10K usuarios)"
        MVP_API["Single Spring Boot<br/>Railway/Render"]
        MVP_DB["PostgreSQL<br/>2GB RAM"]
        MVP_CDN["Cloudflare<br/>Free Tier"]
    end

    subgraph "Fase 2: Crecimiento<br/>(10K-100K usuarios)"
        G_LB["AWS ALB<br/>Load Balancer"]
        G_API1["Spring Boot 1"]
        G_API2["Spring Boot 2"]
        G_API3["Spring Boot 3"]
        G_DB["PostgreSQL<br/>+ Read Replicas"]
        G_Redis["Redis Cluster"]
        G_CDN["CloudFront CDN"]
    end

    subgraph "Fase 3: Escala<br/>(100K-1M usuarios)"
        S_MS1["Auth Service<br/>(Spring Boot)"]
        S_MS2["Events Service<br/>(Spring Boot)"]
        S_MS3["User Service<br/>(Spring Boot)"]
        S_MQ["Message Queue<br/>RabbitMQ/Kafka"]
        S_DB["PostgreSQL<br/>Particionado"]
        S_ES["Elasticsearch"]
        S_Cache["Redis<br/>Sharding"]
    end

    MVP_API --> G_LB
    MVP_DB --> G_DB

    G_LB --> G_API1
    G_LB --> G_API2
    G_LB --> G_API3

    G_API1 --> S_MS1
    G_API2 --> S_MS2
    G_API3 --> S_MS3

    style MVP_API fill:#4CAF50
    style G_LB fill:#FF9800
    style S_MS1 fill:#2196F3
```

---

## 11. FLUJO DE ONBOARDING EN APPS MÓVILES

```mermaid
journey
    title Experiencia de Onboarding - Apps Nativas
    section iOS (Swift)
      Abrir App: 5: Usuario
      Splash Screen (SwiftUI): 3: Usuario
      Welcome View: 4: Usuario
      Seleccionar Gustos: 4: Usuario
      Solicitar Location Permission: 3: Usuario
      Guardar en CoreData: 5: Sistema
    section Android (Kotlin)
      Abrir App: 5: Usuario
      Splash Screen (Compose): 3: Usuario
      Welcome Screen: 4: Usuario
      Seleccionar Gustos: 4: Usuario
      Solicitar Location Permission: 3: Usuario
      Guardar en Room DB: 5: Sistema
    section Primera Experiencia
      Ver Feed personalizado: 5: Usuario
      Descubrir primer evento: 5: Usuario
      Guardar evento: 5: Usuario
      Share via Sistema: 5: Usuario
    section Retención
      Volver al día siguiente: 4: Usuario
      Ver nuevo contenido: 5: Usuario
```

---

## 12. STACK TECNOLÓGICO COMPLETO

```mermaid
graph TB
    subgraph "Frontend Móvil"
        iOS["iOS<br/>Swift 5.9<br/>SwiftUI<br/>Combine"]
        Android["Android<br/>Kotlin 1.9<br/>Jetpack Compose<br/>Hilt"]
    end

    subgraph "Frontend Web"
        Portal["Portal B2B<br/>Angular 17<br/>Material<br/>RxJS"]
        Admin["Admin Panel<br/>Angular 17<br/>Material<br/>NgRx"]
    end

    subgraph "Backend"
        API["Spring Boot 3.2<br/>Java 17<br/>Spring Security<br/>Spring Data JPA"]
    end

    subgraph "Bases de Datos"
        PG["PostgreSQL 16<br/>+ PostGIS"]
        RD["Redis 7<br/>Cache"]
        ES["Elasticsearch 8<br/>Search"]
    end

    subgraph "Infraestructura"
        S3["S3/Cloudinary<br/>Storage"]
        CDN["CloudFront<br/>CDN"]
        Mon["Prometheus<br/>Grafana<br/>Sentry"]
    end

    iOS --> API
    Android --> API
    Portal --> API
    Admin --> API

    API --> PG
    API --> RD
    API --> ES
    API --> S3

    S3 --> CDN
    API --> Mon

    style iOS fill:#007AFF
    style Android fill:#3DDC84
    style Portal fill:#DD0031
    style Admin fill:#FF9800
    style API fill:#6DB33F
    style PG fill:#336791
```

---

## Cómo Usar Estos Diagramas

### En GitHub/GitLab:
Los diagramas Mermaid se renderizan automáticamente en archivos `.md`.

### En Notion:
Copiar el código Mermaid en un bloque de código con tipo `mermaid`.

### En Confluence:
Usar el plugin "Mermaid for Confluence".

### Generar Imágenes:
Usar [Mermaid Live Editor](https://mermaid.live/) para exportar como PNG/SVG.

---

**Nota:** Estos diagramas reflejan la arquitectura actualizada con **Java Spring Boot** (backend), **Swift/SwiftUI** (iOS), **Kotlin/Jetpack Compose** (Android), y **Angular** (web). No hay referencias a tecnologías obsoletas.
