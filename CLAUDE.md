# CLAUDE.md

Este archivo proporciona orientación a Claude Code (claude.ai/code) cuando trabaja con código en este repositorio.

## Visión General del Proyecto

**Amigusto** es una plataforma de descubrimiento de eventos hiper-personalizada con la propuesta de valor única: "Cero ruido, solo tus intereses".

A diferencia de competidores como Fever que priorizan sus propias ventas de tickets, Amigusto es un filtro imparcial que prioriza las preferencias del usuario (gustos), dando el mismo peso a eventos gratuitos y de pago basándose en relevancia.

### Arquitectura de Backend

**IMPORTANTE**: El backend se implementa como **arquitectura de microservicios** usando Spring Cloud:

- **API Gateway** (Spring Cloud Gateway) - Puerto 8080
- **Service Discovery** (Netflix Eureka) - Puerto 8761
- **Config Server** (Spring Cloud Config) - Puerto 8888
- **Auth Service** - Puerto 8081 (Autenticación, JWT)
- **Event Service** - Puerto 8082 (Eventos, búsqueda geoespacial)
- **User Service** - Puerto 8083 (Consumidores, eventos guardados)
- **Promoter Service** - Puerto 8084 (Promotores, verificación)
- **Notification Service** - Puerto 8085 (Emails, push notifications)
- **Storage Service** - Puerto 8086 (Upload imágenes S3/Cloudinary)

**Comunicación**:
- Síncrona: Spring Cloud OpenFeign (REST)
- Asíncrona: RabbitMQ (eventos)
- Cache: Redis compartido
- Tracing: Zipkin + Sleuth

Consulta **ARQUITECTURA_MICROSERVICIOS.md** para detalles completos.

## Lógica de Negocio Central: Flujo de Curación Humana

El diferenciador de la plataforma es su proceso de curación humana:

```
Promotor crea evento (Portal Angular)
    ↓
Evento entra en estado "PENDING_REVIEW" (NO se publica automáticamente)
    ↓
Curador humano revisa y aprueba (Panel Admin Angular)
    ↓
Evento se vuelve visible en Apps Móviles (iOS + Android)
    ↓
Usuarios descubren eventos que coinciden con sus gustos
```

**Crítico:** Los eventos NUNCA se publican automáticamente. Todos los eventos deben pasar por revisión humana para garantizar cero spam y alta calidad de contenido.

## Arquitectura: Tres Pilares

1. **Apps Móviles (B2C)**: iOS nativo (Swift + SwiftUI) y Android nativo (Kotlin + Jetpack Compose) para descubrimiento de eventos por consumidores
2. **Portal Web (B2B)**: Aplicación Angular para que promotores creen y gestionen eventos
3. **Panel Admin**: Aplicación Angular para que curadores humanos revisen y aprueben eventos

Los tres consumen la misma **API Backend** construida con Java Spring Boot.

## Stack Tecnológico

| Componente | Tecnología |
|-----------|-----------|
| Backend API | Java 17 + Spring Boot 3.2 |
| Base de Datos | PostgreSQL 16 + PostGIS (geoespacial) |
| Caché | Redis 7 |
| App iOS | Swift 5.9 + SwiftUI |
| App Android | Kotlin 1.9 + Jetpack Compose |
| Web (Portal + Admin) | Angular 17 (Standalone Components) |

**Nota:** Este proyecto previamente usaba Node.js/React Native/Next.js pero fue completamente migrado al stack actual. Asegúrate de que no queden referencias a tecnologías antiguas.

## Estructura de Repositorios

Este es un repositorio de documentación únicamente. El código real se dividirá en 4 repositorios separados:

```
amigusto-backend/       (Java Spring Boot)
amigusto-ios/          (Swift + SwiftUI)
amigusto-android/      (Kotlin + Jetpack Compose)
amigusto-web/          (Angular - Portal + Admin)
```

## Conceptos Centrales del Esquema de Base de Datos

### Máquina de Estados de Evento

Los eventos siguen esta máquina de estados (aplicada en el backend):

```
DRAFT → PENDING_REVIEW → APPROVED
                       ↘ REJECTED

APPROVED → CANCELLED
        → ENDED
```

### Entidades Clave

- **User**: Consumidores que descubren eventos (rol: `CONSUMER`)
- **Promoter**: Creadores de eventos (rol: `PROMOTER`)
- **Admin**: Curadores que aprueban eventos (rol: `ADMIN`)
- **Event**: Entidad central con geolocalización (lat/lng), gustos (etiquetas) y estado
- **Gusto**: Etiquetas de categoría/interés (ej. "🎵 Música", "🎭 Teatro")
- **SavedEvent**: Relación M:N entre Usuarios y Eventos que han guardado

