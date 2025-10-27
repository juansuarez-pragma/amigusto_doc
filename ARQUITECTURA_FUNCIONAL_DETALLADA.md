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
    ↓     Payload: { userId, email, name, role: "CONSUMER" }
RabbitMQ
    ↓ 17. Enruta mensaje a queues:
    ↓     - user-service.user.created
    ↓     - notification-service.user.created
User Service (:8083)
    ↓ 18. 📨 CONSUME EVENTO user.created
    ↓ 19. Crea Consumer en user_db
user_db (PostgreSQL)
    ↓ 20. INSERT INTO consumers (id, email, name)
    ↓     VALUES ('uuid', 'user@example.com', 'Juan')
Notification Service (:8085)
    ↓ 21. 📨 CONSUME EVENTO user.created
    ↓ 22. Envía email de bienvenida (Spring Mail)
    ↓ 23. Guarda log en notification_db
notification_db (MongoDB)
    ↓ 24. db.email_logs.insertOne({
    ↓       to: 'user@example.com',
    ↓       subject: 'Bienvenido a Amigusto',
    ↓       sentAt: ISODate()
    ↓     })
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

**Tecnologías Utilizadas:**

| Capa | Tecnología | Justificación |
|------|------------|---------------|
| **Validación Frontend** | iOS: Swift Validation, Android: Kotlin Validation | Reducir llamadas innecesarias al backend |
| **Validación Backend** | Jakarta Bean Validation (`@Valid`, `@NotBlank`, `@Email`) | Capa de seguridad adicional, validación consistente |
| **Hash de Contraseña** | BCrypt con factor 12 | Algoritmo diseñado para ser lento, resistente a ataques de fuerza bruta |
| **JWT** | io.jsonwebtoken (jjwt) | Estándar de la industria, stateless, fácil de validar |
| **Storage de Token Móvil** | iOS: Keychain, Android: EncryptedSharedPreferences | Almacenamiento seguro nativo del SO |

**Código Backend (Spring Boot):**

```java
@RestController
@RequestMapping("/api/v1/auth")
@RequiredArgsConstructor
public class AuthController {

    private final AuthService authService;

    @PostMapping("/register/consumer")
    public ResponseEntity<AuthResponse> registerConsumer(
        @Valid @RequestBody RegisterRequest request
    ) {
        AuthResponse response = authService.registerConsumer(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }
}

@Service
@RequiredArgsConstructor
@Transactional
public class AuthService {

    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;  // BCrypt
    private final JwtTokenProvider jwtTokenProvider;
    private final RedisTemplate<String, String> redisTemplate;

    public AuthResponse registerConsumer(RegisterRequest request) {
        // Verificar email único
        if (userRepository.existsByEmail(request.getEmail())) {
            throw new DuplicateEmailException("Email ya registrado");
        }

        // Crear usuario
        User user = User.builder()
            .id(UUID.randomUUID())
            .email(request.getEmail())
            .passwordHash(passwordEncoder.encode(request.getPassword()))
            .name(request.getName())
            .role(UserRole.CONSUMER)
            .createdAt(LocalDateTime.now())
            .build();

        userRepository.save(user);

        // Generar tokens
        String accessToken = jwtTokenProvider.generateAccessToken(user);
        String refreshToken = jwtTokenProvider.generateRefreshToken(user);

        // Guardar refresh token en Redis (7 días)
        String redisKey = "refresh_token:" + user.getId();
        redisTemplate.opsForValue().set(
            redisKey,
            refreshToken,
            7,
            TimeUnit.DAYS
        );

        return AuthResponse.builder()
            .accessToken(accessToken)
            .refreshToken(refreshToken)
            .tokenType("Bearer")
            .expiresIn(900)  // 15 minutos
            .user(UserDto.from(user))
            .build();
    }
}
```

**¿Por qué Redis para Refresh Tokens?**

- **Revocación Instantánea**: Si un usuario cierra sesión, eliminamos el refresh token de Redis → token inválido
- **TTL Automático**: Redis elimina tokens expirados sin lógica adicional
- **Performance**: Validación de refresh token en sub-milisegundos vs query a PostgreSQL

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

**¿Por qué Cachear Gustos en Redis?**

- **Datos Semi-Estáticos**: Los gustos (categorías) cambian raramente
- **Alto Tráfico**: Cada usuario los carga al menos una vez
- **Reducir Carga en DB**: Evitar SELECT en cada onboarding

**Anotación de Caché (Spring Boot):**

```java
@Service
@RequiredArgsConstructor
public class GustoService {

    private final GustoRepository gustoRepository;

    @Cacheable(value = "gustos", key = "'all'")
    public List<GustoResponse> getAllGustos() {
        return gustoRepository.findAll()
            .stream()
            .map(GustoMapper::toResponse)
            .collect(Collectors.toList());
    }
}
```

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

**Tecnologías Clave:**

