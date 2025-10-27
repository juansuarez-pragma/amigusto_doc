# Arquitectura de Proyecto - Amigusto
## Estructura de Microservicios y Organización del Código

---

## 1. ESTRUCTURA GENERAL DE REPOSITORIOS

### 1.1 Enfoque Recomendado: Repositorios Separados por Componente

```
amigusto-microservices/     (Backend - Arquitectura Microservicios)
amigusto-ios/               (Swift + SwiftUI)
amigusto-android/           (Kotlin + Jetpack Compose)
amigusto-web/               (Angular - Portal + Admin)
config-repo/                (Configuración centralizada para Config Server)
```

**Justificación:**
- ✅ Backend de microservicios en un solo repo (monorepo de microservicios)
- ✅ Cada plataforma cliente independiente
- ✅ CI/CD específico por repositorio
- ✅ Config Server con repo Git separado
- ✅ Releases independientes
- ✅ Permisos granulares por repositorio

### 1.2 Alternativa: Monorepo Completo (No Recomendado)

Incluir backend + frontend en un solo repo es complejo porque mezcla tecnologías muy diferentes (Java, Swift, Kotlin, TypeScript) y requiere herramientas especializadas como Nx o Turborepo.

---

## 2. ESTRUCTURA DEL BACKEND (MICROSERVICIOS)

### 2.1 Estructura Completa del Repositorio