### Consultas Geoespaciales

El backend usa **PostGIS** para cálculos geoespaciales con la fórmula de Haversine:

```sql
-- Ejemplo: Encontrar eventos dentro de un radio
SELECT * FROM events e
WHERE (
    6371 * acos(
        cos(radians(:lat)) * cos(radians(e.lat))
        * cos(radians(e.lng) - radians(:lng))
        + sin(radians(:lat)) * sin(radians(e.lat))
    )
) < :radiusKm
```

## Arquitectura Backend (Spring Boot)

### Arquitectura en Capas

```
Controller → Service → Repository → Base de Datos
         ↓
    Mapeo de DTOs
```

**Patrones Clave:**
- Los Controllers usan `@RestController` y retornan DTOs (nunca entidades)
- Los Services contienen lógica de negocio y usan `@Transactional`
- Los Repositories extienden `JpaRepository` con consultas personalizadas
- Seguridad vía Spring Security + tokens JWT
- Caché con `@Cacheable` (Redis)

### Endpoints Importantes

**Públicos:**
- `POST /api/v1/auth/login` - Autenticación
- `POST /api/v1/auth/register` - Registro de usuario
- `GET /api/v1/gustos` - Listar todos los gustos (categorías)

**Consumidor (Apps Móviles):**
- `GET /api/v1/events/discover` - Descubrimiento personalizado de eventos (requiere gustoIds, city, lat, lng)
- `POST /api/v1/saved-events` - Guardar evento en el plan del usuario
- `GET /api/v1/saved-events` - Obtener eventos guardados del usuario

**Promotor (Portal Web):**
- `POST /api/v1/events` - Crear evento (status = DRAFT)
- `POST /api/v1/events/{id}/submit-review` - Enviar a revisión (DRAFT → PENDING_REVIEW)
- `GET /api/v1/events/my-events` - Obtener eventos del promotor

**Admin (Panel):**
- `GET /api/v1/events/pending-review` - Cola de eventos esperando aprobación
- `POST /api/v1/events/{id}/approve` - Aprobar evento (limpia caché)
- `POST /api/v1/events/{id}/reject` - Rechazar evento con razón

### Configuración de Seguridad

- Autenticación JWT con access tokens (15 min) y refresh tokens (7 días)
- Hash de contraseñas con BCrypt
- Control de acceso basado en roles: `CONSUMER`, `PROMOTER`, `ADMIN`
- CORS configurado por dominio
- Rate limiting con Bucket4j

## Arquitectura iOS (Swift + SwiftUI)

### Patrón MVVM

```
View (SwiftUI) ↔ ViewModel (@Published) ↔ Service (API/Local) ↔ Model
```

**Componentes Clave:**
- `APIService`: Networking centralizado con publishers de Combine
- `KeychainManager`: Almacenamiento seguro de JWT
- `@StateObject` para ViewModels
- Propiedades `@Published` para actualizaciones reactivas de UI
- CoreData para caché offline

### Funcionalidades Principales

1. **Onboarding**: Selección de gustos + permiso de ubicación
2. **Discover**: Feed de scroll infinito con filtrado basado en geolocalización
3. **Event Detail**: Información completa del evento con mapa
4. **My Plans**: Lista de eventos guardados

## Arquitectura Android (Kotlin + Jetpack Compose)

### Clean Architecture + MVVM

```
Presentation (Compose) → ViewModel (StateFlow) → UseCase → Repository → Data Source
                                                                        ↓
                                                                   API / Room DB
```

**Componentes Clave:**
- Hilt para inyección de dependencias
- Retrofit + OkHttp para networking
- Room Database para caché offline
- `StateFlow`/`LiveData` para estado reactivo
- Coil para carga de imágenes
- Navigation Compose para enrutamiento

### Patrón Repository

```kotlin
interface EventRepository {
    fun discoverEvents(...): Flow<Resource<List<Event>>>
    suspend fun saveEvent(eventId: UUID): Result<Unit>
}
```

Retorna `Resource<T>` con estados: `Loading`, `Success`, `Error`

## Arquitectura Angular (Portal + Admin)

### Workspace Multi-Proyecto

```
amigusto-web/
├── projects/
│   ├── portal/         # Portal B2B para Promotores
│   └── admin-panel/    # Herramienta de curación Admin
```

### Estructura

