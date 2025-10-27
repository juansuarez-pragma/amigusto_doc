# Plan Técnico - Amigusto MVP
## Motor de Descubrimiento de Eventos Hiper-Personalizado
### Arquitectura de Microservicios

---

## 1. ARQUITECTURA GENERAL DEL SISTEMA

### 1.1 Vista de Alto Nivel (Microservicios)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         CAPA DE CLIENTE                                  │
├──────────────────────┬──────────────────────┬───────────────────────────┤
│   APP iOS (B2C)      │  APP Android (B2C)   │  WEB ANGULAR              │
│   Swift + SwiftUI    │  Kotlin + Compose    │  Portal B2B/Admin         │
└──────────┬───────────┴──────────┬───────────┴─────────┬─────────────────┘
           │                      │                     │
           └──────────────────────┼─────────────────────┘
                                  │ HTTPS/REST
                         ┌────────▼────────┐
                         │  🚪 API GATEWAY │
                         │ Spring Cloud    │ ← Rate Limiting
                         │ Port: 8080      │ ← JWT Validation
                         └────────┬────────┘
                                  │
              ┌───────────────────┼───────────────────┐
              │  🔍 EUREKA SERVER │  ⚙️ CONFIG SERVER │
              │  Service Discovery│  Port: 8888       │
              │  Port: 8761       │  Git-based Config │
              └───────────────────┴───────────────────┘
                                  │
    ┌─────────────┬───────────────┼──────────────┬─────────────┐
    │             │               │              │             │
┌───▼───┐   ┌────▼────┐   ┌──────▼──────┐  ┌───▼────┐  ┌─────▼─────┐
│ AUTH  │   │ EVENT   │   │    USER     │  │PROMOTER│  │NOTIFICATION│
│SERVICE│   │ SERVICE │   │   SERVICE   │  │SERVICE │  │  SERVICE   │
│ :8081 │   │  :8082  │   │    :8083    │  │ :8084  │  │   :8085    │
└───┬───┘   └────┬────┘   └──────┬──────┘  └───┬────┘  └─────┬─────┘
    │            │               │              │             │
    │     ┌──────▼───────────────▼──────────────▼─────────────▼──┐
    │     │         📨 RABBITMQ (Message Broker)                 │
    │     │         Events: user.*, event.*, cache.*             │
    │     └──────────────────────────────────────────────────────┘
    │            │               │              │             │
┌───▼────────────▼───────────────▼──────────────▼─────────────▼─────┐
│                    CAPA DE DATOS (Database per Service)            │
├────────────────────────────────────────────────────────────────────┤
│  Auth DB      Event DB     User DB      Promoter DB   Notif DB    │
│  PostgreSQL   PostGIS      PostgreSQL   PostgreSQL    MongoDB     │
│                                                                     │
│  🗄️ REDIS (Shared Cache)  📦 S3/Cloudinary  📊 Zipkin (Tracing)  │
└────────────────────────────────────────────────────────────────────┘
```

### 1.2 Principios Arquitectónicos de Microservicios

1. **Single Responsibility**: Cada microservicio tiene UNA responsabilidad clara y bien definida
2. **Database per Service**: Cada servicio tiene su propia base de datos independiente
3. **API-First**: Contratos API definidos primero (OpenAPI/Swagger) con versionado
4. **Stateless Services**: Microservicios sin estado, fácil escalado horizontal
5. **Event-Driven Architecture**: Comunicación asíncrona vía RabbitMQ para desacoplamiento
6. **Service Discovery**: Descubrimiento dinámico de servicios con Netflix Eureka
7. **Centralized Configuration**: Config Server para configuración centralizada
8. **Circuit Breaker Pattern**: Resiliencia con Resilience4j (circuit breakers, retries, timeouts)
9. **Distributed Tracing**: Observabilidad completa con Zipkin + Sleuth
10. **API Gateway**: Punto de entrada único con autenticación centralizada

---

## 2. STACK TECNOLÓGICO RECOMENDADO

### 2.1 Frontend - Apps Móviles Nativas

#### 2.1.1 iOS App (B2C)

**Framework Principal: Swift con SwiftUI**

**Justificación:**
- Rendimiento nativo óptimo
- Acceso completo a APIs de iOS
- SwiftUI para UI declarativa moderna
- Mejor integración con el ecosistema Apple

**Stack Completo:**
```yaml
Lenguaje: Swift 5.9+
UI Framework: SwiftUI
Arquitectura: MVVM (Model-View-ViewModel)
Networking: URLSession + Alamofire
State Management: Combine + @StateObject / @ObservedObject
Local Storage: CoreData / UserDefaults / Keychain
Maps: MapKit
Image Handling: SDWebImage o Kingfisher
Location: CoreLocation
Analytics: Firebase Analytics
Dependency Injection: Resolver o Swinject
```

**Estructura de Carpetas:**
```
AmigustoiOS/
├── AmigustoiOS/
│   ├── App/
│   │   └── AmigustoApp.swift
│   ├── Models/
│   │   ├── Event.swift
│   │   ├── Gusto.swift
│   │   └── User.swift
│   ├── ViewModels/
│   │   ├── EventsViewModel.swift
│   │   ├── OnboardingViewModel.swift
│   │   └── MyPlansViewModel.swift
│   ├── Views/
│   │   ├── Onboarding/
│   │   │   ├── WelcomeView.swift
│   │   │   └── GustoSelectionView.swift
│   │   ├── Discover/
│   │   │   ├── DiscoverView.swift
│   │   │   ├── EventCard.swift
│   │   │   └── EventDetailView.swift
│   │   └── MyPlans/
│   │       └── MyPlansView.swift
│   ├── Services/
│   │   ├── APIService.swift
│   │   ├── EventService.swift
│   │   └── LocationService.swift
│   ├── Utilities/
│   │   ├── Extensions/
│   │   ├── Constants.swift
│   │   └── Formatters.swift
│   └── Resources/
│       ├── Assets.xcassets
│       └── Localizable.strings
├── AmigustoiOSTests/
└── AmigustoiOS.xcodeproj
```

#### 2.1.2 Android App (B2C)

**Framework Principal: Kotlin con Jetpack Compose**

**Justificación:**
- Rendimiento nativo óptimo
- Jetpack Compose para UI moderna
- Acceso completo a Android APIs
- Kotlin es el lenguaje oficial de Android

**Stack Completo:**
```yaml
Lenguaje: Kotlin 1.9+
UI Framework: Jetpack Compose
Arquitectura: MVVM (con Clean Architecture)
Networking: Retrofit + OkHttp
State Management: ViewModel + StateFlow / LiveData
Dependency Injection: Hilt (Dagger)
Local Storage: Room Database / DataStore / EncryptedSharedPreferences
Maps: Google Maps SDK for Android
Image Handling: Coil
Location: Google Play Services Location
Navigation: Jetpack Navigation Compose
Analytics: Firebase Analytics
```

**Estructura de Carpetas:**
```
AmigustoAndroid/
├── app/
│   ├── src/
│   │   ├── main/
│   │   │   ├── java/com/amigusto/
│   │   │   │   ├── AmigustoApplication.kt
│   │   │   │   ├── data/
│   │   │   │   │   ├── models/
│   │   │   │   │   │   ├── Event.kt
│   │   │   │   │   │   ├── Gusto.kt
│   │   │   │   │   │   └── User.kt
│   │   │   │   │   ├── repository/
│   │   │   │   │   │   ├── EventRepository.kt
│   │   │   │   │   │   └── UserRepository.kt
│   │   │   │   │   ├── api/
│   │   │   │   │   │   ├── ApiService.kt
│   │   │   │   │   │   └── RetrofitClient.kt
│   │   │   │   │   └── local/
│   │   │   │   │       └── AppDatabase.kt
│   │   │   │   ├── domain/
│   │   │   │   │   ├── usecase/
│   │   │   │   │   │   └── GetEventsUseCase.kt
│   │   │   │   │   └── model/
│   │   │   │   ├── presentation/
│   │   │   │   │   ├── onboarding/
│   │   │   │   │   │   ├── OnboardingViewModel.kt
│   │   │   │   │   │   └── OnboardingScreen.kt
│   │   │   │   │   ├── discover/
│   │   │   │   │   │   ├── DiscoverViewModel.kt
│   │   │   │   │   │   ├── DiscoverScreen.kt
│   │   │   │   │   │   └── EventCard.kt
│   │   │   │   │   └── myplans/
│   │   │   │   │       ├── MyPlansViewModel.kt
│   │   │   │   │       └── MyPlansScreen.kt
│   │   │   │   ├── ui/
│   │   │   │   │   ├── theme/
│   │   │   │   │   └── components/
│   │   │   │   └── utils/
│   │   │   │       ├── Constants.kt
│   │   │   │       └── Extensions.kt
│   │   │   └── res/
│   │   │       ├── values/
│   │   │       ├── drawable/
│   │   │       └── mipmap/
│   │   └── test/
│   └── build.gradle.kts
├── gradle/
└── build.gradle.kts
```

### 2.2 Frontend - Aplicaciones Web (Angular)

**Framework Principal: Angular 17+ (Standalone Components)**

**Justificación:**
- Framework enterprise-grade robusto
- TypeScript nativo
- Dependency Injection potente
- RxJS para programación reactiva
- Angular Material para UI components

#### 2.2.1 Portal Web B2B

**Stack Completo:**
```yaml
Framework: Angular 17+ (Standalone Components)
Lenguaje: TypeScript 5.2+
UI Framework: Angular Material 17+ / PrimeNG
State Management: NgRx (Redux pattern) o RxJS + Services
Forms: Reactive Forms
HTTP: HttpClient (built-in)
Routing: Angular Router
Maps: @angular/google-maps
Date Handling: date-fns / Luxon
Rich Text Editor: ngx-quill / TinyMCE
File Upload: ngx-dropzone / ng2-file-upload
```

**Estructura de Carpetas:**
```
amigusto-portal/
├── src/
│   ├── app/
│   │   ├── core/
│   │   │   ├── services/
│   │   │   │   ├── api.service.ts
│   │   │   │   ├── auth.service.ts
│   │   │   │   └── event.service.ts
│   │   │   ├── guards/
│   │   │   │   └── auth.guard.ts
│   │   │   ├── interceptors/
│   │   │   │   ├── auth.interceptor.ts
│   │   │   │   └── error.interceptor.ts
│   │   │   └── models/
│   │   │       ├── event.model.ts
│   │   │       ├── gusto.model.ts
│   │   │       └── user.model.ts
│   │   ├── features/
│   │   │   ├── auth/
│   │   │   │   ├── login/
│   │   │   │   │   ├── login.component.ts
│   │   │   │   │   ├── login.component.html
│   │   │   │   │   └── login.component.scss
│   │   │   │   └── register/
│   │   │   ├── dashboard/
│   │   │   │   ├── dashboard.component.ts
│   │   │   │   └── events-list/
│   │   │   ├── events/
│   │   │   │   ├── create-event/
│   │   │   │   │   ├── create-event.component.ts
│   │   │   │   │   └── event-form/
│   │   │   │   │       └── event-form.component.ts
│   │   │   │   └── edit-event/
│   │   │   └── profile/
│   │   ├── shared/
│   │   │   ├── components/
│   │   │   │   ├── event-status-badge/
│   │   │   │   ├── gusto-selector/
│   │   │   │   ├── image-uploader/
│   │   │   │   └── location-autocomplete/
│   │   │   ├── directives/
│   │   │   └── pipes/
│   │   │       ├── date-format.pipe.ts
│   │   │       └── price-format.pipe.ts
│   │   ├── app.component.ts
│   │   ├── app.config.ts
│   │   └── app.routes.ts
│   ├── assets/
│   ├── environments/
│   │   ├── environment.ts
│   │   └── environment.prod.ts
│   ├── styles.scss
│   └── main.ts
├── angular.json
├── package.json
└── tsconfig.json
```

#### 2.2.2 Panel de Administración

**Mismo stack que Portal B2B**, puede ser el mismo proyecto con rutas protegidas o un proyecto separado.

**Features Específicos:**
```yaml
Framework Base: Angular 17+
UI: Angular Material + Data Tables (ag-Grid o Material Table)
Charts: ngx-charts / Chart.js
Gestión de Estados: NgRx Store
```

### 2.3 Backend - API y Servicios

**Framework Principal: Java Spring Boot 3.x**

**Justificación:**
- Framework enterprise maduro y robusto
- Ecosistema completo (Spring Data, Security, Cloud)
- Excelente para microservicios
- Alta performance y escalabilidad
- Gran comunidad y documentación

**Stack Completo (Microservicios):**
```yaml
Lenguaje: Java 17+ (LTS)
Framework: Spring Boot 3.2+
Build Tool: Maven 3.9+ o Gradle 8+