```
amigusto-microservices/
│
├── api-gateway/                    # API Gateway (Puerto 8080)
│   ├── src/
│   │   ├── main/
│   │   │   ├── java/com/amigusto/gateway/
│   │   │   │   ├── ApiGatewayApplication.java
│   │   │   │   ├── filter/
│   │   │   │   │   ├── AuthenticationFilter.java
│   │   │   │   │   └── LoggingFilter.java
│   │   │   │   ├── config/
│   │   │   │   │   ├── GatewayConfig.java
│   │   │   │   │   └── CorsConfig.java
│   │   │   │   └── util/
│   │   │   │       └── JwtUtil.java
│   │   │   └── resources/
│   │   │       ├── bootstrap.yml
│   │   │       └── application.yml
│   │   └── test/
│   ├── pom.xml
│   └── Dockerfile
│
├── eureka-server/                  # Service Discovery (Puerto 8761)
│   ├── src/
│   │   ├── main/
│   │   │   ├── java/com/amigusto/eureka/
│   │   │   │   └── EurekaServerApplication.java
│   │   │   └── resources/
│   │   │       └── application.yml
│   │   └── test/
│   ├── pom.xml
│   └── Dockerfile
│
├── config-server/                  # Config Server (Puerto 8888)
│   ├── src/
│   │   ├── main/
│   │   │   ├── java/com/amigusto/config/
│   │   │   │   └── ConfigServerApplication.java
│   │   │   └── resources/
│   │   │       └── application.yml
│   │   └── test/
│   ├── pom.xml
│   └── Dockerfile
│
├── auth-service/                   # Auth Service (Puerto 8081)
│   ├── src/
│   │   ├── main/
│   │   │   ├── java/com/amigusto/auth/
│   │   │   │   ├── AuthServiceApplication.java
│   │   │   │   │
│   │   │   │   ├── controller/
│   │   │   │   │   └── AuthController.java
│   │   │   │   │
│   │   │   │   ├── service/
│   │   │   │   │   ├── AuthService.java
│   │   │   │   │   ├── JwtTokenService.java
│   │   │   │   │   └── UserEventPublisher.java
│   │   │   │   │
│   │   │   │   ├── repository/
│   │   │   │   │   └── UserRepository.java
│   │   │   │   │
│   │   │   │   ├── model/
│   │   │   │   │   ├── entity/
│   │   │   │   │   │   └── User.java
│   │   │   │   │   ├── dto/
│   │   │   │   │   │   ├── request/
│   │   │   │   │   │   │   ├── LoginRequest.java
│   │   │   │   │   │   │   ├── RegisterConsumerRequest.java
│   │   │   │   │   │   │   └── RegisterPromoterRequest.java
│   │   │   │   │   │   └── response/
│   │   │   │   │   │       └── AuthResponse.java
│   │   │   │   │   ├── event/
│   │   │   │   │   │   └── UserCreatedEvent.java
│   │   │   │   │   └── enums/
│   │   │   │   │       └── UserRole.java
│   │   │   │   │
│   │   │   │   ├── security/
│   │   │   │   │   ├── JwtTokenProvider.java
│   │   │   │   │   ├── UserDetailsServiceImpl.java
│   │   │   │   │   └── SecurityConfig.java
│   │   │   │   │
│   │   │   │   ├── config/
│   │   │   │   │   ├── RabbitMQConfig.java
│   │   │   │   │   └── RedisConfig.java
│   │   │   │   │
│   │   │   │   └── exception/
│   │   │   │       ├── GlobalExceptionHandler.java
│   │   │   │       └── DuplicateEmailException.java
│   │   │   │
│   │   │   └── resources/
│   │   │       ├── bootstrap.yml
│   │   │       ├── application.yml
│   │   │       └── db/migration/
│   │   │           └── V1__create_users_table.sql
│   │   │
│   │   └── test/
│   │       └── java/com/amigusto/auth/
│   │           ├── controller/
│   │           └── service/
│   │
│   ├── pom.xml
│   └── Dockerfile
│
├── event-service/                  # Event Service (Puerto 8082)
│   ├── src/
│   │   ├── main/
│   │   │   ├── java/com/amigusto/event/
│   │   │   │   ├── EventServiceApplication.java
│   │   │   │   │
│   │   │   │   ├── controller/
│   │   │   │   │   ├── EventController.java
│   │   │   │   │   └── GustoController.java
│   │   │   │   │
│   │   │   │   ├── service/
│   │   │   │   │   ├── EventService.java
│   │   │   │   │   ├── GustoService.java
│   │   │   │   │   └── EventPublisher.java
│   │   │   │   │
│   │   │   │   ├── repository/
│   │   │   │   │   ├── EventRepository.java
│   │   │   │   │   └── GustoRepository.java
│   │   │   │   │
│   │   │   │   ├── model/
│   │   │   │   │   ├── entity/
│   │   │   │   │   │   ├── Event.java
│   │   │   │   │   │   └── Gusto.java
│   │   │   │   │   ├── dto/
│   │   │   │   │   │   ├── request/
│   │   │   │   │   │   │   ├── CreateEventRequest.java
│   │   │   │   │   │   │   └── EventFilterRequest.java
│   │   │   │   │   │   └── response/
│   │   │   │   │   │       ├── EventResponse.java
│   │   │   │   │   │       └── GustoResponse.java
│   │   │   │   │   ├── event/
│   │   │   │   │   │   ├── EventApprovedEvent.java
│   │   │   │   │   │   ├── EventRejectedEvent.java
│   │   │   │   │   │   └── EventCreatedEvent.java
│   │   │   │   │   └── enums/
│   │   │   │   │       └── EventStatus.java
│   │   │   │   │
│   │   │   │   ├── client/
│   │   │   │   │   ├── PromoterServiceClient.java     # Feign Client
│   │   │   │   │   ├── PromoterServiceFallback.java
│   │   │   │   │   ├── StorageServiceClient.java      # Feign Client
│   │   │   │   │   └── StorageServiceFallback.java
│   │   │   │   │
│   │   │   │   ├── listener/
│   │   │   │   │   └── EventSavedListener.java        # RabbitMQ Listener
│   │   │   │   │
│   │   │   │   ├── config/
│   │   │   │   │   ├── FeignConfig.java
│   │   │   │   │   ├── RabbitMQConfig.java
│   │   │   │   │   ├── RedisConfig.java
│   │   │   │   │   └── ResilienceConfig.java
│   │   │   │   │
│   │   │   │   └── exception/
│   │   │   │       └── GlobalExceptionHandler.java
│   │   │   │
│   │   │   └── resources/
│   │   │       ├── bootstrap.yml
│   │   │       ├── application.yml
│   │   │       └── db/migration/
│   │   │           ├── V1__create_events_table.sql
│   │   │           └── V2__create_gustos_table.sql
│   │   │
│   │   └── test/
│   │       └── java/com/amigusto/event/
│   │           ├── controller/
│   │           ├── service/
│   │           └── repository/
│   │
│   ├── pom.xml
│   └── Dockerfile
│
├── user-service/                   # User Service (Puerto 8083)
│   ├── src/
│   │   ├── main/
│   │   │   ├── java/com/amigusto/user/
│   │   │   │   ├── UserServiceApplication.java
│   │   │   │   │
│   │   │   │   ├── controller/
│   │   │   │   │   ├── UserController.java
│   │   │   │   │   └── SavedEventController.java
│   │   │   │   │
│   │   │   │   ├── service/
│   │   │   │   │   ├── UserService.java
│   │   │   │   │   ├── SavedEventService.java
│   │   │   │   │   └── EventSavedPublisher.java
│   │   │   │   │
│   │   │   │   ├── repository/
│   │   │   │   │   ├── ConsumerRepository.java
│   │   │   │   │   ├── UserGustoRepository.java
│   │   │   │   │   └── SavedEventRepository.java
│   │   │   │   │
│   │   │   │   ├── model/
│   │   │   │   │   ├── entity/
│   │   │   │   │   │   ├── Consumer.java
│   │   │   │   │   │   ├── UserGusto.java
│   │   │   │   │   │   └── SavedEvent.java
│   │   │   │   │   ├── dto/
│   │   │   │   │   ├── event/
│   │   │   │   │   │   ├── UserCreatedEvent.java
│   │   │   │   │   │   └── EventSavedEvent.java
│   │   │   │   │   └── enums/
│   │   │   │   │
│   │   │   │   ├── client/
│   │   │   │   │   ├── EventServiceClient.java        # Feign Client
│   │   │   │   │   └── EventServiceFallback.java
│   │   │   │   │
│   │   │   │   ├── listener/
│   │   │   │   │   └── UserCreatedListener.java       # RabbitMQ Listener
│   │   │   │   │
│   │   │   │   ├── config/
│   │   │   │   │   ├── FeignConfig.java
│   │   │   │   │   ├── RabbitMQConfig.java
│   │   │   │   │   └── ResilienceConfig.java
│   │   │   │   │
│   │   │   │   └── exception/
│   │   │   │
│   │   │   └── resources/
│   │   │       ├── bootstrap.yml
│   │   │       ├── application.yml
│   │   │       └── db/migration/
│   │   │           ├── V1__create_consumers_table.sql
│   │   │           └── V2__create_saved_events_table.sql
│   │   │
│   │   └── test/
│   │
│   ├── pom.xml
│   └── Dockerfile
│
├── promoter-service/               # Promoter Service (Puerto 8084)
│   ├── src/
│   │   ├── main/
│   │   │   ├── java/com/amigusto/promoter/
│   │   │   │   ├── PromoterServiceApplication.java
│   │   │   │   │
│   │   │   │   ├── controller/
│   │   │   │   │   └── PromoterController.java
│   │   │   │   │
│   │   │   │   ├── service/
│   │   │   │   │   └── PromoterService.java
│   │   │   │   │
│   │   │   │   ├── repository/
│   │   │   │   │   └── PromoterRepository.java
│   │   │   │   │
│   │   │   │   ├── model/
│   │   │   │   │   ├── entity/
│   │   │   │   │   │   └── Promoter.java
│   │   │   │   │   ├── dto/
│   │   │   │   │   ├── event/
│   │   │   │   │   │   ├── UserCreatedEvent.java
│   │   │   │   │   │   └── EventCreatedEvent.java
│   │   │   │   │   └── enums/
│   │   │   │   │       └── PromoterStatus.java
│   │   │   │   │
│   │   │   │   ├── client/
│   │   │   │   │   ├── EventServiceClient.java        # Feign Client
│   │   │   │   │   └── EventServiceFallback.java
│   │   │   │   │
│   │   │   │   ├── listener/
│   │   │   │   │   ├── UserCreatedListener.java
│   │   │   │   │   └── EventCreatedListener.java
│   │   │   │   │
│   │   │   │   ├── config/
│   │   │   │   │   ├── FeignConfig.java
│   │   │   │   │   └── RabbitMQConfig.java
│   │   │   │   │
│   │   │   │   └── exception/
│   │   │   │
│   │   │   └── resources/
│   │   │       ├── bootstrap.yml
│   │   │       ├── application.yml
│   │   │       └── db/migration/
│   │   │           └── V1__create_promoters_table.sql
│   │   │
│   │   └── test/
│   │
│   ├── pom.xml
│   └── Dockerfile
│
├── notification-service/           # Notification Service (Puerto 8085)
│   ├── src/
│   │   ├── main/
│   │   │   ├── java/com/amigusto/notification/
│   │   │   │   ├── NotificationServiceApplication.java
│   │   │   │   │
│   │   │   │   ├── service/
│   │   │   │   │   ├── EmailService.java
│   │   │   │   │   ├── PushNotificationService.java
│   │   │   │   │   └── SmsService.java
│   │   │   │   │
│   │   │   │   ├── repository/
│   │   │   │   │   └── NotificationRepository.java   # MongoDB
│   │   │   │   │
│   │   │   │   ├── model/
│   │   │   │   │   ├── document/
│   │   │   │   │   │   └── Notification.java         # MongoDB Document
│   │   │   │   │   ├── dto/
│   │   │   │   │   └── event/
│   │   │   │   │       ├── UserCreatedEvent.java
│   │   │   │   │       ├── EventApprovedEvent.java
│   │   │   │   │       └── EventRejectedEvent.java
│   │   │   │   │
│   │   │   │   ├── client/
│   │   │   │   │   ├── PromoterServiceClient.java     # Feign Client
│   │   │   │   │   └── PromoterServiceFallback.java
│   │   │   │   │
│   │   │   │   ├── listener/
│   │   │   │   │   ├── UserEventListener.java
│   │   │   │   │   └── EventEventListener.java
│   │   │   │   │
│   │   │   │   ├── config/
│   │   │   │   │   ├── MongoConfig.java
│   │   │   │   │   ├── RabbitMQConfig.java
│   │   │   │   │   ├── MailConfig.java
│   │   │   │   │   └── FirebaseConfig.java
│   │   │   │   │
│   │   │   │   └── exception/
│   │   │   │
│   │   │   └── resources/
│   │   │       ├── bootstrap.yml
│   │   │       ├── application.yml
│   │   │       └── templates/
│   │   │           └── email/
│   │   │               ├── welcome.html
│   │   │               ├── event-approved.html
│   │   │               └── event-rejected.html
│   │   │
│   │   └── test/
│   │
│   ├── pom.xml
│   └── Dockerfile
│
├── storage-service/                # Storage Service (Puerto 8086)
│   ├── src/
│   │   ├── main/
│   │   │   ├── java/com/amigusto/storage/
│   │   │   │   ├── StorageServiceApplication.java
│   │   │   │   │
│   │   │   │   ├── controller/
│   │   │   │   │   └── StorageController.java
│   │   │   │   │
│   │   │   │   ├── service/
│   │   │   │   │   ├── S3StorageService.java
│   │   │   │   │   └── CloudinaryStorageService.java
│   │   │   │   │
│   │   │   │   ├── model/
│   │   │   │   │   └── dto/
│   │   │   │   │       ├── UploadResponse.java
│   │   │   │   │       └── DeleteResponse.java
│   │   │   │   │
│   │   │   │   ├── config/
│   │   │   │   │   ├── S3Config.java
│   │   │   │   │   └── CloudinaryConfig.java
│   │   │   │   │
│   │   │   │   └── exception/
│   │   │   │       ├── GlobalExceptionHandler.java
│   │   │   │   │   └── InvalidFileException.java
│   │   │   │
│   │   │   └── resources/
│   │   │       ├── bootstrap.yml
│   │   │       └── application.yml
│   │   │
│   │   └── test/
│   │
│   ├── pom.xml
│   └── Dockerfile
│
├── docker-compose.yml              # Orquestación local de todos los servicios
├── docker-compose.dev.yml          # Override para desarrollo
├── docker-compose.prod.yml         # Override para producción
│
├── k8s/                            # Kubernetes manifests
│   ├── namespace.yml
│   ├── configmap.yml
│   ├── secrets.yml
│   │
│   ├── infrastructure/
│   │   ├── eureka-deployment.yml
│   │   ├── eureka-service.yml
│   │   ├── config-server-deployment.yml
│   │   ├── rabbitmq-deployment.yml
│   │   ├── rabbitmq-service.yml
│   │   ├── redis-deployment.yml
│   │   ├── redis-service.yml
│   │   ├── postgres-deployment.yml
│   │   ├── postgres-service.yml
│   │   ├── mongodb-deployment.yml
│   │   └── mongodb-service.yml
│   │
│   ├── microservices/
│   │   ├── api-gateway-deployment.yml
│   │   ├── api-gateway-service.yml
│   │   ├── auth-service-deployment.yml
│   │   ├── auth-service-service.yml
│   │   ├── event-service-deployment.yml
│   │   ├── event-service-service.yml
│   │   ├── user-service-deployment.yml
│   │   ├── user-service-service.yml
│   │   ├── promoter-service-deployment.yml
│   │   ├── promoter-service-service.yml
│   │   ├── notification-service-deployment.yml
│   │   ├── notification-service-service.yml
│   │   ├── storage-service-deployment.yml
│   │   └── storage-service-service.yml
│   │
│   └── ingress/
│       └── ingress.yml
│
├── .github/workflows/              # CI/CD Pipelines
│   ├── api-gateway-ci.yml
│   ├── auth-service-ci.yml
│   ├── event-service-ci.yml
│   ├── user-service-ci.yml
│   ├── promoter-service-ci.yml
│   ├── notification-service-ci.yml
│   └── storage-service-ci.yml
│
├── scripts/                        # Scripts útiles
│   ├── start-local.sh             # Levantar todo en local
│   ├── stop-local.sh
│   ├── build-all.sh               # Build de todos los servicios
│   └── deploy-k8s.sh              # Deploy a Kubernetes
│
├── pom.xml                         # Parent POM
├── .gitignore
├── README.md
└── .env.example
```