```
app/
├── core/               # Servicios singleton, guards, interceptors
│   ├── services/      # Comunicación API
│   ├── guards/        # Protección de rutas
│   ├── interceptors/  # Manejo de peticiones/respuestas HTTP
│   └── models/        # Interfaces TypeScript
├── features/          # Módulos de funcionalidades (lazy loaded)
│   ├── auth/
│   ├── dashboard/
│   └── events/
└── shared/            # Componentes reutilizables, pipes, directivas
```

**Servicios Clave:**
- `AuthService`: Gestión de JWT + login/logout
- `EventService`: Operaciones CRUD de eventos
- `GustoService`: Cargar gustos disponibles

**Importante:** Usar Standalone Components de Angular 17+ (sin NgModules).

## Comandos de Desarrollo

### Backend (Spring Boot)

```bash
# Ejecutar localmente
cd amigusto-backend
./mvnw spring-boot:run

# Ejecutar tests
./mvnw test

# Build
./mvnw clean package

# Puntos de acceso:
# API: http://localhost:8080
# Swagger UI: http://localhost:8080/swagger-ui.html
```

### iOS

```bash
cd amigusto-ios
open AmigustoiOS.xcodeproj
# Ejecutar en Xcode: ⌘R
# Ejecutar tests: ⌘U
```

### Android

```bash
cd amigusto-android

# Build debug APK
./gradlew assembleDebug

# Ejecutar tests unitarios
./gradlew test

# Ejecutar tests de instrumentación
./gradlew connectedAndroidTest

# Abrir en Android Studio y ejecutar
```

### Web (Angular)

```bash
cd amigusto-web

# Instalar dependencias
npm install

# Ejecutar Portal (servidor dev)
npm run start:portal
# Acceder: http://localhost:4200

# Ejecutar Panel Admin
npm run start:admin
# Acceder: http://localhost:4201

# Lint
npm run lint

# Ejecutar tests
npm test

# Build para producción
npm run build:portal
npm run build:admin
```

## Estrategia de Testing

### Backend (JUnit 5 + Mockito)

- Tests unitarios para Services con repositories mockeados
- Tests de integración para Controllers con `@WebMvcTest`
- Tests de Repository con `@DataJpaTest`

### iOS (XCTest)

- Tests unitarios para ViewModels
- Mock de `APIService` para tests de red
- Tests de UI con XCUITest para flujos críticos

### Android (JUnit + Espresso)

- Tests unitarios para ViewModels y UseCases
- Tests de Repository con implementaciones fake
- Tests de UI con librería de testing de Compose

### Web (Jasmine + Karma + Cypress)

- Tests unitarios para servicios y componentes
- Tests de integración con `TestBed`
- Tests E2E con Cypress para flujos de usuario

## Flujo de Trabajo con Git

### Estrategia de Ramas

- `main` - Producción
- `staging` - Entorno de staging
- `develop` - Integración de desarrollo
- `feature/nombre-feature` - Ramas de funcionalidades
- `bugfix/nombre-bug` - Corrección de bugs

### Convención de Commits

Usar **Conventional Commits**:

```
feat(backend): agregar filtrado geolocalizado de eventos
feat(ios): implementar scroll infinito en feed de descubrimiento
feat(android): agregar pull-to-refresh en eventos guardados
feat(web): crear cola de aprobación de eventos para admins
fix(backend): resolver bug de expiración de token JWT
docs(readme): actualizar instrucciones de setup
```

### Requisitos de Code Review

- Todas las features requieren PR con ≥1 aprobación
- Los tests deben pasar (CI/CD)
- El linting debe pasar:
  - Backend: Checkstyle/SpotBugs
  - iOS: SwiftLint
  - Android: ktlint/detekt
  - Angular: ESLint

## Configuración de Entornos

### Backend (application.yml)

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/amigusto_dev
  jpa:
    hibernate:
      ddl-auto: validate  # Usar Flyway para migraciones
  data:
    redis:
      host: localhost
      port: 6379

jwt:
  secret: ${JWT_SECRET}
  expiration: 900000  # 15 minutos