# Microservices Infrastructure
API Gateway: Spring Cloud Gateway 4.0+
Service Discovery: Netflix Eureka Server
Config Server: Spring Cloud Config Server
Load Balancing: Spring Cloud LoadBalancer (client-side)

# Inter-Service Communication
Sync (REST): Spring Cloud OpenFeign
Async (Messaging): Spring AMQP + RabbitMQ 3.12+
Message Format: JSON (Jackson)

# Resilience
Circuit Breaker: Resilience4j
Retry: Resilience4j Retry
Timeout: Resilience4j TimeLimiter
Rate Limiting: Resilience4j RateLimiter

# Data Layer
ORM: Spring Data JPA + Hibernate
Database: PostgreSQL 16+ (con PostGIS para Event Service)
NoSQL: MongoDB 7+ (para Notification Service)
Cache: Spring Data Redis + Lettuce (compartido)

# Security
Auth: Spring Security + JWT (jjwt library)
Validation: Jakarta Bean Validation
Token Storage: Redis (refresh tokens)

# Observability
Distributed Tracing: Spring Cloud Sleuth + Zipkin
Metrics: Micrometer + Prometheus
Logging: SLF4J + Logback + ELK Stack
Health Checks: Spring Actuator

# API Documentation
API Docs: Springdoc OpenAPI (Swagger) por servicio
Contract Testing: Spring Cloud Contract (opcional)

# Testing
Unit Tests: JUnit 5 + Mockito
Integration: Spring Boot Test + Testcontainers
E2E: Rest Assured

# Otros
File Upload: AWS S3 SDK / Cloudinary SDK
Email: Spring Mail + Thymeleaf
Task Scheduling: Spring Scheduler
```

**Estructura de Carpetas (Proyecto Maven):**
```
amigusto-backend/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/amigusto/
│   │   │       ├── AmigustoApplication.java
│   │   │       ├── config/
│   │   │       │   ├── SecurityConfig.java
│   │   │       │   ├── RedisConfig.java
│   │   │       │   ├── S3Config.java
│   │   │       │   └── OpenApiConfig.java
│   │   │       ├── controller/
│   │   │       │   ├── AuthController.java
│   │   │       │   ├── EventController.java
│   │   │       │   ├── GustoController.java
│   │   │       │   ├── PromoterController.java
│   │   │       │   └── AdminController.java
│   │   │       ├── service/
│   │   │       │   ├── AuthService.java
│   │   │       │   ├── EventService.java
│   │   │       │   ├── GustoService.java
│   │   │       │   ├── UserService.java
│   │   │       │   └── EmailService.java
│   │   │       ├── repository/
│   │   │       │   ├── UserRepository.java
│   │   │       │   ├── EventRepository.java
│   │   │       │   ├── GustoRepository.java
│   │   │       │   └── PromoterRepository.java
│   │   │       ├── model/
│   │   │       │   ├── entity/
│   │   │       │   │   ├── User.java
│   │   │       │   │   ├── Event.java
│   │   │       │   │   ├── Gusto.java
│   │   │       │   │   ├── Promoter.java
│   │   │       │   │   └── SavedEvent.java
│   │   │       │   ├── dto/
│   │   │       │   │   ├── request/
│   │   │       │   │   │   ├── LoginRequest.java
│   │   │       │   │   │   ├── CreateEventRequest.java
│   │   │       │   │   │   └── EventFilterRequest.java
│   │   │       │   │   └── response/
│   │   │       │   │       ├── AuthResponse.java
│   │   │       │   │       ├── EventResponse.java
│   │   │       │   │       └── ApiResponse.java
│   │   │       │   └── enums/
│   │   │       │       ├── UserRole.java
│   │   │       │       ├── EventStatus.java
│   │   │       │       └── PromoterStatus.java
│   │   │       ├── security/
│   │   │       │   ├── JwtTokenProvider.java
│   │   │       │   ├── JwtAuthenticationFilter.java
│   │   │       │   └── UserDetailsServiceImpl.java
│   │   │       ├── exception/
│   │   │       │   ├── GlobalExceptionHandler.java
│   │   │       │   ├── ResourceNotFoundException.java
│   │   │       │   └── UnauthorizedException.java
│   │   │       └── util/
│   │   │           ├── DateUtil.java
│   │   │           ├── GeoUtil.java
│   │   │           └── ValidationUtil.java
│   │   └── resources/
│   │       ├── application.yml
│   │       ├── application-dev.yml
│   │       ├── application-prod.yml
│   │       ├── db/
│   │       │   └── migration/
│   │       │       └── V1__initial_schema.sql
│   │       └── templates/
│   │           └── email/
│   │               ├── event-approved.html
│   │               └── event-rejected.html
│   └── test/
│       └── java/
│           └── com/amigusto/
│               ├── controller/
│               ├── service/
│               └── repository/
├── pom.xml
├── .gitignore
└── README.md
```

### 2.4 Base de Datos

**Base de Datos Principal: PostgreSQL 16+**

**Justificación:**
- Robusta para datos relacionales
- Soporte para búsquedas geoespaciales (PostGIS)
- JSON/JSONB para datos flexibles
- Excelente rendimiento
- Compatible con Spring Data JPA

**Complementos:**
```yaml
PostgreSQL 16+: Base de datos principal
PostGIS: Extensión para queries geoespaciales
Redis 7+: Cache de sesiones, rate limiting
Elasticsearch 8+: Búsqueda full-text (opcional fase 2)
S3/Cloudinary: Storage de imágenes
Flyway/Liquibase: Migraciones de BD
```

### 2.5 Infraestructura y DevOps

**Hosting Recomendado (MVP):**

**Opción 1 - Bajo Costo:**
```yaml
Backend API: Railway.app / Render.com / Heroku ($10-25/mes)
Base de Datos: Railway Postgres / Supabase
Redis: Upstash Free Tier
Storage: Cloudinary Free Tier (10GB)
CDN: Cloudflare Free
Frontend Web: Vercel / Netlify Free Tier
Apps Móviles: App Store + Google Play
```

**Opción 2 - Escalable (Producción):**
```yaml
Backend API: AWS EC2 / ECS (Fargate) o GCP Cloud Run
Base de Datos: AWS RDS PostgreSQL / GCP Cloud SQL
Redis: AWS ElastiCache / GCP Memorystore
Storage: AWS S3 + CloudFront
Frontend Web: Vercel Pro / AWS S3 + CloudFront
Apps Móviles: App Store + Google Play + Firebase
Load Balancer: AWS ALB / GCP Load Balancer
```

**CI/CD:**
```yaml
Repository: GitHub/GitLab
CI/CD: GitHub Actions / GitLab CI / Jenkins
Workflows:
  - Build Spring Boot con Maven/Gradle
  - Tests unitarios + integración
  - Build iOS con Xcode Cloud / Fastlane
  - Build Android con Gradle
  - Build Angular con npm/Angular CLI
  - Deploy automático a Staging
  - Deploy manual a Producción