### 2.2 Convenciones de Código Java (Microservicios)

**Naming Conventions:**
- Clases: PascalCase (`EventService`, `EventController`)
- Métodos: camelCase (`getEventById`, `saveEvent`)
- Constantes: UPPER_SNAKE_CASE (`MAX_FILE_SIZE`, `JWT_SECRET_KEY`)
- Packages: lowercase (`com.amigusto.event.service`)

**Anotaciones Spring Boot:**
```java
@RestController
@RequestMapping("/api/v1/events")
@RequiredArgsConstructor
@Slf4j
@Validated
public class EventController {
    // ...
}
```

**Anotaciones Spring Cloud:**
```java
// Feign Client
@FeignClient(
    name = "PROMOTER-SERVICE",
    fallback = PromoterServiceFallback.class
)
public interface PromoterServiceClient {
    @GetMapping("/api/v1/promoters/{id}")
    PromoterResponse getPromoter(@PathVariable UUID id);
}

// RabbitMQ Listener
@Component
@RequiredArgsConstructor
@Slf4j
public class UserCreatedListener {

    @RabbitListener(queues = "user-service.user.created")
    public void handleUserCreated(UserCreatedEvent event) {
        log.info("Usuario creado: {}", event.getUserId());
        // Process event...
    }
}

// Circuit Breaker
@Service
@RequiredArgsConstructor
public class EventService {

    @CircuitBreaker(name = "promoterService", fallbackMethod = "fallback")
    @Retry(name = "promoterService")
    @Timeout(value = 3, unit = ChronoUnit.SECONDS)
    public PromoterResponse getPromoter(UUID id) {
        return promoterClient.getPromoter(id);
    }

    private PromoterResponse fallback(UUID id, Exception e) {
        log.error("Fallback activado", e);
        throw new ServiceUnavailableException("Promoter Service no disponible");
    }
}
```

