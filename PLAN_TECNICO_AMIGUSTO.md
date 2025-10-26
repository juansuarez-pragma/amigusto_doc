# Plan Técnico - Amigusto MVP
## Motor de Descubrimiento de Eventos Hiper-Personalizado

---

## 1. ARQUITECTURA GENERAL DEL SISTEMA

### 1.1 Vista de Alto Nivel

```
┌─────────────────────────────────────────────────────────────────┐
│                         CAPA DE CLIENTE                          │
├──────────────────────┬──────────────────────┬───────────────────┤
│   APP iOS (B2C)      │  APP Android (B2C)   │  WEB ANGULAR      │
│   Swift + SwiftUI    │  Kotlin + Compose    │  Portal B2B/Admin │
└──────────┬───────────┴──────────┬───────────┴─────────┬─────────┘
           │                      │                     │
           └──────────────────────┼─────────────────────┘
                                  │
                         ┌────────▼────────┐
                         │   API GATEWAY   │
                         │  (Spring Cloud) │
                         └────────┬────────┘
                                  │
           ┌──────────────────────┼──────────────────────┐
           │                      │                      │
    ┌──────▼──────┐      ┌───────▼───────┐    ┌────────▼────────┐
    │   API Core  │      │  Content API  │    │   Auth Service  │
    │(Spring Boot)│      │ (Spring Boot) │    │  (Spring Boot)  │
    └──────┬──────┘      └───────┬───────┘    └────────┬────────┘
           │                     │                      │
           └─────────────────────┼──────────────────────┘
                                 │
                    ┌────────────▼─────────────┐
                    │   CAPA DE DATOS          │
                    ├──────────────────────────┤
                    │  PostgreSQL (Principal)  │
                    │  Redis (Cache/Sessions)  │
                    │  S3/Cloudinary (Assets)  │
                    │  Elasticsearch (Search)  │
                    └──────────────────────────┘
```

### 1.2 Principios Arquitectónicos

1. **Separación de Concerns**: Cada plataforma (iOS, Android, Web) consume los mismos servicios backend REST.
2. **API-First**: Todo el backend expone APIs RESTful siguiendo estándares Spring Boot.
3. **Stateless Backend**: Las APIs no mantienen estado, autenticación vía JWT tokens.
4. **Event-Driven**: Flujo de aprobación con sistema de estados y eventos Spring.
5. **Geo-Awareness**: Queries geoespaciales con PostGIS.

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

**Stack Completo:**
```yaml
Lenguaje: Java 17+ (LTS)
Framework: Spring Boot 3.2+
Build Tool: Maven 3.9+ o Gradle 8+
ORM: Spring Data JPA + Hibernate
Database: PostgreSQL 16+ (driver: postgresql)
Cache: Spring Data Redis + Lettuce
Security: Spring Security + JWT (jjwt library)
Validation: Jakarta Bean Validation (Hibernate Validator)
API Docs: Springdoc OpenAPI (Swagger)
Testing: JUnit 5 + Mockito + Spring Boot Test
Logging: SLF4J + Logback
File Upload: MultipartFile + AWS S3 SDK / Cloudinary SDK
Email: Spring Mail + Thymeleaf (templates)
Task Scheduling: Spring Scheduler / Spring Batch
Monitoring: Spring Actuator + Micrometer
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

## 3. DISEÑO DE BASE DE DATOS

### 3.1 Esquema SQL (PostgreSQL)

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

## 4. ARQUITECTURA DE API

### 4.1 Endpoints Principales

**BASE URL:** `https://api.amigusto.com/v1`

#### 4.1.1 Autenticación

```
POST   /api/v1/auth/register/consumer
POST   /api/v1/auth/register/promoter
POST   /api/v1/auth/login
POST   /api/v1/auth/logout
POST   /api/v1/auth/refresh
GET    /api/v1/auth/me
POST   /api/v1/auth/forgot-password
POST   /api/v1/auth/reset-password
```

#### 4.1.2 Eventos - Consumer (Apps Móviles)

```
GET    /api/v1/events
  Query Params:
    - city: string
    - gustoIds: UUID[]
    - startDate: ISO Date
    - endDate: ISO Date
    - isFree: boolean
    - lat: decimal
    - lng: decimal
    - page: number (default: 0)
    - size: number (default: 20)

GET    /api/v1/events/{id}

POST   /api/v1/events/{id}/save

DELETE /api/v1/events/{id}/save

POST   /api/v1/events/{id}/share

GET    /api/v1/events/saved
```

#### 4.1.3 Eventos - Promoter (Web Angular)

```
GET    /api/v1/promoter/events

POST   /api/v1/promoter/events

GET    /api/v1/promoter/events/{id}

PUT    /api/v1/promoter/events/{id}

DELETE /api/v1/promoter/events/{id}

POST   /api/v1/promoter/events/{id}/submit-review
```

#### 4.1.4 Admin Panel

```
GET    /api/v1/admin/events

PUT    /api/v1/admin/events/{id}

POST   /api/v1/admin/events/{id}/approve

POST   /api/v1/admin/events/{id}/reject

GET    /api/v1/admin/promoters

PUT    /api/v1/admin/promoters/{id}/verify
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

## 5. PLAN DE IMPLEMENTACIÓN POR FASES

### FASE 0: Setup y Arquitectura (Semana 1-2)

**Tareas:**
- [ ] Setup de repositorios Git (3 repos o monorepo)
- [ ] Configuración de CI/CD (GitHub Actions)
- [ ] Setup de PostgreSQL + Redis (Docker local)
- [ ] Configuración de entornos (Dev, Staging, Prod)
- [ ] Setup Spring Boot inicial con Spring Initializr
- [ ] Setup proyecto iOS con Xcode
- [ ] Setup proyecto Android con Android Studio
- [ ] Setup proyectos Angular (Portal + Admin)
- [ ] Configuración de Firebase (para analytics móvil)

---

### FASE 1: Backend Core (Semana 3-5)

#### Sprint 1.1: Autenticación y Usuarios
- [ ] Entidades JPA: User, Promoter
- [ ] Repositorios Spring Data
- [ ] Servicio de autenticación con JWT
- [ ] Endpoints de registro y login
- [ ] Spring Security configuration
- [ ] Tests unitarios con JUnit 5

#### Sprint 1.2: Gustos y Ciudades
- [ ] Entidades: Gusto, City
- [ ] Controllers y Services
- [ ] Seed inicial de datos (SQL scripts)

#### Sprint 1.3: Eventos - Core
- [ ] Entidad Event con relaciones JPA
- [ ] EventRepository con queries personalizados
- [ ] CRUD completo para promotores
- [ ] State machine de eventos
- [ ] Upload de imágenes (S3/Cloudinary)
- [ ] Validaciones con Bean Validation

#### Sprint 1.4: Eventos - Consumer
- [ ] Query con filtrado geoespacial (PostGIS)
- [ ] Paginación con Spring Data Pageable
- [ ] Endpoint de guardado (SavedEvent)
- [ ] Métricas (view/save/share count)

**Entregables Fase 1:**
- API Spring Boot funcional
- OpenAPI docs (Swagger UI)
- Tests (>80% coverage)
- Deployado en Railway/Render

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