```

### iOS (Configuration.plist - en gitignore)

```xml
<key>API_BASE_URL</key>
<string>https://api.amigusto.com</string>
```

### Android (local.properties - en gitignore)

```properties
API_BASE_URL=https://api.amigusto.com
GOOGLE_MAPS_API_KEY=tu_clave_aqui
```

### Angular (environment.ts)

```typescript
export const environment = {
  production: false,
  apiBaseUrl: 'http://localhost:8080/api/v1'
};
```

## Objetivos de Rendimiento Clave (MVP)

- Latencia API p95: <500ms
- Uptime: >99.9%
- Tasa de apps libre de crashes: >99.5%
- Tiempo de aprobación de eventos: <24 horas
- DAU (Día 1): 500+ usuarios activos

## Consideraciones de Seguridad

- **NUNCA** commitear secretos JWT, API keys o credenciales de base de datos
- Usar variables de entorno para configuración sensible
- iOS: Almacenar tokens en Keychain
- Android: Usar EncryptedSharedPreferences
- Web: Usar cookies HttpOnly para refresh tokens
- Backend: Siempre validar entrada con Jakarta Validation (`@Valid`)
- Backend: Usar consultas parametrizadas (JPA/Hibernate protege contra SQL injection)

## Archivos de Documentación

### Arquitectura
- **ARQUITECTURA_MICROSERVICIOS.md**: 🔴 Arquitectura completa de microservicios (PRINCIPAL)
- **MICROSERVICIOS_RESUMEN.md**: Resumen ejecutivo de microservicios
- **ARQUITECTURA_FUNCIONAL_DETALLADA.md**: Flujos funcionales detallados con arquitectura de microservicios

### Planificación
- **PLAN_TECNICO_AMIGUSTO.md**: Plan de implementación por fases
- **GUIA_INICIO_RAPIDO.md**: Setup local de microservicios

### Estructura de Código
- **ARQUITECTURA_PROYECTO.md**: Estructura de carpetas por microservicio
- **EJEMPLOS_CODIGO.md**: Ejemplos de código (Controllers, Services, Feign, RabbitMQ)

### Visualización
- **DIAGRAMAS.md**: Diagramas de arquitectura (Mermaid)

## Patrones Comunes a Seguir

### Backend (Microservicio): Crear una nueva entidad

**IMPORTANTE**: Primero determina en QUÉ microservicio debe vivir la entidad.

1. Identificar microservicio responsable (Event Service, User Service, etc.)
2. Crear clase `@Entity` en `{servicio}/model/entity/`
3. Crear DTOs en `{servicio}/model/dto/request/` y `response/`
4. Crear interfaz `Repository` extendiendo `JpaRepository`
5. Crear `Service` con métodos `@Transactional`
6. Crear `Controller` con `@RestController`
7. Agregar migración Flyway en `{servicio}/resources/db/migration/`
8. **Si otros servicios necesitan los datos**:
   - Opción A: Crear `FeignClient` en el servicio consumidor
   - Opción B: Publicar eventos vía RabbitMQ para replicación
9. Actualizar Swagger/OpenAPI docs del servicio
10. Configurar Circuit Breaker si hay llamadas Feign

### iOS: Crear una nueva funcionalidad

1. Crear Model en `Core/Models/`
2. Agregar métodos API a `APIService` o crear servicio dedicado
3. Crear ViewModel con propiedades `@Published`
4. Crear View de SwiftUI usando patrón MVVM
5. Agregar navegación en `Navigation/`

### Android: Crear una nueva funcionalidad

1. Crear modelo de dominio en `domain/model/`
2. Crear DTO en `data/remote/dto/`
3. Agregar endpoint API a `ApiService`
4. Crear interfaz Repository + implementación
5. Crear UseCase en `domain/usecase/`
6. Crear ViewModel con `StateFlow`
7. Crear pantalla Composable

### Angular: Crear una nueva funcionalidad

1. Crear módulo/componente de feature en `features/`
2. Definir interfaces TypeScript en `core/models/`
3. Crear servicio en `core/services/`
4. Crear componente con Reactive Forms si es necesario
5. Agregar ruta en `app.routes.ts`
6. Agregar guard si requiere autenticación

## Reglas de Negocio Críticas

1. **Transiciones de Estado de Evento**: Solo se permiten transiciones de estado válidas (aplicadas en `EventService`)
2. **Requisito de Gusto**: Los eventos deben tener ≥1 gusto para ser creados
3. **Geolocalización Requerida**: Todos los eventos deben tener coordenadas lat/lng válidas
4. **No Auto-Aprobación**: Los eventos solo pueden moverse a APPROVED vía acción de admin
5. **Invalidación de Caché**: Aprobar/rechazar eventos debe limpiar caché de `discover-events`
6. **Aplicación de Roles**: Los endpoints deben verificar roles de usuario (Spring Security `@PreAuthorize`)

## Resolución de Problemas

### El backend no inicia
- Verificar que PostgreSQL esté ejecutándose: `docker ps` o `brew services list`
- Verificar que Redis esté ejecutándose
- Verificar credenciales de base de datos en `application.yml`

### El build de iOS falla
- Limpiar carpeta de build: Xcode → Product → Clean Build Folder (⌘⇧K)
- Eliminar derived data: `rm -rf ~/Library/Developer/Xcode/DerivedData`
- Actualizar dependencias de Swift Package

### El build de Android falla
- Invalidar cachés: Android Studio → File → Invalidate Caches / Restart
- Limpiar proyecto: `./gradlew clean`
- Verificar `gradle.properties` y `local.properties`

### Errores de compilación Angular
- Eliminar node_modules: `rm -rf node_modules package-lock.json`
- Reinstalar: `npm install`
- Verificar versión de Angular CLI: `ng version`

## Fases de Implementación

El proyecto está planificado en 8 fases (~23 semanas en total):

1. **Fase 0** (1-2 semanas): Setup de arquitectura
2. **Fase 1** (3 semanas): API Backend con Spring Boot
3. **Fase 2** (5 semanas): Apps nativas iOS + Android
4. **Fase 3** (3 semanas): Portal Web (Angular B2B)
5. **Fase 4** (2 semanas): Panel Admin (Angular)
6. **Fase 5** (2 semanas): Testing & QA
7. **Fase 6** (2 semanas): Preparación pre-lanzamiento
8. **Fase 7** (3 semanas): Beta pública
9. **Fase 8** (1 semana): Lanzamiento público (App Store + Google Play)

Consultar PLAN_TECNICO_AMIGUSTO.md para el desglose detallado de tareas por fase.

## Convenciones de Código

### Java (Backend)

**Nomenclatura:**
- Clases: PascalCase (`EventService`, `UserController`)
- Métodos: camelCase (`getEventById`, `saveEvent`)
- Constantes: UPPER_SNAKE_CASE (`MAX_FILE_SIZE`, `JWT_SECRET_KEY`)
- Packages: lowercase (`com.amigusto.service`)

**Estructura de anotaciones Spring:**
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

### Swift (iOS)

**Nomenclatura:**
- Tipos: PascalCase (`EventViewModel`, `DiscoverView`)
- Funciones/Variables: camelCase (`fetchEvents`, `selectedGustos`)
- Protocolos: PascalCase con sufijo "able" o "ing" (`Codable`, `ObservableObject`)

**Estructura de Views SwiftUI:**
```swift
struct DiscoverView: View {
    @StateObject private var viewModel = DiscoverViewModel()