### 2.3 Configuración Bootstrap vs Application

**bootstrap.yml** (cargado ANTES, para Config Server):
```yaml
spring:
  application:
    name: event-service
  cloud:
    config:
      uri: http://localhost:8888
      fail-fast: true
  profiles:
    active: dev
```

**application.yml** (configuración local, overrideada por Config Server):
```yaml
server:
  port: 8082

eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
  instance:
    preferIpAddress: true

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: always
```

---

## 3. CONFIGURACIÓN CENTRALIZADA (CONFIG REPO)

### 3.1 Estructura del Repositorio de Configuración

```
config-repo/
├── application.yml                 # Configuración común para TODOS los servicios
├── application-dev.yml             # Override para entorno DEV
├── application-staging.yml         # Override para entorno STAGING
├── application-prod.yml            # Override para entorno PROD
│
├── api-gateway.yml                 # Config específico de API Gateway
├── api-gateway-dev.yml
├── api-gateway-prod.yml
│
├── auth-service.yml                # Config específico de Auth Service
├── auth-service-dev.yml
├── auth-service-prod.yml
│
├── event-service.yml
├── event-service-dev.yml
├── event-service-prod.yml
│
├── user-service.yml
├── user-service-dev.yml
├── user-service-prod.yml
│
├── promoter-service.yml
├── promoter-service-dev.yml
├── promoter-service-prod.yml
│
├── notification-service.yml
├── notification-service-dev.yml
├── notification-service-prod.yml
│
├── storage-service.yml
├── storage-service-dev.yml
├── storage-service-prod.yml
│
└── README.md
```