| Componente | Tecnología | Justificación |
|------------|------------|---------------|
| **Geolocalización** | PostGIS (Haversine formula) | Cálculo preciso de distancias geográficas en la DB |
| **Índices Espaciales** | PostgreSQL GIST Index | Búsquedas geoespaciales 100x más rápidas que sin índice |
| **Caché de Resultados** | Redis con TTL 5 min | Misma ciudad + gustos = mismo resultado por 5 min |
| **Paginación** | Spring Data Pageable | Evitar cargar todos los eventos en memoria |
| **Infinite Scroll** | iOS: onAppear, Android: LazyColumn | UX fluida, carga progresiva |

**¿Por qué TTL de 5 minutos en caché de descubrimiento?**

- **Balance entre Frescura y Performance**:
  - Eventos APROBADOS no cambian frecuentemente (quizás 1-2 por hora)
  - Usuarios toleran ver eventos con hasta 5 min de retraso
  - Reduce carga en DB en ~95% durante picos de tráfico

**Query Optimizada con PostGIS:**

```java
@Repository
public interface EventRepository extends JpaRepository<Event, UUID> {

    @Query(value = """
        SELECT DISTINCT e.*
        FROM events e
        INNER JOIN event_gustos eg ON e.id = eg.event_id
        WHERE e.status = 'APPROVED'
          AND e.city = :city
          AND e.start_date > CURRENT_TIMESTAMP
          AND eg.gusto_id IN :gustoIds
          AND (
              6371 * acos(
                  cos(radians(:lat)) * cos(radians(e.lat))
                  * cos(radians(e.lng) - radians(:lng))
                  + sin(radians(:lat)) * sin(radians(e.lat))
              )
          ) < :radiusKm
        ORDER BY e.start_date ASC
        """,
        nativeQuery = true)
    Page<Event> findByGustosAndCity(
        @Param("gustoIds") List<UUID> gustoIds,
        @Param("city") String city,
        @Param("lat") BigDecimal lat,
        @Param("lng") BigDecimal lng,
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

**¿Por qué no usar Redis para eventos guardados?**

- **Persistencia Crítica**: Los eventos guardados son datos importantes del usuario
- **Integridad Referencial**: FK a `users` y `events` garantiza consistencia
- **Métricas**: `save_count` se actualiza transaccionalmente
- **Redis es volátil**: Riesgo de pérdida de datos si Redis se reinicia

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

**¿Por qué TTL de solo 2 minutos?**

- **Datos Personales Dinámicos**: Usuario puede guardar/desguardar eventos frecuentemente
- **Lista Pequeña**: Típicamente <100 eventos por usuario, query rápida

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

**¿Por qué incrementar view_count de forma asíncrona?**

- **Performance**: No bloquear respuesta HTTP esperando UPDATE
- **Tolerancia a Errores**: Si falla el UPDATE, la vista del evento sigue funcionando
- **Métricas No-Críticas**: view_count es analítico, no afecta lógica de negocio

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

**Tecnologías Clave:**

| Componente | Tecnología | Justificación |
|------------|------------|---------------|
| **Frontend Form** | Angular Reactive Forms | Validación compleja, control fino de estado |
| **Autocomplete Ubicación** | Google Places Autocomplete API | UX mejorada, lat/lng automáticos |
| **Upload Imagen** | Angular HttpClient + Spring MultipartFile | Subida directa al backend, validación |
| **Storage** | AWS S3 / Cloudinary | Escalabilidad, CDN global |
| **Validación** | Jakarta Bean Validation | Validación declarativa, consistente |

**¿Por qué eventos inician como DRAFT?**

- **Flujo de Curación**: Promotor puede revisar antes de enviar a aprobación
- **Prevenir Spam**: No se publican eventos automáticamente
- **Edición Flexible**: Promotor puede editar sin restricciones

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

**¿Por qué invalidar caché de eventos pendientes?**

- **Cola Actualizada**: Admins deben ver el nuevo evento inmediatamente
- **TTL Corto No Es Suficiente**: Podría haber delay de hasta 1 minuto

**Código de Máquina de Estados:**

```java
@Service
@RequiredArgsConstructor
@Transactional
public class EventService {

    private final EventRepository eventRepository;

