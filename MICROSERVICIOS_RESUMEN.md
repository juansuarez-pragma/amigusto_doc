# Arquitectura de Microservicios - Amigusto
## Resumen Ejecutivo

---

## 📋 Microservicios Implementados

### 1. **API Gateway** (Puerto 8080)
- **Tecnología**: Spring Cloud Gateway
- **Responsabilidad**: Punto de entrada único, routing, autenticación centralizada
- **Características**:
  - Validación JWT
  - Rate limiting
  - Load balancing
  - CORS
  - Request/Response logging

### 2. **Auth Service** (Puerto 8081)
- **Tecnología**: Spring Boot + Spring Security
- **Base de Datos**: PostgreSQL (users)
- **Responsabilidad**: Autenticación y autorización
- **APIs**:
  - POST /auth/register/consumer
  - POST /auth/register/promoter
  - POST /auth/login
  - POST /auth/refresh
  - POST /auth/logout
- **Eventos Publicados**: `user.created`

### 3. **Event Service** (Puerto 8082)
- **Tecnología**: Spring Boot + PostGIS
- **Base de Datos**: PostgreSQL + PostGIS (events, gustos)
- **Responsabilidad**: Gestión de eventos, búsqueda geoespacial
- **APIs**:
  - POST /events
  - GET /events/discover (búsqueda geolocalizada)
  - POST /events/{id}/approve (Admin)
  - POST /events/{id}/reject (Admin)
  - GET /gustos
- **Clientes Feign**: Promoter Service, Storage Service
- **Eventos Publicados**: `event.approved`, `event.rejected`, `event.created`

### 4. **User Service** (Puerto 8083)
- **Tecnología**: Spring Boot
- **Base de Datos**: PostgreSQL (consumers, saved_events)
- **Responsabilidad**: Perfiles de consumidores, eventos guardados
- **APIs**:
  - GET /users/me
  - PUT /users/me/gustos
  - POST /saved-events
  - GET /saved-events
- **Clientes Feign**: Event Service
- **Eventos Publicados**: `event.saved`
- **Eventos Consumidos**: `user.created`

### 5. **Promoter Service** (Puerto 8084)
- **Tecnología**: Spring Boot
- **Base de Datos**: PostgreSQL (promoters)
- **Responsabilidad**: Gestión de promotores, verificación
- **APIs**:
  - POST /promoters
  - GET /promoters/me
  - POST /promoters/{id}/verify (Admin)
- **Eventos Consumidos**: `user.created`, `event.created`

### 6. **Notification Service** (Puerto 8085)
- **Tecnología**: Spring Boot + Spring Mail
- **Base de Datos**: MongoDB (notifications)
- **Responsabilidad**: Envío de emails, push notifications
- **Eventos Consumidos**: `user.created`, `event.approved`, `event.rejected`
- **Integraciones**: Firebase Cloud Messaging, SMTP

### 7. **Storage Service** (Puerto 8086)
- **Tecnología**: Spring Boot + AWS SDK
- **Base de Datos**: Ninguna (stateless)
- **Responsabilidad**: Upload de imágenes a S3/Cloudinary
- **APIs**:
  - POST /storage/upload
  - DELETE /storage/{key}

---

## 🏗️ Infraestructura Compartida

### Service Discovery (Eureka Server) - Puerto 8761
```yaml
Función: Registro y descubrimiento de microservicios
Health checks: Automáticos cada 30s
Tecnología: Netflix Eureka
```

### Config Server - Puerto 8888
```yaml
Función: Configuración centralizada
Storage: Git repository
Refresh: Dinámico vía /actuator/refresh
```

### Message Broker (RabbitMQ) - Puerto 5672
```yaml
Exchanges:
  - user.events (Topic)
  - event.events (Topic)
  - cache.events (Fanout)

Queues por servicio:
  - notification-service.user.created
  - notification-service.event.approved
  - user-service.user.created
  - event-service.event.saved
```

