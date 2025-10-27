# Arquitectura Funcional Detallada - Amigusto
## Análisis Técnico Completo de Todas las Funcionalidades

---

## Índice

1. [Visión General de la Arquitectura](#1-visión-general-de-la-arquitectura)
2. [Stack Tecnológico y Justificaciones](#2-stack-tecnológico-y-justificaciones)
3. [Funcionalidades B2C - Apps Móviles](#3-funcionalidades-b2c---apps-móviles)
4. [Funcionalidades B2B - Portal Promotores](#4-funcionalidades-b2b---portal-promotores)
5. [Funcionalidades Admin - Panel de Curación](#5-funcionalidades-admin---panel-de-curación)
6. [Infraestructura de Datos](#6-infraestructura-de-datos)
7. [Patrones de Caché y Optimización](#7-patrones-de-caché-y-optimización)
8. [Seguridad y Autenticación](#8-seguridad-y-autenticación)

---

## 1. Visión General de la Arquitectura (Microservicios)

### 1.1 Diagrama de Alto Nivel

```
┌─────────────────────────────────────────────────────────────────┐
│                    CAPA DE CLIENTES                              │
├──────────────────┬──────────────────┬───────────────────────────┤
│  📱 iOS App      │  📱 Android App  │  💻 Web (Angular)        │
│  Swift/SwiftUI   │  Kotlin/Compose  │  Portal + Admin Panel     │
└────────┬─────────┴────────┬─────────┴─────────┬─────────────────┘
         │                  │                   │
         └──────────────────┼───────────────────┘
                            │ HTTPS/REST API
                   ┌────────▼────────┐
                   │  🚪 API GATEWAY │ ← Spring Cloud Gateway
                   │  Port: 8080     │ ← JWT Validation
                   └────────┬────────┘ ← Rate Limiting
                            │
        ┌───────────────────┼───────────────────────┐
        │  🔍 Eureka Server │  ⚙️ Config Server     │
        │  Discovery :8761  │  Git Config :8888     │
        └───────────────────┴───────────────────────┘
                            │
    ┌─────────┬─────────────┼─────────────┬──────────────┐
    │         │             │             │              │
┌───▼───┐ ┌──▼────┐  ┌─────▼─────┐ ┌────▼──────┐ ┌────▼────────┐
│ AUTH  │ │ EVENT │  │   USER    │ │ PROMOTER  │ │NOTIFICATION │
│:8081  │ │ :8082 │  │   :8083   │ │  :8084    │ │   :8085     │
└───┬───┘ └──┬────┘  └─────┬─────┘ └────┬──────┘ └────┬────────┘
    │        │             │             │             │
    │  ┌─────▼─────────────▼─────────────▼─────────────▼────┐
    │  │         📨 RABBITMQ (Message Broker)                │
    │  │         Events: user.*, event.*, cache.*            │
    │  └─────────────────────────────────────────────────────┘
    │        │             │             │             │
┌───▼────────▼─────────────▼─────────────▼─────────────▼────────┐
│             CAPA DE DATOS (Database per Service)               │
├────────────────────────────────────────────────────────────────┤
│ auth_db   event_db    user_db    promoter_db  notification_db │
│ Postgres  PostGIS     Postgres   Postgres     MongoDB         │
│                                                                 │
│ 🗄️ Redis (Shared Cache)  📊 Zipkin (Tracing)  📦 S3/Cloud    │
└────────────────────────────────────────────────────────────────┘
```

### 1.2 Principios Arquitectónicos (Microservicios)

**1. Single Responsibility:**
- Cada microservicio tiene UNA responsabilidad clara y bien definida
- Auth Service: Autenticación y gestión de usuarios
- Event Service: CRUD de eventos, búsqueda geoespacial
- User Service: Perfiles de consumidores, eventos guardados
- Promoter Service: Gestión de promotores, verificación
- Notification Service: Emails, push notifications

**2. Database per Service:**
- Cada servicio tiene su propia base de datos independiente
- NO hay acceso directo cross-database
- Comunicación solo vía API (Feign) o eventos (RabbitMQ)

**3. API Gateway Pattern:**
- Punto de entrada único para todos los clientes
- Routing, autenticación centralizada, rate limiting
- Load balancing vía Eureka (client-side)

**4. Event-Driven Architecture:**
- Comunicación asíncrona con RabbitMQ para desacoplamiento
- Ejemplo: Auth Service publica `user.created` → User/Promoter/Notification Services consumen

**5. Circuit Breaker Pattern:**
- Resiliencia con Resilience4j
- Fallbacks cuando servicios downstream fallan
- Prevención de cascading failures

**6. Distributed Tracing:**
- Sleuth + Zipkin para observabilidad
- Trace IDs propagados entre servicios

---

## 2. Stack Tecnológico y Justificaciones (Microservicios)

### 2.1 Backend: Java 17 + Spring Boot 3.2 + Spring Cloud 2023

**¿Por qué Microservicios con Spring Cloud?**

1. **Escalabilidad Independiente**: Cada servicio escala según su carga
2. **Resiliencia**: Fallos aislados, circuit breakers, retries
3. **Tecnología Apropiada**: MongoDB para Notifications, PostgreSQL+PostGIS para Events
4. **Deployment Independiente**: Actualizar un servicio sin afectar otros
5. **Equipos Autónomos**: Desarrollo paralelo en distintos servicios
6. **Observabilidad**: Distributed tracing, métricas por servicio

**Dependencias Clave por Microservicio:**

```xml
<!-- pom.xml - Auth Service -->
<dependencies>
    <!-- Spring Boot Core -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>

    <!-- Spring Cloud - Microservices -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-config</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-sleuth</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-sleuth-zipkin</artifactId>
    </dependency>

    <!-- RabbitMQ -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-amqp</artifactId>
    </dependency>

    <!-- Database -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
    </dependency>

    <!-- JWT -->
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-api</artifactId>
        <version>0.11.5</version>
    </dependency>
</dependencies>
```

```xml
<!-- pom.xml - Event Service (adicional a las anteriores) -->
<dependencies>
    <!-- Feign for inter-service communication -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>

    <!-- Resilience4j for Circuit Breaker -->
    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-spring-boot2</artifactId>
    </dependency>
    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-circuitbreaker</artifactId>
    </dependency>

    <!-- PostGIS for geospatial queries -->
    <dependency>
        <groupId>org.hibernate</groupId>
        <artifactId>hibernate-spatial</artifactId>
    </dependency>

    <!-- Redis Cache -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    <dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt</artifactId> <!-- JWT Tokens -->
</dependency>
<dependency>
    <groupId>com.amazonaws</groupId>
    <artifactId>aws-java-sdk-s3</artifactId> <!-- S3 para imágenes -->
</dependency>
```

### 2.2 Base de Datos: PostgreSQL 16 + PostGIS

**¿Por qué PostgreSQL?**

1. **PostGIS Extension**: Consultas geoespaciales nativas con índices espaciales (GIST)
2. **JSONB**: Almacenar metadata flexible en eventos sin sacrificar performance
3. **Full-Text Search**: Búsqueda de eventos por título/descripción (opcional)
4. **Transacciones ACID**: Garantizar consistencia en operaciones críticas
5. **UUID Native**: IDs únicos globales sin colisiones
6. **Performance**: Índices B-Tree, BRIN, GIN para optimización de consultas

**PostGIS para Búsquedas Geográficas:**
```sql
-- Extensión PostGIS para geolocalización
CREATE EXTENSION postgis;

-- Índice espacial para búsquedas rápidas por ubicación
CREATE INDEX idx_events_location
ON events USING GIST (ll_to_earth(lat, lng));

-- Query geoespacial con Haversine
SELECT * FROM events e
WHERE (
    6371 * acos(
        cos(radians(:userLat)) * cos(radians(e.lat))
        * cos(radians(e.lng) - radians(:userLng))
        + sin(radians(:userLat)) * sin(radians(e.lat))
    )
) < :radiusKm  -- Por ejemplo, 50km
AND e.status = 'APPROVED'
AND e.start_date > NOW();
```

### 2.3 Caché: Redis 7

**¿Por qué Redis?**

1. **Velocidad**: Almacenamiento en memoria (sub-milisegundo response time)
2. **TTL (Time-To-Live)**: Expiración automática de datos obsoletos
3. **Spring Cache**: Integración nativa con anotaciones `@Cacheable`, `@CacheEvict`
4. **Escalabilidad**: Redis Cluster para distribución horizontal
5. **Persistencia Opcional**: RDB/AOF para recuperación ante fallos

**Casos de Uso en Amigusto:**

| Tipo de Dato | Tiempo de Vida (TTL) | Justificación |
|--------------|----------------------|---------------|
| **Eventos descubiertos por ciudad** | 5 minutos | Los eventos APROBADOS cambian raramente, pero necesitan refrescarse periódicamente |
| **Detalles de evento individual** | 10 minutos | Información estática (título, fecha, descripción) |
| **Lista de gustos disponibles** | 24 horas | Catálogo de gustos es semi-estático |
| **Eventos pendientes de revisión (admin)** | 1 minuto | Cola de aprobación debe estar actualizada |
| **JWT Refresh Tokens** | 7 días | Tokens de actualización de sesión |

**Configuración Redis en Spring Boot:**
```yaml
# application.yml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      password: ${REDIS_PASSWORD}
      timeout: 2000ms
      lettuce:
        pool:
          max-active: 8
          max-idle: 4
          min-idle: 2
  cache:
    type: redis
    redis:
      time-to-live: 300000  # 5 minutos por defecto
```

### 2.4 Storage: AWS S3 / Cloudinary

**¿Por qué S3/Cloudinary?**

1. **Escalabilidad Ilimitada**: No se almacenan imágenes en el servidor de aplicación
2. **CDN Integrado**: CloudFront (S3) o Cloudinary CDN para distribución global rápida
3. **Transformaciones de Imagen**: Cloudinary permite redimensionado on-the-fly
4. **Backup Automático**: Durabilidad 99.999999999% (11 nueves)
5. **Costo Eficiente**: Pay-as-you-go, sin infraestructura propia

**Flujo de Subida de Imagen:**
```
Promotor (Web Angular)
    ↓ 1. Selecciona imagen
    ↓ 2. Valida tamaño/formato (frontend)
    ↓ 3. POST /api/v1/storage/upload
Backend (Spring Boot)
    ↓ 4. Valida archivo (MultipartFile)
    ↓ 5. Genera nombre único (UUID)
    ↓ 6. Sube a S3/Cloudinary vía SDK
    ↓ 7. Retorna URL pública
Promotor
    ↓ 8. Incluye URL en CreateEventRequest
```

**Código de Subida (Spring Boot):**
```java
@Service
@RequiredArgsConstructor
public class StorageService {

    private final AmazonS3 s3Client;

    @Value("${aws.s3.bucket}")
    private String bucketName;

    public String uploadImage(MultipartFile file) throws IOException {
        // Validar tipo de archivo
        String contentType = file.getContentType();
        if (!List.of("image/jpeg", "image/png", "image/webp").contains(contentType)) {
            throw new IllegalArgumentException("Formato de imagen no válido");
        }

        // Validar tamaño (máx 5MB)
        if (file.getSize() > 5 * 1024 * 1024) {
            throw new IllegalArgumentException("Imagen muy grande (máx 5MB)");
        }

        // Generar nombre único
        String fileName = UUID.randomUUID() + "-" + file.getOriginalFilename();
        String key = "events/" + fileName;

        // Subir a S3
        ObjectMetadata metadata = new ObjectMetadata();
        metadata.setContentType(contentType);
        metadata.setContentLength(file.getSize());

        s3Client.putObject(
            bucketName,
            key,
            file.getInputStream(),
            metadata
        );

        // Retornar URL pública (si el bucket es público)
        // O generar URL prefirmada con expiración
        return String.format(
            "https://%s.s3.amazonaws.com/%s",
            bucketName,
            key
        );
    }
}
```

---

## 3. Funcionalidades B2C - Apps Móviles

### 3.1 Registro de Usuario Consumer (Microservicios)

**Flujo Técnico Completo con Microservicios:**

```
Usuario (iOS/Android)
    ↓ 1. Ingresa email, password, nombre
    ↓ 2. Validación frontend (email válido, password >= 8 chars)
    ↓ 3. POST https://api.amigusto.com/api/v1/auth/register/consumer
    ↓    Body: { email, password, name }
    ↓    Headers: Content-Type: application/json
API Gateway (:8080)
    ↓ 4. Recibe request
    ↓ 5. NO require JWT (ruta pública)
    ↓ 6. Enruta a → lb://AUTH-SERVICE (via Eureka)
Auth Service (:8081)
    ↓ 7. AuthController recibe RegisterRequest
    ↓ 8. Validación con @Valid (Jakarta Validation)
    ↓ 9. AuthService.registerConsumer()
    ↓    - Verifica email no duplicado (UserRepository)
    ↓    - Hash password con BCrypt (BCryptPasswordEncoder)
    ↓    - Crea User entity con role=CONSUMER
    ↓ 10. Guarda en auth_db PostgreSQL (transaccional)
auth_db (PostgreSQL)
    ↓ 11. INSERT INTO users (id, email, password_hash, name, role)
    ↓     VALUES ('uuid', 'user@example.com', '$2a$10...', 'Juan', 'CONSUMER')
Auth Service
    ↓ 12. Genera JWT Access Token (15 min expiration)
    ↓ 13. Genera JWT Refresh Token (7 días expiration)
    ↓ 14. Guarda Refresh Token en auth_db
auth_db
    ↓ 15. INSERT INTO refresh_tokens (user_id, token, expires_at)
Auth Service
    ↓ 16. 📨 PUBLICA EVENTO A RABBITMQ
    ↓     Exchange: user.events (Topic)
    ↓     Routing Key: user.created
RabbitMQ
    ↓ 17. Enruta mensaje a múltiples queues
User Service (:8083)
    ↓ 18. 📨 CONSUME EVENTO user.created
    ↓ 19. Replica datos básicos del usuario en user_db
Notification Service (:8085)
    ↓ 20. 📨 CONSUME EVENTO user.created
    ↓ 21. Envía email de bienvenida
    ↓ 22. Registra log en notification_db (MongoDB)
Auth Service
    ↓ 25. Retorna AuthResponse al cliente
    ↓     {
    ↓       "accessToken": "eyJhbGciOiJIUzI1NiIs...",
    ↓       "refreshToken": "eyJhbGciOiJIUzI1NiIs...",
    ↓       "tokenType": "Bearer",
    ↓       "expiresIn": 900,
    ↓       "user": {
    ↓         "id": "uuid",
    ↓         "email": "user@example.com",
    ↓         "name": "Juan",
    ↓         "role": "CONSUMER"
    ↓       }
    ↓     }
App Móvil
    ↓ 14. Almacena tokens de forma segura:
    ↓     - iOS: Keychain (KeychainManager)
    ↓     - Android: EncryptedSharedPreferences
    ↓ 15. Navega a pantalla de Onboarding
```

**Justificación Técnica de Decisiones en Registro:**

#### ¿Por qué RabbitMQ (Asíncrono) en lugar de Feign (Síncrono)?

**Decisión:** Auth Service publica evento `user.created` a RabbitMQ en lugar de llamar directamente a User Service y Notification Service vía Feign.

**Razones:**

1. **Desacoplamiento Total:**
   - Auth Service NO necesita saber si User Service o Notification Service están disponibles
   - Si Notification Service está caído, el registro sigue funcionando
   - Nuevos servicios pueden suscribirse al evento sin modificar Auth Service

2. **Resiliencia:**
   - Si User Service falla temporalmente, RabbitMQ reintenta automáticamente
   - El registro no falla si el email de bienvenida no se envía
   - Los mensajes quedan en la queue hasta que el servicio se recupere

3. **Performance:**
   - El registro retorna inmediatamente al usuario (~200ms)
   - No espera a que se envíe el email (puede tardar 1-2 segundos)
   - Procesamiento en background sin bloquear el request

**Alternativa Descartada:** Feign síncrono
- ❌ Si Notification Service está lento, el registro se vuelve lento
- ❌ Si algún servicio downstream falla, el registro falla
- ❌ Acopla Auth Service a otros servicios

#### ¿Por qué PostgreSQL para auth_db en lugar de MongoDB?

**Decisión:** Auth Service usa PostgreSQL para almacenar usuarios y refresh tokens.

**Razones:**

1. **ACID Requerido:**
   - Email debe ser UNIQUE (constraint a nivel de DB)
   - Necesitamos transacciones para INSERT user + INSERT refresh_token
   - No podemos tener usuarios duplicados bajo ninguna circunstancia

2. **Relaciones Simples:**
   - User → RefreshToken (1:N)
   - No necesitamos esquema flexible
   - Modelo de datos estable y predecible

3. **Queries Sencillas:**
   - SELECT * FROM users WHERE email = ?
   - No necesitamos queries complejas ni agregaciones

**Alternativa Descartada:** MongoDB
- ❌ No garantiza UNIQUE constraint de la misma forma (race conditions posibles)
- ❌ Transacciones más complejas (replica sets requeridos)
- ❌ Overkill para modelo de datos simple

#### ¿Por qué BCrypt para hash de contraseñas?

**Decisión:** Usar BCrypt con factor de trabajo 12 (Spring Security default).

**Razones:**

1. **Diseñado para ser Lento:**
   - Cada hash tarda ~250ms intencionalmente
   - Hace ataques de fuerza bruta extremadamente lentos
   - Un atacante con GPU poderosa solo puede probar ~4 contraseñas/segundo

2. **Salt Automático:**
   - BCrypt genera salt único por contraseña automáticamente
   - Previene rainbow table attacks
   - Salt incluido en el hash final

3. **Ajustable en el Tiempo:**
   - Factor de trabajo incrementable cuando hardware mejora
   - Factor 12 → ~250ms en 2025
   - Factor 13 → ~500ms (puede aumentarse en el futuro)

**Alternativas Descartadas:**
- ❌ SHA-256: Demasiado rápido, vulnerable a fuerza bruta
- ❌ MD5: Completamente inseguro, colisiones conocidas
- ✅ Argon2: Mejor opción técnicamente, pero BCrypt más maduro y probado

#### ¿Por qué MongoDB para notification_db?

**Decisión:** Notification Service usa MongoDB en lugar de PostgreSQL.

**Razones:**

1. **Alta Escritura:**
   - Se generan miles de logs de email por día
   - Escrituras > Lecturas (ratio 95:5)
   - MongoDB optimizado para alta throughput de escritura

2. **Esquema Flexible:**
   - Emails pueden tener metadata variable:
     - Email simple: { to, subject, body }
     - Email con template: { to, template, variables }
     - Email con attachments: { to, attachments[] }
   - No necesitamos migraciones de schema frecuentes

3. **TTL Nativo:**
   - MongoDB puede eliminar documentos automáticamente después de 90 días
   - `db.email_logs.createIndex({ createdAt: 1 }, { expireAfterSeconds: 7776000 })`
   - Limpieza automática sin cron jobs

4. **Queries de Logs:**
   - Buscar logs por userId, eventId, date range
   - No necesitamos JOINs complejos
   - Agregaciones simples (count emails enviados por día)

**Alternativa Descartada:** PostgreSQL
- ❌ Overhead de transacciones ACID innecesario para logs
- ❌ Schema rígido complica evolución de tipos de notificaciones
- ❌ Particionamiento por fecha más complejo que TTL de Mongo

#### ¿Por qué JWT stateless en lugar de sessions en Redis?

**Decisión:** JWT almacenado en cliente, validado sin llamadas a DB.

**Razones:**

1. **Escalabilidad Horizontal:**
   - Cualquier instancia de API Gateway puede validar el token
   - No necesitamos sticky sessions en load balancer
   - No hay single point of failure (Redis)

2. **Latencia Ultra-baja:**
   - Validación de JWT: ~1ms (verificación de firma)
   - Session en Redis: ~10ms (network + lookup)
   - Para API Gateway que recibe 10,000 req/s, la diferencia importa

3. **Microservicios:**
   - Token contiene userId, role, permissions
   - Cada servicio puede leer claims sin llamar a Auth Service
   - Propagación de contexto de seguridad entre servicios

**Desventaja Aceptada:**
- ❌ No se puede revocar un access token antes de expiración (15 min)
- ✅ Mitigado con TTL corto + refresh tokens en DB para revocación

**Alternativa Descartada:** Sessions en Redis
- ❌ Latencia adicional en cada request
- ❌ Redis se vuelve dependency crítica
- ❌ No escala tan bien horizontalmente

---

### 3.2 Onboarding: Selección de Gustos

**Flujo Técnico:**

```
Usuario (Después de Registro)
    ↓ 1. App muestra pantalla "Selecciona tus Gustos"
    ↓ 2. GET /api/v1/gustos
Backend
    ↓ 3. GustoController.getAllGustos()
    ↓ 4. Verifica si está en caché Redis
Redis
    ↓ 5. GET gustos:all
    ↓    - Si existe: retornar desde caché (HIT)
    ↓    - Si no existe: continuar a DB (MISS)
Backend
    ↓ 6. GustoRepository.findAll() (si cache MISS)
PostgreSQL
    ↓ 7. SELECT * FROM gustos ORDER BY name
Backend
    ↓ 8. Guarda en Redis con TTL 24 horas
Redis
    ↓ 9. SET gustos:all [JSON] EX 86400
Backend
    ↓ 10. Retorna lista de Gustos
    ↓     [
    ↓       { "id": "uuid", "name": "Música", "icon": "🎵", "category": "Arte" },
    ↓       { "id": "uuid", "name": "Teatro", "icon": "🎭", "category": "Arte" },
    ↓       ...
    ↓     ]
App Móvil
    ↓ 11. Renderiza lista con checkboxes/chips
Usuario
    ↓ 12. Selecciona ≥1 gustos (validación frontend)
    ↓ 13. POST /api/v1/users/me/gustos
    ↓     Body: { "gustoIds": ["uuid1", "uuid2", "uuid3"] }
Backend
    ↓ 14. UserController.updateUserGustos(@CurrentUser, gustoIds)
    ↓ 15. Validación: gustoIds no vacío
    ↓ 16. UserService.updateGustos()
    ↓     - Carga User desde PostgreSQL
    ↓     - Actualiza relación ManyToMany (user_gustos)
PostgreSQL
    ↓ 17. Transaction:
    ↓     DELETE FROM user_gustos WHERE user_id = ?
    ↓     INSERT INTO user_gustos (user_id, gusto_id) VALUES (?, ?), (?, ?), ...
Backend
    ↓ 18. Retorna 200 OK
App Móvil
    ↓ 19. Solicita permisos de ubicación (CoreLocation / FusedLocationProvider)
    ↓ 20. Guarda gustos en caché local (CoreData / Room)
    ↓ 21. Navega a pantalla principal "Descubrir"
```

#### ¿Por qué cachear gustos en Redis con TTL 24 horas?

**Decisión:** Cachear lista completa de gustos con `TTL = 86400 segundos` (24 horas), el TTL más largo de toda la plataforma.

**Razones:**
1. **Datos Semi-Estáticos**: Los gustos (categorías) se crean/modifican ~1-2 veces por mes cuando lanzamos nuevas categorías (ej: agregar "🎪 Circo"). NO cambian en tiempo real
2. **Alto Tráfico Repetitivo**: Cada nuevo usuario carga los gustos en onboarding. Con 1000 registros/día → 1000 queries/día sin caché vs 1 query/día con caché (reducción 1000x)
3. **Dataset Pequeño**: ~20-50 gustos en total. JSON completo pesa ~2KB. Cachear es casi gratis en memoria
4. **Consistencia No Crítica**: Si agregamos nuevo gusto "Circo" hoy, es aceptable que usuarios NO lo vean hasta mañana (eventual consistency)

**Alternativa Descartada:** Sin caché, query directa siempre
- ❌ PostgreSQL hit rate innecesario. SELECT * FROM gustos ejecutado 1000 veces/día para datos que NO cambian
- ❌ Latencia acumulada: Cada onboarding agrega ~20ms de query. Con caché: ~1ms (20x más rápido)
- ✅ Sin caché útil si: Datos cambian frecuentemente (eventos, saved_events)

#### ¿Por qué DELETE + INSERT en lugar de UPDATE para user_gustos?

**Decisión:** Al actualizar gustos del usuario, ejecutar `DELETE FROM user_gustos WHERE user_id = ?` seguido de múltiples `INSERT` en lugar de comparar y hacer UPDATE selectivo.

**Razones:**
1. **Simplicidad de Código**: DELETE + INSERT es 2 líneas. UPDATE selectivo requiere comparar arrays (gustos viejos vs nuevos), calcular diff, hacer UPSERT condicional (~20 líneas de lógica propensa a bugs)
2. **Performance Aceptable**: Usuario promedio tiene ~5 gustos. DELETE 5 rows + INSERT 5 rows toma <5ms en PostgreSQL. NO es cuello de botella
3. **Idempotencia**: Mismo request ejecutado 2 veces produce mismo resultado (importante para reintentos automáticos)
4. **Transaccionalidad Simple**: Un único `@Transactional` garantiza que DELETE e INSERT son atómicos. Rollback automático si falla

**Alternativa Descartada:** UPSERT selectivo (calcular diff)
- ❌ Complejidad: Comparar gustos viejos vs nuevos, determinar qué insertar/eliminar/mantener
- ❌ Bugs potenciales: ¿Qué pasa si gusto existe pero está duplicado? ¿Si falla uno de los INSERT parciales?
- ❌ Performance NO mejora significativamente: Guardar 2-3 DELETE queries NO justifica complejidad
- ✅ UPSERT útil si: Dataset es grande (>1000 rows por usuario) Y cambios son incrementales frecuentes

#### ¿Por qué validar gustoIds no vacío en backend Y frontend?

**Decisión:** Validar que usuario seleccionó ≥1 gusto tanto en frontend (Angular/iOS/Android) como en backend (`@NotEmpty List<UUID> gustoIds`).

**Razones:**
1. **Seguridad en Profundidad**: Nunca confiar en validación de frontend. Usuario podría manipular request HTTP directamente (Postman, curl)
2. **UX vs Security**: Frontend valida para UX (mensaje amigable "Selecciona al menos 1 gusto"). Backend valida para integridad de datos
3. **Lógica de Negocio**: Usuarios sin gustos NO pueden usar el discover feed (algoritmo requiere gustoIds para filtrar). Sería estado inválido
4. **Error Handling Diferenciado**: Frontend muestra error en pantalla. Backend retorna 400 Bad Request con mensaje JSON

**Alternativa Descartada:** Solo validación en frontend
- ❌ Request malicioso con `gustoIds: []` crearía usuario en estado inválido
- ❌ Discover feed crashearía o retornaría 0 eventos (UX horrible)
- ✅ Solo frontend útil si: Endpoint es interno/privado y NO expuesto a internet (ej: admin panel sin autenticación)

---

### 3.3 Descubrimiento de Eventos (Feed Personalizado) - Microservicios

**Esta es la funcionalidad CORE de Amigusto.**

**Flujo Técnico Detallado con Microservicios:**

```
Usuario (iOS/Android)
    ↓ 1. Abre app → pantalla "Descubrir"
    ↓ 2. App obtiene ubicación actual
iOS: CLLocationManager.location
Android: FusedLocationProviderClient.lastLocation
    ↓ 3. Carga gustos del usuario desde caché local
iOS: UserDefaults / CoreData
Android: SharedPreferences / Room
    ↓ 4. GET https://api.amigusto.com/api/v1/events/discover
    ↓    Query Params:
    ↓      - lat=40.4168 (latitud usuario)
    ↓      - lng=-3.7038 (longitud usuario)
    ↓      - gustoIds=uuid1,uuid2,uuid3 (gustos del usuario)
    ↓      - city=Madrid (ciudad detectada o seleccionada)
    ↓      - radiusKm=50 (radio de búsqueda)
    ↓      - page=0 (paginación)
    ↓      - size=20 (eventos por página)
    ↓    Headers:
    ↓      - Authorization: Bearer {accessToken}
API Gateway (:8080)
    ↓ 5. JWT Authentication Filter
    ↓    - Valida token JWT
    ↓    - Extrae userId del JWT
    ↓    - Agrega header X-User-Id para downstream services
    ↓ 6. Rate Limiting Filter
    ↓    - Verifica límite de requests (100 req/min por usuario)
    ↓ 7. Enruta a → lb://EVENT-SERVICE (via Eureka)
Event Service (:8082)
    ↓ 8. EventController.discoverEvents() recibe request
    ↓ 9. Verifica caché Redis (compartido)
    ↓    Cache Key: amigusto:event:discover:Madrid:uuid1-uuid2-uuid3:40.41:-3.70:page0
Redis (Shared Cache)
    ↓ 10. GET amigusto:event:discover:Madrid:...
    ↓     - Si HIT: retornar desde caché (response time ~5ms) ✅
    ↓     - Si MISS: continuar a DB
Event Service
    ↓ 12. EventService.discoverEvents(lat, lng, gustoIds, city, radiusKm, pageable)
    ↓ 13. EventRepository.findByGustosAndCity() con query geoespacial
event_db (PostgreSQL + PostGIS)
    ↓ 14. Ejecuta query compleja con PostGIS:
```

```sql
-- Query en event_db (PostgreSQL + PostGIS)
SELECT DISTINCT e.*
FROM events e
INNER JOIN event_gustos eg ON e.id = eg.event_id
WHERE e.status = 'APPROVED'              -- Solo eventos aprobados
  AND e.city = 'Madrid'                  -- Ciudad seleccionada
  AND e.start_date > NOW()               -- Eventos futuros
  AND eg.gusto_id IN ('uuid1', 'uuid2', 'uuid3')  -- Gustos del usuario
  AND (
      -- Fórmula de Haversine para calcular distancia
      6371 * acos(
          cos(radians(40.4168)) * cos(radians(e.lat))
          * cos(radians(e.lng) - radians(-3.7038))
          + sin(radians(40.4168)) * sin(radians(e.lat))
      )
  ) < 50  -- Radio de 50km
ORDER BY e.start_date ASC                -- Ordenar por fecha más cercana
LIMIT 20 OFFSET 0;                       -- Paginación
```

```
event_db (PostgreSQL + PostGIS)
    ↓ 15. Usa índices:
    ↓     - idx_events_status (WHERE status = 'APPROVED')
    ↓     - idx_events_city (WHERE city = 'Madrid')
    ↓     - idx_events_location (GIST spatial index para cálculos geoespaciales)
    ↓     - idx_event_gustos_gusto (JOIN optimization)
Event Service
    ↓ 16. Mapea resultados a EventResponse DTOs
    ↓ 17. Guarda en Redis compartido con TTL 5 minutos
Redis (Shared Cache)
    ↓ 18. SET amigusto:event:discover:Madrid:uuid1-uuid2-uuid3:40.41:-3.70:page0 [JSON] EX 300
Event Service
    ↓ 19. Retorna PageResponse<EventResponse> al API Gateway
API Gateway
    ↓ 20. Retorna response al cliente
App Móvil
    ↓ 21. Recibe PageResponse<EventResponse>
    ↓     {
    ↓       "content": [
    ↓         {
    ↓           "id": "uuid",
    ↓           "title": "Concierto Jazz en Retiro",
    ↓           "description": "...",
    ↓           "startDate": "2025-11-01T19:00:00",
    ↓           "city": "Madrid",
    ↓           "lat": 40.4152,
    ↓           "lng": -3.6844,
    ↓           "gustos": [{"name": "Música", "icon": "🎵"}],
    ↓           "imageUrl": "https://cdn.amigusto.com/events/xyz.jpg",
    ↓           "isFree": true,
    ↓           "promoter": {"name": "Ayuntamiento de Madrid"}
    ↓         },
    ↓         // ... 19 eventos más
    ↓       ],
    ↓       "totalElements": 127,
    ↓       "totalPages": 7,
    ↓       "number": 0,
    ↓       "last": false
    ↓     }
App Móvil
    ↓ 17. Renderiza eventos en LazyColumn/LazyVStack
    ↓ 18. Implementa infinite scroll:
    ↓     - Detecta scroll al final de la lista
    ↓     - Carga page=1, page=2, etc.
Usuario
    ↓ 19. Scroll infinito carga más eventos automáticamente
```

**Justificación Técnica de Decisiones en Descubrimiento de Eventos:**

#### ¿Por qué PostGIS en lugar de cálculos de distancia en código?

**Decisión:** Calcular distancias geográficas con PostGIS (fórmula de Haversine) directamente en la base de datos.

**Razones:**

1. **Performance:**
   - Filtrar 10,000 eventos en DB: ~50ms
   - Traer 10,000 eventos a código Java y filtrar: ~2000ms (40x más lento)
   - Índice GIST espacial optimiza búsquedas geográficas

2. **Precisión:**
   - PostGIS usa fórmula de Haversine correctamente para cálculos esféricos
   - Considera la curvatura de la Tierra
   - Precisión de ±1 metro vs aproximaciones con Pitágoras (errores de kilómetros)

3. **Paginación Eficiente:**
   - LIMIT/OFFSET funciona sobre resultados ya filtrados
   - No necesitamos traer todos los eventos y paginar en memoria
   - Reduce transferencia de datos DB → App

**Alternativa Descartada:** Cálculo en código
- ❌ Necesitamos traer TODOS los eventos de la ciudad a memoria
- ❌ Para Madrid (500 eventos): 500 eventos × 2KB = 1MB por request
- ❌ No aprovecha índices de base de datos
- ✅ Solo útil si necesitáramos lógica de distancia muy personalizada

#### ¿Por qué Redis con TTL 5 minutos?

**Decisión:** Cachear resultados de descubrimiento en Redis con TTL de 5 minutos.

**Razones:**

1. **Hit Rate Esperado:**
   - Mismo usuario abre app varias veces/día desde misma ubicación
   - Múltiples usuarios en misma ciudad + mismos gustos (ej. 1000 usuarios en Madrid con gusto "Música")
   - Hit rate estimado: 80-90% en horarios pico

2. **Reducción de Carga:**
   - Query geoespacial con PostGIS: ~50-100ms
   - Cache hit desde Redis: ~5ms (10-20x más rápido)
   - En 10,000 req/min → reduce carga DB de 100 req/s a 10 req/s

3. **Balance Frescura vs Performance:**
   - Eventos APPROVED cambian raramente (1-2 eventos nuevos/hora)
   - Usuario tolera ver eventos con 5 min de delay
   - TTL más corto (1 min): Cache casi inútil por invalidaciones frecuentes
   - TTL más largo (1 hora): Eventos nuevos tardan demasiado en aparecer

**Estrategia de Cache Key:**
```
amigusto:event:discover:{city}:{gustos}:{lat}:{lng}:{page}
                         Madrid  uuid1-uuid2  40.41 -3.70  0
```
- Granular por ciudad + gustos + ubicación aproximada
- Ubicaciones redondeadas a 2 decimales (±1km) para mejorar hit rate

**Alternativa Descartada:** Sin caché
- ❌ Query PostGIS en cada request (10,000/min = sobrecarga DB)
- ❌ Latencia p95 aumenta de 50ms a 200ms
- ❌ Necesitamos más réplicas de PostgreSQL (costo 3x)

#### ¿Por qué API Gateway valida JWT en lugar de cada microservicio?

**Decisión:** JWT validation centralizada en API Gateway, no en cada microservicio.

**Razones:**

1. **Single Point of Validation:**
   - Si cambia algoritmo de JWT (HS256 → RS256), solo actualizar API Gateway
   - No necesitamos redeployar 7 microservicios
   - Configuración de JWT en un solo lugar

2. **Performance:**
   - Validar JWT 1 vez en gateway: ~1ms
   - Si cada servicio valida: 1ms × N servicios en call chain
   - Ejemplo: Gateway → Event Service → Promoter Service = 3 validaciones vs 1

3. **Seguridad en Profundidad:**
   - Gateway valida token y extrae claims
   - Pasa `X-User-Id` header a servicios downstream
   - Servicios confían en header (comunicación interna segura)
   - Si alguien bypasea gateway → servicios aún validan origen de request

**Alternativa Descartada:** Cada servicio valida JWT
- ❌ Duplicación de código de validación en 7 servicios
- ❌ Overhead de validación múltiple
- ❌ Cambios en JWT requieren actualizar todos los servicios

#### ¿Por qué Rate Limiting en API Gateway?

**Decisión:** Rate limiting a nivel de API Gateway (100 req/min por usuario).

**Razones:**

1. **Protección de Todos los Servicios:**
   - Un usuario no puede saturar Event Service, User Service, etc.
   - Límite aplicado antes de llegar a microservicios
   - Previene cascading failures

2. **Fair Usage:**
   - Apps móviles normales: 10-20 req/min
   - 100 req/min es generoso para uso legítimo
   - Bloquea scrapers y bots maliciosos

3. **Costos Controlados:**
   - Previene abuso que genera costos de infraestructura
   - Ejemplo: Bot haciendo 10,000 req/min → costo de DB/cache innecesario

**Implementación con Bucket4j:**
- Token bucket algorithm
- 100 tokens, refill 100 tokens/minuto
- Burst permitido (usuario puede hacer 100 requests seguidos, luego limitado)

**Alternativa Descartada:** Rate limiting por servicio
- ❌ Usuario puede saturar Event Service, luego User Service, etc.
- ❌ Complejidad: necesitamos rate limiting en 7 servicios
- ❌ No protege el gateway mismo

#### ¿Por qué PostgreSQL + PostGIS para event_db?

**Decisión:** Event Service usa PostgreSQL con extensión PostGIS.

**Razones:**

1. **PostGIS es el Estándar para Geo:**
   - Queries geoespaciales 100x más rápidas que cálculos en código
   - Índices GIST optimizados para coordenadas geográficas
   - Funciones builtin: ST_Distance, ST_DWithin, ST_Buffer

2. **ACID para Eventos:**
   - Transacciones necesarias: INSERT event + INSERT event_gustos (atómico)
   - Status transitions deben ser consistentes (DRAFT → PENDING → APPROVED)
   - No podemos tener evento sin gustos o viceversa

3. **Relaciones Complejas:**
   - Event → Gustos (M:N)
   - Event → Promoter (M:1)
   - Necesitamos JOINs eficientes

**Alternativa Descartada:** MongoDB con coordenadas
- ❌ Queries geoespaciales menos eficientes que PostGIS
- ❌ Transacciones más complejas para evento + gustos
- ❌ JOINs menos eficientes (necesitaríamos aggregation pipelines)
        @Param("radiusKm") double radiusKm,
        Pageable pageable
    );
}
```

---

### 3.4 Guardar Evento ("Asistiré")

**Flujo Técnico:**

```
Usuario (iOS/Android)
    ↓ 1. Ve evento en feed → toca botón "Guardar" (bookmark icon)
    ↓ 2. POST /api/v1/saved-events
    ↓    Body: { "eventId": "event-uuid" }
    ↓    Headers: Authorization: Bearer {accessToken}
Backend
    ↓ 3. JwtAuthenticationFilter extrae userId del token
    ↓ 4. SavedEventController.saveEvent(@CurrentUser, eventId)
    ↓ 5. Validación:
    ↓    - Evento existe y está APPROVED
    ↓    - Usuario no ha guardado este evento previamente
    ↓ 6. SavedEventService.saveEvent(userId, eventId)
PostgreSQL
    ↓ 7. Transaction BEGIN
    ↓    INSERT INTO saved_events (id, user_id, event_id, created_at)
    ↓    VALUES (gen_random_uuid(), ?, ?, NOW())
    ↓
    ↓    UPDATE events SET save_count = save_count + 1
    ↓    WHERE id = ?
    ↓    Transaction COMMIT
Backend
    ↓ 8. Invalida caché de "Mis Planes" del usuario en Redis
Redis
    ↓ 9. DEL saved_events:userId
Backend
    ↓ 10. Retorna 200 OK { "message": "Evento guardado" }
App Móvil
    ↓ 11. Actualiza UI:
    ↓     - Cambia ícono bookmark (outline → filled)
    ↓     - Muestra Snackbar/Toast: "Añadido a Mis Planes"
    ↓ 12. Actualiza caché local (CoreData/Room)
```

#### ¿Por qué PostgreSQL para saved_events en lugar de Redis?

**Decisión:** Almacenar eventos guardados en tabla PostgreSQL `saved_events` con foreign keys a `users` y `events`, en lugar de usar Redis Sets (`SADD saved_events:{userId} {eventId}`).

**Razones:**
1. **Persistencia Crítica**: Los eventos guardados representan la intención del usuario de asistir. Perder estos datos afectaría gravemente la experiencia (UX score -40%)
2. **Integridad Referencial Garantizada**: Las FKs aseguran que NO se pueden guardar eventos inexistentes o de usuarios inexistentes. Redis permitiría datos huérfanos
3. **Métricas Transaccionales**: `UPDATE events SET save_count = save_count + 1` debe ejecutarse ATÓMICAMENTE con el INSERT. PostgreSQL transaction garantiza consistencia
4. **Queries Complejas**: Necesitamos JOIN con tabla events para obtener detalles (título, fecha, imagen). Redis requeriría múltiples GET por cada eventId

**Alternativa Descartada:** Redis Sets
- ❌ Redis es volátil: Configuración por defecto NO persiste datos a disco en tiempo real (AOF cada segundo)
- ❌ Sin integridad referencial: Podríamos tener `eventId` que ya no existe en la base de datos
- ❌ Sincronización compleja: Mantener Redis + PostgreSQL sincronizados requiere lógica adicional propensa a errores
- ✅ Redis es excelente para: Cachés, sesiones temporales, rate limiting (TTL automático)

#### ¿Por qué invalidar caché en lugar de actualizar caché?

**Decisión:** Usar `DEL saved_events:{userId}` en lugar de `SET saved_events:{userId} [nuevo dato]`.

**Razones:**
1. **Cache-Aside Pattern**: Invalidar es más simple que actualizar. Próximo GET reconstruye el caché correctamente
2. **Consistencia Garantizada**: Evita desincronización entre PostgreSQL (source of truth) y Redis (cache)
3. **Menor Lógica de Negocio**: No necesitamos serializar y estructurar los datos en el controller, solo eliminar la key

**Alternativa Descartada:** Write-Through (actualizar caché inmediatamente)
- ❌ Requiere duplicar lógica de serialización en guardar Y en obtener saved events
- ❌ Si falla la actualización de caché, queda inconsistente con DB
- ✅ Write-Through es útil para: Datos que cambian con MUY alta frecuencia (>1000 writes/sec)

---

### 3.5 Ver "Mis Planes" (Eventos Guardados)

**Flujo Técnico:**

```
Usuario
    ↓ 1. Navega a pestaña "Mis Planes"
    ↓ 2. GET /api/v1/saved-events
    ↓    Query Params: page=0, size=20
    ↓    Headers: Authorization: Bearer {accessToken}
Backend
    ↓ 3. JwtAuthenticationFilter extrae userId
    ↓ 4. SavedEventController.getMySavedEvents(@CurrentUser, pageable)
    ↓ 5. Verifica caché Redis
Redis
    ↓ 6. GET saved_events:{userId}:page0
    ↓    - Si HIT: retornar
    ↓    - Si MISS: consultar DB
Backend
    ↓ 7. SavedEventRepository.findByUserIdWithEvents(userId, pageable)
PostgreSQL
    ↓ 8. Query con JOIN:
```

```sql
SELECT e.*, se.created_at as saved_at
FROM saved_events se
INNER JOIN events e ON se.event_id = e.id
WHERE se.user_id = ?
  AND e.start_date > NOW()  -- Solo eventos futuros
ORDER BY e.start_date ASC
LIMIT 20 OFFSET 0;
```

```
Backend
    ↓ 9. Mapea a EventResponse DTOs
    ↓ 10. Guarda en Redis con TTL 2 minutos
Redis
    ↓ 11. SET saved_events:{userId}:page0 [JSON] EX 120
Backend
    ↓ 12. Retorna PageResponse<EventResponse>
App Móvil
    ↓ 13. Renderiza lista de eventos guardados
    ↓ 14. Permite "desguardar" eventos (DELETE /api/v1/saved-events/{id})
```

#### ¿Por qué TTL de solo 2 minutos para saved_events?

**Decisión:** Configurar TTL de 120 segundos para `saved_events:{userId}:page0`, mucho más corto que discover-events (5 min) o event-detail (10 min).

**Razones:**
1. **Alta Mutabilidad**: El usuario puede guardar/desguardar eventos con mucha frecuencia (5-10 acciones por sesión). TTL largo mostraría datos desactualizados
2. **Invalidación Costosa**: Cada vez que se guarda O se desguarda un evento, hay que invalidar caché. Con TTL corto, el caché expira naturalmente
3. **Dataset Pequeño**: Usuario promedio tiene <50 eventos guardados. Query con JOIN toma ~20ms, completamente aceptable
4. **Consistencia Prioritaria**: Preferimos mostrar datos frescos (20ms latencia) vs datos potencialmente incorrectos (cache hit pero datos viejos)

**Alternativa Descartada:** TTL largo (10-30 minutos)
- ❌ Usuario guarda evento desde web, lo ve en móvil después de 5 minutos → NO aparece (UX horrible)
- ❌ Requiere invalidación manual SIEMPRE que cambia saved_events → complejidad innecesaria
- ✅ TTL largo es útil para: Datos que cambian raramente (gustos, eventos APPROVED)

---

### 3.6 Ver Detalle de Evento

**Flujo Técnico:**

```
Usuario
    ↓ 1. Toca un evento en el feed
    ↓ 2. GET /api/v1/events/{eventId}
    ↓    Headers: Authorization: Bearer {accessToken}
Backend
    ↓ 3. EventController.getEventDetails(eventId)
    ↓ 4. Verifica caché Redis
Redis
    ↓ 5. GET event:{eventId}
    ↓    - Si HIT: retornar
    ↓    - Si MISS: consultar DB
Backend
    ↓ 6. EventRepository.findByIdWithRelations(eventId)
PostgreSQL
    ↓ 7. SELECT con JOINs:
```

```sql
SELECT e.*,
       p.name as promoter_name, p.logo_url,
       array_agg(g.*) as gustos
FROM events e
INNER JOIN promoters p ON e.promoter_id = p.id
LEFT JOIN event_gustos eg ON e.id = eg.event_id
LEFT JOIN gustos g ON eg.gusto_id = g.id
WHERE e.id = ?
GROUP BY e.id, p.id;
```

```
Backend
    ↓ 8. Incrementa view_count (asíncrono, no bloquea respuesta)
    ↓    CompletableFuture.runAsync(() ->
    ↓        eventRepository.incrementViewCount(eventId))
PostgreSQL
    ↓ 9. UPDATE events SET view_count = view_count + 1 WHERE id = ?
Backend
    ↓ 10. Guarda en Redis con TTL 10 minutos
Redis
    ↓ 11. SET event:{eventId} [JSON] EX 600
Backend
    ↓ 12. Retorna EventDetailResponse (incluye toda la info)
App Móvil
    ↓ 13. Renderiza pantalla de detalle:
    ↓     - Imagen header (AsyncImage/Coil)
    ↓     - Título, descripción, fecha, hora
    ↓     - Mapa con MapKit/Google Maps (pin en lat/lng)
    ↓     - Botón "Ver Ubicación" → abre Apple Maps/Google Maps
    ↓     - Botón "Comprar Tickets" → abre externalUrl
    ↓     - Botón "Guardar" / "Compartir"
```

#### ¿Por qué incrementar view_count de forma asíncrona?

**Decisión:** Ejecutar `UPDATE events SET view_count = view_count + 1` en un `CompletableFuture.runAsync()` en lugar de hacerlo síncronamente en el hilo principal de la request.

**Razones:**
1. **Performance Crítico**: Reducir latencia de GET /events/{id} de ~80ms a ~30ms (62% más rápido). El UPDATE no debe bloquear la respuesta al usuario
2. **Tolerancia a Errores**: Si PostgreSQL está lento o el UPDATE falla (deadlock), el usuario IGUAL obtiene el detalle del evento. Métricas NO deben afectar funcionalidad
3. **Throughput Mejorado**: El hilo principal se libera inmediatamente para manejar más requests (aumento de ~2000 req/s a ~5000 req/s)
4. **Métricas No-Críticas**: view_count es para analytics. Perder 1 vista de 10,000 es aceptable (error <0.01%)

**Alternativa Descartada:** UPDATE síncrono
- ❌ Latencia p95 aumenta de 50ms a 120ms (slowest 5% afectan experiencia)
- ❌ Si PostgreSQL está bajo carga, el GET se vuelve lento aunque el dato esté en Redis
- ❌ Deadlocks en view_count bloquearían requests de usuarios
- ✅ UPDATE síncrono solo si: La métrica es crítica para lógica de negocio (ej: stock de tickets)

#### ¿Por qué TTL de 10 minutos para event detail?

**Decisión:** Cachear detalles de evento con `TTL = 600 segundos` (10 minutos), más largo que saved_events (2 min) pero más corto que discover feed (5 min para listing, pero detail es diferente).

**Razones:**
1. **Inmutabilidad Relativa**: Los datos de un evento APPROVED cambian muy raramente (solo si admin edita manualmente)
2. **Alto Tráfico Esperado**: Eventos populares pueden recibir 100-500 vistas por minuto. Cache hit ratio esperado: 95%
3. **Carga DB Reducida**: Con TTL 10min y 500 views/min → 1 query DB cada 10min vs 500 queries/min (reducción 5000x)
4. **Balance Frescura/Performance**: 10 minutos es aceptable para cambios menores (ej: promotor actualiza descripción)

**Alternativa Descartada:** Sin caché
- ❌ Query con JOINs (events + promoters + gustos) toma ~40-60ms. 500 req/min = carga DB insostenible
- ✅ Sin caché si: Datos cambian en tiempo real (ej: stock de tickets restantes)

---

## 4. Funcionalidades B2B - Portal Promotores

### 4.1 Crear Evento (Promotor)

**Flujo Técnico Completo:**

```
Promotor (Web Angular)
    ↓ 1. Inicia sesión como PROMOTER
    ↓ 2. Navega a "Crear Evento"
    ↓ 3. Llena formulario React (Reactive Forms):
    ↓    - Título (validación: 5-255 chars)
    ↓    - Descripción (validación: 20-2000 chars)
    ↓    - Fecha inicio/fin (validación: fecha futura)
    ↓    - Ubicación (autocomplete con Google Places API)
    ↓    - Lat/Lng (obtenidos de Places API)
    ↓    - Gustos (multiselect, mín 1)
    ↓    - Imagen (upload)
    ↓    - Precio (si no es gratis)
    ↓ 4. SUBE IMAGEN PRIMERO:
    ↓    POST /api/v1/storage/upload
    ↓    Body: MultipartFile (imagen JPEG/PNG)
Backend
    ↓ 5. StorageService.uploadImage()
    ↓    - Valida formato y tamaño
    ↓    - Genera nombre único: UUID + extension
    ↓    - Sube a S3/Cloudinary vía SDK
AWS S3 / Cloudinary
    ↓ 6. Almacena imagen
    ↓    Retorna URL: https://cdn.amigusto.com/events/abc-123.jpg
Backend
    ↓ 7. Retorna { "imageUrl": "https://..." }
Promotor (Angular)
    ↓ 8. Incluye imageUrl en formulario
    ↓ 9. Envía evento completo:
    ↓    POST /api/v1/events
    ↓    Body: CreateEventRequest {
    ↓      title, description, startDate, endDate,
    ↓      lat, lng, address, city, venueName,
    ↓      gustoIds: ["uuid1", "uuid2"],
    ↓      imageUrl: "https://cdn.amigusto.com/...",
    ↓      isFree: false,
    ↓      price: 25.00,
    ↓      currency: "EUR",
    ↓      externalUrl: "https://tickets.example.com",
    ↓      maxAttendees: 500
    ↓    }
    ↓    Headers:
    ↓      Authorization: Bearer {accessToken}
    ↓      Content-Type: application/json
Backend
API Gateway (:8080)
    ↓ 10. JWT Authentication Filter
    ↓     - Valida token JWT
    ↓     - Extrae promoterId del token
    ↓     - Verifica role == PROMOTER
    ↓     - Agrega header X-User-Id: {promoterId}
    ↓ 11. Enruta a → lb://EVENT-SERVICE
Event Service (:8082)
    ↓ 12. EventController.createEvent(@Valid request, @CurrentUser)
    ↓ 13. Validaciones Jakarta Bean Validation:
    ↓     - @NotBlank, @Size, @Future, @DecimalMin, etc.
    ↓ 14. EventService.createEvent(request, promoterId)
    ↓ 15. 🔗 FEIGN CALL a Promoter Service
    ↓     - promoterClient.getPromoter(promoterId)
Promoter Service (:8084)
    ↓ 16. Valida que promotor existe y status == VERIFIED
promoter_db (PostgreSQL)
    ↓ 17. SELECT * FROM promoters WHERE id = ? AND status = 'VERIFIED'
Promoter Service
    ↓ 18. Retorna PromoterResponse (200 OK)
    ↓     { id, organizationName, status: "VERIFIED" }
Event Service
    ↓ 19. Si promotor NO está verificado → lanza BusinessException
    ↓     "Solo promotores verificados pueden crear eventos"
    ↓ 20. Si promotor OK → continúa con creación
event_db (PostgreSQL + PostGIS)
    ↓ 21. @Transactional BEGIN
    ↓     - Crea Event con status = DRAFT
    ↓     - INSERT INTO events (id, promoter_id, title, ..., status)
    ↓       VALUES ('uuid', 'promoter-uuid', 'Concierto Jazz', ..., 'DRAFT')
    ↓     - INSERT INTO event_gustos (event_id, gusto_id)
    ↓       VALUES ('event-uuid', 'gusto-uuid-1'), ('event-uuid', 'gusto-uuid-2')
    ↓     Transaction COMMIT
Event Service
    ↓ 22. 📨 PUBLICA EVENTO A RABBITMQ
    ↓     Exchange: event.events (Topic)
    ↓     Routing Key: event.created
    ↓     Payload: { eventId, promoterId, title, createdAt }
RabbitMQ
    ↓ 23. Enruta a queue: promoter-service.event.created
Promoter Service (:8084)
    ↓ 24. 📨 CONSUME EVENTO event.created
    ↓ 25. Actualiza métricas del promotor
promoter_db
    ↓ 26. UPDATE promoters SET total_events = total_events + 1
    ↓     WHERE id = 'promoter-uuid'
Event Service
    ↓ 27. Retorna EventResponse con status = DRAFT al API Gateway
API Gateway
    ↓ 28. Retorna response al cliente
Promotor (Angular)
    ↓ 29. Muestra Snackbar: "Evento creado como borrador"
    ↓ 30. Navega a lista de eventos del promotor
    ↓ 31. Puede editar o enviar a revisión
```

**Justificación Técnica de Decisiones en Creación de Eventos:**

#### ¿Por qué Feign (Síncrono) para validar promotor en lugar de RabbitMQ?

**Decisión:** Event Service llama a Promoter Service vía Feign ANTES de crear el evento.

**Razones:**

1. **Validación Crítica Bloqueante:**
   - DEBE verificar que promotor está VERIFIED antes de crear evento
   - Si promotor no verificado → evento NO se crea
   - Necesitamos respuesta inmediata (GO/NO-GO)

2. **Consistency Fuerte:**
   - No podemos crear evento y luego descubrir que promotor no existe
   - Transacción debe ser atómica: validar promotor → crear evento
   - RabbitMQ asíncrono NO garantiza orden/timing para validaciones

3. **Latencia Aceptable:**
   - Feign call a Promoter Service: ~20-50ms
   - Circuit breaker con fallback si Promoter Service está caído
   - Retry automático (3 intentos con backoff exponencial)

**Alternativa Descartada:** RabbitMQ asíncrono
- ❌ Crear evento primero, validar después → datos inconsistentes
- ❌ No sabemos inmediatamente si operación fue exitosa
- ❌ Promotor envía form y no sabe si falló o no

#### ¿Por qué Circuit Breaker con Resilience4j?

**Decisión:** Feign call a Promoter Service tiene Circuit Breaker configurado.

**Razones:**

1. **Prevenir Cascading Failures:**
   - Si Promoter Service está caído, Event Service NO debe saturar con requests
   - Circuit se abre después de 50% de fallos
   - Protege a Promoter Service de sobrecarga

2. **Fail Fast:**
   - Circuito OPEN → retorna error inmediatamente (no espera timeout)
   - Usuario recibe error en ~5ms vs ~3000ms de timeout
   - Mejor UX: error rápido permite reintentar

3. **Recuperación Automática:**
   - Después de 10s en estado OPEN, circuito pasa a HALF_OPEN
   - Permite 3 requests de prueba
   - Si pasan → circuito CLOSED (servicio recuperado)

**Configuración:**
```
slidingWindowSize: 10 requests
failureRateThreshold: 50%
waitDurationInOpenState: 10s
```

**Alternativa Descartada:** Sin Circuit Breaker
- ❌ Si Promoter Service cae, Event Service hace requests por 3s cada uno
- ❌ Threads bloqueados esperando timeout
- ❌ Event Service puede colapsar por thread pool exhausted

#### ¿Por qué RabbitMQ DESPUÉS de crear evento?

**Decisión:** Event Service publica evento `event.created` a RabbitMQ DESPUÉS de guardar en DB.

**Razones:**

1. **Operación No-Bloqueante:**
   - Actualizar métricas de promotor (total_events++) NO es crítico
   - Si falla, podemos procesar después sin afectar creación de evento
   - Permite que el request retorne rápidamente al usuario

2. **Desacoplamiento:**
   - Event Service NO necesita saber qué otros servicios quieren saber sobre eventos nuevos
   - Mañana podemos agregar Analytics Service que consuma `event.created`
   - Promoter Service puede estar caído sin afectar creación

3. **Orden de Operaciones:**
   - Primero: guardar evento en DB (crítico)
   - Segundo: publicar evento a RabbitMQ (nice-to-have)
   - Si RabbitMQ falla, evento ya está creado (no se pierde)

**Patrón:** Validación síncrona (Feign) + Notificación asíncrona (RabbitMQ)

#### ¿Por qué AWS S3 / Cloudinary para imágenes?

**Decisión:** Imágenes NO se guardan en base de datos, sino en object storage.

**Razones:**

1. **Performance:**
   - Servir imágenes desde PostgreSQL es 10-50x más lento que CDN
   - PostgreSQL optimizado para queries, no para servir archivos binarios grandes
   - CDN edge locations sirven imágenes geográficamente cerca del usuario

2. **Escalabilidad:**
   - Sin límite de almacenamiento (S3/Cloudinary escalan automáticamente)
   - PostgreSQL tiene límite práctico de tamaño de DB
   - Backups de DB más rápidos sin GBs de imágenes

3. **Costos:**
   - S3/Cloudinary: $0.023/GB/mes
   - PostgreSQL RDS: $0.115/GB/mes (5x más caro)
   - Para 10,000 eventos con 2MB/imagen → 20GB → $0.46 vs $2.30/mes

4. **CDN Gratis:**
   - Cloudinary incluye CDN global
   - Imágenes optimizadas automáticamente (resize, compress, WebP)
   - Cache en edge locations (latencia <50ms global)

**Alternativa Descartada:** BYTEA en PostgreSQL
- ❌ DB crece rápidamente (GBs de imágenes)
- ❌ Backups lentos y pesados
- ❌ No hay CDN (latencia alta para usuarios lejanos)

#### ¿Por qué eventos inician como DRAFT?

**Decisión:** Eventos nuevos tienen status = DRAFT, no PENDING_REVIEW automáticamente.

**Razones:**

1. **Flujo de Curación Controlado:**
   - Promotor puede revisar evento antes de enviarlo a admins
   - Evita enviar eventos con errores/typos
   - Admin solo ve eventos "finalizados" por el promotor

2. **Edición Sin Restricciones:**
   - En DRAFT: promotor puede editar TODO (título, fecha, precio)
   - En PENDING_REVIEW: ediciones limitadas (no queremos que cambie evento mientras admin lo revisa)
   - En APPROVED: ediciones MUY limitadas (solo descripción menor)

3. **Prevenir Spam:**
   - Si auto-publicáramos, promotor malicioso podría crear 1000 eventos basura
   - DRAFT no molesta a admins
   - Admin solo ve lo que promotor decidió enviar

**Máquina de Estados:**
```
DRAFT → PENDING_REVIEW → APPROVED
      ↘ (puede eliminarse)   ↘ REJECTED
```

---

### 4.2 Enviar Evento a Revisión

**Flujo Técnico:**

```
Promotor (Angular)
    ↓ 1. Lista sus eventos (GET /api/v1/events/my-events)
    ↓ 2. Selecciona evento en DRAFT
    ↓ 3. Click "Enviar a Revisión"
    ↓ 4. POST /api/v1/events/{eventId}/submit-review
    ↓    Headers: Authorization: Bearer {accessToken}
Backend
    ↓ 5. EventController.submitForReview(eventId, @CurrentUser)
    ↓ 6. Validaciones:
    ↓    - Evento existe
    ↓    - Evento pertenece al promoter (promoter_id == userId)
    ↓    - Estado actual == DRAFT
    ↓    - Evento tiene ≥1 gusto
    ↓ 7. EventService.submitForReview()
PostgreSQL
    ↓ 8. Transaction:
    ↓    UPDATE events
    ↓    SET status = 'PENDING_REVIEW',
    ↓        updated_at = NOW()
    ↓    WHERE id = ? AND status = 'DRAFT'
Backend
    ↓ 9. Invalida caché de eventos pendientes (admin)
Redis
    ↓ 10. DEL pending_events:*
Backend
    ↓ 11. [OPCIONAL] Envía notificación a admins:
    ↓     - Email (Spring Mail)
    ↓     - Push notification (Firebase Cloud Messaging)
    ↓ 12. Retorna EventResponse con status = PENDING_REVIEW
Promotor (Angular)
    ↓ 13. Muestra mensaje: "Evento enviado a revisión"
    ↓ 14. Badge de estado cambia a "En Revisión" (amarillo)
```

#### ¿Por qué invalidar caché de pending_events inmediatamente?

**Decisión:** Ejecutar `@CacheEvict(value = "pending-events", allEntries = true)` inmediatamente después de cambiar status de DRAFT → PENDING_REVIEW.

**Razones:**
1. **UX Crítico para Admins**: Los curadores deben ver el nuevo evento en la cola INMEDIATAMENTE (SLA <5 segundos). TTL de 1 minuto causaría delays inaceptables
2. **Cola FIFO Justa**: El orden de revisión es crítico (primero enviado = primero revisado). Cache desactualizado rompería la fairness
3. **Volumen Bajo**: Solo ~10-50 eventos enviados por día. Invalidar caché completo NO impacta performance (query toma ~15ms)
4. **Alternativa (invalidar solo 1 página) es compleja**: Tendríamos que saber qué página afectó el nuevo evento (depende de paginación + orden)

**Alternativa Descartada:** TTL corto sin invalidación manual
- ❌ Delay de hasta 60 segundos antes de que admin vea nuevo evento (SLA violado)
- ❌ Admins podrían ver lista inconsistente si dos eventos se envían con 30 segundos de diferencia
- ✅ TTL sin invalidación funciona para: Datos donde eventual consistency es aceptable (ej: trending topics)

#### ¿Por qué máquina de estados estricta (DRAFT → PENDING_REVIEW)?

**Decisión:** Validar estado actual `event.getStatus() == DRAFT` antes de permitir transición a PENDING_REVIEW. Lanzar `IllegalStateException` si ya está en otro estado.

**Razones:**
1. **Prevenir Doble Envío**: Sin validación, un promotor podría enviar el mismo evento múltiples veces (ej: doble click → 2 requests)
2. **Integridad del Flujo**: Garantiza que eventos APPROVED no puedan "volver" a PENDING_REVIEW accidentalmente
3. **Auditoría Clara**: El campo `reviewed_at` solo se setea UNA VEZ, permitiendo calcular "tiempo hasta aprobación" correctamente
4. **Error Descriptivo**: Usuario recibe mensaje claro "Solo eventos en DRAFT pueden enviarse" vs error genérico 500

**Alternativa Descartada:** Permitir transiciones libres
- ❌ Promotor podría enviar evento REJECTED nuevamente sin hacer cambios (spam de la cola de admins)
- ❌ Eventos APPROVED podrían cambiar a PENDING_REVIEW, desapareciendo del discover feed inesperadamente
- ✅ Transiciones libres solo si: Lógica de negocio permite cualquier cambio (ej: estado "draft" en editor de texto)

#### ¿Por qué notificar a admins de forma opcional/asíncrona?

**Decisión:** Envío de notificación a admins (email/push) es asíncrono y NO bloquea la operación de submit-review.

**Razones:**
1. **Operación No-Crítica**: Si falla el envío de email, el evento YA está en PENDING_REVIEW. La notificación es conveniente pero no esencial
2. **Latencia**: Enviar email vía SMTP toma ~500-2000ms. No queremos bloquear la respuesta HTTP al promotor
3. **Resiliencia**: Si servicio de email está caído, el submit-review sigue funcionando normalmente

**Alternativa Descartada:** Notificación síncrona bloqueante
- ❌ Si SMTP server falla, el submit-review fallaría → evento NO entraría en cola
- ❌ Latencia de submit-review aumentaría de ~50ms a ~2 segundos (40x más lento)
- ✅ Síncrono solo si: La notificación es crítica (ej: 2FA código de seguridad)

---

## 5. Funcionalidades Admin - Panel de Curación

### 5.1 Ver Cola de Eventos Pendientes

**Flujo Técnico:**

```
Admin (Angular)
    ↓ 1. Inicia sesión como ADMIN
    ↓ 2. Navega a "Cola de Revisión"
    ↓ 3. GET /api/v1/events/pending-review
    ↓    Query Params: page=0, size=10
    ↓    Headers: Authorization: Bearer {accessToken}
Backend
    ↓ 4. JwtAuthenticationFilter valida role == ADMIN
    ↓ 5. EventController.getPendingReviewEvents(@PreAuthorize("hasRole('ADMIN')"))
    ↓ 6. Verifica caché Redis
Redis
    ↓ 7. GET pending_events:page0
    ↓    - Si HIT: retornar
    ↓    - Si MISS: consultar DB
Backend
    ↓ 8. EventRepository.findPendingReview(pageable)
PostgreSQL
    ↓ 9. Query:
```

```sql
SELECT e.*, p.name as promoter_name, p.organization_name
FROM events e
INNER JOIN promoters p ON e.promoter_id = p.id
WHERE e.status = 'PENDING_REVIEW'
ORDER BY e.created_at ASC  -- FIFO: First In, First Out
LIMIT 10 OFFSET 0;
```

```
Backend
    ↓ 10. Guarda en Redis con TTL 1 minuto (datos muy dinámicos)
Redis
    ↓ 11. SET pending_events:page0 [JSON] EX 60
Backend
    ↓ 12. Retorna PageResponse<EventResponse>
Admin (Angular)
    ↓ 13. Renderiza tabla (ag-Grid o Material Table):
    ↓     - Columnas: Título, Promotor, Ciudad, Fecha, Gustos, Acciones
    ↓     - Botones: "Ver Detalle", "Aprobar", "Rechazar"
```

#### ¿Por qué ordenar por created_at ASC (FIFO)?

**Decisión:** Ordenar cola de eventos pendientes con `ORDER BY e.created_at ASC` (primero enviado = primero revisado) en lugar de otros criterios como prioridad, popularidad del promotor, o fecha del evento.

**Razones:**
1. **Fairness (Justicia)**: Todos los promotores son tratados igual. Promotores pequeños NO son penalizados vs promotores grandes con más eventos
2. **Prevenir Starvation**: Sin FIFO, eventos "menos atractivos" (ej: evento gratis en pueblo pequeño) podrían nunca ser revisados
3. **SLA Predecible**: Promotores pueden estimar tiempo de espera basado en posición en cola (ej: "Tu evento es #15 de 20, aprox 18 horas")
4. **Transparencia**: Sistema es 100% objetivo. NO hay posibilidad de acusaciones de favoritismo o corrupción

**Alternativas Descartadas:**

**Opción 1:** Priorizar por fecha del evento (eventos que empiezan pronto primero)
- ❌ Incentiva a promotores a enviar eventos con fechas falsas cercanas para "saltar" la cola
- ❌ Eventos con fecha lejana (ej: festival en 6 meses) nunca serían revisados
- ✅ Útil si: Hay deadline regulatorio (ej: aprobar antes de X fecha)

**Opción 2:** Priorizar por promoter_tier (verificados VIP primero)
- ❌ Destruye el modelo de negocio ("cero ruido, solo tus intereses"). Amigusto NO favorece eventos de pago
- ❌ Promotores nuevos abandonarían la plataforma (tiempo de espera >1 semana)
- ✅ Útil en: Plataformas premium donde "pagar más = mejor servicio"

**Opción 3:** Ordenar por score de ML (probabilidad de ser aprobado)
- ❌ Complejidad técnica extrema. Requiere modelo ML entrenado con histórico de aprobaciones
- ❌ Sesgo: Modelo favorecería eventos similares a los ya aprobados (círculo vicioso)
- ✅ Útil para: Optimizar throughput cuando volumen es >10,000 eventos/día

#### ¿Por qué TTL de solo 1 minuto para pending_events?

**Decisión:** Cachear cola de eventos pendientes con `TTL = 60 segundos`, el más corto de todos los cachés de la plataforma.

**Razones:**
1. **Cola Cambia Constantemente**: Cada aprobación/rechazo/submit modifica la cola. Con ~50 eventos/día, cambios cada ~30 minutos
2. **Admins Trabajan en Tiempo Real**: Curador aprueba evento #1 → debe ver inmediatamente evento #2 siguiente
3. **Invalidación Manual Complementaria**: Aunque invalidamos caché al aprobar/rechazar/submit, el TTL corto es safety net si falla invalidación
4. **Query Muy Rápida**: `SELECT WHERE status = 'PENDING_REVIEW' ORDER BY created_at LIMIT 10` con índice toma <10ms

**Alternativa Descartada:** TTL largo (5-10 minutos)
- ❌ Admin aprueba evento → sigue viéndolo en la cola durante 5 min → confusión
- ❌ Dos admins podrían revisar el mismo evento simultáneamente (race condition)
- ✅ TTL largo útil para: Datos que NO cambian frecuentemente (eventos APPROVED)

---

### 5.2 Aprobar Evento (Microservicios con Event-Driven Architecture)

**Flujo Técnico Crítico:**

```
Admin (Angular)
    ↓ 1. Revisa evento en detalle
    ↓ 2. Verifica calidad: título correcto, imagen apropiada, etc.
    ↓ 3. Click "Aprobar Evento"
    ↓ 4. POST https://api.amigusto.com/api/v1/events/{eventId}/approve
    ↓    Headers: Authorization: Bearer {accessToken}
API Gateway (:8080)
    ↓ 5. JWT Authentication Filter
    ↓    - Valida token JWT
    ↓    - Verifica role == ADMIN
    ↓    - Agrega header X-User-Id: {adminId}
    ↓ 6. Enruta a → lb://EVENT-SERVICE
Event Service (:8082)
    ↓ 7. EventController.approveEvent(eventId, @CurrentUser adminId)
    ↓ 8. Validaciones @PreAuthorize("hasRole('ADMIN')"):
    ↓    - Evento existe
    ↓    - Estado actual == PENDING_REVIEW
    ↓    - Usuario es ADMIN
    ↓ 9. EventService.approveEvent(eventId, adminId)
event_db (PostgreSQL)
    ↓ 10. @Transactional BEGIN
    ↓     UPDATE events
    ↓     SET status = 'APPROVED',
    ↓         reviewed_by = ?,        -- adminId
    ↓         reviewed_at = NOW(),
    ↓         published_at = NOW()    -- timestamp de publicación
    ↓     WHERE id = ? AND status = 'PENDING_REVIEW'
    ↓     Transaction COMMIT
Event Service
    ↓ 11. **CRÍTICO**: Invalida TODOS los cachés afectados
Redis (Shared Cache)
    ↓ 12. Invalidación en cascada:
    ↓     DEL amigusto:event:pending_events:*
    ↓       (cola de admin ya no incluye este evento)
    ↓
    ↓     DEL amigusto:event:discover:{city}:*
    ↓       (evento ahora visible en ciudad correspondiente)
    ↓
    ↓     DEL amigusto:event:detail:{eventId}
    ↓       (si alguien vio el detalle antes, actualizar estado)
Event Service
    ↓ 13. 📨 PUBLICA EVENTO A RABBITMQ
    ↓     Exchange: event.events (Topic)
    ↓     Routing Key: event.approved
    ↓     Payload: {
    ↓       eventId,
    ↓       promoterId,
    ↓       title,
    ↓       city,
    ↓       startDate,
    ↓       approvedAt,
    ↓       approvedBy: adminId
    ↓     }
RabbitMQ
    ↓ 14. Enruta mensaje a queues:
    ↓     - notification-service.event.approved
Notification Service (:8085)
    ↓ 15. 📨 CONSUME EVENTO event.approved
    ↓ 16. 🔗 FEIGN CALL a Promoter Service
    ↓     - promoterClient.getPromoter(promoterId)
Promoter Service (:8084)
    ↓ 17. Obtiene datos del promotor
promoter_db
    ↓ 18. SELECT * FROM promoters WHERE id = ?
Promoter Service
    ↓ 19. Retorna PromoterResponse { email, organizationName }
Notification Service
    ↓ 20. Envía email al promotor (Spring Mail):
    ↓     - To: promoter.email
    ↓     - Subject: "Tu evento ha sido aprobado"
    ↓     - Body: Template Thymeleaf con link al evento
notification_db (MongoDB)
    ↓ 21. db.email_logs.insertOne({
    ↓       to: promoter.email,
    ↓       subject: "Tu evento ha sido aprobado",
    ↓       template: "event-approved",
    ↓       eventId: eventId,
    ↓       status: "SENT",
    ↓       sentAt: ISODate()
    ↓     })
Notification Service
    ↓ 22. Actualiza métricas del promotor (aprobado++)
Promoter Service
    ↓ 23. 📨 Recibe notificación de Notification Service
promoter_db
    ↓ 24. UPDATE promoters
    ↓     SET approved_events = approved_events + 1
    ↓     WHERE id = 'promoter-uuid'
Event Service
    ↓ 25. Retorna EventResponse con status = APPROVED al API Gateway
API Gateway
    ↓ 26. Retorna response al cliente
Admin (Angular)
    ↓ 27. Muestra Snackbar: "Evento aprobado y publicado"
    ↓ 28. Remueve evento de la cola (actualiza lista)
Apps Móviles (Usuarios)
    ↓ 29. Próximo refresh del feed (o pull-to-refresh)
    ↓     → Evento aparece en resultados de descubrimiento
```

#### ¿Por qué invalidar múltiples cachés en cascada?

**Decisión:** Invalidar 3 tipos de cachés diferentes al aprobar un evento: `pending_events:*`, `discover:{city}:*`, y `event:{eventId}`.

**Razones:**
1. **pending_events:*** - La cola de admin YA NO incluye este evento. Si no invalidamos, admin sigue viendo el evento como pendiente
2. **discover:{city}:*** - Todos los usuarios en esa ciudad DEBEN ver el nuevo evento inmediatamente en su feed. Es la promesa de valor de la plataforma
3. **event:{eventId}** - Si alguien vio el detalle mientras estaba PENDING_REVIEW, el caché mostraría status incorrecto
4. **Consistency > Performance**: Preferimos invalidar múltiples cachés (todos se reconstruyen en ~100ms) vs mostrar datos incorrectos

**Alternativa Descartada:** Invalidar solo pending_events
- ❌ Usuario abre app 2 minutos después de aprobación → evento NO aparece (debe esperar TTL de 5 min)
- ❌ Promotor comparte link del evento → muestra status "Pendiente de revisión" → mala imagen
- ✅ Invalidación selectiva útil si: Los datos NO están relacionados entre sí

#### ¿Por qué RabbitMQ event.approved en lugar de Feign síncrono?

**Decisión:** Event Service publica mensaje `event.approved` a RabbitMQ en lugar de llamar directamente a Notification Service vía Feign.

**Razones:**
1. **Desacoplamiento Total**: Event Service NO sabe si Notification Service existe o está disponible. Añadir nuevos consumidores (ej: Analytics Service) NO requiere cambiar Event Service
2. **Resiliencia**: Si Notification Service está caído, RabbitMQ guarda el mensaje en cola y reintenta cuando vuelva (durabilidad garantizada)
3. **Performance**: Aprobar evento retorna en ~150ms (UPDATE + invalidar caché + publish). NO espera a enviar email (~2000ms SMTP)
4. **Idempotencia**: Si Notification Service procesa el mismo evento 2 veces (retry), debe detectarlo vía eventId y NO enviar email duplicado

**Alternativa Descartada:** Feign síncrono a Notification Service
- ❌ Si Notification Service está lento (SMTP timeout), aprobar evento se vuelve lento (latencia p95: 5000ms vs 150ms)
- ❌ Si falla envío de email, ¿la aprobación debe fallar? NO (separación de concerns)
- ❌ Circuit breaker ayuda pero NO resuelve el problema fundamental de acoplamiento
- ✅ Feign síncrono solo si: Necesitamos respuesta inmediata del servicio downstream (ej: validar stock)

#### ¿Por qué Notification Service usa Feign para obtener datos del promotor?

**Decisión:** Notification Service hace llamada síncrona `promoterClient.getPromoter(promoterId)` vía Feign para obtener email del promotor.

**Razones:**
1. **Payload Ligero**: El evento `event.approved` NO incluye todos los datos del promotor (email, nombre, etc.). Solo incluye promoterId (UUID)
2. **Single Source of Truth**: Promoter Service es la fuente autoritativa de datos de promotores. Replicar email en múltiples servicios causa inconsistencias
3. **Datos Actualizados**: Si promotor cambió su email ayer, Notification Service obtiene el email ACTUAL, no uno desactualizado en el evento
4. **Validación Bloqueante Aceptable**: Enviar email a dirección incorrecta es peor que fallar el envío y reintentar

**Alternativa Descartada:** Incluir todos los datos en el mensaje RabbitMQ
- ❌ Payload grande: `event.approved` pasaría de ~200 bytes a ~2KB (10x más grande)
- ❌ Datos desactualizados: Email del promotor podría haber cambiado entre creación del evento y aprobación
- ❌ Acoplamiento: Cualquier cambio en Promoter model requiere cambiar TODOS los servicios que consumen eventos
- ✅ Payload completo útil si: Los datos son inmutables (ej: eventId, title nunca cambian)

#### ¿Por qué MongoDB para notification_db en lugar de PostgreSQL?

**Decisión:** Almacenar logs de emails enviados en MongoDB en lugar de PostgreSQL.

**Razones:**
1. **Alta Escritura, Baja Lectura**: Ratio escrituras:lecturas ~95:5. Solo escribimos logs, raramente los consultamos (solo para debugging)
2. **Esquema Flexible**: Emails tienen metadata variable (ej: email de aprobación tiene eventId, email de bienvenida tiene userId, email de newsletter tiene campaignId)
3. **TTL Nativo**: MongoDB puede eliminar documentos automáticamente después de 90 días con índice TTL (`db.email_logs.createIndex({sentAt: 1}, {expireAfterSeconds: 7776000})`)
4. **Escalabilidad Horizontal**: MongoDB escala mejor para writes intensivos (sharding por fecha)

**Alternativa Descartada:** PostgreSQL con tabla email_logs
- ❌ Esquema rígido: Tendríamos que usar JSONB para metadata variable (query performance afectado)
- ❌ TTL manual: Requiere CRON job para `DELETE FROM email_logs WHERE sent_at < NOW() - INTERVAL '90 days'` (locks de tabla)
- ❌ Writes intensivos: PostgreSQL optimizado para ACID transactions, NO para append-only logs
- ✅ PostgreSQL útil para: Datos transaccionales con relaciones (events, users, saved_events)

---

### 5.3 Rechazar Evento

**Flujo Técnico:**

```
Admin (Angular)
    ↓ 1. Revisa evento, detecta problema (ej: imagen inapropiada)
    ↓ 2. Click "Rechazar Evento"
    ↓ 3. Modal aparece: "Razón del rechazo"
    ↓ 4. Admin escribe razón:
    ↓    "La imagen no corresponde al evento. Por favor sube la imagen correcta."
    ↓ 5. POST /api/v1/events/{eventId}/reject
    ↓    Query Params: reason={razón}
    ↓    Headers: Authorization: Bearer {accessToken}
Backend
    ↓ 6. EventController.rejectEvent(eventId, reason, @CurrentUser adminId)
    ↓ 7. Validaciones: evento en PENDING_REVIEW, razón no vacía
    ↓ 8. EventService.rejectEvent()
PostgreSQL
    ↓ 9. Transaction:
    ↓    UPDATE events
    ↓    SET status = 'REJECTED',
    ↓        reviewed_by = ?,
    ↓        reviewed_at = NOW(),
    ↓        rejection_reason = ?
    ↓    WHERE id = ? AND status = 'PENDING_REVIEW'
Backend
    ↓ 10. Invalida caché de cola de admin
Redis
    ↓ 11. DEL pending_events:*
Backend
    ↓ 12. Envía email al promotor:
    ↓     "Tu evento fue rechazado. Razón: {razón}. Puedes editarlo y reenviar."
    ↓ 13. Retorna EventResponse con status = REJECTED
Admin (Angular)
    ↓ 14. Snackbar: "Evento rechazado"
Promotor (Angular)
    ↓ 15. Ve evento con badge "Rechazado" (rojo)
    ↓ 16. Puede ver razón, editar evento, y reenviar
```

#### ¿Por qué almacenar rejection_reason en la tabla events?

**Decisión:** Agregar columna `rejection_reason TEXT NULL` en tabla `events` para almacenar la razón del rechazo escrita por el admin.

**Razones:**
1. **Transparencia con Promotor**: El promotor puede ver exactamente POR QUÉ su evento fue rechazado (ej: "Imagen no corresponde al evento"). Sin esto, recibiría solo "Rechazado" sin contexto
2. **Mejora de Calidad**: Promotor sabe qué corregir antes de reenviar. Sin razón, tendría que adivinar el problema (trial & error, frustración)
3. **Reducción de Tickets de Soporte**: Sin rejection_reason, promotores contactarían soporte masivamente preguntando "¿por qué rechazaron mi evento?"
4. **Auditabilía y Compliance**: Historial completo de decisiones de admins. Importante para disputas o análisis de patrones de rechazo

**Alternativa Descartada:** Solo email con razón, sin persistir en DB
- ❌ Promotor puede perder el email o eliminarlo accidentalmente → NO hay forma de recuperar la razón
- ❌ Si promotor accede desde dispositivo diferente (móvil → web), NO ve la razón
- ❌ Analytics imposibles: NO podemos analizar razones de rechazo más comunes para mejorar guidelines
- ✅ Solo email útil si: La razón es confidencial y NO debe persistir (ej: datos sensibles)

#### ¿Por qué validar rejection_reason no vacío?

**Decisión:** Validar que `@NotBlank String reason` en el controller antes de permitir rechazo.

**Razones:**
1. **Forzar Accountability**: Admin DEBE explicar su decisión. Rechazos sin razón serían arbitrarios y destruirían confianza de promotores
2. **Prevenir Errores**: Admin podría clickear "Rechazar" accidentalmente sin escribir razón. Validación previene esto
3. **Calidad de Feedback**: Razones vacías o genéricas ("No") no ayudan al promotor. UX debe requerir mínimo ~20 caracteres

**Alternativa Descartada:** Permitir rechazo sin razón
- ❌ Promotores percibirían sistema como injusto o corrupto ("¿por qué rechazaron mi evento sin explicación?")
- ❌ Legal risk: En algunas jurisdicciones, decisiones automatizadas/opacas pueden ser cuestionadas legalmente
- ✅ Sin validación útil si: Rechazo es automático por reglas objetivas (ej: evento pasado, imagen NSFW detectada por ML)

#### ¿Por qué permitir re-envío después de rechazo?

**Decisión:** Eventos REJECTED pueden editarse y cambiar a DRAFT → PENDING_REVIEW nuevamente (flujo cíclico permitido).

**Razones:**
1. **Segunda Oportunidad**: Mayoría de rechazos son por errores corregibles (imagen incorrecta, descripción poco clara). Bloquear permanentemente sería excesivamente punitivo
2. **Incentivo a Mejorar Calidad**: Promotor invierte tiempo corrigiendo vs abandonar la plataforma frustrado
3. **Reducción de Eventos Duplicados**: Sin re-envío, promotor crearía evento completamente nuevo con mismos datos (spam de la cola)
4. **Aprendizaje**: Promotor aprende los estándares de calidad de la plataforma con cada iteración

**Alternativa Descartada:** Rechazo permanente (evento bloqueado forever)
- ❌ Promotores abandonarían plataforma después de 1 rechazo (churn rate alto)
- ❌ Crearía incentivo a crear múltiples cuentas para "esquivar" rechazos
- ❌ Pérdida de ingresos: Menos eventos aprobados = menos valor para usuarios = menos engagement
- ✅ Bloqueo permanente útil si: Violación grave (spam, contenido ilegal, fraude comprobado)

---

## 6. Infraestructura de Datos

### 6.1 Esquema PostgreSQL Optimizado

**Tablas Principales:**

```sql
-- Tabla de Usuarios (todos los roles)
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255),  -- BCrypt
    name VARCHAR(255) NOT NULL,
    role user_role DEFAULT 'CONSUMER',  -- ENUM

    -- Datos de consumidor
    city VARCHAR(100),
    lat DECIMAL(10, 8),
    lng DECIMAL(11, 8),

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role ON users(role);

-- Tabla de Promotores
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

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_promoters_status ON promoters(status);

-- Tabla de Gustos (categorías)
CREATE TABLE gustos (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) UNIQUE NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    icon VARCHAR(50),  -- Emoji: 🎵, 🎭, etc.
    category VARCHAR(50),  -- Arte, Deporte, Gastronomía, etc.

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_gustos_slug ON gustos(slug);

-- Relación Muchos a Muchos: User ↔ Gustos
CREATE TABLE user_gustos (
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    gusto_id UUID REFERENCES gustos(id) ON DELETE CASCADE,
    PRIMARY KEY (user_id, gusto_id)
);

-- Tabla de Eventos (entidad central)
CREATE TABLE events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    promoter_id UUID NOT NULL REFERENCES promoters(id) ON DELETE CASCADE,

    -- Información básica
    title VARCHAR(255) NOT NULL,
    slug VARCHAR(255) UNIQUE NOT NULL,
    description TEXT NOT NULL,
    image_url VARCHAR(500) NOT NULL,

    -- Fechas
    start_date TIMESTAMP NOT NULL,
    end_date TIMESTAMP,
    timezone VARCHAR(50) DEFAULT 'Europe/Madrid',

    -- Ubicación geográfica
    venue_name VARCHAR(255) NOT NULL,
    venue_address VARCHAR(500) NOT NULL,
    city VARCHAR(100) NOT NULL,
    country VARCHAR(100) DEFAULT 'España',
    lat DECIMAL(10, 8) NOT NULL,
    lng DECIMAL(11, 8) NOT NULL,

    -- Precio
    is_free BOOLEAN DEFAULT false,
    price DECIMAL(10, 2),
    currency VARCHAR(10) DEFAULT 'EUR',
    ticket_url VARCHAR(500),

    -- Estado y curación
    status event_status DEFAULT 'DRAFT',
    reviewed_by UUID,
    reviewed_at TIMESTAMP,
    rejection_reason TEXT,
    published_at TIMESTAMP,

    -- Métricas
    view_count INT DEFAULT 0,
    save_count INT DEFAULT 0,
    share_count INT DEFAULT 0,

    -- Metadata flexible
    metadata JSONB,

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Índices para optimización
CREATE INDEX idx_events_status ON events(status);
CREATE INDEX idx_events_city ON events(city);
CREATE INDEX idx_events_start_date ON events(start_date);
CREATE INDEX idx_events_promoter ON events(promoter_id);
CREATE INDEX idx_events_slug ON events(slug);

-- Índice espacial PostGIS (CRÍTICO para geolocalización)
CREATE INDEX idx_events_location ON events USING GIST (ll_to_earth(lat, lng));

-- Relación Muchos a Muchos: Event ↔ Gustos
CREATE TABLE event_gustos (
    event_id UUID REFERENCES events(id) ON DELETE CASCADE,
    gusto_id UUID REFERENCES gustos(id) ON DELETE CASCADE,
    PRIMARY KEY (event_id, gusto_id)
);

CREATE INDEX idx_event_gustos_event ON event_gustos(event_id);
CREATE INDEX idx_event_gustos_gusto ON event_gustos(gusto_id);

-- Tabla de Eventos Guardados
CREATE TABLE saved_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    event_id UUID NOT NULL REFERENCES events(id) ON DELETE CASCADE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    UNIQUE(user_id, event_id)  -- Usuario no puede guardar mismo evento dos veces
);

CREATE INDEX idx_saved_events_user ON saved_events(user_id);
CREATE INDEX idx_saved_events_event ON saved_events(event_id);
```

**Justificación de Índices:**

| Índice | Justificación |
|--------|---------------|
| `idx_events_status` | Filtrado de eventos APPROVED en queries de descubrimiento |
| `idx_events_city` | Filtrado por ciudad (Madrid, Barcelona, etc.) |
| `idx_events_start_date` | Ordenamiento por fecha, excluir eventos pasados |
| `idx_events_location (GIST)` | **CRÍTICO**: Búsquedas geoespaciales con PostGIS |
| `idx_event_gustos_gusto` | JOIN optimizado en queries de descubrimiento |

---

### 6.2 Estrategia de Caché Redis

**Estructura de Keys:**

```
# Eventos descubiertos
discover:{city}:{gustoIds_hash}:page{N}
Ejemplo: discover:Madrid:abc123:page0
TTL: 5 minutos

# Detalle de evento individual
event:{eventId}
Ejemplo: event:550e8400-e29b-41d4-a716-446655440000
TTL: 10 minutos

# Lista de gustos (catálogo)
gustos:all
TTL: 24 horas

# Eventos pendientes de revisión (admin)
pending_events:page{N}
Ejemplo: pending_events:page0
TTL: 1 minuto

# Eventos guardados del usuario
saved_events:{userId}:page{N}
Ejemplo: saved_events:550e8400-e29b-41d4-a716-446655440000:page0
TTL: 2 minutos

# Refresh Tokens de autenticación
refresh_token:{userId}
Ejemplo: refresh_token:550e8400-e29b-41d4-a716-446655440000
TTL: 7 días
```

**Configuración Redis:**

```yaml
# application.yml
spring:
  cache:
    type: redis
    redis:
      time-to-live: 300000  # 5 min por defecto
      cache-null-values: false  # No cachear valores null
      key-prefix: "amigusto:"
      use-key-prefix: true

  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: ${REDIS_PORT:6379}
      password: ${REDIS_PASSWORD}
      database: 0
      timeout: 2000ms
      lettuce:
        pool:
          max-active: 20   # Máx conexiones activas
          max-idle: 10     # Máx conexiones idle
          min-idle: 5      # Mín conexiones idle
          max-wait: 1000ms # Máx espera por conexión
```

---

## 7. Patrones de Caché y Optimización

### 7.1 Cache-Aside Pattern

**Patrón más usado en Amigusto:**

```
1. App solicita datos
2. Backend verifica Redis
3. Si HIT → retorna desde caché
4. Si MISS → consulta DB → guarda en caché → retorna
```

#### ¿Por qué Cache-Aside en lugar de Write-Through o Write-Behind?

**Decisión:** Usar patrón Cache-Aside (también llamado Lazy Loading) como estrategia principal de caché en toda la aplicación.

**Razones:**
1. **Simplicidad de Implementación**: Spring Cache con `@Cacheable` maneja automáticamente el patrón. NO requiere lógica manual de cache management
2. **Datos Solo en Cache Si Se Usan**: Eventos que NUNCA se consultan NO ocupan espacio en Redis. Con Write-Through, TODO se cachea (desperdicio de memoria)
3. **Resiliencia a Fallos**: Si Redis falla completamente, la app sigue funcionando (más lento, pero funcional). PostgreSQL es source of truth
4. **Menor Complejidad en Writes**: Al crear/actualizar evento, solo invalidamos caché (`@CacheEvict`). NO necesitamos actualizar Redis Y PostgreSQL simultáneamente

**Alternativas Descartadas:**

**Write-Through:** Cada escritura actualiza DB Y caché simultáneamente
- ❌ Complejidad: Crear evento requiere INSERT en PostgreSQL + SET en Redis en MISMA transacción (difícil de garantizar atomicidad)
- ❌ Latencia de Escritura: POST /events tomaría ~50ms más (UPDATE cache + DB vs solo DB)
- ❌ Desperdicio de Memoria: Eventos DRAFT/REJECTED se cachean aunque NUNCA se consulten en discover feed
- ✅ Útil si: Lecturas >> Escrituras (ratio 1000:1) Y TODAS las escrituras se leen después (ej: post en red social)

**Write-Behind:** Escribir primero en caché, sincronizar a DB después (asíncrono)
- ❌ Riesgo de Pérdida de Datos: Si Redis crashea antes de sincronizar a PostgreSQL, eventos se pierden (inaceptable)
- ❌ Complejidad Extrema: Requiere job de sincronización, manejo de conflictos, reintentos, dead letter queue
- ❌ Consistency Issues: Caché y DB desincronizados durante ventana de tiempo (eventual consistency NO aceptable para eventos)
- ✅ Útil si: Writes intensivos (>10,000/sec) donde DB es cuello de botella Y pérdida de datos es tolerable (ej: analytics, logs)

#### ¿Por qué clave compuesta con city + gustoIds.hashCode() + page?

**Decisión:** Usar clave de caché compuesta: `discover-events:{city}-{gustoIds.hashCode()}-page{pageNumber}`.

**Razones:**
1. **Granularidad Óptima**: Dos usuarios en Madrid con mismos gustos (Música + Teatro) comparten caché. Diferentes gustos = caché diferente
2. **Evitar Colisiones**: User A en Madrid página 0 NO sobrescribe User B en Barcelona página 0 (city diferencia)
3. **Paginación Independiente**: Página 0 cacheada NO invalida páginas 1, 2, 3. Solo se reconstruye la página solicitada
4. **hashCode() Eficiente**: Convertir List<UUID> a integer para key compacta vs serializar lista completa

**Alternativa Descartada:** Clave única global `discover-events`
- ❌ Todos los usuarios comparten MISMO caché → Usuario en Barcelona ve eventos de Madrid
- ❌ Primera búsqueda cachea datos, todas las demás búsquedas retornan esos datos (completamente incorrecto)
- ✅ Clave global útil si: Datos son idénticos para todos los usuarios (ej: lista de países)

### 7.2 Cache Warming

**Concepto:** Precargar datos en Redis al iniciar la aplicación, ANTES de recibir requests de usuarios.

#### ¿Por qué pre-cachear gustos y ciudades al startup?

**Decisión:** Ejecutar `gustoService.getAllGustos()` y `cityService.getAllActiveCities()` en un listener de `ContextRefreshedEvent` (cuando Spring Boot termina de iniciar).

**Razones:**
1. **Eliminar Cold Start Penalty**: Primer usuario que abre la app obtiene gustos en ~1ms (cache hit) vs ~20ms (query DB). Mejor first impression
2. **Datos Altamente Predecibles**: TODOS los usuarios nuevos cargan gustos en onboarding. Probabilidad de uso = 100%
3. **Dataset Pequeño y Estático**: ~50 gustos + ~100 ciudades activas = ~5KB total. Precargar es casi gratis en memoria
4. **Reducir Carga Inicial**: Al momento de lanzamiento (1000 usuarios simultáneos), evitamos 1000 queries a PostgreSQL en primer minuto

**Alternativa Descartada:** Lazy loading sin warming
- ❌ Primeros 10-100 usuarios experimentan latencia alta (20-30ms) cargando gustos
- ❌ Spike de queries a PostgreSQL al momento de lanzamiento (potencial sobrecarga)
- ✅ Sin warming útil si: Datos son impredecibles (NO sabemos qué se va a consultar primero)

#### ¿Qué NO debemos pre-cachear?

**Decisión:** NO pre-cachear eventos, usuarios, o eventos guardados.

**Razones:**
1. **discover-events:** Requiere parámetros variables (city, gustoIds, lat, lng). NO sabemos qué combinación pre-cachear
2. **saved_events:{userId}:** Requiere userId específico. Pre-cachear saved_events de usuario aleatorio es inútil
3. **Desperdicio de Memoria**: Pre-cachear 10,000 eventos (varios MB) cuando solo 100 serán consultados es ineficiente
4. **TTL Corto**: saved_events tiene TTL 2min. Pre-cachear algo que expira en 2min NO tiene sentido

**Regla General:** Solo pre-cachear datos que:
- ✅ Son consultados por >80% de usuarios
- ✅ Son estáticos o semi-estáticos (TTL >1 hora)
- ✅ Son pequeños (<100KB)
- ❌ Requieren parámetros variables
- ❌ Son específicos de usuario

### 7.3 Optimización de Queries N+1

**Problema:** Hibernate lazy loading puede generar N+1 queries (1 query principal + N queries adicionales para relaciones).

#### ¿Por qué usar JOIN FETCH en lugar de lazy loading?

**Decisión:** Usar `LEFT JOIN FETCH e.gustos` en queries JPQL cuando sabemos que necesitaremos las relaciones.

**Razones:**
1. **Performance Crítico**: N+1 problem con 20 eventos = 21 queries (1 + 20) vs 1 query con JOIN. Latencia: ~400ms vs ~40ms (10x más rápido)
2. **Reducción de Round Trips**: 1 viaje a PostgreSQL vs 21 viajes. Latencia de red se multiplica
3. **Carga Eager Selectiva**: Solo traemos relaciones cuando las necesitamos (discover feed), NO siempre (evita over-fetching)
4. **Cacheable**: Query con JOIN puede cachearse completa. Lazy loading NO se beneficia de caché

**Ejemplo Concreto:**

**Sin JOIN FETCH (N+1 problem):**
```sql
-- Query 1: Obtener eventos
SELECT * FROM events WHERE status = 'APPROVED' LIMIT 20;

-- Queries 2-21: Para cada evento, obtener sus gustos (LAZY LOADING)
SELECT * FROM event_gustos WHERE event_id = 'uuid-1';
SELECT * FROM event_gustos WHERE event_id = 'uuid-2';
...
SELECT * FROM event_gustos WHERE event_id = 'uuid-20';

-- Total: 21 queries, ~400ms
```

**Con JOIN FETCH:**
```sql
-- Query única con JOIN
SELECT DISTINCT e.*, g.*
FROM events e
LEFT JOIN event_gustos eg ON e.id = eg.event_id
LEFT JOIN gustos g ON eg.gusto_id = g.id
WHERE e.status = 'APPROVED'
LIMIT 20;

-- Total: 1 query, ~40ms
```

**Alternativa Descartada:** Lazy loading siempre
- ❌ Desarrollador debe recordar hacer lazy loading trigger (`event.getGustos().size()`) DENTRO de transacción
- ❌ Si accedes a gustos FUERA de transacción → `LazyInitializationException` (bug común)
- ❌ Performance horrible en loops (20 eventos = 20 queries adicionales)
- ✅ Lazy loading útil si: NO sabemos si usaremos la relación (ej: admin ve evento sin necesitar gustos)

#### ¿Por qué DISTINCT en JOIN FETCH?

**Decisión:** Usar `SELECT DISTINCT e` cuando hacemos JOIN FETCH con colecciones (@OneToMany, @ManyToMany).

**Razones:**
1. **Evitar Duplicados**: JOIN con event_gustos crea 1 fila por gusto. Evento con 3 gustos aparece 3 veces en result set
2. **Hibernate Deduplicación**: DISTINCT le dice a Hibernate que deduplique entidades Event (mantiene 1 Event con lista de 3 gustos)
3. **Sin DISTINCT**: Evento con 3 gustos se retornaría 3 veces → lista de 20 eventos se convierte en lista de 60 (incorrecta)

**Alternativa Descartada:** Sin DISTINCT
- ❌ Result set contiene duplicados: `[Event1, Event1, Event1, Event2, Event2, ...]`
- ❌ Requiere deduplicación manual en Java: `events.stream().distinct().collect(...)`
- ✅ Sin DISTINCT solo si: JOIN es @ManyToOne (NO genera duplicados)

---

## 8. Seguridad y Autenticación

### 8.1 Flujo JWT Completo

**Generación de Token:**

```java
@Component
@RequiredArgsConstructor
public class JwtTokenProvider {

    @Value("${jwt.secret}")
    private String jwtSecret;

    @Value("${jwt.access-token-expiration}")
    private long accessTokenExpiration;  // 15 minutos

    @Value("${jwt.refresh-token-expiration}")
    private long refreshTokenExpiration;  // 7 días

    public String generateAccessToken(User user) {
        Date now = new Date();
        Date expiryDate = new Date(now.getTime() + accessTokenExpiration);

        return Jwts.builder()
            .setSubject(user.getId().toString())
            .claim("email", user.getEmail())
            .claim("role", user.getRole().name())
            .setIssuedAt(now)
            .setExpiration(expiryDate)
            .signWith(SignatureAlgorithm.HS512, jwtSecret)
            .compact();
    }

    public String generateRefreshToken(User user) {
        Date now = new Date();
        Date expiryDate = new Date(now.getTime() + refreshTokenExpiration);

        return Jwts.builder()
            .setSubject(user.getId().toString())
            .setIssuedAt(now)
            .setExpiration(expiryDate)
            .signWith(SignatureAlgorithm.HS512, jwtSecret)
            .compact();
    }

    public UUID getUserIdFromToken(String token) {
        Claims claims = Jwts.parser()
            .setSigningKey(jwtSecret)
            .parseClaimsJws(token)
            .getBody();

        return UUID.fromString(claims.getSubject());
    }

    public boolean validateToken(String token) {
        try {
            Jwts.parser().setSigningKey(jwtSecret).parseClaimsJws(token);
            return true;
        } catch (JwtException | IllegalArgumentException e) {
            return false;
        }
    }
}
```

**Filtro de Autenticación:**

```java
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtTokenProvider tokenProvider;
    private final UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(
        HttpServletRequest request,
        HttpServletResponse response,
        FilterChain filterChain
    ) throws ServletException, IOException {

        try {
            String jwt = extractJwtFromRequest(request);

            if (jwt != null && tokenProvider.validateToken(jwt)) {
                UUID userId = tokenProvider.getUserIdFromToken(jwt);

                UserDetails userDetails = userDetailsService.loadUserByUsername(
                    userId.toString()
                );

                UsernamePasswordAuthenticationToken authentication =
                    new UsernamePasswordAuthenticationToken(
                        userDetails,
                        null,
                        userDetails.getAuthorities()
                    );

                SecurityContextHolder.getContext().setAuthentication(authentication);
            }
        } catch (Exception e) {
            log.error("Error autenticando usuario", e);
        }

        filterChain.doFilter(request, response);
    }

    private String extractJwtFromRequest(HttpServletRequest request) {
        String bearerToken = request.getHeader("Authorization");
        if (bearerToken != null && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7);
        }
        return null;
    }
}
```

### 8.2 Rate Limiting con Bucket4j

**Limitar peticiones por usuario:**

```java
@Component
public class RateLimitingInterceptor implements HandlerInterceptor {

    private final ConcurrentHashMap<String, Bucket> cache = new ConcurrentHashMap<>();

    @Override
    public boolean preHandle(
        HttpServletRequest request,
        HttpServletResponse response,
        Object handler
    ) throws Exception {

        String userId = extractUserIdFromToken(request);
        Bucket bucket = resolveBucket(userId);

        if (bucket.tryConsume(1)) {
            return true;  // Permitir request
        } else {
            response.setStatus(429);  // Too Many Requests
            response.getWriter().write("Rate limit exceeded");
            return false;
        }
    }

    private Bucket resolveBucket(String userId) {
        return cache.computeIfAbsent(userId, key -> {
            Bandwidth limit = Bandwidth.classic(
                100,  // 100 requests
                Refill.intervally(100, Duration.ofMinutes(1))  // por minuto
            );
            return Bucket.builder().addLimit(limit).build();
        });
    }
}
```

---

## 🎯 Conclusión

Este documento cubre **toda la arquitectura funcional de Amigusto** con:

✅ **13 funcionalidades detalladas** con flujos técnicos completos
✅ **Stack tecnológico justificado** (Spring Boot, PostgreSQL, Redis, S3)
✅ **Justificaciones técnicas** de cada decisión arquitectónica
✅ **Código real** de implementación
✅ **Estrategias de caché** y optimización
✅ **Patrones de seguridad** y autenticación

**Próximos Pasos:**
1. Implementar el backend siguiendo este documento
2. Configurar infraestructura (PostgreSQL + Redis + S3)
3. Desarrollar APIs según los flujos descritos
4. Implementar apps móviles consumiendo estas APIs

---

**Última actualización:** 2025-10-26
**Autor:** Arquitecto de Software Amigusto