Container: Docker para Spring Boot
Orchestration: Kubernetes (opcional para escala)
```

**Monitoreo:**
```yaml
Logs: ELK Stack (Elasticsearch, Logstash, Kibana) o Datadog
APM: New Relic / Datadog / Dynatrace
Errors: Sentry
Uptime: UptimeRobot / Better Uptime
Analytics App: Firebase Analytics + Mixpanel
Analytics Web: Google Analytics 4
Spring Boot: Actuator + Prometheus + Grafana
```

---

## 3. DISEÑO DE BASE DE DATOS (Database per Service Pattern)

### 3.1 Visión General de Bases de Datos

En la arquitectura de microservicios, cada servicio tiene su propia base de datos independiente:

```
┌──────────────────┬──────────────────┬──────────────────┐
│  Auth Service    │  Event Service   │  User Service    │
│  ↓               │  ↓               │  ↓               │
│  auth_db         │  event_db        │  user_db         │
│  (PostgreSQL)    │  (PostgreSQL +   │  (PostgreSQL)    │
│                  │   PostGIS)       │                  │
├──────────────────┼──────────────────┼──────────────────┤
│ Tables:          │ Tables:          │ Tables:          │
│ - users          │ - events         │ - consumers      │
│ - refresh_tokens │ - event_gustos   │ - user_gustos    │
│                  │ - gustos         │ - saved_events   │
│                  │ - cities         │                  │
└──────────────────┴──────────────────┴──────────────────┘

┌──────────────────┬──────────────────┬──────────────────┐
│ Promoter Service │ Notification Svc │ Storage Service  │
│  ↓               │  ↓               │  ↓               │
│  promoter_db     │  notification_db │  (Stateless)     │
│  (PostgreSQL)    │  (MongoDB)       │                  │
├──────────────────┼──────────────────┼──────────────────┤
│ Tables:          │ Collections:     │ - AWS S3 /       │
│ - promoters      │ - notifications  │   Cloudinary     │
│                  │ - email_logs     │                  │
└──────────────────┴──────────────────┴──────────────────┘
```

**Principios:**
- ✅ Cada servicio tiene FULL ownership de su base de datos
- ✅ NO hay acceso directo cross-database
- ✅ Comunicación solo vía API (Feign) o eventos (RabbitMQ)
- ✅ Eventual consistency aceptada para datos replicados

### 3.2 Auth Service Database (auth_db)

**Propósito**: Autenticación y gestión de usuarios

```sql
-- ============================================
-- AUTH SERVICE DATABASE
-- ============================================

CREATE TYPE user_role AS ENUM ('CONSUMER', 'PROMOTER', 'ADMIN');

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    name VARCHAR(255) NOT NULL,
    role user_role DEFAULT 'CONSUMER',

    -- OAuth
    google_id VARCHAR(255) UNIQUE,
    apple_id VARCHAR(255) UNIQUE,

    is_active BOOLEAN DEFAULT true,
    email_verified BOOLEAN DEFAULT false,
    email_verified_at TIMESTAMP,

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE refresh_tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token VARCHAR(500) UNIQUE NOT NULL,
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    UNIQUE(user_id, token)
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role ON users(role);
CREATE INDEX idx_refresh_tokens_user ON refresh_tokens(user_id);
CREATE INDEX idx_refresh_tokens_token ON refresh_tokens(token);
```

### 3.3 Event Service Database (event_db)

**Propósito**: Gestión de eventos, gustos, búsqueda geoespacial

```sql
-- ============================================
-- EVENT SERVICE DATABASE
-- ============================================

-- Habilitar PostGIS
CREATE EXTENSION IF NOT EXISTS postgis;

CREATE TYPE event_status AS ENUM (
    'DRAFT',
    'PENDING_REVIEW',
    'APPROVED',
    'REJECTED',
    'CANCELLED',
    'ENDED'
);

CREATE TABLE events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    promoter_id UUID NOT NULL, -- FK lógico a Promoter Service

    -- Información básica
    title VARCHAR(255) NOT NULL,
    slug VARCHAR(255) UNIQUE NOT NULL,
    description TEXT NOT NULL,
    image_url VARCHAR(500),
    image_gallery TEXT[],

    -- Fechas
    start_date TIMESTAMP NOT NULL,
    end_date TIMESTAMP,
    start_time TIME,
    end_time TIME,
    timezone VARCHAR(50) DEFAULT 'America/Bogota',

    -- Ubicación (con PostGIS)
    venue_name VARCHAR(255) NOT NULL,
    venue_address VARCHAR(500) NOT NULL,
    city VARCHAR(100) NOT NULL,
    country VARCHAR(100) DEFAULT 'Colombia',
    lat DECIMAL(10, 8) NOT NULL,
    lng DECIMAL(11, 8) NOT NULL,
    location GEOGRAPHY(POINT, 4326), -- PostGIS point

    -- Precio
    is_free BOOLEAN DEFAULT false,
    price DECIMAL(10, 2),
    currency VARCHAR(10) DEFAULT 'EUR',
    external_url VARCHAR(500),

    -- Estado de curación
    status event_status DEFAULT 'DRAFT',
    reviewed_by UUID,
    reviewed_at TIMESTAMP,
    rejection_reason TEXT,

    -- Métricas
    view_count INT DEFAULT 0,
    save_count INT DEFAULT 0,

    -- Metadata
    metadata JSONB,

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Trigger para actualizar PostGIS point automáticamente
CREATE OR REPLACE FUNCTION update_location_point()
RETURNS TRIGGER AS $$
BEGIN
    NEW.location = ST_SetSRID(ST_MakePoint(NEW.lng, NEW.lat), 4326);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_update_location
    BEFORE INSERT OR UPDATE ON events
    FOR EACH ROW
    EXECUTE FUNCTION update_location_point();

CREATE INDEX idx_events_status ON events(status);
CREATE INDEX idx_events_city ON events(city);
CREATE INDEX idx_events_start_date ON events(start_date);
CREATE INDEX idx_events_promoter ON events(promoter_id);
CREATE INDEX idx_events_location ON events USING GIST(location);

-- ============================================
-- GUSTOS (Categorías)
-- ============================================

CREATE TABLE gustos (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) UNIQUE NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    icon VARCHAR(50),
    description TEXT,
    color VARCHAR(20),
    is_active BOOLEAN DEFAULT true,

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_gustos_slug ON gustos(slug);
CREATE INDEX idx_gustos_active ON gustos(is_active);