### Distributed Tracing (Zipkin) - Puerto 9411
```yaml
Función: Trazabilidad de requests cross-service
Tecnología: Zipkin + Spring Cloud Sleuth
Sampling: 100% en dev, 10% en prod
```

### Cache (Redis) - Puerto 6379
```yaml
Uso: Cache compartido entre servicios
Namespaces:
  - amigusto:auth:*
  - amigusto:event:*
  - amigusto:user:*
TTL: Variable por tipo de dato (1min - 24h)
```

---

## 🔄 Comunicación entre Servicios

### Síncrona (OpenFeign)
```
Event Service ──Feign──> Promoter Service (validar promotor)
Event Service ──Feign──> Storage Service (validar imagen)
User Service  ──Feign──> Event Service (obtener detalles evento)
Notification  ──Feign──> Promoter Service (obtener email)
```

**¿Cuándo usar?**
- ✅ Validaciones críticas antes de procesar request
- ✅ Lectura de datos necesarios para completar operación
- ❌ Notificaciones (usar RabbitMQ)
- ❌ Operaciones que pueden fallar sin afectar flujo principal

### Asíncrona (RabbitMQ)
```
Auth Service     ──publish──> user.created ──consume──> User Service
                                           ──consume──> Notification Service

Event Service    ──publish──> event.approved ──consume──> Notification Service
                 ──publish──> event.created ──consume──> Promoter Service

User Service     ──publish──> event.saved ──consume──> Event Service
```

**¿Cuándo usar?**
- ✅ Notificaciones (emails, push)
- ✅ Actualizar métricas/contadores
- ✅ Operaciones que pueden ejecutarse en background
- ✅ Desacoplamiento entre servicios

---

## 🛡️ Resiliencia

### Circuit Breaker (Resilience4j)
```java
@CircuitBreaker(name = "promoterService", fallbackMethod = "fallback")
@Retry(name = "promoterService", maxAttempts = 3)
@Timeout(value = 3, unit = ChronoUnit.SECONDS)
public PromoterResponse getPromoter(UUID id) {
    return promoterClient.getPromoter(id);
}
```

**Estados**:
- CLOSED → OPEN (50% fallos)
- OPEN → HALF_OPEN (esperar 10s)
- HALF_OPEN → CLOSED (3 requests exitosos)

### Configuración
```yaml
resilience4j:
  circuitbreaker:
    instances:
      promoterService:
        slidingWindowSize: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 10s
  retry:
    instances:
      promoterService:
        maxAttempts: 3
        waitDuration: 500ms
```

---

## 📊 Observabilidad

### Distributed Tracing (Zipkin)
```
Request trace completo:
API Gateway (10ms)
  └─> Event Service (150ms)
       └─> Promoter Service (80ms)
            └─> PostgreSQL (60ms)

Total: 300ms
```

### Métricas (Prometheus)
```
http_server_requests_seconds_count{service="event-service"} 15234
http_server_requests_seconds_sum{service="event-service"} 4567.89
jvm_memory_used_bytes{service="event-service"} 524288000
```

### Logging (ELK Stack)
```
Logs centralizados con metadata:
- traceId (correlación cross-service)
- spanId (identificación de operación)
- service (nombre del microservicio)
- userId (usuario que hizo el request)
```

---

## 🗄️ Gestión de Datos

### Database per Service Pattern

| Microservicio | Base de Datos | Justificación |
|---------------|---------------|---------------|
| Auth Service | PostgreSQL | Datos de usuarios (ACID) |
| Event Service | PostgreSQL + PostGIS | Búsquedas geoespaciales |
| User Service | PostgreSQL | Eventos guardados (ACID) |
| Promoter Service | PostgreSQL | Promotores (ACID) |
| Notification Service | MongoDB | Alta escritura, logs flexibles |

### Data Replication (Event-Driven)
```
Auth Service crea usuario
  ↓ Publica evento user.created
User Service consume evento
  ↓ Replica email, name (eventual consistency)
PostgreSQL (User DB) almacena datos replicados
```