### 3.2 Ejemplo de Configuración Común

**application.yml:**
```yaml
# Configuración compartida por TODOS los microservicios

spring:
  jpa:
    hibernate:
      ddl-auto: validate  # Usar Flyway para migraciones
    show-sql: false
    properties:
      hibernate:
        format_sql: true

  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: 6379
      timeout: 2000ms

  rabbitmq:
    host: ${RABBITMQ_HOST:localhost}
    port: 5672
    username: ${RABBITMQ_USER:rabbitmq}
    password: ${RABBITMQ_PASS:rabbitmq}

  sleuth:
    sampler:
      probability: 0.1  # 10% de requests trackeados

  zipkin:
    base-url: http://localhost:9411

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  metrics:
    export:
      prometheus:
        enabled: true

logging:
  level:
    root: INFO
    com.amigusto: DEBUG
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} [%X{traceId}/%X{spanId}] %-5level %logger{36} - %msg%n"
```

**event-service-prod.yml:**
```yaml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST:postgres-prod}:5432/event_service
    username: ${DB_USER}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5

  cache:
    type: redis
    redis:
      time-to-live: 300000  # 5 minutos

resilience4j:
  circuitbreaker:
    instances:
      promoterService:
        registerHealthIndicator: true
        slidingWindowSize: 10
        minimumNumberOfCalls: 5
        permittedNumberOfCallsInHalfOpenState: 3
        waitDurationInOpenState: 10s
        failureRateThreshold: 50

  retry:
    instances:
      promoterService:
        maxAttempts: 3
        waitDuration: 500ms

logging:
  level:
    root: WARN
    com.amigusto: INFO
```