    var body: some View {
        // UI aquí
    }
}
```

### Kotlin (Android)

**Nomenclatura:**
- Clases/Interfaces: PascalCase (`EventRepository`, `DiscoverViewModel`)
- Funciones/Variables: camelCase (`fetchEvents`, `savedEvents`)
- Constantes: UPPER_SNAKE_CASE (`API_BASE_URL`, `MAX_RETRY_COUNT`)
- Packages: lowercase (`com.amigusto.data.repository`)

**Estructura de Composables:**
```kotlin
@Composable
fun DiscoverScreen(
    viewModel: DiscoverViewModel = hiltViewModel()
) {
    val uiState by viewModel.uiState.collectAsState()

    // UI aquí
}
```

### TypeScript (Angular)

**Nomenclatura:**
- Components: kebab-case archivos, PascalCase clases (`event-card.component.ts`, `EventCardComponent`)
- Services: kebab-case archivos con sufijo `.service.ts` (`auth.service.ts`, `AuthService`)
- Interfaces: PascalCase con prefijo "I" opcional (`IEvent` o `Event`)
- Enums: PascalCase (`EventStatus`)

**Estructura de Component:**
```typescript
@Component({
  selector: 'app-event-card',
  standalone: true,
  imports: [CommonModule, MatCardModule],
  templateUrl: './event-card.component.html',
  styleUrls: ['./event-card.component.scss']
})
export class EventCardComponent implements OnInit {
  // ...
}
```

## Glosario de Términos del Dominio

- **Gusto**: Categoría o etiqueta de interés del usuario (música, teatro, deportes, etc.)
- **Curación**: Proceso de revisión humana de eventos antes de publicación
- **Promotor**: Usuario con rol de creador de eventos (B2B)
- **Consumidor**: Usuario final que descubre eventos (B2C)
- **Discovery Feed**: Feed personalizado de eventos basado en gustos del usuario
- **Saved Events / My Plans**: Eventos que el usuario ha guardado para asistir
- **Event Status**: Estado del evento en la máquina de estados (DRAFT, PENDING_REVIEW, etc.)
- **Geofiltrado**: Filtrado de eventos basado en ubicación geográfica y radio

## Contacto y Recursos

Para preguntas sobre la arquitectura o implementación, consultar:
- **PLAN_TECNICO_AMIGUSTO.md** - Plan técnico completo
- **ARQUITECTURA_PROYECTO.md** - Estructura detallada de código
- **EJEMPLOS_CODIGO.md** - Ejemplos prácticos de implementación