**Datos replicados**:
- ✅ ID (mismo entre servicios)
- ✅ Email, Name (inmutables)
- ❌ Password hash (NUNCA)
- ❌ Datos que cambian frecuentemente

---

## 🚀 Despliegue

### Docker Compose (Local)
```bash
docker-compose up -d

# Servicios levantados:
# - eureka-server :8761
# - config-server :8888
# - postgres :5432
# - redis :6379
# - rabbitmq :5672, :15672
# - auth-service :8081
# - event-service :8082
# - user-service :8083
# - promoter-service :8084
# - notification-service :8085
# - storage-service :8086
# - api-gateway :8080
```

### Kubernetes (Producción)
```yaml
Deployment per service:
- Replicas: 3 (auth, event, user)
- Replicas: 2 (promoter, notification, storage)
- Resources:
    requests: 512Mi memory, 500m CPU
    limits: 1Gi memory, 1000m CPU
- Health checks: liveness + readiness probes
- Auto-scaling: HPA (CPU > 70%)
```

### CI/CD (GitHub Actions)
```
1. Push a main
2. Run tests (JUnit + Testcontainers)
3. Build Docker image
4. Push to Docker Hub
5. Deploy to Kubernetes
6. Run smoke tests
```

---

## 📈 Ventajas de Microservicios

1. **Escalado Independiente**: Event Service puede tener 5 réplicas, Auth Service solo 2
2. **Despliegue Independiente**: Actualizar Notification Service sin tocar Event Service
3. **Tecnología Apropiada**: MongoDB para Notification, PostgreSQL para Event
4. **Resiliencia**: Fallo en Notification Service no afecta Event Service
5. **Equipos Autónomos**: Equipo A trabaja en Event Service, Equipo B en User Service
6. **Desarrollo Paralelo**: Múltiples features en distintos servicios simultáneamente

---

## ⚠️ Desafíos de Microservicios

1. **Complejidad Operacional**: 7 servicios + infraestructura vs 1 monolito
2. **Debugging Distribuido**: Tracing esencial, más difícil que monolito
3. **Latencia de Red**: Llamadas Feign más lentas que llamadas locales
4. **Consistencia Eventual**: Datos replicados pueden estar desincronizados
5. **Testing E2E**: Requiere todos los servicios corriendo
6. **DevOps Expertise**: Requiere conocimiento de Docker, Kubernetes, etc.

---

## 🎯 Recomendaciones

### Para MVP:
✅ **Empezar con Microservicios** si:
- Equipo tiene experiencia con Spring Cloud
- Presupuesto para infraestructura (Kubernetes cluster)
- Plan de escalar rápidamente (100k+ usuarios)

❌ **Empezar con Monolito** si:
- Equipo pequeño (<3 developers)
- MVP para validar idea rápidamente
- Presupuesto limitado

### Migración Gradual:
1. **Fase 1**: Monolito (primeros 6 meses)
2. **Fase 2**: Extraer Auth Service (cuando >10k usuarios)
3. **Fase 3**: Extraer Event Service (cuando búsqueda es lenta)
4. **Fase 4**: Extraer Notification Service (cuando emails son cuello de botella)

---

## 📚 Recursos Adicionales

### Documentos:
- **ARQUITECTURA_MICROSERVICIOS.md**: Detalles técnicos completos
- **ARQUITECTURA_FUNCIONAL_DETALLADA.md**: Flujos de funcionalidades
- **PLAN_TECNICO_AMIGUSTO.md**: Plan de implementación

### Tecnologías:
- Spring Cloud: https://spring.io/projects/spring-cloud
- Resilience4j: https://resilience4j.readme.io
- Zipkin: https://zipkin.io
- RabbitMQ: https://www.rabbitmq.com

---

**Última actualización:** 2025-10-26
**Versión:** 1.0.0