---

## 4. ESTRUCTURA DE iOS (SWIFT + SWIFTUI)

### 4.1 Estructura Completa (Sin cambios respecto a monolito)

```
AmigustoiOS/
├── AmigustoiOS/
│   ├── App/
│   │   ├── AmigustoiOSApp.swift
│   │   └── AppDelegate.swift
│   │
│   ├── Core/
│   │   ├── Models/
│   │   │   ├── Event.swift
│   │   │   ├── Gusto.swift
│   │   │   └── User.swift
│   │   │
│   │   ├── Networking/
│   │   │   ├── APIService.swift            # Apunta a API Gateway :8080
│   │   │   ├── Endpoints.swift
│   │   │   └── NetworkError.swift
│   │   │
│   │   └── Persistence/
│   │       ├── CoreDataStack.swift
│   │       └── KeychainManager.swift
│   │
│   ├── Features/
│   │   ├── Auth/
│   │   ├── Onboarding/
│   │   ├── Discover/
│   │   └── MyPlans/
│   │
│   └── Resources/
│
└── AmigustoiOS.xcodeproj
```

**IMPORTANTE:** Las apps móviles NO necesitan saber que el backend son microservicios. Solo se comunican con **API Gateway** en puerto 8080.

---

## 5. ESTRUCTURA DE ANDROID (KOTLIN + JETPACK COMPOSE)

### 5.1 Estructura Completa (Sin cambios respecto a monolito)

```
AmigustoAndroid/
├── app/
│   ├── src/
│   │   ├── main/
│   │   │   ├── java/com/amigusto/
│   │   │   │   ├── AmigustoApplication.kt
│   │   │   │   ├── data/
│   │   │   │   │   ├── remote/
│   │   │   │   │   │   └── api/
│   │   │   │   │   │       └── ApiService.kt  # Base URL: API Gateway :8080
│   │   │   │   │   └── local/
│   │   │   │   ├── domain/
│   │   │   │   └── presentation/
│   │   │   └── res/
│   │   └── test/
│   └── build.gradle.kts
└── build.gradle.kts
```

**IMPORTANTE:** Android también solo conoce API Gateway. La arquitectura de microservicios es transparente para clientes.

---

## 6. ESTRUCTURA DE WEB (ANGULAR)

### 6.1 Estructura Completa (Sin cambios respecto a monolito)

```
amigusto-web/
├── projects/
│   ├── portal/                     # Portal B2B
│   │   ├── src/
│   │   │   ├── app/
│   │   │   │   ├── core/
│   │   │   │   │   └── services/
│   │   │   │   │       └── api.service.ts  # Base URL: API Gateway :8080
│   │   │   │   ├── features/
│   │   │   │   └── shared/
│   │   │   └── environments/
│   │   │       ├── environment.ts
│   │   │       │   # apiBaseUrl: 'http://localhost:8080/api/v1'
│   │   │       └── environment.prod.ts
│   │   │           # apiBaseUrl: 'https://api.amigusto.com/api/v1'
│   │   └── project.json
│   │
│   └── admin-panel/                # Admin Panel
│       └── (estructura similar)
│
├── angular.json
├── package.json
└── tsconfig.json
```

---

## 7. DOCKER COMPOSE PARA DESARROLLO LOCAL

### 7.1 docker-compose.yml