-- ============================================
-- EVENT_GUSTOS (Muchos a Muchos)
-- ============================================

CREATE TABLE event_gustos (
    event_id UUID REFERENCES events(id) ON DELETE CASCADE,
    gusto_id UUID REFERENCES gustos(id) ON DELETE CASCADE,
    PRIMARY KEY (event_id, gusto_id)
);

CREATE INDEX idx_event_gustos_event ON event_gustos(event_id);
CREATE INDEX idx_event_gustos_gusto ON event_gustos(gusto_id);

-- ============================================
-- CITIES
-- ============================================

CREATE TABLE cities (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) UNIQUE NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    country VARCHAR(100) DEFAULT 'España',
    lat DECIMAL(10, 8) NOT NULL,
    lng DECIMAL(11, 8) NOT NULL,
    radius DECIMAL(6, 2) DEFAULT 50.0,
    is_active BOOLEAN DEFAULT true,

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_cities_slug ON cities(slug);
CREATE INDEX idx_cities_active ON cities(is_active);
```

**Query Geoespacial (PostGIS):**

```sql
-- Encontrar eventos dentro de 50km de Madrid
SELECT e.id, e.title,
       ST_Distance(e.location, ST_SetSRID(ST_MakePoint(-3.7038, 40.4168), 4326)::geography) / 1000 AS distance_km
FROM events e
WHERE e.status = 'APPROVED'
  AND e.start_date > NOW()
  AND ST_DWithin(
      e.location,
      ST_SetSRID(ST_MakePoint(-3.7038, 40.4168), 4326)::geography,
      50000  -- 50km en metros
  )
ORDER BY distance_km ASC
LIMIT 20;
```

### 3.4 User Service Database (user_db)

**Propósito**: Perfiles de consumidores, eventos guardados

```sql
-- ============================================
-- USER SERVICE DATABASE
-- ============================================

CREATE TABLE consumers (
    id UUID PRIMARY KEY, -- Mismo ID que users de Auth Service
    email VARCHAR(255) UNIQUE NOT NULL, -- Replicado vía RabbitMQ
    name VARCHAR(255) NOT NULL,         -- Replicado vía RabbitMQ

    -- Preferencias del usuario
    city VARCHAR(100),
    lat DECIMAL(10, 8),
    lng DECIMAL(11, 8),

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_consumers_city ON consumers(city);
CREATE INDEX idx_consumers_email ON consumers(email);

-- ============================================
-- USER_GUSTOS
-- ============================================

CREATE TABLE user_gustos (
    user_id UUID REFERENCES consumers(id) ON DELETE CASCADE,
    gusto_id UUID NOT NULL, -- FK lógico a Event Service
    PRIMARY KEY (user_id, gusto_id)
);

CREATE INDEX idx_user_gustos_user ON user_gustos(user_id);

-- ============================================
-- SAVED_EVENTS
-- ============================================

CREATE TABLE saved_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES consumers(id) ON DELETE CASCADE,
    event_id UUID NOT NULL, -- FK lógico a Event Service

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    UNIQUE(user_id, event_id)
);

CREATE INDEX idx_saved_events_user ON saved_events(user_id);
CREATE INDEX idx_saved_events_event ON saved_events(event_id);
```

### 3.5 Promoter Service Database (promoter_db)

**Propósito**: Perfiles de promotores, verificación

```sql
-- ============================================
-- PROMOTER SERVICE DATABASE
-- ============================================

CREATE TYPE promoter_status AS ENUM (
    'PENDING_VERIFICATION',
    'VERIFIED',
    'REJECTED',
    'SUSPENDED'
);

CREATE TABLE promoters (
    id UUID PRIMARY KEY, -- Mismo ID que users de Auth Service
    email VARCHAR(255) UNIQUE NOT NULL, -- Replicado
    name VARCHAR(255) NOT NULL,         -- Replicado

    -- Información del promotor
    organization_name VARCHAR(255),
    phone VARCHAR(50),
    website VARCHAR(255),
    description TEXT,
    logo_url VARCHAR(500),

    -- Verificación
    status promoter_status DEFAULT 'PENDING_VERIFICATION',
    verified_at TIMESTAMP,
    verified_by UUID,
    verification_notes TEXT,

    -- Métricas
    total_events INT DEFAULT 0,
    approved_events INT DEFAULT 0,

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_promoters_status ON promoters(status);
CREATE INDEX idx_promoters_email ON promoters(email);
```

### 3.6 Notification Service Database (notification_db - MongoDB)

**Propósito**: Almacenar notificaciones enviadas

```javascript
// MongoDB Collection: notifications
{
  _id: ObjectId,
  userId: UUID,
  type: String, // "EMAIL", "PUSH", "SMS"
  channel: String, // "event.approved", "event.rejected", "user.created"
  subject: String,
  body: String,
  metadata: Object,
  status: String, // "SENT", "FAILED", "PENDING"
  sentAt: ISODate,
  createdAt: ISODate
}

// Indexes
db.notifications.createIndex({ userId: 1, createdAt: -1 });
db.notifications.createIndex({ type: 1, status: 1 });
db.notifications.createIndex({ createdAt: 1 }, { expireAfterSeconds: 7776000 }); // TTL 90 días

// MongoDB Collection: email_logs
{
  _id: ObjectId,
  to: String,
  from: String,
  subject: String,
  template: String,
  variables: Object,
  provider: String, // "SMTP", "SendGrid", "SES"
  messageId: String,
  status: String,
  error: String,
  sentAt: ISODate,
  createdAt: ISODate
}
```

### 3.7 Esquema SQL (PostgreSQL)

```sql
-- ============================================
-- ENUMS
-- ============================================

CREATE TYPE user_role AS ENUM ('CONSUMER', 'PROMOTER', 'ADMIN');
CREATE TYPE promoter_status AS ENUM ('PENDING_VERIFICATION', 'VERIFIED', 'SUSPENDED');
CREATE TYPE event_status AS ENUM ('DRAFT', 'PENDING_REVIEW', 'APPROVED', 'REJECTED', 'CANCELLED', 'ENDED');
CREATE TYPE event_type AS ENUM ('SINGLE', 'MULTI_DAY', 'RECURRING');

-- ============================================
-- TABLA: users
-- ============================================

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255),
    name VARCHAR(255) NOT NULL,
    role user_role DEFAULT 'CONSUMER',

    -- Datos de consumidor
    city VARCHAR(100),
    lat DECIMAL(10, 8),
    lng DECIMAL(11, 8),

    -- OAuth
    google_id VARCHAR(255) UNIQUE,
    facebook_id VARCHAR(255) UNIQUE,

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role ON users(role);
CREATE INDEX idx_users_city ON users(city);

-- ============================================
-- TABLA: promoters
-- ============================================

CREATE TABLE promoters (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID UNIQUE NOT NULL REFERENCES users(id) ON DELETE CASCADE,

    organization_name VARCHAR(255) NOT NULL,
    phone VARCHAR(50),
    website VARCHAR(255),
    description TEXT,

    status promoter_status DEFAULT 'PENDING_VERIFICATION',
    verified_at TIMESTAMP,
    verified_by UUID,

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_promoters_status ON promoters(status);
CREATE INDEX idx_promoters_user_id ON promoters(user_id);

-- ============================================
-- TABLA: gustos
-- ============================================

CREATE TABLE gustos (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) UNIQUE NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    icon VARCHAR(50),
    description TEXT,
    color VARCHAR(20),

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_gustos_slug ON gustos(slug);

-- ============================================
-- TABLA: user_gustos (Muchos a Muchos)
-- ============================================

CREATE TABLE user_gustos (
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    gusto_id UUID REFERENCES gustos(id) ON DELETE CASCADE,
    PRIMARY KEY (user_id, gusto_id)
);

-- ============================================
-- TABLA: events
-- ============================================

CREATE TABLE events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    promoter_id UUID NOT NULL REFERENCES promoters(id) ON DELETE CASCADE,

    -- Información básica
    title VARCHAR(255) NOT NULL,
    slug VARCHAR(255) UNIQUE NOT NULL,
    description TEXT NOT NULL,
    image_url VARCHAR(500) NOT NULL,
    image_gallery TEXT[], -- Array de URLs

    -- Fechas
    event_type event_type DEFAULT 'SINGLE',
    start_date TIMESTAMP NOT NULL,
    end_date TIMESTAMP,
    start_time TIME,
    end_time TIME,
    timezone VARCHAR(50) DEFAULT 'America/Bogota',

    -- Ubicación
    venue_name VARCHAR(255) NOT NULL,
    venue_address VARCHAR(500) NOT NULL,
    city VARCHAR(100) NOT NULL,
    country VARCHAR(100) DEFAULT 'Colombia',
    lat DECIMAL(10, 8) NOT NULL,
    lng DECIMAL(11, 8) NOT NULL,

    -- Precio
    is_free BOOLEAN DEFAULT false,
    price DECIMAL(10, 2),
    currency VARCHAR(10) DEFAULT 'COP',
    ticket_url VARCHAR(500),

    -- Estado
    status event_status DEFAULT 'DRAFT',
    reviewed_by UUID,
    reviewed_at TIMESTAMP,
    rejection_reason TEXT,
    published_at TIMESTAMP,

    -- Métricas
    view_count INT DEFAULT 0,
    save_count INT DEFAULT 0,
    share_count INT DEFAULT 0,

    -- Metadata
    metadata JSONB,

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_events_status ON events(status);
CREATE INDEX idx_events_city ON events(city);
CREATE INDEX idx_events_start_date ON events(start_date);
CREATE INDEX idx_events_promoter ON events(promoter_id);
CREATE INDEX idx_events_slug ON events(slug);
CREATE INDEX idx_events_location ON events USING GIST (ll_to_earth(lat, lng));

-- ============================================
-- TABLA: event_gustos (Muchos a Muchos)
-- ============================================

CREATE TABLE event_gustos (
    event_id UUID REFERENCES events(id) ON DELETE CASCADE,
    gusto_id UUID REFERENCES gustos(id) ON DELETE CASCADE,
    PRIMARY KEY (event_id, gusto_id)
);

CREATE INDEX idx_event_gustos_event ON event_gustos(event_id);
CREATE INDEX idx_event_gustos_gusto ON event_gustos(gusto_id);

-- ============================================
-- TABLA: saved_events
-- ============================================

CREATE TABLE saved_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    event_id UUID NOT NULL REFERENCES events(id) ON DELETE CASCADE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    UNIQUE(user_id, event_id)
);

