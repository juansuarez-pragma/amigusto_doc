# Arquitectura de Microservicios - Amigusto
## Diseño Completo de Backend Distribuido

---

## Índice

1. [Visión General de Microservicios](#1-visión-general-de-microservicios)
2. [Descomposición de Servicios](#2-descomposición-de-servicios)
3. [Infraestructura y Componentes Compartidos](#3-infraestructura-y-componentes-compartidos)
4. [Comunicación entre Microservicios](#4-comunicación-entre-microservicios)
5. [Gestión de Datos Distribuidos](#5-gestión-de-datos-distribuidos)
6. [Flujos de Funcionalidades con Microservicios](#6-flujos-de-funcionalidades-con-microservicios)
7. [Resiliencia y Tolerancia a Fallos](#7-resiliencia-y-tolerancia-a-fallos)
8. [Observabilidad y Monitoreo](#8-observabilidad-y-monitoreo)
9. [Despliegue y CI/CD](#9-despliegue-y-cicd)

---

## 1. Visión General de Microservicios

### 1.1 Arquitectura Completa

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           CAPA DE CLIENTES                                   │
├─────────────────┬─────────────────┬──────────────────────────────────────────┤
│  📱 iOS App     │  📱 Android App │  💻 Web Angular (Portal + Admin)       │
└────────┬────────┴────────┬────────┴─────────┬────────────────────────────────┘
         │                 │                  │
         └─────────────────┼──────────────────┘
                           │ HTTPS/REST
              ┌────────────▼────────────┐
              │   🚪 API GATEWAY        │
              │ Spring Cloud Gateway    │ ← SSL/TLS Termination
              │ Port: 8080              │ ← Rate Limiting
              │                         │ ← Load Balancing
              │                         │ ← Authentication Filter
              └────────────┬────────────┘
                           │
          ┌────────────────┼────────────────────────────┐
          │                │                            │
          │    ┌───────────▼──────────┐                │
          │    │  🔍 SERVICE DISCOVERY│                │
          │    │  Netflix Eureka      │                │
          │    │  Port: 8761          │                │
          │    └──────────────────────┘                │
          │                                             │
┌─────────▼─────────┬──────────────┬───────────────┬──────────────┬────────────▼────┐
│                   │              │               │              │                 │
│  🔐 AUTH SERVICE  │  📅 EVENT    │  👤 USER     │  🏢 PROMOTER │  📧 NOTIFICATION│
│  Port: 8081       │  SERVICE     │  SERVICE     │  SERVICE     │  SERVICE        │
│                   │  Port: 8082  │  Port: 8083  │  Port: 8084  │  Port: 8085     │
│                   │              │               │              │                 │
│  - Login/Register │  - Events    │  - Consumers │  - Promoters │  - Email        │
│  - JWT Tokens     │  - Gustos    │  - Gustos    │  - Orgs      │  - Push Notif   │
│  - Refresh        │  - Discovery │  - Saved     │  - Verify    │  - SMS          │
│  - Validation     │  - Search    │  - Profile   │  - Events    │                 │
│                   │              │               │              │                 │
│  DB: PostgreSQL   │  DB: PostGIS │  DB: Postgres│  DB: Postgres│  DB: MongoDB    │
│  (users)          │  (events)    │  (consumers) │  (promoters) │  (notifications)│
└───────────────────┴──────────────┴───────────────┴──────────────┴─────────────────┘
         │                  │               │              │              │
         └──────────────────┼───────────────┼──────────────┼──────────────┘
                            │               │              │
              ┌─────────────▼───────────────▼──────────────▼─────┐
              │         📨 MESSAGE BROKER (RabbitMQ)             │
              │         Port: 5672                               │
              │                                                  │
              │  Exchanges:                                      │
              │  - event.approved → notification-service        │
              │  - event.rejected → notification-service        │
              │  - event.saved → analytics-service              │
              │  - cache.invalidate → all services              │
              └──────────────────────────────────────────────────┘
         │                  │               │              │
┌────────▼────────┐  ┌──────▼──────┐  ┌────▼─────┐  ┌────▼──────────┐
│  📦 STORAGE     │  │  ⚙️ CONFIG  │  │ 🗄️ REDIS │  │  📊 ZIPKIN    │
│  SERVICE        │  │  SERVER     │  │  Cache   │  │  Distributed  │
│  Port: 8086     │  │  Port: 8888 │  │  Shared  │  │  Tracing      │
│                 │  │             │  │          │  │  Port: 9411   │
│  - S3 Upload    │  │  Git-based  │  │  Multi   │  │               │
│  - Cloudinary   │  │  Config     │  │  tenant  │  │  Traces all   │
│  - CDN URLs     │  │  Refresh    │  │          │  │  requests     │
│                 │  │             │  │          │  │               │
│  DB: None       │  │  Git Repo   │  │  Port:   │  │  DB: MySQL    │
│  (stateless)    │  │             │  │  6379    │  │               │
└─────────────────┘  └─────────────┘  └──────────┘  └───────────────┘
```

### 1.2 Principios de Microservicios

**1. Single Responsibility Principle (SRP)**
- Cada microservicio tiene UNA responsabilidad clara
- Auth Service → Solo autenticación y autorización
- Event Service → Solo gestión de eventos

**2. Database per Service**
- Cada microservicio tiene su propia base de datos
- No hay acceso directo entre bases de datos
- Comunicación vía APIs o eventos

**3. API First**
- Contratos API definidos primero (OpenAPI/Swagger)
- Versionado de APIs (/v1/, /v2/)
- Retrocompatibilidad

**4. Stateless Services**
- No almacenar estado en memoria del servicio
- Estado en Redis/Database
- Fácil escalado horizontal

**5. Event-Driven Architecture**
- Comunicación asíncrona con RabbitMQ
- Desacoplamiento entre servicios
- Eventual consistency

**6. Resilience by Design**
- Circuit Breakers (Resilience4j)
- Timeouts y Retries
- Fallbacks

---

## 2. Descomposición de Servicios

### 2.1 Auth Service (Puerto 8081)

**Responsabilidades:**
- ✅ Registro de usuarios (Consumer, Promoter)
- ✅ Login (email + password)
- ✅ Generación de JWT tokens (access + refresh)
- ✅ Validación de tokens
- ✅ Refresh tokens
- ✅ Logout (invalidación de tokens)

**Tecnologías:**
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
</dependencies>
```

**Base de Datos:** PostgreSQL
```sql
-- Auth Service DB Schema
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    name VARCHAR(255) NOT NULL,
    role user_role NOT NULL,  -- CONSUMER, PROMOTER, ADMIN

    -- OAuth
    google_id VARCHAR(255) UNIQUE,
    facebook_id VARCHAR(255) UNIQUE,

    email_verified BOOLEAN DEFAULT false,
    enabled BOOLEAN DEFAULT true,

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role ON users(role);

-- Refresh tokens almacenados en Redis con TTL
```

**Endpoints:**
```
POST   /api/v1/auth/register/consumer
POST   /api/v1/auth/register/promoter
POST   /api/v1/auth/login
POST   /api/v1/auth/refresh
POST   /api/v1/auth/logout
GET    /api/v1/auth/validate  (interno, no expuesto por Gateway)
POST   /api/v1/auth/forgot-password
POST   /api/v1/auth/reset-password
```

**Comunicación con otros servicios:**
- **User Service**: Al registrar, envía evento `user.created` vía RabbitMQ
- **Notification Service**: Envía email de bienvenida vía evento

---

### 2.2 Event Service (Puerto 8082)

**Responsabilidades:**
- ✅ CRUD de eventos
- ✅ Búsqueda y descubrimiento de eventos (geolocalización)
- ✅ Gestión de gustos (categorías)
- ✅ Máquina de estados de eventos (DRAFT → PENDING_REVIEW → APPROVED)
- ✅ Estadísticas de eventos (view_count, save_count)

**Tecnologías:**
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-amqp</artifactId>
    </dependency>
    <dependency>
        <groupId>org.hibernatespatial</groupId>
        <artifactId>hibernate-spatial</artifactId>
    </dependency>
</dependencies>
```

**Base de Datos:** PostgreSQL + PostGIS
```sql
-- Event Service DB Schema
CREATE EXTENSION postgis;

CREATE TABLE events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    promoter_id UUID NOT NULL,  -- FK lógico a Promoter Service

    title VARCHAR(255) NOT NULL,
    slug VARCHAR(255) UNIQUE NOT NULL,
    description TEXT NOT NULL,
    image_url VARCHAR(500) NOT NULL,

    start_date TIMESTAMP NOT NULL,
    end_date TIMESTAMP,
    timezone VARCHAR(50) DEFAULT 'Europe/Madrid',

    venue_name VARCHAR(255) NOT NULL,
    venue_address VARCHAR(500) NOT NULL,
    city VARCHAR(100) NOT NULL,
    country VARCHAR(100) DEFAULT 'España',
    lat DECIMAL(10, 8) NOT NULL,
    lng DECIMAL(11, 8) NOT NULL,

    is_free BOOLEAN DEFAULT false,
    price DECIMAL(10, 2),
    currency VARCHAR(10) DEFAULT 'EUR',
    ticket_url VARCHAR(500),

    status event_status DEFAULT 'DRAFT',
    reviewed_by UUID,
    reviewed_at TIMESTAMP,
    rejection_reason TEXT,
    published_at TIMESTAMP,

    view_count INT DEFAULT 0,
    save_count INT DEFAULT 0,
    share_count INT DEFAULT 0,

    metadata JSONB,

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_events_status ON events(status);
CREATE INDEX idx_events_city ON events(city);
CREATE INDEX idx_events_start_date ON events(start_date);
CREATE INDEX idx_events_promoter ON events(promoter_id);
CREATE INDEX idx_events_location ON events USING GIST (ll_to_earth(lat, lng));

CREATE TABLE gustos (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) UNIQUE NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    icon VARCHAR(50),
    category VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE event_gustos (
    event_id UUID REFERENCES events(id) ON DELETE CASCADE,
    gusto_id UUID REFERENCES gustos(id) ON DELETE CASCADE,
    PRIMARY KEY (event_id, gusto_id)
);

CREATE INDEX idx_event_gustos_event ON event_gustos(event_id);
CREATE INDEX idx_event_gustos_gusto ON event_gustos(gusto_id);
```

**Endpoints:**
```
POST   /api/v1/events
GET    /api/v1/events/{id}
PUT    /api/v1/events/{id}
DELETE /api/v1/events/{id}
GET    /api/v1/events/discover
GET    /api/v1/events/my-events
POST   /api/v1/events/{id}/submit-review
POST   /api/v1/events/{id}/approve     (Admin)
POST   /api/v1/events/{id}/reject      (Admin)
GET    /api/v1/events/pending-review   (Admin)

GET    /api/v1/gustos
```

**Comunicación con otros servicios:**
- **Promoter Service (Feign)**: Validar que promoter_id existe y está verificado
- **Storage Service (Feign)**: Validar que image_url existe antes de crear evento
- **Notification Service (RabbitMQ)**: Notificar aprobación/rechazo de evento
- **User Service (RabbitMQ)**: Notificar guardado de evento para actualizar métricas

---

### 2.3 User Service (Puerto 8083)

**Responsabilidades:**
- ✅ Gestión de perfiles de consumidores
- ✅ Gustos del usuario (preferencias)
- ✅ Eventos guardados ("Mis Planes")
- ✅ Ubicación del usuario
- ✅ Historial de eventos

**Tecnologías:**
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-amqp</artifactId>
    </dependency>
</dependencies>
```

**Base de Datos:** PostgreSQL
```sql
-- User Service DB Schema
CREATE TABLE consumers (
    id UUID PRIMARY KEY,  -- Mismo ID que user en Auth Service
    name VARCHAR(255),
    email VARCHAR(255),   -- Duplicado de Auth Service (eventual consistency)

    city VARCHAR(100),
    lat DECIMAL(10, 8),
    lng DECIMAL(11, 8),

    avatar_url VARCHAR(500),
    bio TEXT,

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE user_gustos (
    user_id UUID REFERENCES consumers(id) ON DELETE CASCADE,
    gusto_id UUID NOT NULL,  -- FK lógico a Event Service
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (user_id, gusto_id)
);

CREATE TABLE saved_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES consumers(id) ON DELETE CASCADE,
    event_id UUID NOT NULL,  -- FK lógico a Event Service
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(user_id, event_id)
);

CREATE INDEX idx_saved_events_user ON saved_events(user_id);
CREATE INDEX idx_saved_events_event ON saved_events(event_id);
```

**Endpoints:**
```
GET    /api/v1/users/me
PUT    /api/v1/users/me
GET    /api/v1/users/me/gustos
PUT    /api/v1/users/me/gustos

POST   /api/v1/saved-events
DELETE /api/v1/saved-events/{id}
GET    /api/v1/saved-events
```

**Comunicación con otros servicios:**
- **Event Service (Feign)**: Obtener detalles completos de eventos guardados
- **Auth Service**: Recibe evento `user.created` para crear perfil de consumer
- **Event Service (RabbitMQ)**: Notificar guardado de evento (incrementar save_count)

---

### 2.4 Promoter Service (Puerto 8084)

**Responsabilidades:**
- ✅ Gestión de promotores (organizaciones)
- ✅ Verificación de promotores
- ✅ Listado de eventos del promotor
- ✅ Estadísticas del promotor

**Tecnologías:**
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
</dependencies>
```

**Base de Datos:** PostgreSQL
```sql
-- Promoter Service DB Schema
CREATE TABLE promoters (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID UNIQUE NOT NULL,  -- FK lógico a Auth Service

    organization_name VARCHAR(255) NOT NULL,
    phone VARCHAR(50),
    website VARCHAR(255),
    description TEXT,
    logo_url VARCHAR(500),

    status promoter_status DEFAULT 'PENDING_VERIFICATION',
    verified_at TIMESTAMP,
    verified_by UUID,

    -- Métricas
    total_events INT DEFAULT 0,
    approved_events INT DEFAULT 0,
    rejected_events INT DEFAULT 0,

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_promoters_status ON promoters(status);
CREATE INDEX idx_promoters_user_id ON promoters(user_id);
```

**Endpoints:**
```
POST   /api/v1/promoters
GET    /api/v1/promoters/me
PUT    /api/v1/promoters/me
GET    /api/v1/promoters/{id}
GET    /api/v1/promoters/me/stats

POST   /api/v1/promoters/{id}/verify   (Admin)
POST   /api/v1/promoters/{id}/suspend  (Admin)
```

**Comunicación con otros servicios:**
- **Event Service (Feign)**: Obtener eventos del promotor
- **Auth Service**: Recibe evento `user.created` con role=PROMOTER

---

### 2.5 Notification Service (Puerto 8085)

**Responsabilidades:**
- ✅ Envío de emails (Welcome, Event Approved/Rejected, etc.)
- ✅ Push notifications (iOS + Android vía Firebase)
- ✅ SMS (Twilio - opcional)
- ✅ Historial de notificaciones

**Tecnologías:**
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-mail</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-amqp</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-mongodb</artifactId>
    </dependency>
    <dependency>
        <groupId>com.google.firebase</groupId>
        <artifactId>firebase-admin</artifactId>
    </dependency>
</dependencies>
```

**Base de Datos:** MongoDB (NoSQL por alta escritura)
```javascript
// MongoDB Schema
{
  _id: ObjectId,
  type: "EMAIL" | "PUSH" | "SMS",
  recipient: "user@example.com",
  subject: "Evento aprobado",
  body: "Tu evento ha sido aprobado...",
  status: "PENDING" | "SENT" | "FAILED",
  metadata: {
    eventId: "uuid",
    userId: "uuid"
  },
  sentAt: ISODate,
  createdAt: ISODate
}
```

**Eventos Consumidos (RabbitMQ):**
- `user.created` → Enviar email de bienvenida
- `event.approved` → Notificar promotor
- `event.rejected` → Notificar promotor con razón
- `event.saved` → Notificar promotor (opcional)

**Endpoints:**
```
GET    /api/v1/notifications/my-notifications
POST   /api/v1/notifications/send-test  (Admin)
```

---

### 2.6 Storage Service (Puerto 8086)

**Responsabilidades:**
- ✅ Upload de imágenes a S3/Cloudinary
- ✅ Validación de archivos (tipo, tamaño)
- ✅ Generación de URLs firmadas
- ✅ Transformaciones de imagen (resize, crop)

**Tecnologías:**
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>com.amazonaws</groupId>
        <artifactId>aws-java-sdk-s3</artifactId>
    </dependency>
    <!-- O -->
    <dependency>
        <groupId>com.cloudinary</groupId>
        <artifactId>cloudinary-http44</artifactId>
    </dependency>
</dependencies>
```

**Base de Datos:** Ninguna (servicio stateless)

**Endpoints:**
```
POST   /api/v1/storage/upload
DELETE /api/v1/storage/{key}
GET    /api/v1/storage/{key}/signed-url
```

---

## 3. Infraestructura y Componentes Compartidos

### 3.1 API Gateway (Spring Cloud Gateway)

**Puerto:** 8080

**Responsabilidades:**
- ✅ Punto de entrada único para clientes
- ✅ Routing a microservicios
- ✅ Rate Limiting global
- ✅ Autenticación centralizada (validar JWT)
- ✅ CORS
- ✅ Request/Response logging

**Configuración:**
```yaml
# application.yml
spring:
  cloud:
    gateway:
      routes:
        # Auth Service
        - id: auth-service
          uri: lb://AUTH-SERVICE  # Load balanced vía Eureka
          predicates:
            - Path=/api/v1/auth/**
          filters:
            - RewritePath=/api/v1/auth/(?<segment>.*), /$\{segment}

        # Event Service
        - id: event-service
          uri: lb://EVENT-SERVICE
          predicates:
            - Path=/api/v1/events/**, /api/v1/gustos/**
          filters:
            - AuthenticationFilter  # Custom filter para validar JWT

        # User Service
        - id: user-service
          uri: lb://USER-SERVICE
          predicates:
            - Path=/api/v1/users/**, /api/v1/saved-events/**
          filters:
            - AuthenticationFilter

        # Promoter Service
        - id: promoter-service
          uri: lb://PROMOTER-SERVICE
          predicates:
            - Path=/api/v1/promoters/**
          filters:
            - AuthenticationFilter

        # Storage Service
        - id: storage-service
          uri: lb://STORAGE-SERVICE
          predicates:
            - Path=/api/v1/storage/**
          filters:
            - AuthenticationFilter

      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins:
              - "http://localhost:4200"
              - "https://amigusto.com"
              - "https://admin.amigusto.com"
            allowedMethods:
              - GET
              - POST
              - PUT
              - DELETE
              - PATCH
            allowedHeaders: "*"
            allowCredentials: true

      default-filters:
        - name: Retry
          args:
            retries: 3
            statuses: BAD_GATEWAY, SERVICE_UNAVAILABLE
        - name: CircuitBreaker
          args:
            name: defaultCircuitBreaker
            fallbackUri: forward:/fallback
```

**Custom Authentication Filter:**
```java
@Component
public class AuthenticationFilter implements GatewayFilter, Ordered {

    @Autowired
    private JwtUtil jwtUtil;

    @Autowired
    private WebClient.Builder webClient;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();

        // Extraer token
        String authHeader = request.getHeaders().getFirst("Authorization");
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            return onError(exchange, "Missing authorization header", HttpStatus.UNAUTHORIZED);
        }

        String token = authHeader.substring(7);

        // Validar token localmente (JWT signature)
        if (!jwtUtil.validateToken(token)) {
            return onError(exchange, "Invalid token", HttpStatus.UNAUTHORIZED);
        }

        // Extraer claims
        String userId = jwtUtil.getUserIdFromToken(token);
        String role = jwtUtil.getRoleFromToken(token);

        // Agregar headers para microservicios downstream
        ServerHttpRequest modifiedRequest = exchange.getRequest()
            .mutate()
            .header("X-User-Id", userId)
            .header("X-User-Role", role)
            .build();

        return chain.filter(exchange.mutate().request(modifiedRequest).build());
    }

    private Mono<Void> onError(ServerWebExchange exchange, String message, HttpStatus status) {
        ServerHttpResponse response = exchange.getResponse();
        response.setStatusCode(status);
        return response.setComplete();
    }

    @Override
    public int getOrder() {
        return -1;  // Ejecutar primero
    }
}
```

**¿Por qué validar JWT en el Gateway?**
- **Centralización**: Un solo punto de validación
- **Performance**: Microservicios no necesitan validar JWT
- **Seguridad**: Solo requests autenticados llegan a microservicios

---

### 3.2 Service Discovery (Netflix Eureka)

**Puerto:** 8761

**Responsabilidades:**
- ✅ Registro de microservicios
- ✅ Health checks
- ✅ Load balancing (lado cliente)
- ✅ Descubrimiento dinámico de servicios

**Configuración (Eureka Server):**
```yaml
# application.yml
server:
  port: 8761

eureka:
  instance:
    hostname: localhost
  client:
    registerWithEureka: false  # No se registra a sí mismo
    fetchRegistry: false
  server:
    enableSelfPreservation: false  # Desactivar en dev
```

**Configuración (Microservicio Cliente):**
```yaml
# application.yml (Event Service ejemplo)
spring:
  application:
    name: EVENT-SERVICE

eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
  instance:
    preferIpAddress: true
    leaseRenewalIntervalInSeconds: 10
    metadataMap:
      instanceId: ${spring.application.name}:${random.value}
```

**¿Por qué Eureka?**
- **Descubrimiento Dinámico**: Gateway no necesita IPs fijas de microservicios
- **High Availability**: Múltiples instancias del mismo servicio
- **Health Monitoring**: Detecta servicios caídos automáticamente

---

### 3.3 Config Server (Spring Cloud Config)

**Puerto:** 8888

**Responsabilidades:**
- ✅ Configuración centralizada para todos los microservicios
- ✅ Almacenamiento en Git
- ✅ Refresh dinámico de configuración (sin reiniciar servicios)
- ✅ Profiles (dev, staging, prod)

**Estructura Git:**
```
config-repo/
├── application.yml              # Config común para todos
├── application-dev.yml          # Override para dev
├── application-prod.yml         # Override para prod
├── auth-service.yml             # Config específico Auth Service
├── event-service.yml            # Config específico Event Service
└── ...
```

**Configuración Config Server:**
```yaml
# application.yml
server:
  port: 8888

spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/amigusto/config-repo
          default-label: main
          search-paths: '{application}'
  security:
    user:
      name: config-admin
      password: ${CONFIG_SERVER_PASSWORD}
```

**Cliente (Microservicio):**
```yaml
# bootstrap.yml (cargado antes que application.yml)
spring:
  application:
    name: event-service
  cloud:
    config:
      uri: http://localhost:8888
      username: config-admin
      password: ${CONFIG_SERVER_PASSWORD}
      fail-fast: true
  profiles:
    active: dev
```

**Refresh dinámico:**
```bash
# Actualizar config sin reiniciar servicio
curl -X POST http://localhost:8082/actuator/refresh
```

---

### 3.4 Redis Distribuido

**Puerto:** 6379

**Uso Multi-Tenant:**
```
# Namespace por servicio
amigusto:auth:refresh_token:{userId}
amigusto:event:discover:{city}:{gustoIds}
amigusto:user:saved_events:{userId}
```

**Configuración:**
```yaml
spring:
  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: 6379
      password: ${REDIS_PASSWORD}
      database: 0
      timeout: 2000ms
      lettuce:
        pool:
          max-active: 20
          max-idle: 10
  cache:
    type: redis
    redis:
      time-to-live: 300000  # 5 min
      key-prefix: "amigusto:${spring.application.name}:"
```

**¿Por qué Redis Compartido?**
- **Consistencia**: Cache invalidation cross-service
- **Performance**: Un solo cluster Redis para todos
- **Costo**: Más económico que Redis por servicio

---

### 3.5 Message Broker (RabbitMQ)

**Puerto:** 5672 (AMQP), 15672 (Management UI)

**Exchanges y Queues:**

```
┌─────────────────────────────────────────────────────┐
│            RABBITMQ EXCHANGES & QUEUES              │
├─────────────────────────────────────────────────────┤
│                                                     │
│  📧 Exchange: user.events (Topic)                  │
│     ├─ Queue: user-service.user.created            │
│     ├─ Queue: notification-service.user.created    │
│     └─ Routing Key: user.created                   │
│                                                     │
│  📅 Exchange: event.events (Topic)                 │
│     ├─ Queue: notification-service.event.approved  │
│     ├─ Queue: notification-service.event.rejected  │
│     ├─ Queue: event-service.event.saved            │
│     └─ Routing Keys:                               │
│        - event.approved                            │
│        - event.rejected                            │
│        - event.saved                               │
│                                                     │
│  🔄 Exchange: cache.events (Fanout)                │
│     ├─ Queue: auth-service.cache.invalidate        │
│     ├─ Queue: event-service.cache.invalidate       │
│     ├─ Queue: user-service.cache.invalidate        │
│     └─ Fanout: todos reciben el mensaje            │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Configuración Producer (Auth Service):**
```java
@Configuration
public class RabbitMQConfig {

    @Bean
    public TopicExchange userExchange() {
        return new TopicExchange("user.events");
    }

    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
        RabbitTemplate template = new RabbitTemplate(connectionFactory);
        template.setMessageConverter(jsonMessageConverter());
        return template;
    }

    @Bean
    public MessageConverter jsonMessageConverter() {
        return new Jackson2JsonMessageConverter();
    }
}

@Service
@RequiredArgsConstructor
public class UserEventPublisher {

    private final RabbitTemplate rabbitTemplate;

    public void publishUserCreated(User user) {
        UserCreatedEvent event = UserCreatedEvent.builder()
            .userId(user.getId())
            .email(user.getEmail())
            .name(user.getName())
            .role(user.getRole())
            .createdAt(user.getCreatedAt())
            .build();

        rabbitTemplate.convertAndSend(
            "user.events",         // exchange
            "user.created",        // routing key
            event
        );
    }
}
```

**Configuración Consumer (Notification Service):**
```java
@Configuration
public class RabbitMQConfig {

    @Bean
    public Queue userCreatedQueue() {
        return new Queue("notification-service.user.created", true);
    }

    @Bean
    public Binding userCreatedBinding(Queue userCreatedQueue, TopicExchange userExchange) {
        return BindingBuilder
            .bind(userCreatedQueue)
            .to(userExchange)
            .with("user.created");
    }
}

@Component
@RequiredArgsConstructor
@Slf4j
public class UserEventListener {

    private final EmailService emailService;

    @RabbitListener(queues = "notification-service.user.created")
    public void handleUserCreated(UserCreatedEvent event) {
        log.info("Usuario creado recibido: {}", event.getUserId());

        emailService.sendWelcomeEmail(
            event.getEmail(),
            event.getName()
        );
    }
}
```

**¿Por qué RabbitMQ?**
- **Desacoplamiento**: Servicios no se llaman directamente para eventos
- **Asincronía**: Operaciones no-críticas se procesan en background
- **Resiliencia**: Si Notification Service está caído, mensajes se acumulan
- **Escalabilidad**: Múltiples consumers por queue (competing consumers)

---

## 4. Comunicación entre Microservicios

### 4.1 Comunicación Síncrona (OpenFeign)

**Caso de Uso:** Event Service necesita validar que promoter existe

**Feign Client (Event Service):**
```java
@FeignClient(
    name = "PROMOTER-SERVICE",  // Nombre en Eureka
    fallback = PromoterServiceFallback.class
)
public interface PromoterServiceClient {

    @GetMapping("/api/v1/promoters/{id}")
    PromoterResponse getPromoter(@PathVariable("id") UUID id);

    @GetMapping("/api/v1/promoters/{id}/status")
    PromoterStatus getPromoterStatus(@PathVariable("id") UUID id);
}

@Component
@Slf4j
public class PromoterServiceFallback implements PromoterServiceClient {

    @Override
    public PromoterResponse getPromoter(UUID id) {
        log.error("Fallback: Promoter Service no disponible para ID {}", id);
        throw new ServiceUnavailableException("Promoter Service no disponible");
    }

    @Override
    public PromoterStatus getPromoterStatus(UUID id) {
        log.error("Fallback: Status no disponible");
        return PromoterStatus.UNKNOWN;
    }
}
```

**Uso en Event Service:**
```java
@Service
@RequiredArgsConstructor
@Transactional
public class EventService {

    private final EventRepository eventRepository;
    private final PromoterServiceClient promoterServiceClient;

    public EventResponse createEvent(CreateEventRequest request, UUID promoterId) {
        // Validar promoter existe y está verificado (llamada síncrona)
        try {
            PromoterResponse promoter = promoterServiceClient.getPromoter(promoterId);

            if (promoter.getStatus() != PromoterStatus.VERIFIED) {
                throw new BusinessException("Promotor no verificado");
            }
        } catch (FeignException.NotFound e) {
            throw new ResourceNotFoundException("Promotor no encontrado");
        }

        // Crear evento...
        Event event = eventMapper.toEntity(request);
        event.setPromoterId(promoterId);
        event.setStatus(EventStatus.DRAFT);

        Event saved = eventRepository.save(event);

        return eventMapper.toResponse(saved);
    }
}
```

**¿Cuándo usar Feign (Síncrono)?**
- ✅ Operaciones críticas que requieren respuesta inmediata
- ✅ Validaciones antes de procesar request
- ✅ Leer datos necesarios para completar operación

**¿Cuándo NO usar Feign?**
- ❌ Operaciones que pueden fallar sin afectar el flujo principal
- ❌ Notificaciones (usar RabbitMQ)
- ❌ Cadenas largas de llamadas (crea acoplamiento)

---

### 4.2 Comunicación Asíncrona (RabbitMQ)

**Caso de Uso:** Admin aprueba evento → Notificar promotor

**Event Service (Publisher):**
```java
@Service
@RequiredArgsConstructor
@Transactional
public class EventService {

    private final EventRepository eventRepository;
    private final EventPublisher eventPublisher;

    @Caching(evict = {
        @CacheEvict(value = "discover-events", allEntries = true),
        @CacheEvict(value = "pending-events", allEntries = true)
    })
    public EventResponse approveEvent(UUID eventId, UUID adminId) {
        Event event = eventRepository.findById(eventId)
            .orElseThrow(() -> new ResourceNotFoundException("Evento no encontrado"));

        if (event.getStatus() != EventStatus.PENDING_REVIEW) {
            throw new IllegalStateException("Solo eventos PENDING_REVIEW pueden aprobarse");
        }

        event.setStatus(EventStatus.APPROVED);
        event.setReviewedBy(adminId);
        event.setReviewedAt(LocalDateTime.now());
        event.setPublishedAt(LocalDateTime.now());

        Event updated = eventRepository.save(event);

        // Publicar evento de forma asíncrona
        eventPublisher.publishEventApproved(updated);

        log.info("Evento {} aprobado por admin {}", eventId, adminId);

        return EventMapper.toResponse(updated);
    }
}

@Component
@RequiredArgsConstructor
public class EventPublisher {

    private final RabbitTemplate rabbitTemplate;

    public void publishEventApproved(Event event) {
        EventApprovedEvent message = EventApprovedEvent.builder()
            .eventId(event.getId())
            .promoterId(event.getPromoterId())
            .title(event.getTitle())
            .approvedAt(event.getReviewedAt())
            .build();

        rabbitTemplate.convertAndSend(
            "event.events",       // exchange
            "event.approved",     // routing key
            message
        );
    }
}
```

**Notification Service (Consumer):**
```java
@Component
@RequiredArgsConstructor
@Slf4j
public class EventListener {

    private final EmailService emailService;
    private final PromoterServiceClient promoterClient;

    @RabbitListener(queues = "notification-service.event.approved")
    public void handleEventApproved(EventApprovedEvent event) {
        log.info("Evento aprobado recibido: {}", event.getEventId());

        try {
            // Obtener datos del promotor para enviar email
            PromoterResponse promoter = promoterClient.getPromoter(event.getPromoterId());

            emailService.sendEventApprovedEmail(
                promoter.getEmail(),
                event.getTitle(),
                event.getEventId()
            );

            log.info("Email de aprobación enviado a {}", promoter.getEmail());

        } catch (Exception e) {
            log.error("Error enviando email de aprobación", e);
            // El mensaje se reintentará automáticamente (RabbitMQ retry)
        }
    }

    @RabbitListener(queues = "notification-service.event.rejected")
    public void handleEventRejected(EventRejectedEvent event) {
        log.info("Evento rechazado recibido: {}", event.getEventId());

        try {
            PromoterResponse promoter = promoterClient.getPromoter(event.getPromoterId());

            emailService.sendEventRejectedEmail(
                promoter.getEmail(),
                event.getTitle(),
                event.getRejectionReason()
            );

        } catch (Exception e) {
            log.error("Error enviando email de rechazo", e);
        }
    }
}
```

---

### 4.3 Patrón Saga para Transacciones Distribuidas

**Problema:** Crear evento requiere operaciones en múltiples servicios:
1. Event Service: Crear evento
2. Promoter Service: Incrementar contador de eventos
3. Storage Service: Reservar cuota de almacenamiento

**Solución: Choreography-based Saga con RabbitMQ**

```java
// Event Service
@Transactional
public EventResponse createEvent(CreateEventRequest request, UUID promoterId) {
    // 1. Crear evento en estado DRAFT
    Event event = eventRepository.save(buildEvent(request, promoterId));

    // 2. Publicar evento creado
    eventPublisher.publishEventCreated(event);

    return EventMapper.toResponse(event);
}

// Promoter Service (Listener)
@RabbitListener(queues = "promoter-service.event.created")
@Transactional
public void handleEventCreated(EventCreatedEvent event) {
    try {
        // Incrementar contador
        promoterRepository.incrementEventCount(event.getPromoterId());

        // Publicar éxito
        publishEventCreatedSuccess(event);

    } catch (Exception e) {
        // Publicar fallo (compensating transaction)
        publishEventCreatedFailed(event);
    }
}

// Event Service (Listener de compensación)
@RabbitListener(queues = "event-service.event.created.failed")
@Transactional
public void handleEventCreatedFailed(EventCreatedFailedEvent event) {
    // Rollback: Eliminar evento
    eventRepository.deleteById(event.getEventId());
    log.error("Evento {} eliminado debido a fallo en saga", event.getEventId());
}
```

---

## 5. Gestión de Datos Distribuidos

### 5.1 Database per Service Pattern

**Cada microservicio tiene su propia base de datos:**

```
Auth Service     → PostgreSQL DB (users)
Event Service    → PostgreSQL DB (events, gustos)
User Service     → PostgreSQL DB (consumers, saved_events)
Promoter Service → PostgreSQL DB (promoters)
Notification Svc → MongoDB (notifications)
```

**¿Por qué DB separadas?**
- ✅ **Autonomía**: Servicio puede cambiar esquema sin afectar otros
- ✅ **Escalado Independiente**: Event Service puede tener DB más grande
- ✅ **Resiliencia**: Fallo en una DB no afecta otros servicios
- ✅ **Tecnología Apropiada**: MongoDB para notificaciones (alta escritura)

**Desventajas:**
- ❌ **No hay JOINs cross-DB**: Necesitas Feign o eventos
- ❌ **Consistencia Eventual**: Datos duplicados pueden estar desincronizados
- ❌ **Complejidad**: Más infraestructura que gestionar

---

### 5.2 Data Replication Pattern

**Problema:** User Service necesita email del usuario, pero está en Auth Service

**Solución: Event-Driven Replication**

```java
// Auth Service: Publicar cambios de usuario
@Service
@RequiredArgsConstructor
public class AuthService {

    private final UserRepository userRepository;
    private final UserEventPublisher eventPublisher;

    @Transactional
    public AuthResponse registerConsumer(RegisterRequest request) {
        User user = userRepository.save(buildUser(request));

        // Publicar evento
        eventPublisher.publishUserCreated(user);

        return generateAuthResponse(user);
    }
}

// User Service: Replicar datos
@RabbitListener(queues = "user-service.user.created")
@Transactional
public void handleUserCreated(UserCreatedEvent event) {
    Consumer consumer = Consumer.builder()
        .id(event.getUserId())  // Mismo ID
        .email(event.getEmail())  // Replicado
        .name(event.getName())    // Replicado
        .build();

    consumerRepository.save(consumer);
}
```

**¿Qué datos replicar?**
- ✅ Datos inmutables: ID, email, nombre
- ✅ Datos de lectura frecuente
- ❌ Datos sensibles: password_hash (NUNCA)
- ❌ Datos que cambian frecuentemente

---

### 5.3 CQRS Pattern (Command Query Responsibility Segregation)

**Separar lectura (Query) y escritura (Command)**

**Event Service - Command Side:**
```java
// Escritura: Crear/Actualizar eventos
@PostMapping("/api/v1/events")
public EventResponse createEvent(@RequestBody CreateEventRequest request) {
    return eventCommandService.createEvent(request);
}
```

**Event Service - Query Side (Read Model):**
```java
// Lectura: Vista materializada optimizada para descubrimiento
CREATE MATERIALIZED VIEW event_discovery_view AS
SELECT
    e.id,
    e.title,
    e.city,
    e.lat,
    e.lng,
    e.start_date,
    e.is_free,
    e.price,
    e.image_url,
    array_agg(g.name) as gusto_names,
    array_agg(g.icon) as gusto_icons,
    p.organization_name as promoter_name
FROM events e
JOIN promoters p ON e.promoter_id = p.id
LEFT JOIN event_gustos eg ON e.id = eg.event_id
LEFT JOIN gustos g ON eg.gusto_id = g.id
WHERE e.status = 'APPROVED'
  AND e.start_date > NOW()
GROUP BY e.id, p.id;

CREATE INDEX idx_discovery_city ON event_discovery_view(city);
CREATE INDEX idx_discovery_location ON event_discovery_view USING GIST (ll_to_earth(lat, lng));

REFRESH MATERIALIZED VIEW CONCURRENTLY event_discovery_view;
```

**Query Service:**
```java
@Repository
public interface EventDiscoveryRepository extends JpaRepository<EventDiscoveryView, UUID> {

    @Query(value = """
        SELECT * FROM event_discovery_view
        WHERE city = :city
          AND (6371 * acos(...)) < :radiusKm
        ORDER BY start_date ASC
        """, nativeQuery = true)
    Page<EventDiscoveryView> findDiscoveryEvents(...);
}
```

**¿Por qué CQRS?**
- ✅ **Performance**: Query optimizado sin JOINs costosos
- ✅ **Escalado**: Read replicas separadas de Write DB
- ✅ **Simplicidad**: Queries simples y rápidas

---

## 6. Flujos de Funcionalidades con Microservicios

### 6.1 Registro de Usuario Consumer

```
Usuario (iOS/Android)
    ↓ 1. POST /api/v1/auth/register/consumer
API Gateway (8080)
    ↓ 2. Route a AUTH-SERVICE
Auth Service (8081)
    ↓ 3. Validar email único
    ↓ 4. Hash password (BCrypt)
    ↓ 5. INSERT INTO users
PostgreSQL (Auth DB)
    ↓ 6. Guardar usuario
Auth Service
    ↓ 7. Generar JWT tokens
    ↓ 8. Guardar refresh token
Redis
    ↓ 9. SET refresh_token:{userId} EX 7d
Auth Service
    ↓ 10. Publicar evento user.created
RabbitMQ
    ↓ 11. Exchange: user.events, Key: user.created
    ↓ 12. Fan out a:
    ↓     - user-service.user.created
    ↓     - notification-service.user.created
User Service (8083)
    ↓ 13. Consume evento
    ↓ 14. Crear perfil Consumer
PostgreSQL (User DB)
    ↓ 15. INSERT INTO consumers
Notification Service (8085)
    ↓ 16. Consume evento
    ↓ 17. Enviar email bienvenida
    ↓ 18. Guardar notificación
MongoDB
    ↓ 19. INSERT notification document
Auth Service
    ↓ 20. Retornar AuthResponse con tokens
API Gateway
    ↓ 21. Forward response a cliente
App Móvil
    ↓ 22. Almacenar tokens en Keychain/EncryptedPrefs
    ↓ 23. Navegar a Onboarding
```

**Servicios Involucrados:**
- API Gateway (routing)
- Auth Service (registro, JWT)
- User Service (perfil consumer)
- Notification Service (email)
- Redis (refresh tokens)
- RabbitMQ (eventos)

---

### 6.2 Descubrimiento de Eventos

```
Usuario (iOS/Android)
    ↓ 1. GET /api/v1/events/discover
    ↓    Params: lat, lng, gustoIds, city, page=0
    ↓    Header: Authorization: Bearer {token}
API Gateway (8080)
    ↓ 2. AuthenticationFilter valida JWT
    ↓ 3. Extrae userId, role de token
    ↓ 4. Agrega headers: X-User-Id, X-User-Role
    ↓ 5. Route a EVENT-SERVICE
Event Service (8082)
    ↓ 6. Controller recibe request con headers
    ↓ 7. Verifica caché Redis
Redis
    ↓ 8. GET amigusto:event:discover:{city}:{gustoIds}:page0
    ↓    - Si HIT: retornar (5ms)
    ↓    - Si MISS: continuar
Event Service
    ↓ 9. EventRepository.findByGustosAndCity()
PostgreSQL (Event DB)
    ↓ 10. Query geoespacial con PostGIS:
```

```sql
SELECT DISTINCT e.*
FROM events e
INNER JOIN event_gustos eg ON e.id = eg.event_id
WHERE e.status = 'APPROVED'
  AND e.city = 'Madrid'
  AND e.start_date > NOW()
  AND eg.gusto_id IN (...)
  AND (6371 * acos(...)) < 50
ORDER BY e.start_date ASC
LIMIT 20 OFFSET 0;
```

```
Event Service
    ↓ 11. Mapear a EventResponse DTOs
    ↓ 12. Guardar en Redis con TTL 5 min
Redis
    ↓ 13. SET discover:{city}:{gustoIds}:page0 EX 300
Event Service
    ↓ 14. Retornar PageResponse<EventResponse>
API Gateway
    ↓ 15. Forward response
App Móvil
    ↓ 16. Renderizar eventos en feed
    ↓ 17. Guardar en caché local (CoreData/Room)
```

**Servicios Involucrados:**
- API Gateway (autenticación, routing)
- Event Service (búsqueda geoespacial)
- Redis (caché)

**Tiempo de Respuesta:**
- Cache HIT: ~10ms
- Cache MISS: ~150ms (query + mapeo + cache set)

---

### 6.3 Guardar Evento ("Asistiré")

```
Usuario (iOS/Android)
    ↓ 1. Toca botón "Guardar"
    ↓ 2. POST /api/v1/saved-events
    ↓    Body: { eventId: "uuid" }
    ↓    Header: Authorization: Bearer {token}
API Gateway
    ↓ 3. Valida JWT, extrae userId
    ↓ 4. Route a USER-SERVICE
User Service (8083)
    ↓ 5. SavedEventController.saveEvent()
    ↓ 6. Validar evento existe (Feign call)
Event Service (8082)
    ↓ 7. Feign: GET /api/v1/events/{eventId}
    ↓ 8. Verificar status = APPROVED
    ↓ 9. Retornar EventResponse
User Service
    ↓ 10. Validar no duplicado
    ↓ 11. INSERT INTO saved_events
PostgreSQL (User DB)
    ↓ 12. Guardar evento
User Service
    ↓ 13. Publicar evento event.saved
RabbitMQ
    ↓ 14. Exchange: event.events, Key: event.saved
    ↓ 15. Queue: event-service.event.saved
Event Service
    ↓ 16. Consume evento
    ↓ 17. UPDATE events SET save_count = save_count + 1
PostgreSQL (Event DB)
    ↓ 18. Actualizar métrica
User Service
    ↓ 19. Invalidar caché de "Mis Planes"
Redis
    ↓ 20. DEL amigusto:user:saved_events:{userId}:*
User Service
    ↓ 21. Retornar 200 OK
API Gateway
    ↓ 22. Forward response
App Móvil
    ↓ 23. Actualizar UI (bookmark filled)
    ↓ 24. Mostrar Snackbar "Añadido a Mis Planes"
```

**Servicios Involucrados:**
- API Gateway
- User Service (guardar evento)
- Event Service (validar + actualizar métrica)
- RabbitMQ (evento asíncrono)
- Redis (invalidación caché)

**Comunicación:**
- **Síncrona (Feign)**: User Service → Event Service (validar evento)
- **Asíncrona (RabbitMQ)**: User Service → Event Service (incrementar save_count)

---

### 6.4 Crear Evento (Promotor)

```
Promotor (Angular Web)
    ↓ 1. Llena formulario de evento
    ↓ 2. SUBE IMAGEN PRIMERO:
    ↓    POST /api/v1/storage/upload
    ↓    Body: MultipartFile
API Gateway
    ↓ 3. Route a STORAGE-SERVICE
Storage Service (8086)
    ↓ 4. Validar archivo (tipo, tamaño)
    ↓ 5. Generar key único: events/{uuid}.jpg
    ↓ 6. Upload a S3/Cloudinary
AWS S3
    ↓ 7. Almacenar imagen
    ↓ 8. Retornar URL: https://cdn.amigusto.com/events/xyz.jpg
Storage Service
    ↓ 9. Retornar { imageUrl: "..." }
Promotor
    ↓ 10. Incluye imageUrl en formulario
    ↓ 11. POST /api/v1/events
    ↓     Body: CreateEventRequest
API Gateway
    ↓ 12. Valida JWT, extrae promoterId
    ↓ 13. Route a EVENT-SERVICE
Event Service (8082)
    ↓ 14. Validar promotor existe y verificado (Feign)
Promoter Service (8084)
    ↓ 15. Feign: GET /api/v1/promoters/{promoterId}
    ↓ 16. Verificar status = VERIFIED
    ↓ 17. Retornar PromoterResponse
Event Service
    ↓ 18. Validar imageUrl existe (Feign)
Storage Service
    ↓ 19. Feign: HEAD /api/v1/storage/verify?url={imageUrl}
    ↓ 20. Retornar 200 OK si existe
Event Service
    ↓ 21. Validaciones Jakarta Bean Validation
    ↓ 22. Crear Event con status = DRAFT
PostgreSQL (Event DB)
    ↓ 23. Transaction BEGIN
    ↓     INSERT INTO events
    ↓     INSERT INTO event_gustos (para cada gusto)
    ↓     Transaction COMMIT
Event Service
    ↓ 24. Publicar evento event.created
RabbitMQ
    ↓ 25. Exchange: event.events, Key: event.created
Promoter Service
    ↓ 26. Consume evento
    ↓ 27. UPDATE promoters SET total_events++
Event Service
    ↓ 28. Retornar EventResponse
API Gateway
    ↓ 29. Forward response
Promotor (Angular)
    ↓ 30. Mostrar Snackbar "Evento creado"
    ↓ 31. Navegar a lista de eventos
```

**Servicios Involucrados:**
- API Gateway
- Storage Service (upload imagen)
- Event Service (crear evento)
- Promoter Service (validar + actualizar contador)
- RabbitMQ (evento asíncrono)

---

### 6.5 Aprobar Evento (Admin)

```
Admin (Angular)
    ↓ 1. Revisa evento en cola
    ↓ 2. POST /api/v1/events/{eventId}/approve
API Gateway
    ↓ 3. Valida JWT, verifica role = ADMIN
    ↓ 4. Route a EVENT-SERVICE
Event Service (8082)
    ↓ 5. EventController.approveEvent()
    ↓ 6. Validar estado = PENDING_REVIEW
    ↓ 7. UPDATE events SET status = 'APPROVED', ...
PostgreSQL (Event DB)
    ↓ 8. Actualizar evento
Event Service
    ↓ 9. Publicar evento event.approved
RabbitMQ
    ↓ 10. Exchange: event.events, Key: event.approved
    ↓ 11. Fan out a:
    ↓     - notification-service.event.approved
    ↓     - cache-invalidation.event.approved
Notification Service (8085)
    ↓ 12. Consume evento
    ↓ 13. Obtener datos promotor (Feign)
Promoter Service (8084)
    ↓ 14. Feign: GET /api/v1/promoters/{promoterId}
    ↓ 15. Retornar email del promotor
Notification Service
    ↓ 16. Enviar email aprobación
    ↓ 17. Guardar notificación en MongoDB
Event Service
    ↓ 18. Consume evento cache.invalidate
    ↓ 19. Invalidar cachés afectados
Redis
    ↓ 20. DEL amigusto:event:discover:{city}:*
    ↓     DEL amigusto:event:pending_events:*
    ↓     DEL amigusto:event:event:{eventId}
Event Service
    ↓ 21. UPDATE promoters SET approved_events++
Event Service
    ↓ 22. Retornar EventResponse
API Gateway
    ↓ 23. Forward response
Admin (Angular)
    ↓ 24. Mostrar Snackbar "Evento aprobado"
    ↓ 25. Remover de cola
Apps Móviles (Usuarios)
    ↓ 26. Próximo refresh del feed
    ↓ 27. Evento aparece en descubrimiento
```

**Servicios Involucrados:**
- API Gateway
- Event Service (cambiar estado)
- Notification Service (email promotor)
- Promoter Service (obtener email)
- RabbitMQ (eventos múltiples)
- Redis (invalidación caché)

**Patrón:** Event-Driven con Choreography (cada servicio reacciona a eventos)

---

## 7. Resiliencia y Tolerancia a Fallos

### 7.1 Circuit Breaker (Resilience4j)

**Problema:** Si Promoter Service está caído, Event Service no puede validar promotores

**Solución: Circuit Breaker + Fallback**

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class EventService {

    private final PromoterServiceClient promoterClient;

    @CircuitBreaker(
        name = "promoterService",
        fallbackMethod = "createEventFallback"
    )
    @Retry(name = "promoterService")
    @Timeout(value = 3, unit = ChronoUnit.SECONDS)
    public EventResponse createEvent(CreateEventRequest request, UUID promoterId) {
        // Validar promotor con Feign (puede fallar)
        PromoterResponse promoter = promoterClient.getPromoter(promoterId);

        if (promoter.getStatus() != PromoterStatus.VERIFIED) {
            throw new BusinessException("Promotor no verificado");
        }

        // Crear evento...
        return eventMapper.toResponse(eventRepository.save(buildEvent(request)));
    }

    // Fallback method
    private EventResponse createEventFallback(
        CreateEventRequest request,
        UUID promoterId,
        Exception e
    ) {
        log.error("Promoter Service no disponible, usando validación local", e);

        // Opción 1: Retornar error al usuario
        throw new ServiceUnavailableException("Servicio temporalmente no disponible");

        // Opción 2: Continuar sin validación (guardar como DRAFT)
        // Event event = buildEvent(request);
        // event.setStatus(EventStatus.DRAFT);
        // return eventMapper.toResponse(eventRepository.save(event));
    }
}
```

**Configuración:**
```yaml
# application.yml (Event Service)
resilience4j:
  circuitbreaker:
    instances:
      promoterService:
        registerHealthIndicator: true
        slidingWindowSize: 10
        minimumNumberOfCalls: 5
        permittedNumberOfCallsInHalfOpenState: 3
        automaticTransitionFromOpenToHalfOpenEnabled: true
        waitDurationInOpenState: 10s
        failureRateThreshold: 50
        eventConsumerBufferSize: 10

  retry:
    instances:
      promoterService:
        maxAttempts: 3
        waitDuration: 500ms
        exponentialBackoffMultiplier: 2

  timelimiter:
    instances:
      promoterService:
        timeoutDuration: 3s
```

**Estados del Circuit Breaker:**
```
CLOSED (Normal)
    ↓ 50% de requests fallan
OPEN (Bloqueado)
    ↓ Esperar 10s
HALF_OPEN (Probando)
    ↓ 3 requests exitosos
CLOSED (Recuperado)
```

---

### 7.2 Bulkhead Pattern

**Limitar recursos por dependencia externa:**

```java
@Service
public class EventService {

    @Bulkhead(name = "promoterService", type = Bulkhead.Type.THREADPOOL)
    public PromoterResponse getPromoter(UUID id) {
        return promoterClient.getPromoter(id);
    }
}
```

```yaml
resilience4j:
  bulkhead:
    instances:
      promoterService:
        maxConcurrentCalls: 10
        maxWaitDuration: 1s
```

**¿Por qué Bulkhead?**
- ✅ Evita que un servicio lento bloquee todos los threads
- ✅ Recursos dedicados por dependencia

---

### 7.3 Health Checks

**Endpoint de Health:**
```yaml
# application.yml (todos los microservicios)
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: always
      probes:
        enabled: true
  health:
    circuitbreakers:
      enabled: true
    ratelimiters:
      enabled: true
```

**Response:**
```json
GET /actuator/health

{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP",
      "details": {
        "database": "PostgreSQL",
        "validationQuery": "isValid()"
      }
    },
    "redis": {
      "status": "UP",
      "details": {
        "version": "7.0.0"
      }
    },
    "circuitBreakers": {
      "status": "UP",
      "details": {
        "promoterService": {
          "status": "CLOSED"
        }
      }
    },
    "diskSpace": {
      "status": "UP",
      "details": {
        "total": 500000000000,
        "free": 200000000000
      }
    }
  }
}
```

---

## 8. Observabilidad y Monitoreo

### 8.1 Distributed Tracing (Zipkin + Sleuth)

**Problema:** Request pasa por API Gateway → Auth Service → Event Service → Promoter Service. ¿Dónde está el cuello de botella?

**Solución: Distributed Tracing**

```xml
<!-- Agregar a todos los microservicios -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-sleuth-zipkin</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  sleuth:
    sampler:
      probability: 1.0  # 100% en dev, 0.1 (10%) en prod
  zipkin:
    base-url: http://localhost:9411
    sender:
      type: web
```

**Trace Example:**
```
Trace ID: abc123def456
Span 1: API Gateway         [0ms - 250ms]   Total: 250ms
  Span 2: Event Service     [10ms - 200ms]  Total: 190ms
    Span 3: Promoter Svc    [50ms - 180ms]  Total: 130ms
      Span 4: PostgreSQL    [60ms - 170ms]  Total: 110ms
```

**Zipkin UI:**
```
http://localhost:9411

- Ver todos los traces
- Filtrar por servicio
- Ver latencia por servicio
- Detectar servicios lentos
```

---

### 8.2 Métricas (Prometheus + Grafana)

**Exportar métricas:**
```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

```yaml
management:
  metrics:
    export:
      prometheus:
        enabled: true
  endpoint:
    prometheus:
      enabled: true
```

**Endpoint:**
```
GET /actuator/prometheus

# HELP http_server_requests_seconds
# TYPE http_server_requests_seconds summary
http_server_requests_seconds_count{method="GET",uri="/api/v1/events/discover",status="200"} 15234
http_server_requests_seconds_sum{method="GET",uri="/api/v1/events/discover",status="200"} 4567.89
```

**Prometheus Config:**
```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'event-service'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['localhost:8082']

  - job_name: 'auth-service'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['localhost:8081']
```

**Grafana Dashboards:**
- **Request Rate**: Requests/segundo por servicio
- **Error Rate**: % de errors (5xx)
- **Latency**: p50, p95, p99
- **JVM Metrics**: Heap, GC, Threads
- **Database Connections**: Pool size, active

---

### 8.3 Logging Centralizado (ELK Stack)

**Logstash config:**
```yaml
# logstash.conf
input {
  tcp {
    port => 5000
    codec => json
  }
}

filter {
  if [service] == "event-service" {
    mutate {
      add_field => { "environment" => "production" }
    }
  }
}

output {
  elasticsearch {
    hosts => ["localhost:9200"]
    index => "amigusto-logs-%{+YYYY.MM.dd}"
  }
}
```

**Application logs:**
```java
@Slf4j
@RestController
public class EventController {

    @GetMapping("/api/v1/events/discover")
    public ResponseEntity<Page<EventResponse>> discoverEvents(...) {
        MDC.put("userId", userId);
        MDC.put("city", city);

        log.info("Discovering events for user {} in city {}", userId, city);

        Page<EventResponse> events = eventService.discoverEvents(...);

        log.info("Found {} events", events.getTotalElements());

        MDC.clear();

        return ResponseEntity.ok(events);
    }
}
```

**Kibana Queries:**
```
# Ver todos los errores de Event Service
service:"event-service" AND level:ERROR

# Ver requests lentos (>1s)
service:"event-service" AND duration_ms:>1000

# Ver eventos de un usuario
userId:"550e8400-e29b-41d4-a716-446655440000"
```

---

## 9. Despliegue y CI/CD

### 9.1 Containerización (Docker)

**Dockerfile (Event Service):**
```dockerfile
FROM openjdk:17-jdk-slim

WORKDIR /app

COPY target/event-service-1.0.0.jar app.jar

EXPOSE 8082

ENV SPRING_PROFILES_ACTIVE=prod
ENV JAVA_OPTS="-Xms512m -Xmx1024m"

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

**docker-compose.yml (Local Development):**
```yaml
version: '3.8'

services:
  # Infraestructura
  eureka-server:
    build: ./eureka-server
    ports:
      - "8761:8761"
    environment:
      - SPRING_PROFILES_ACTIVE=dev

  config-server:
    build: ./config-server
    ports:
      - "8888:8888"
    environment:
      - SPRING_PROFILES_ACTIVE=dev
      - GIT_URI=https://github.com/amigusto/config-repo

  postgres:
    image: postgis/postgis:16-3.4
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: amigusto
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - postgres-data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --requirepass redispass

  rabbitmq:
    image: rabbitmq:3-management-alpine
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: rabbitmq
      RABBITMQ_DEFAULT_PASS: rabbitmq

  # Microservicios
  auth-service:
    build: ./auth-service
    ports:
      - "8081:8081"
    depends_on:
      - eureka-server
      - postgres
      - redis
    environment:
      - SPRING_PROFILES_ACTIVE=dev
      - EUREKA_CLIENT_SERVICEURL_DEFAULTZONE=http://eureka-server:8761/eureka/
      - SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/amigusto
      - SPRING_DATA_REDIS_HOST=redis

  event-service:
    build: ./event-service
    ports:
      - "8082:8082"
    depends_on:
      - eureka-server
      - postgres
      - redis
      - rabbitmq
    environment:
      - SPRING_PROFILES_ACTIVE=dev
      - EUREKA_CLIENT_SERVICEURL_DEFAULTZONE=http://eureka-server:8761/eureka/
      - SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/amigusto
      - SPRING_RABBITMQ_HOST=rabbitmq

  api-gateway:
    build: ./api-gateway
    ports:
      - "8080:8080"
    depends_on:
      - eureka-server
      - auth-service
      - event-service
    environment:
      - SPRING_PROFILES_ACTIVE=dev
      - EUREKA_CLIENT_SERVICEURL_DEFAULTZONE=http://eureka-server:8761/eureka/

volumes:
  postgres-data:
```

**Iniciar todos los servicios:**
```bash
docker-compose up -d
```

---

### 9.2 CI/CD Pipeline (GitHub Actions)

**.github/workflows/event-service-ci.yml:**
```yaml
name: Event Service CI/CD

on:
  push:
    branches: [main, develop]
    paths:
      - 'event-service/**'
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgis/postgis:16-3.4
        env:
          POSTGRES_DB: test
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports:
          - 5432:5432
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v3

      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: maven

      - name: Run tests
        working-directory: ./event-service
        run: mvn clean test

      - name: Run integration tests
        working-directory: ./event-service
        run: mvn verify -P integration-tests

  build:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v3

      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Build with Maven
        working-directory: ./event-service
        run: mvn clean package -DskipTests

      - name: Build Docker image
        working-directory: ./event-service
        run: |
          docker build -t amigusto/event-service:${{ github.sha }} .
          docker tag amigusto/event-service:${{ github.sha }} amigusto/event-service:latest

      - name: Login to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Push Docker image
        run: |
          docker push amigusto/event-service:${{ github.sha }}
          docker push amigusto/event-service:latest

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - name: Deploy to Kubernetes
        uses: azure/k8s-deploy@v4
        with:
          manifests: |
            k8s/event-service-deployment.yml
            k8s/event-service-service.yml
          images: |
            amigusto/event-service:${{ github.sha }}
```

---

### 9.3 Kubernetes Deployment

**k8s/event-service-deployment.yml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: event-service
  labels:
    app: event-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: event-service
  template:
    metadata:
      labels:
        app: event-service
    spec:
      containers:
      - name: event-service
        image: amigusto/event-service:latest
        ports:
        - containerPort: 8082
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "prod"
        - name: EUREKA_CLIENT_SERVICEURL_DEFAULTZONE
          value: "http://eureka-service:8761/eureka/"
        - name: SPRING_DATASOURCE_URL
          valueFrom:
            secretKeyRef:
              name: event-service-secrets
              key: database-url
        - name: SPRING_DATASOURCE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: event-service-secrets
              key: database-password
        - name: SPRING_DATA_REDIS_HOST
          value: "redis-service"
        - name: SPRING_RABBITMQ_HOST
          value: "rabbitmq-service"
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8082
          initialDelaySeconds: 60
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8082
          initialDelaySeconds: 30
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: event-service
spec:
  selector:
    app: event-service
  ports:
  - protocol: TCP
    port: 8082
    targetPort: 8082
  type: ClusterIP
```

---

## 🎯 Resumen de Arquitectura de Microservicios

### Microservicios:
1. **Auth Service** (8081) - Autenticación, JWT
2. **Event Service** (8082) - Eventos, búsqueda geoespacial
3. **User Service** (8083) - Consumidores, eventos guardados
4. **Promoter Service** (8084) - Promotores, verificación
5. **Notification Service** (8085) - Emails, push, SMS
6. **Storage Service** (8086) - Upload imágenes S3

### Infraestructura:
- **API Gateway** (8080) - Spring Cloud Gateway
- **Service Discovery** (8761) - Netflix Eureka
- **Config Server** (8888) - Spring Cloud Config
- **Message Broker** (5672) - RabbitMQ
- **Distributed Tracing** (9411) - Zipkin
- **Redis** (6379) - Caché compartido

### Bases de Datos:
- Auth Service → PostgreSQL
- Event Service → PostgreSQL + PostGIS
- User Service → PostgreSQL
- Promoter Service → PostgreSQL
- Notification Service → MongoDB

### Comunicación:
- **Síncrona**: OpenFeign (REST)
- **Asíncrona**: RabbitMQ (Eventos)

### Resiliencia:
- Circuit Breakers (Resilience4j)
- Retries con backoff exponencial
- Timeouts
- Bulkheads
- Fallbacks

### Observabilidad:
- Distributed Tracing (Zipkin)
- Métricas (Prometheus + Grafana)
- Logs centralizados (ELK Stack)
- Health checks

---

**Última actualización:** 2025-10-26
**Versión:** 1.0.0 - Arquitectura de Microservicios