```yaml
version: '3.8'

services:
  # ========== INFRAESTRUCTURA ==========

  postgres:
    image: postgis/postgis:16-3.4
    container_name: amigusto-postgres
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: amigusto
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - amigusto-network

  redis:
    image: redis:7-alpine
    container_name: amigusto-redis
    ports:
      - "6379:6379"
    command: redis-server --requirepass redispass
    networks:
      - amigusto-network

  rabbitmq:
    image: rabbitmq:3-management-alpine
    container_name: amigusto-rabbitmq
    ports:
      - "5672:5672"   # AMQP
      - "15672:15672" # Management UI
    environment:
      RABBITMQ_DEFAULT_USER: rabbitmq
      RABBITMQ_DEFAULT_PASS: rabbitmq
    networks:
      - amigusto-network

  zipkin:
    image: openzipkin/zipkin:latest
    container_name: amigusto-zipkin
    ports:
      - "9411:9411"
    networks:
      - amigusto-network

  # ========== SPRING CLOUD INFRASTRUCTURE ==========

  eureka-server:
    build: ./eureka-server
    container_name: amigusto-eureka
    ports:
      - "8761:8761"
    networks:
      - amigusto-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8761/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 5

  config-server:
    build: ./config-server
    container_name: amigusto-config-server
    ports:
      - "8888:8888"
    environment:
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://eureka-server:8761/eureka/
      GIT_URI: https://github.com/amigusto/config-repo
    depends_on:
      - eureka-server
    networks:
      - amigusto-network

  # ========== MICROSERVICIOS ==========

  auth-service:
    build: ./auth-service
    container_name: amigusto-auth-service
    ports:
      - "8081:8081"
    environment:
      SPRING_PROFILES_ACTIVE: dev
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://eureka-server:8761/eureka/
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/amigusto
      SPRING_DATASOURCE_USERNAME: postgres
      SPRING_DATASOURCE_PASSWORD: postgres
      SPRING_DATA_REDIS_HOST: redis
      SPRING_DATA_REDIS_PASSWORD: redispass
      SPRING_RABBITMQ_HOST: rabbitmq
      SPRING_ZIPKIN_BASEURL: http://zipkin:9411
    depends_on:
      - postgres
      - redis
      - rabbitmq
      - eureka-server
      - config-server
    networks:
      - amigusto-network

  event-service:
    build: ./event-service
    container_name: amigusto-event-service
    ports:
      - "8082:8082"
    environment:
      SPRING_PROFILES_ACTIVE: dev
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://eureka-server:8761/eureka/
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/amigusto
      SPRING_DATA_REDIS_HOST: redis
      SPRING_RABBITMQ_HOST: rabbitmq
      SPRING_ZIPKIN_BASEURL: http://zipkin:9411
    depends_on:
      - postgres
      - redis
      - rabbitmq
      - eureka-server
      - config-server
    networks:
      - amigusto-network

  user-service:
    build: ./user-service
    container_name: amigusto-user-service
    ports:
      - "8083:8083"
    environment:
      SPRING_PROFILES_ACTIVE: dev
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://eureka-server:8761/eureka/
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/amigusto
      SPRING_RABBITMQ_HOST: rabbitmq
    depends_on:
      - postgres
      - rabbitmq
      - eureka-server
      - config-server
    networks:
      - amigusto-network

  promoter-service:
    build: ./promoter-service
    container_name: amigusto-promoter-service
    ports:
      - "8084:8084"
    environment:
      SPRING_PROFILES_ACTIVE: dev
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://eureka-server:8761/eureka/
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/amigusto
      SPRING_RABBITMQ_HOST: rabbitmq
    depends_on:
      - postgres
      - rabbitmq
      - eureka-server
    networks:
      - amigusto-network

  notification-service:
    build: ./notification-service
    container_name: amigusto-notification-service
    ports:
      - "8085:8085"
    environment:
      SPRING_PROFILES_ACTIVE: dev
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://eureka-server:8761/eureka/
      SPRING_DATA_MONGODB_URI: mongodb://mongo:27017/notifications
      SPRING_RABBITMQ_HOST: rabbitmq
    depends_on:
      - rabbitmq
      - eureka-server
    networks:
      - amigusto-network

  storage-service:
    build: ./storage-service
    container_name: amigusto-storage-service
    ports:
      - "8086:8086"
    environment:
      SPRING_PROFILES_ACTIVE: dev
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://eureka-server:8761/eureka/
      AWS_ACCESS_KEY_ID: ${AWS_ACCESS_KEY_ID}
      AWS_SECRET_ACCESS_KEY: ${AWS_SECRET_ACCESS_KEY}
    depends_on:
      - eureka-server
    networks:
      - amigusto-network

  api-gateway:
    build: ./api-gateway
    container_name: amigusto-api-gateway
    ports:
      - "8080:8080"
    environment:
      SPRING_PROFILES_ACTIVE: dev
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://eureka-server:8761/eureka/
    depends_on:
      - eureka-server
      - auth-service
      - event-service
      - user-service
      - promoter-service
      - notification-service
      - storage-service
    networks:
      - amigusto-network

networks:
  amigusto-network:
    driver: bridge

volumes:
  postgres-data:
```