CREATE INDEX idx_saved_events_user ON saved_events(user_id);
CREATE INDEX idx_saved_events_event ON saved_events(event_id);

-- ============================================
-- TABLA: cities
-- ============================================

CREATE TABLE cities (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) UNIQUE NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    country VARCHAR(100) DEFAULT 'Colombia',
    lat DECIMAL(10, 8) NOT NULL,
    lng DECIMAL(11, 8) NOT NULL,
    radius DECIMAL(6, 2) DEFAULT 50.0,
    is_active BOOLEAN DEFAULT true,

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_cities_slug ON cities(slug);
CREATE INDEX idx_cities_active ON cities(is_active);

-- ============================================
-- FUNCIONES ÚTILES
-- ============================================

-- Función para calcular distancia (Haversine)
CREATE OR REPLACE FUNCTION calculate_distance(
    lat1 DECIMAL, lng1 DECIMAL,
    lat2 DECIMAL, lng2 DECIMAL
) RETURNS DECIMAL AS $$
DECLARE
    earth_radius DECIMAL := 6371; -- km
    dlat DECIMAL;
    dlng DECIMAL;
    a DECIMAL;
    c DECIMAL;
BEGIN
    dlat := radians(lat2 - lat1);
    dlng := radians(lng2 - lng1);

    a := sin(dlat/2) * sin(dlat/2) +
         cos(radians(lat1)) * cos(radians(lat2)) *
         sin(dlng/2) * sin(dlng/2);

    c := 2 * atan2(sqrt(a), sqrt(1-a));

    RETURN earth_radius * c;
END;
$$ LANGUAGE plpgsql IMMUTABLE;

-- Trigger para actualizar updated_at
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_users_updated_at BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_events_updated_at BEFORE UPDATE ON events
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_promoters_updated_at BEFORE UPDATE ON promoters
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

### 3.2 Entidades JPA (Spring Boot)

**Ejemplo de Entidad Event:**

```java
@Entity
@Table(name = "events", indexes = {
    @Index(name = "idx_events_status", columnList = "status"),
    @Index(name = "idx_events_city", columnList = "city"),
    @Index(name = "idx_events_start_date", columnList = "start_date")
})
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Event {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "promoter_id", nullable = false)
    private Promoter promoter;

    @Column(nullable = false)
    private String title;

    @Column(unique = true, nullable = false)
    private String slug;

    @Column(columnDefinition = "TEXT", nullable = false)
    private String description;

    @Column(name = "image_url", nullable = false, length = 500)
    private String imageUrl;

    @Type(type = "json")
    @Column(name = "image_gallery", columnDefinition = "text[]")
    private List<String> imageGallery;

    @Enumerated(EnumType.STRING)
    @Column(name = "event_type")
    private EventType eventType = EventType.SINGLE;

    @Column(name = "start_date", nullable = false)
    private LocalDateTime startDate;

    @Column(name = "end_date")
    private LocalDateTime endDate;

    @Column(name = "start_time")
    private LocalTime startTime;

    @Column(name = "end_time")
    private LocalTime endTime;

    @Column(name = "venue_name", nullable = false)
    private String venueName;

    @Column(name = "venue_address", nullable = false, length = 500)
    private String venueAddress;

    @Column(nullable = false, length = 100)
    private String city;

    @Column(nullable = false, precision = 10, scale = 8)
    private BigDecimal lat;

    @Column(nullable = false, precision = 11, scale = 8)
    private BigDecimal lng;

    @Column(name = "is_free")
    private Boolean isFree = false;

    @Column(precision = 10, scale = 2)
    private BigDecimal price;

    @Column(name = "ticket_url", length = 500)
    private String ticketUrl;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private EventStatus status = EventStatus.DRAFT;

    @Column(name = "reviewed_by")
    private UUID reviewedBy;

    @Column(name = "reviewed_at")
    private LocalDateTime reviewedAt;

    @Column(name = "rejection_reason", columnDefinition = "TEXT")
    private String rejectionReason;

    @Column(name = "published_at")
    private LocalDateTime publishedAt;

    @ManyToMany
    @JoinTable(
        name = "event_gustos",
        joinColumns = @JoinColumn(name = "event_id"),
        inverseJoinColumns = @JoinColumn(name = "gusto_id")
    )
    private Set<Gusto> gustos = new HashSet<>();

    @Column(name = "view_count")
    private Integer viewCount = 0;

    @Column(name = "save_count")
    private Integer saveCount = 0;

    @Column(name = "share_count")
    private Integer shareCount = 0;

    @Type(type = "jsonb")
    @Column(columnDefinition = "jsonb")
    private Map<String, Object> metadata;

    @CreatedDate
    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
}
```

---

## 4. ARQUITECTURA DE API (API Gateway + Microservicios)

### 4.1 Flujo de Requests

```
Cliente (iOS/Android/Web)
    ↓ HTTPS
API Gateway (:8080)
    ├─ JWT Validation
    ├─ Rate Limiting
    ├─ Request Logging
    └─ Routing
        ↓
    ┌───┴───┬───────┬────────┬──────────┐
    ↓       ↓       ↓        ↓          ↓
 Auth    Event    User   Promoter   Notification
 :8081   :8082    :8083   :8084       :8085
```

**BASE URL:** `https://api.amigusto.com`

Todos los requests pasan por el API Gateway (:8080), que enruta a los microservicios correspondientes.

### 4.2 API Gateway Routes Configuration

```yaml
# Configuración Spring Cloud Gateway

# Auth Service (público)
/api/v1/auth/** → lb://AUTH-SERVICE

# Event Service
/api/v1/events/** → lb://EVENT-SERVICE
/api/v1/gustos/** → lb://EVENT-SERVICE

# User Service
/api/v1/users/** → lb://USER-SERVICE
/api/v1/saved-events/** → lb://USER-SERVICE

# Promoter Service
/api/v1/promoters/** → lb://PROMOTER-SERVICE

# Notification Service (interno, NO expuesto)
# Notification Service solo consume eventos de RabbitMQ

# Storage Service
/api/v1/storage/** → lb://STORAGE-SERVICE
```

### 4.3 Endpoints por Microservicio

#### 4.3.1 Auth Service (Puerto 8081)

**Público** - No requiere autenticación

```
POST   /api/v1/auth/register/consumer
Request:
{
  "email": "user@example.com",
  "password": "securePass123",
  "name": "Juan Pérez"
}
Response:
{
  "accessToken": "eyJhbGc...",
  "refreshToken": "eyJhbGc...",
  "user": {
    "id": "uuid",
    "email": "user@example.com",
    "name": "Juan Pérez",
    "role": "CONSUMER"
  }
}

POST   /api/v1/auth/register/promoter
POST   /api/v1/auth/login
POST   /api/v1/auth/logout
POST   /api/v1/auth/refresh
GET    /api/v1/auth/me
POST   /api/v1/auth/forgot-password
POST   /api/v1/auth/reset-password
```