    @CacheEvict(value = "pending-events", allEntries = true)
    public EventResponse submitForReview(UUID eventId, UUID promoterId) {
        Event event = eventRepository.findById(eventId)
            .orElseThrow(() -> new ResourceNotFoundException("Evento no encontrado"));

        // Verificar ownership
        if (!event.getPromoter().getId().equals(promoterId)) {
            throw new UnauthorizedException("No autorizado para este evento");
        }

        // Verificar estado válido
        if (event.getStatus() != EventStatus.DRAFT) {
            throw new IllegalStateException(
                "Solo eventos en DRAFT pueden enviarse a revisión. " +
                "Estado actual: " + event.getStatus()
            );
        }

        // Cambiar estado
        event.setStatus(EventStatus.PENDING_REVIEW);
        Event updated = eventRepository.save(event);

        // [OPCIONAL] Notificar admins
        notificationService.notifyAdmins(
            "Nuevo evento pendiente de revisión: " + event.getTitle()
        );

        return EventMapper.toResponse(updated);
    }
}
```

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

**¿Por qué ordenar por created_at ASC (FIFO)?**

- **Fairness**: Primer evento enviado = primer evento revisado
- **Prevenir Starvation**: Eventos antiguos no se quedan sin revisar
- **SLA Predecible**: Promotores saben cuánto esperar (~24 horas)

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

**¿Por qué invalidar múltiples cachés?**

- **Consistencia**: El evento APROBADO debe ser visible inmediatamente
- **discover:{city}:*** : Todos los usuarios en esa ciudad deben ver el nuevo evento
- **pending_events:** : La cola de admin ya no incluye este evento
- **event:{eventId}** : Si alguien tenía el detalle cacheado con status=PENDING, debe actualizarse

**Código con Anotaciones de Caché:**

```java
@Service
@RequiredArgsConstructor
@Transactional
public class EventService {

    private final EventRepository eventRepository;
    private final EmailService emailService;

    @Caching(evict = {
        @CacheEvict(value = "discover-events", allEntries = true),
        @CacheEvict(value = "pending-events", allEntries = true),
        @CacheEvict(value = "events", key = "#eventId")
    })
    public EventResponse approveEvent(UUID eventId, UUID adminId) {
        Event event = eventRepository.findById(eventId)
            .orElseThrow(() -> new ResourceNotFoundException("Evento no encontrado"));

        // Validar estado
        if (event.getStatus() != EventStatus.PENDING_REVIEW) {
            throw new IllegalStateException(
                "Solo eventos en PENDING_REVIEW pueden aprobarse"
            );
        }

        // Cambiar estado
        event.setStatus(EventStatus.APPROVED);
        event.setReviewedBy(adminId);
        event.setReviewedAt(LocalDateTime.now());
        event.setPublishedAt(LocalDateTime.now());

        Event updated = eventRepository.save(event);

        // Notificar promotor (asíncrono)
        CompletableFuture.runAsync(() -> {
            emailService.sendEventApprovedEmail(
                event.getPromoter().getUser().getEmail(),
                event.getTitle()
            );
        });

        log.info("Evento {} aprobado por admin {}", eventId, adminId);

        return EventMapper.toResponse(updated);
    }
}
```

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

**¿Por qué almacenar rejection_reason?**

- **Transparencia**: Promotor entiende el problema
- **Mejora de Calidad**: Promotor puede corregir y reenviar
- **Auditabilidad**: Historial de decisiones de admin

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

**Implementación con Spring Cache:**

```java
@Service
@RequiredArgsConstructor
public class EventService {

    private final EventRepository eventRepository;

    // Cache-Aside: busca primero en caché, si no está, consulta DB y guarda
    @Cacheable(
        value = "discover-events",
        key = "#city + '-' + #gustoIds.hashCode() + '-page' + #pageable.pageNumber"
    )
    public Page<EventResponse> discoverEvents(
        BigDecimal lat,
        BigDecimal lng,
        List<UUID> gustoIds,
        String city,
        Pageable pageable
    ) {
        // Esta query solo se ejecuta si hay CACHE MISS
        Page<Event> events = eventRepository.findByGustosAndCity(
            gustoIds, city, lat, lng, 50.0, pageable
        );

        return events.map(EventMapper::toResponse);
    }

    // Invalidación de caché al aprobar evento
    @Caching(evict = {
        @CacheEvict(value = "discover-events", allEntries = true),
        @CacheEvict(value = "pending-events", allEntries = true)
    })
    public EventResponse approveEvent(UUID eventId, UUID adminId) {
        // ... lógica de aprobación ...
    }
}
```

### 7.2 Cache Warming

**Precargar caché al iniciar la aplicación:**

```java
@Component
@RequiredArgsConstructor
public class CacheWarmer implements ApplicationListener<ContextRefreshedEvent> {

    private final GustoService gustoService;
    private final CityService cityService;

    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        // Precargar datos estáticos en Redis
        gustoService.getAllGustos();  // → Redis: gustos:all
        cityService.getAllActiveCities();  // → Redis: cities:active

        log.info("Caché precargado con éxito");
    }
}
```

### 7.3 Optimización de Queries N+1

**Problema: Cargar eventos con sus gustos (N+1 queries)**

```java
// ❌ MAL: N+1 problem
List<Event> events = eventRepository.findAll();
events.forEach(event -> {
    event.getGustos().size();  // Lazy loading → 1 query por evento
});
```

**Solución: Fetch JOIN**

```java
// ✅ BIEN: Único query con JOIN
@Query("""
    SELECT DISTINCT e FROM Event e
    LEFT JOIN FETCH e.gustos
    LEFT JOIN FETCH e.promoter
    WHERE e.status = 'APPROVED'
    """)
List<Event> findAllWithRelations();
```

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