### 7.2 Comandos Docker Compose

```bash
# Levantar toda la infraestructura
docker-compose up -d

# Ver logs de un servicio específico
docker-compose logs -f event-service

# Ver logs de todos los servicios
docker-compose logs -f

# Reiniciar un servicio
docker-compose restart event-service

# Parar todos los servicios
docker-compose down

# Parar y eliminar volúmenes
docker-compose down -v

# Rebuild de un servicio específico
docker-compose build event-service
docker-compose up -d event-service
```

---

## 8. GITIGNORE RECOMENDADO (POR TECNOLOGÍA)

### 8.1 Backend (Microservicios - Java)

```gitignore
# Maven
target/
pom.xml.tag
pom.xml.releaseBackup

# Gradle
.gradle/
build/

# IntelliJ IDEA
.idea/
*.iml
*.iws

# Eclipse
.classpath
.project
.settings/

# Application config (con secretos)
**/application-local.yml
**/application-dev.yml
**/bootstrap-local.yml
*.env
.env.local

# Logs
logs/
*.log

# OS
.DS_Store
Thumbs.db
```

### 8.2 iOS

```gitignore
# Xcode
xcuserdata/
*.xcodeproj/*
!*.xcodeproj/project.pbxproj

# SPM
.swiftpm/
.build/

# CocoaPods
Pods/

# Build
DerivedData/
build/

# Configuration
Configuration.plist

# OS
.DS_Store
```

### 8.3 Android

```gitignore
# Gradle
.gradle/
build/
local.properties

# Android Studio
.idea/
*.iml

# Keystore
*.jks
*.keystore

# OS
.DS_Store
```

### 8.4 Web (Angular)

```gitignore
# Dependencies
node_modules/
npm-debug.log

# Build
dist/
.angular/

# Environment
.env
.env.local

# IDE
.vscode/
.idea/

# OS
.DS_Store
```

---

## 9. CI/CD PIPELINE (GITHUB ACTIONS)

### 9.1 Ejemplo: Event Service CI/CD

```yaml
# .github/workflows/event-service-ci.yml
name: Event Service CI/CD

on:
  push:
    branches: [main, develop]
    paths:
      - 'event-service/**'
      - '.github/workflows/event-service-ci.yml'
  pull_request:
    branches: [main]
    paths:
      - 'event-service/**'

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

      - name: Upload test coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./event-service/target/site/jacoco/jacoco.xml

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
      - uses: actions/checkout@v3

      - name: Deploy to Kubernetes
        uses: azure/k8s-deploy@v4
        with:
          manifests: |
            k8s/microservices/event-service-deployment.yml
            k8s/microservices/event-service-service.yml
          images: |
            amigusto/event-service:${{ github.sha }}
          kubectl-version: 'latest'
```

---

## 🎯 Resumen de Arquitectura

### Microservicios (7 servicios):
1. **API Gateway** :8080 - Punto de entrada único
2. **Auth Service** :8081 - Autenticación, JWT
3. **Event Service** :8082 - Eventos, búsqueda geoespacial
4. **User Service** :8083 - Consumidores, eventos guardados
5. **Promoter Service** :8084 - Promotores, verificación
6. **Notification Service** :8085 - Emails, push
7. **Storage Service** :8086 - Upload imágenes S3

### Infraestructura:
- **Eureka Server** :8761 - Service Discovery
- **Config Server** :8888 - Configuración centralizada
- **RabbitMQ** :5672 - Message Broker
- **Redis** :6379 - Cache compartido
- **Zipkin** :9411 - Distributed Tracing
- **PostgreSQL** :5432 - Databases (DB per service)
- **MongoDB** :27017 - Notification Service DB

### Clientes (transparentes a microservicios):
- **iOS** - Swift + SwiftUI → API Gateway :8080
- **Android** - Kotlin + Compose → API Gateway :8080
- **Web** - Angular → API Gateway :8080

---

**Última actualización:** 2025-10-26
**Versión:** 2.0.0 - Arquitectura de Microservicios