**Eventos Publicados:**
- `user.created` → RabbitMQ (consumido por User Service, Promoter Service, Notification Service)

#### 4.3.2 Event Service (Puerto 8082)

**Endpoints Públicos:**

```
GET    /api/v1/gustos
Response:
[
  {
    "id": "uuid",
    "name": "Música",
    "icon": "🎵",
    "slug": "musica",
    "color": "#FF5733"
  }
]
```

**Endpoints Autenticados (CONSUMER):**

```
GET    /api/v1/events/discover
Query Params:
  - city: string (required)
  - gustoIds: UUID[] (required)
  - lat: decimal (required)
  - lng: decimal (required)
  - radiusKm: number (default: 50)
  - page: number (default: 0)
  - size: number (default: 20)

Response:
{
  "content": [...],
  "pageable": {...},
  "totalElements": 150,
  "totalPages": 8
}

GET    /api/v1/events/{id}
```

**Endpoints Autenticados (PROMOTER):**

```
POST   /api/v1/events
Request:
{
  "title": "Concierto de Jazz",
  "description": "...",
  "startDate": "2025-12-01T20:00:00Z",
  "venueName": "Teatro Nacional",
  "city": "Madrid",
  "lat": 40.4168,
  "lng": -3.7038,
  "gustoIds": ["uuid1", "uuid2"],
  "isFree": false,
  "price": 25.00,
  "currency": "EUR",
  "imageUrl": "https://..."
}

GET    /api/v1/events/my-events
PUT    /api/v1/events/{id}
DELETE /api/v1/events/{id}
POST   /api/v1/events/{id}/submit-review
```

**Endpoints Autenticados (ADMIN):**

```
GET    /api/v1/events/pending-review
POST   /api/v1/events/{id}/approve
POST   /api/v1/events/{id}/reject
```

**Comunicación con otros servicios:**
- **Feign → Promoter Service**: Validar que el promotor existe y está verificado
- **Feign → Storage Service**: Validar que la imagen subida existe
- **RabbitMQ**: Publica `event.approved`, `event.rejected`, `event.created`

#### 4.3.3 User Service (Puerto 8083)

**Endpoints Autenticados (CONSUMER):**

```
GET    /api/v1/users/me
Response:
{
  "id": "uuid",
  "email": "user@example.com",
  "name": "Juan Pérez",
  "city": "Madrid",
  "gustos": [
    { "id": "uuid", "name": "Música", "icon": "🎵" }
  ]
}

PUT    /api/v1/users/me/gustos
Request:
{
  "gustoIds": ["uuid1", "uuid2", "uuid3"]
}

POST   /api/v1/saved-events
Request:
{
  "eventId": "uuid"
}

GET    /api/v1/saved-events
DELETE /api/v1/saved-events/{eventId}
```

**Comunicación con otros servicios:**
- **Feign → Event Service**: Obtener detalles de eventos guardados
- **RabbitMQ**: Publica `event.saved`, consume `user.created`

#### 4.3.4 Promoter Service (Puerto 8084)

**Endpoints Autenticados (PROMOTER):**

```
GET    /api/v1/promoters/me
Response:
{
  "id": "uuid",
  "organizationName": "Eventos SA",
  "phone": "+34 600 000 000",
  "website": "https://eventossa.com",
  "status": "VERIFIED",
  "totalEvents": 25,
  "approvedEvents": 20
}

PUT    /api/v1/promoters/me
Request:
{
  "organizationName": "Eventos SA",
  "phone": "+34 600 000 000",
  "website": "https://eventossa.com",
  "description": "Empresa organizadora de eventos..."
}
```

**Endpoints Autenticados (ADMIN):**

```
GET    /api/v1/promoters
GET    /api/v1/promoters/{id}
POST   /api/v1/promoters/{id}/verify
POST   /api/v1/promoters/{id}/reject
POST   /api/v1/promoters/{id}/suspend
```

**Comunicación con otros servicios:**
- **RabbitMQ**: Consume `user.created`, `event.created`

#### 4.3.5 Notification Service (Puerto 8085)

**NO tiene endpoints públicos.** Solo consume eventos de RabbitMQ.

**Eventos Consumidos:**
- `user.created` → Enviar email de bienvenida
- `event.approved` → Notificar al promotor que su evento fue aprobado
- `event.rejected` → Notificar al promotor que su evento fue rechazado
- `event.saved` → (Opcional) Notificar al usuario que guardó un evento

**Comunicación con otros servicios:**
- **Feign → Promoter Service**: Obtener email del promotor para notificaciones
- **Feign → User Service**: Obtener email del usuario para notificaciones

#### 4.3.6 Storage Service (Puerto 8086)

**Endpoints Autenticados (PROMOTER):**

```
POST   /api/v1/storage/upload
Request: multipart/form-data
  - file: File (image/jpeg, image/png, max 5MB)

Response:
{
  "url": "https://s3.amazonaws.com/amigusto/events/uuid.jpg",
  "key": "events/uuid.jpg",
  "size": 1234567
}

DELETE /api/v1/storage/{key}
```

**Integraciones:**
- AWS S3 SDK / Cloudinary API

### 4.4 Comunicación entre Microservicios

#### 4.4.1 Síncrona (Spring Cloud OpenFeign)

**Ejemplo: Event Service → Promoter Service**

```java
// Event Service
@FeignClient(name = "PROMOTER-SERVICE", fallback = PromoterServiceFallback.class)
public interface PromoterServiceClient {
    @GetMapping("/api/v1/promoters/{id}")
    PromoterResponse getPromoter(@PathVariable("id") UUID id);
}

// Uso en EventService
public EventResponse createEvent(EventRequest request, UUID promoterId) {
    // Validar que el promotor existe y está verificado
    PromoterResponse promoter = promoterClient.getPromoter(promoterId);

    if (!promoter.isVerified()) {
        throw new BusinessException("Promoter must be verified to create events");
    }

    // Continuar creación del evento...
}
```

**Casos de uso:**
- Event Service → Promoter Service (validar promotor al crear evento)
- Event Service → Storage Service (validar imagen antes de guardar URL)
- User Service → Event Service (obtener detalles de eventos guardados)

#### 4.4.2 Asíncrona (RabbitMQ)

**Ejemplo: Auth Service publica user.created**

```java
// Auth Service - Publisher
@Component
public class UserEventPublisher {
    private final RabbitTemplate rabbitTemplate;

    public void publishUserCreated(User user) {
        UserCreatedEvent event = UserCreatedEvent.builder()
            .userId(user.getId())
            .email(user.getEmail())
            .name(user.getName())
            .role(user.getRole())
            .build();

        rabbitTemplate.convertAndSend("user.events", "user.created", event);
    }
}

// User Service - Consumer
@Component
public class UserEventConsumer {
    @RabbitListener(queues = "user-service.user.created")
    public void handleUserCreated(UserCreatedEvent event) {
        // Crear consumer en user_db
        Consumer consumer = new Consumer();
        consumer.setId(event.getUserId());
        consumer.setEmail(event.getEmail());
        consumer.setName(event.getName());

        consumerRepository.save(consumer);
    }
}
```

**Ventajas:**
- Desacoplamiento total entre servicios
- Tolerancia a fallos (si User Service está caído, el mensaje queda en queue)
- Procesamiento asíncrono (Auth Service no espera a que User Service procese)
- Broadcast (múltiples servicios pueden consumir el mismo evento)

### 4.5 Manejo de Errores Cross-Service

**Circuit Breaker con Resilience4j:**

```java
@Service
public class EventService {

    @CircuitBreaker(name = "promoterService", fallbackMethod = "getPromoterFallback")
    @Retry(name = "promoterService")
    public PromoterResponse getPromoter(UUID id) {
        return promoterClient.getPromoter(id);
    }

    private PromoterResponse getPromoterFallback(UUID id, Throwable ex) {
        log.error("Circuit breaker activated for promoter service", ex);
        // Retornar datos en caché o datos por defecto
        return PromoterResponse.builder()
            .id(id)
            .verified(false)
            .build();
    }
}
```

**Configuración:**

```yaml
resilience4j:
  circuitbreaker:
    instances:
      promoterService:
        slidingWindowSize: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 10s
```

### 4.2 Ejemplo de Controller (Spring Boot)

```java
@RestController
@RequestMapping("/api/v1/events")
@RequiredArgsConstructor
@Validated
public class EventController {

    private final EventService eventService;

    @GetMapping
    public ResponseEntity<Page<EventResponse>> getEvents(
            @RequestParam String city,
            @RequestParam(required = false) List<UUID> gustoIds,
            @RequestParam(required = false) Boolean isFree,
            @RequestParam(required = false) LocalDateTime startDate,
            @RequestParam(required = false) LocalDateTime endDate,
            @RequestParam(required = false) BigDecimal lat,
            @RequestParam(required = false) BigDecimal lng,
            @RequestParam(required = false) BigDecimal radius,
            Pageable pageable
    ) {
        EventFilterRequest filter = EventFilterRequest.builder()
            .city(city)
            .gustoIds(gustoIds)
            .isFree(isFree)
            .startDate(startDate)
            .endDate(endDate)
            .lat(lat)
            .lng(lng)
            .radius(radius)
            .build();

        Page<EventResponse> events = eventService.findEvents(filter, pageable);
        return ResponseEntity.ok(events);
    }

    @GetMapping("/{id}")
    public ResponseEntity<EventResponse> getEventById(@PathVariable UUID id) {
        EventResponse event = eventService.findById(id);
        return ResponseEntity.ok(event);
    }

    @PostMapping("/{id}/save")
    @PreAuthorize("hasRole('CONSUMER')")
    public ResponseEntity<ApiResponse> saveEvent(
            @PathVariable UUID id,
            @AuthenticationPrincipal UserDetails userDetails
    ) {
        UUID userId = ((CustomUserDetails) userDetails).getId();
        eventService.saveEvent(userId, id);
        return ResponseEntity.ok(new ApiResponse(true, "Event saved successfully"));
    }
}
```

---

## 5. PLAN DE IMPLEMENTACIÓN POR FASES (Microservicios)

### FASE 0: Setup Infraestructura Microservicios (Semana 1-2)

**Objetivo**: Configurar la infraestructura base de microservicios antes de implementar la lógica de negocio.

#### Sprint 0.1: Repositorio y CI/CD
- [ ] Setup de monorepo Git o repos separados
  ```
  amigusto-backend/
  ├── eureka-server/
  ├── config-server/
  ├── api-gateway/
  ├── auth-service/
  ├── event-service/
  ├── user-service/
  ├── promoter-service/
  ├── notification-service/
  ├── storage-service/
  └── config-repo/
  ```
- [ ] Setup CI/CD por microservicio (GitHub Actions / GitLab CI)
- [ ] Docker Compose para desarrollo local
- [ ] Setup de PostgreSQL multi-database (auth_db, event_db, user_db, promoter_db)
- [ ] Setup de MongoDB para notification_db
- [ ] Setup de Redis compartido
- [ ] Setup de RabbitMQ

**Docker Compose Base:**

```yaml
version: '3.8'
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: amigusto
      POSTGRES_PASSWORD: dev_password
    ports:
      - "5432:5432"
    volumes:
      - ./init-dbs.sql:/docker-entrypoint-initdb.d/init.sql

  mongodb:
    image: mongo:7
    ports:
      - "27017:27017"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  rabbitmq:
    image: rabbitmq:3.12-management-alpine
    ports:
      - "5672:5672"
      - "15672:15672"  # Management UI

  zipkin:
    image: openzipkin/zipkin:latest
    ports:
      - "9411:9411"
```

#### Sprint 0.2: Eureka Server (Service Discovery)
- [ ] Crear proyecto `eureka-server` con Spring Initializr
- [ ] Agregar dependencias: `spring-cloud-starter-netflix-eureka-server`
- [ ] Configurar `@EnableEurekaServer`
- [ ] Configurar application.yml:
  ```yaml
  server:
    port: 8761
  eureka:
    client:
      registerWithEureka: false
      fetchRegistry: false
  ```
- [ ] Deployar en contenedor Docker
- [ ] Verificar dashboard en `http://localhost:8761`

#### Sprint 0.3: Config Server
- [ ] Crear proyecto `config-server`
- [ ] Agregar dependencias: `spring-cloud-config-server`
- [ ] Configurar `@EnableConfigServer`
- [ ] Setup repositorio Git para configuraciones
- [ ] Crear configs: `application.yml`, `event-service.yml`, `auth-service.yml`
- [ ] Configurar encriptación de secrets
- [ ] Deployar en Docker

#### Sprint 0.4: API Gateway
- [ ] Crear proyecto `api-gateway`
- [ ] Agregar dependencias: `spring-cloud-starter-gateway`, `spring-cloud-starter-netflix-eureka-client`
- [ ] Configurar rutas en application.yml
- [ ] Implementar JWT Authentication Filter
- [ ] Implementar Rate Limiting Filter
- [ ] Implementar Global Logging Filter
- [ ] Configurar CORS
- [ ] Deployar en Docker
- [ ] Tests de routing

**Entregables Fase 0:**
- Infraestructura de microservicios completa
- Eureka Server funcionando
- Config Server con repositorio Git
- API Gateway con filtros configurados
- Docker Compose funcional con todos los servicios base
- Scripts de inicialización de bases de datos

---

### FASE 1: Microservicios Core (Semana 3-6)

#### Sprint 1.1: Auth Service (Semana 3)

**Setup:**
- [ ] Crear proyecto `auth-service`
- [ ] Agregar dependencias:
  - `spring-boot-starter-data-jpa`
  - `spring-boot-starter-security`
  - `spring-cloud-starter-netflix-eureka-client`
  - `spring-cloud-starter-config`
  - `spring-cloud-starter-sleuth`
  - `spring-cloud-sleuth-zipkin`
  - `spring-boot-starter-amqp`
  - `jjwt` (JWT library)
  - `postgresql`

**Implementación:**
- [ ] Configurar conexión a `auth_db` PostgreSQL
- [ ] Configurar Eureka Client
- [ ] Entidades JPA: `User`, `RefreshToken`
- [ ] Repository: `UserRepository`, `RefreshTokenRepository`
- [ ] Service: `AuthService` con JWT generation/validation
- [ ] Controller: `AuthController`
- [ ] Endpoints:
  - POST /auth/register/consumer
  - POST /auth/register/promoter
  - POST /auth/login
  - POST /auth/refresh
  - POST /auth/logout
- [ ] RabbitMQ Publisher: `UserEventPublisher`
  - Publicar evento `user.created` al registrar usuario
- [ ] Spring Security configuration
- [ ] Tests unitarios (JUnit 5 + Mockito)
- [ ] Tests de integración (Testcontainers)
- [ ] Configurar en Config Server
- [ ] Deployar en Docker
- [ ] Registrar en Eureka

**Verificación:**
- Endpoint `/actuator/health` responde OK
- Servicio visible en Eureka Dashboard
- JWT generation funciona correctamente
- Evento `user.created` se publica a RabbitMQ

#### Sprint 1.2: Event Service (Semana 4)

**Setup:**
- [ ] Crear proyecto `event-service`
- [ ] Agregar todas las dependencias de microservicio + PostGIS
- [ ] Configurar conexión a `event_db` PostgreSQL con PostGIS

**Implementación:**
- [ ] Entidades JPA: `Event`, `Gusto`, `City`, `EventGusto`
- [ ] Repositories con queries PostGIS
- [ ] Feign Client: `PromoterServiceClient`
- [ ] Feign Client: `StorageServiceClient`
- [ ] Service: `EventService` con Circuit Breaker
- [ ] RabbitMQ Publisher: `EventPublisher`
  - Eventos: `event.created`, `event.approved`, `event.rejected`
- [ ] Controller: `EventController`
- [ ] Endpoints:
  - POST /events (crear evento)
  - GET /events/discover (búsqueda geolocalizada)
  - GET /events/my-events (promotor)
  - POST /events/{id}/submit-review
  - POST /events/{id}/approve (admin)
  - POST /events/{id}/reject (admin)
  - GET /gustos (público)
- [ ] Resilience4j configuration (Circuit Breaker, Retry, Timeout)
- [ ] Redis caching para /events/discover
- [ ] Seed data: Gustos iniciales, Ciudades
- [ ] Tests con Testcontainers (PostgreSQL + PostGIS)

**Verificación:**
- Query geoespacial con PostGIS funciona
- Feign calls a Promoter Service funcionan con fallback
- RabbitMQ events se publican correctamente
- Circuit Breaker se activa al fallar Promoter Service
- Cache Redis funciona correctamente

#### Sprint 1.3: User Service (Semana 5)

**Implementación:**
- [ ] Crear proyecto `user-service`
- [ ] Configurar conexión a `user_db` PostgreSQL
- [ ] Entidades: `Consumer`, `UserGusto`, `SavedEvent`
- [ ] RabbitMQ Consumer: `UserEventConsumer`
  - Consumir `user.created` de Auth Service
  - Crear registro en `consumers` table
- [ ] Feign Client: `EventServiceClient`
- [ ] Service: `UserService`, `SavedEventService`
- [ ] Controller: `UserController`, `SavedEventController`
- [ ] Endpoints:
  - GET /users/me
  - PUT /users/me/gustos
  - POST /saved-events
  - GET /saved-events
  - DELETE /saved-events/{eventId}
- [ ] RabbitMQ Publisher: Publicar `event.saved`
- [ ] Tests unitarios + integración

**Verificación:**
- Consumer de RabbitMQ procesa evento `user.created`
- Usuario se replica correctamente de auth_db a user_db
- Guardar evento funciona correctamente

#### Sprint 1.4: Promoter Service (Semana 6)

**Implementación:**
- [ ] Crear proyecto `promoter-service`
- [ ] Configurar conexión a `promoter_db` PostgreSQL
- [ ] Entidad: `Promoter`
- [ ] RabbitMQ Consumer: `PromoterEventConsumer`
  - Consumir `user.created` (si role = PROMOTER)
  - Consumir `event.created` (actualizar métricas)
- [ ] Service: `PromoterService`
- [ ] Controller: `PromoterController`, `AdminPromoterController`
- [ ] Endpoints:
  - GET /promoters/me
  - PUT /promoters/me
  - POST /promoters/{id}/verify (admin)
  - POST /promoters/{id}/reject (admin)
- [ ] Tests

#### Sprint 1.5: Notification Service (Semana 6)

**Implementación:**
- [ ] Crear proyecto `notification-service`
- [ ] Configurar conexión a MongoDB
- [ ] RabbitMQ Consumers:
  - `user.created` → Email de bienvenida
  - `event.approved` → Email al promotor
  - `event.rejected` → Email al promotor
- [ ] Feign Clients: `PromoterServiceClient`, `UserServiceClient`
- [ ] Service: `EmailService` (Spring Mail + Thymeleaf)
- [ ] Service: `PushNotificationService` (Firebase Cloud Messaging)
- [ ] MongoDB Documents: `Notification`, `EmailLog`
- [ ] Templates de email en Thymeleaf
- [ ] Tests con mock SMTP

#### Sprint 1.6: Storage Service (Semana 6)

**Implementación:**
- [ ] Crear proyecto `storage-service`
- [ ] Configurar AWS S3 SDK / Cloudinary SDK
- [ ] Service: `ImageUploadService`
- [ ] Controller: `StorageController`
- [ ] Endpoints:
  - POST /storage/upload
  - DELETE /storage/{key}
- [ ] Validaciones: tipo de archivo, tamaño máximo
- [ ] Tests con AWS S3 mock (LocalStack)

**Entregables Fase 1:**
- 7 microservicios funcionando independientemente
- Todos registrados en Eureka
- Comunicación Feign entre servicios funciona
- RabbitMQ messaging funciona
- Circuit Breakers configurados
- Distributed tracing con Zipkin
- OpenAPI docs por servicio
- Tests >80% coverage por servicio
- Docker Compose completo con todos los servicios

---

### FASE 2: Apps Móviles Nativas (Semana 6-10)

#### Sprint 2.1: iOS - Setup y Autenticación (Semana 6)
- [ ] Setup del proyecto Xcode
- [ ] Configuración de URLSession/Alamofire
- [ ] Modelos (Codable)
- [ ] APIService layer
- [ ] Screens de Login/Registro
- [ ] Keychain para tokens

#### Sprint 2.2: iOS - Onboarding y Feed (Semana 7-8)
- [ ] Onboarding con SwiftUI
- [ ] GustoSelectionView
- [ ] DiscoverView (feed)
- [ ] EventCard component
- [ ] Infinite scroll con Combine
- [ ] CoreLocation integration

#### Sprint 2.3: iOS - Detalle y Mis Planes (Semana 9)
- [ ] EventDetailView
- [ ] MapKit integration
- [ ] Share functionality
- [ ] MyPlansView
- [ ] CoreData para cache offline

#### Sprint 2.4: Android - Setup y Autenticación (Semana 6)
- [ ] Setup Gradle + Hilt
- [ ] Retrofit + OkHttp setup
- [ ] Modelos con Kotlin Data Classes
- [ ] Repository pattern
- [ ] Login/Register screens con Compose
- [ ] EncryptedSharedPreferences

#### Sprint 2.5: Android - Onboarding y Feed (Semana 7-8)
- [ ] Onboarding flow con Compose
- [ ] GustoSelector con LazyVerticalGrid
- [ ] DiscoverScreen con LazyColumn
- [ ] EventCard composable
- [ ] Paging 3 library para infinite scroll
- [ ] Google Play Services Location

#### Sprint 2.6: Android - Detalle y Mis Planes (Semana 9)
- [ ] EventDetailScreen
- [ ] Google Maps SDK integration
- [ ] Share Intent
- [ ] MyPlansScreen
- [ ] Room database para cache

#### Sprint 2.7: Testing y Pulido (Semana 10)
- [ ] Unit tests (iOS: XCTest, Android: JUnit)
- [ ] UI tests (iOS: XCUITest, Android: Espresso)
- [ ] Bug fixing
- [ ] Performance optimization
- [ ] Firebase Analytics integration

**Entregables Fase 2:**
- App iOS completa (TestFlight)
- App Android completa (Internal Testing)

---

### FASE 3: Portal Web Angular (Semana 11-13)

#### Sprint 3.1: Setup y Autenticación
- [ ] ng new con standalone components
- [ ] Angular Material setup
- [ ] HttpClient + Interceptors
- [ ] Auth service con JWT
- [ ] Login/Register components
- [ ] AuthGuard

#### Sprint 3.2: Dashboard y Listado
- [ ] Dashboard component
- [ ] EventsList con Material Table
- [ ] Status badges
- [ ] Filtros y búsqueda

#### Sprint 3.3: Formulario de Evento
- [ ] Reactive Forms
- [ ] Validaciones custom
- [ ] GustoSelector component
- [ ] ImageUploader component
- [ ] Google Maps Autocomplete
- [ ] Date/Time pickers

#### Sprint 3.4: Edición y Gestión
- [ ] Edit Event component
- [ ] Delete confirmation dialog
- [ ] Toast notifications (Material Snackbar)
- [ ] Profile component

**Entregables Fase 3:**
- Portal web funcional
- Deployado en Vercel

---

### FASE 4: Panel Admin Angular (Semana 14-15)

- [ ] Cola de aprobación con ag-Grid
- [ ] EventReview component
- [ ] Approve/Reject actions
- [ ] PromoterManagement
- [ ] Métricas dashboard con ngx-charts

**Entregables Fase 4:**
- Panel admin funcional

---

### FASE 5: Testing y QA (Semana 16-17)

- [ ] Testing E2E (Cypress para Angular)
- [ ] Testing de integración API
- [ ] Testing manual de apps móviles
- [ ] Performance testing (JMeter)
- [ ] Security audit
- [ ] Bug fixing

---

### FASE 6-8: Lanzamiento (Semana 18-23)

Similar al plan original, con:
- Seed de contenido
- Submission a App Store y Google Play
- Beta pública
- Lanzamiento final

---

## 6. STACK RESUMEN

| Componente | Tecnología |
|------------|------------|
| **Backend API** | Java 17 + Spring Boot 3.2 |
| **Base de Datos** | PostgreSQL 16 + PostGIS |
| **ORM** | Spring Data JPA + Hibernate |
| **Cache** | Redis 7 + Spring Data Redis |
| **App iOS** | Swift 5.9 + SwiftUI |
| **App Android** | Kotlin 1.9 + Jetpack Compose |
| **Portal Web** | Angular 17 + Material |
| **Panel Admin** | Angular 17 + Material |
| **Build (Backend)** | Maven 3.9 / Gradle 8 |
| **Build (iOS)** | Xcode 15 + Swift Package Manager |
| **Build (Android)** | Gradle 8.2 + Kotlin DSL |
| **Build (Angular)** | npm + Angular CLI |
| **Testing (Backend)** | JUnit 5 + Mockito + Spring Boot Test |
| **Testing (iOS)** | XCTest + XCUITest |
| **Testing (Android)** | JUnit + Espresso + Compose Testing |
| **Testing (Angular)** | Jasmine + Karma + Cypress |

---

## 7. ESTIMACIÓN DE EQUIPO

**Para MVP (6 meses):**
- 1 Tech Lead (Full-Stack)
- 2 Backend Developers (Java/Spring Boot)
- 1 iOS Developer (Swift)
- 1 Android Developer (Kotlin)
- 2 Frontend Developers (Angular)
- 1 UI/UX Designer
- 1 QA Engineer
- 1 DevOps Engineer (part-time)

**Total:** 9-10 personas

---

Esta es la versión actualizada del plan técnico con el stack que solicitaste. ¿Quieres que continúe actualizando los demás documentos (ARQUITECTURA_PROYECTO, EJEMPLOS_CODIGO, etc.)?
