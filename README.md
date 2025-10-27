# Amigusto - Documentación Técnica
## Motor de Descubrimiento de Eventos Hiper-Personalizado con Microservicios

> **"Cero ruido, solo tus intereses"**

[![Java Version](https://img.shields.io/badge/Java-17-blue)](https://openjdk.org/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.2-green)](https://spring.io/projects/spring-boot)
[![Spring Cloud](https://img.shields.io/badge/Spring%20Cloud-2023-green)](https://spring.io/projects/spring-cloud)
[![Microservices](https://img.shields.io/badge/Architecture-Microservices-orange)]()

---

## Índice de Documentación

Este repositorio contiene toda la documentación técnica necesaria para implementar el MVP de Amigusto con arquitectura de **microservicios**. A continuación, encontrarás una guía de qué documento revisar según tu necesidad.

---

## 📚 Documentos Principales

### 1. [PLAN_TECNICO_AMIGUSTO.md](./PLAN_TECNICO_AMIGUSTO.md)
**Documento maestro con la arquitectura completa del sistema (MICROSERVICIOS)**

📋 **Contenido:**
- Arquitectura de microservicios (diagramas de alto nivel)
- Stack tecnológico completo:
  - **Backend**: Java 17 + Spring Boot 3.2 + **Spring Cloud** (Microservicios)
  - **iOS**: Swift 5.9 + SwiftUI
  - **Android**: Kotlin 1.9 + Jetpack Compose
  - **Web**: Angular 17 + Material
- **Database per Service Pattern** (PostgreSQL por servicio, MongoDB para Notifications)
- Arquitectura de API con API Gateway (Spring Cloud Gateway)
- Comunicación entre servicios (Feign + RabbitMQ)
- Plan de implementación por fases con setup de infraestructura de microservicios
- Consideraciones de seguridad, escalabilidad y costos
- KPIs y métricas de éxito

👥 **Para quién:**
- Tech Leads
- Arquitectos de Software
- Product Managers
- Stakeholders técnicos

---

### 1.1 [ARQUITECTURA_MICROSERVICIOS.md](./ARQUITECTURA_MICROSERVICIOS.md)
**Documentación técnica detallada de la arquitectura de microservicios**

📋 **Contenido:**
- Listado completo de los 7 microservicios (Auth, Event, User, Promoter, Notification, Storage, API Gateway)
- Infraestructura compartida (Eureka, Config Server, RabbitMQ, Redis, Zipkin)
- Patrones de comunicación:
  - **Síncrona**: Spring Cloud OpenFeign con Circuit Breakers
  - **Asíncrona**: RabbitMQ con Topic/Fanout exchanges
- Resilience patterns (Circuit Breaker, Retry, Timeout con Resilience4j)
- Distributed Tracing con Zipkin + Sleuth
- Docker Compose completo
- Manifiestos de Kubernetes
- Pipeline CI/CD por microservicio

👥 **Para quién:**
- Backend Developers
- DevOps Engineers
- Arquitectos de Software

---

### 1.2 [MICROSERVICIOS_RESUMEN.md](./MICROSERVICIOS_RESUMEN.md)
**Resumen ejecutivo de la arquitectura de microservicios**

📋 **Contenido:**
- Resumen de cada microservicio con responsabilidades
- Ventajas de microservicios (escalado independiente, resiliencia, etc.)
- Desafíos de microservicios (complejidad operacional, debugging)
- Recomendaciones para MVP (cuándo usar microservicios vs monolito)
- Estrategia de migración gradual

👥 **Para quién:**
- Product Managers
- Tech Leads que necesitan decisión arquitectónica
- Stakeholders no técnicos

---

### 2. [ARQUITECTURA_PROYECTO.md](./ARQUITECTURA_PROYECTO.md)
**Estructura de carpetas y organización del código (MICROSERVICIOS)**

📋 **Contenido:**
- Estructura de monorepo para microservicios
- Organización de cada microservicio (Auth, Event, User, Promoter, Notification, Storage)
- Estructura del Config Server repository
- Docker Compose completo con todos los servicios
- Kubernetes manifests organization
- CI/CD pipelines por microservicio
- Git ignore patterns por tecnología

👥 **Para quién:**
- Backend Developers
- DevOps Engineers
- Nuevos developers uniéndose al proyecto

---

### 3. [EJEMPLOS_CODIGO.md](./EJEMPLOS_CODIGO.md)
**Ejemplos de código para cada plataforma (incluye patrones de microservicios)**

📋 **Contenido:**
- Ejemplos de backend (Spring Boot + Spring Cloud):
  - **Feign Clients** con Circuit Breakers
  - **RabbitMQ Publishers y Consumers**
  - **Resilience4j** (Circuit Breaker, Retry, Timeout)
  - **API Gateway Filters** (JWT, Rate Limiting)
  - **Config Server** setup
  - **Distributed Tracing** con Sleuth
- Ejemplos de iOS (Swift + SwiftUI + MVVM)
- Ejemplos de Android (Kotlin + Compose + Clean Architecture)
- Ejemplos de Angular (Standalone Components + Reactive Forms)

👥 **Para quién:**
- Todos los developers
- Excelente para copiar/pegar código base

---

### 4. [GUIA_INICIO_RAPIDO.md](./GUIA_INICIO_RAPIDO.md)
**Setup inicial paso a paso para comenzar a desarrollar**

📋 **Contenido:**
- Prerequisites (Java, Docker, Xcode, Android Studio, Node.js)
- **Setup de microservicios** con Docker Compose
- Setup de proyectos iOS (Xcode) y Android (Android Studio)
- Setup de proyectos Angular (Portal + Admin)
- Configuración de PostgreSQL multi-database + MongoDB + Redis + RabbitMQ
- Levantar Eureka Server, Config Server, API Gateway
- Flujo de trabajo diario
- Troubleshooting común

👥 **Para quién:**
- Nuevos developers uniéndose al proyecto
- Cualquiera que necesite configurar su entorno de desarrollo

---

### 3. [DIAGRAMAS.md](./DIAGRAMAS.md)
**Diagramas visuales de arquitectura y flujos**

📋 **Contenido:**
- Diagramas de arquitectura general
- Flujo de curación de eventos
- Modelo de datos (Entity-Relationship)
- State machine de eventos
- Pipeline CI/CD
- Diagramas de despliegue

👥 **Para quién:**
- Todos (excelente punto de entrada visual)
- Arquitectos
- Para presentaciones a stakeholders

---

## 🚀 ¿Por Dónde Empezar?

### Si eres nuevo en el proyecto:

1. **Lee primero:** [PLAN_TECNICO_AMIGUSTO.md](./PLAN_TECNICO_AMIGUSTO.md) (Sección 1 y 2)
   - Entenderás el "por qué" del producto y el stack tecnológico

2. **Visualiza:** [DIAGRAMAS.md](./DIAGRAMAS.md)
   - Comprende visualmente la arquitectura

3. **Configura tu entorno:** [GUIA_INICIO_RAPIDO.md](./GUIA_INICIO_RAPIDO.md)
   - Setup de tu stack específico

---

### Si eres Backend Developer (Java Spring Boot):

1. [PLAN_TECNICO_AMIGUSTO.md](./PLAN_TECNICO_AMIGUSTO.md) → Secciones 2.3, 3, 4
2. [GUIA_INICIO_RAPIDO.md](./GUIA_INICIO_RAPIDO.md) → Sección de Backend

**Primera tarea recomendada:** Implementar módulo de autenticación con Spring Security + JWT

**Stack a dominar:**
- Java 17+
- Spring Boot 3.2+ (Web, Data JPA, Security)
- PostgreSQL + Spring Data JPA
- Redis + Spring Data Redis
- Maven/Gradle
- JUnit 5 + Mockito

---

### Si eres iOS Developer (Swift):

1. [PLAN_TECNICO_AMIGUSTO.md](./PLAN_TECNICO_AMIGUSTO.md) → Sección 2.1.1
2. [GUIA_INICIO_RAPIDO.md](./GUIA_INICIO_RAPIDO.md) → Sección de iOS

**Primera tarea recomendada:** Implementar onboarding (selección de gustos)

**Stack a dominar:**
- Swift 5.9+
- SwiftUI
- Combine (reactive programming)
- URLSession/Alamofire
- CoreData + Keychain
- MapKit + CoreLocation

---

### Si eres Android Developer (Kotlin):

1. [PLAN_TECNICO_AMIGUSTO.md](./PLAN_TECNICO_AMIGUSTO.md) → Sección 2.1.2
2. [GUIA_INICIO_RAPIDO.md](./GUIA_INICIO_RAPIDO.md) → Sección de Android

**Primera tarea recomendada:** Implementar onboarding (selección de gustos)

**Stack a dominar:**
- Kotlin 1.9+
- Jetpack Compose
- ViewModel + StateFlow/LiveData
- Retrofit + OkHttp
- Room Database + DataStore
- Hilt (Dependency Injection)
- Google Maps SDK

---

### Si eres Frontend Developer (Angular):

1. [PLAN_TECNICO_AMIGUSTO.md](./PLAN_TECNICO_AMIGUSTO.md) → Sección 2.2
2. [GUIA_INICIO_RAPIDO.md](./GUIA_INICIO_RAPIDO.md) → Sección de Angular

**Primera tarea recomendada:** Implementar formulario de creación de eventos

**Stack a dominar:**
- Angular 17+ (Standalone Components)
- TypeScript 5.2+
- Angular Material / PrimeNG
- Reactive Forms
- RxJS
- NgRx (opcional, state management)

---

### Si eres Product Manager o Tech Lead:

1. **Lee completo:** [PLAN_TECNICO_AMIGUSTO.md](./PLAN_TECNICO_AMIGUSTO.md)
   - Especialmente las secciones 5 (Plan de Implementación) y 7 (Estimaciones)
2. **Revisa:** [DIAGRAMAS.md](./DIAGRAMAS.md)
3. **Define prioridades** usando las fases del plan técnico

---

## 🎯 Conceptos Clave del Proyecto

### La Propuesta de Valor Única (PUV)

**"Cero ruido, solo tus intereses"**

Amigusto es un **filtro imparcial** que pone el "Gusto" del usuario por encima de todo. A diferencia de competidores como Fever (que priorizan sus propios tickets), nuestro modelo se basa en **relevancia**, dando igual prioridad a eventos gratuitos y de pago.

### Los Tres Pilares de la Arquitectura

1. **Apps Móviles Nativas (iOS + Android)**: Descubrimiento personalizado B2C
2. **Portal Web Angular (B2B)**: Creación y gestión de eventos para promotores
3. **Panel Admin Angular**: Curación humana de contenido (el "filtro anti-ruido")

### El Flujo de Curación (La "Salsa Secreta")

```
Promotor crea evento (Portal Angular)
    ↓
Evento entra a "PENDING_REVIEW" (NO se publica automáticamente)
    ↓
Curador humano revisa y aprueba (Panel Admin Angular)
    ↓
Evento se vuelve visible en Apps Móviles (iOS + Android)
    ↓
Usuarios descubren eventos relevantes a sus gustos
```

Este flujo garantiza **cero spam** y **alta calidad** de contenido.

---

## 🛠️ Stack Tecnológico (Microservicios)

### Backend (Microservicios)

| Componente | Tecnología |
|------------|------------|
| **Framework** | Java 17 + Spring Boot 3.2 |
| **Microservices Infrastructure** | Spring Cloud 2023 |
| **API Gateway** | Spring Cloud Gateway 4.0+ |
| **Service Discovery** | Netflix Eureka |
| **Config Server** | Spring Cloud Config |
| **Load Balancing** | Spring Cloud LoadBalancer |
| **Inter-Service Communication (Sync)** | Spring Cloud OpenFeign |
| **Inter-Service Communication (Async)** | Spring AMQP + RabbitMQ 3.12 |
| **Circuit Breaker** | Resilience4j |
| **Distributed Tracing** | Spring Cloud Sleuth + Zipkin |
| **Base de Datos** | PostgreSQL 16 + PostGIS, MongoDB 7 |
| **Database Pattern** | Database per Service |
| **ORM** | Spring Data JPA + Hibernate |
| **Cache** | Redis 7 (shared) + Lettuce |
| **Security** | Spring Security + JWT |
| **API Docs** | Springdoc OpenAPI (Swagger) por servicio |
| **Build Tool** | Maven 3.9+ |
| **Testing** | JUnit 5 + Mockito + Testcontainers |
| **Storage** | AWS S3 / Cloudinary |
| **Email** | Spring Mail + Thymeleaf |

### Frontend & Mobile

| Componente | Tecnología |
|------------|------------|
| **App iOS** | Swift 5.9 + SwiftUI + MVVM |
| **App Android** | Kotlin 1.9 + Jetpack Compose + Clean Architecture |
| **Portal Web B2B** | Angular 17 (Standalone) + Material |
| **Panel Admin** | Angular 17 (Standalone) + Material |
| **Build (iOS)** | Xcode 15 + Swift Package Manager |
| **Build (Android)** | Gradle 8.2 + Kotlin DSL |
| **Build (Web)** | npm + Angular CLI |
| **Testing (iOS)** | XCTest + XCUITest |
| **Testing (Android)** | JUnit + Espresso + Compose Testing |
| **Testing (Web)** | Jasmine + Karma + Cypress |

### DevOps & Infrastructure

| Componente | Tecnología |
|------------|------------|
| **Containerization** | Docker + Docker Compose |
| **Orchestration** | Kubernetes |
| **CI/CD** | GitHub Actions |
| **Monitoring** | Prometheus + Grafana + Zipkin |
| **Logging** | ELK Stack (Elasticsearch, Logback, Kibana) |
| **APM** | Micrometer + Actuator |

---

## 📊 Plan de Implementación (Resumen)

| Fase | Duración | Entregable Principal |
|------|----------|----------------------|
| **Fase 0** | 1-2 semanas | Setup de arquitectura (Spring Boot, Xcode, Android Studio, Angular) |
| **Fase 1** | 3 semanas | Backend Spring Boot funcional con API REST |
| **Fase 2** | 5 semanas | Apps iOS y Android nativas completas |
| **Fase 3** | 3 semanas | Portal Web Angular B2B completo |
| **Fase 4** | 2 semanas | Panel Admin Angular operativo |
| **Fase 5** | 2 semanas | Testing y QA completo (JUnit, XCTest, Espresso, Cypress) |
| **Fase 6** | 2 semanas | Preparación para lanzamiento |
| **Fase 7** | 3 semanas | Beta pública y feedback |
| **Fase 8** | 1 semana | Lanzamiento público (App Store + Google Play) |

**Total estimado:** 22-23 semanas (~5.5 meses)

---

## 🔐 Consideraciones de Seguridad

- ✅ Autenticación JWT con Spring Security
- ✅ BCrypt password hashing
- ✅ HTTPS obligatorio en producción
- ✅ Bean Validation en backend (Jakarta Validation)
- ✅ Keychain (iOS) y EncryptedSharedPreferences (Android)
- ✅ HttpOnly cookies para refresh tokens
- ✅ Rate limiting con Spring (Bucket4j)
- ✅ CORS configurado por dominio
- ✅ SQL Injection protection (JPA/Hibernate)

---

## 📈 Métricas de Éxito (MVP)

### Producto:
- **DAU (Day 1):** 500+ usuarios activos
- **Eventos guardados por usuario:** >3
- **Share rate:** >10% de eventos vistos

### Contenido:
- **Eventos aprobados por semana:** 50+
- **Tiempo de aprobación:** <24 horas
- **% eventos rechazados:** <20%

### Técnicas:
- **API latency p95:** <500ms
- **Uptime:** >99.9%
- **Crash-free rate (apps):** >99.5%

---

## 🤝 Convenciones de Desarrollo

### Estructura de Repositorios:

**Opción recomendada: 3 repositorios separados**

```
amigusto-backend/       (Java Spring Boot)
amigusto-ios/          (Swift + SwiftUI)
amigusto-android/      (Kotlin + Jetpack Compose)
amigusto-web/          (Angular - Portal + Admin)
```

### Git Workflow:

- **Branch principal:** `main` (producción)
- **Branch de staging:** `staging`
- **Branch de desarrollo:** `develop`
- **Feature branches:** `feature/nombre-feature`
- **Bugfix branches:** `bugfix/nombre-bug`

### Commits:

Seguir **Conventional Commits**:

```
feat(backend): add event filtering by gustos
feat(ios): implement event detail screen
feat(android): add infinite scroll to discover feed
feat(web): create event form with validation
fix(backend): resolve JWT token expiration issue
```

### Code Review:

- Toda feature debe pasar por PR (Pull Request)
- Mínimo 1 approval de otro developer
- Tests deben pasar (CI/CD)
- Linting debe pasar:
  - Backend: Checkstyle / SpotBugs
  - iOS: SwiftLint
  - Android: ktlint / detekt
  - Angular: ESLint

---

## 📞 Contacto y Soporte

- **Slack del equipo:** [Crear canales]
- **Notion/Confluence:** [Link a documentación]
- **JIRA/Linear:** [Link a board de tareas]
- **GitHub Issues:** Para bugs técnicos

---

## 🎓 Recursos de Aprendizaje

Si eres nuevo en alguna de las tecnologías del stack:

### Backend (Java Spring Boot):
- [Spring Boot Documentation](https://docs.spring.io/spring-boot/docs/current/reference/html/)
- [Spring Data JPA Guide](https://spring.io/guides/gs/accessing-data-jpa/)
- [Spring Security Reference](https://docs.spring.io/spring-security/reference/index.html)
- [Baeldung Spring Tutorials](https://www.baeldung.com/spring-tutorial)

### iOS (Swift + SwiftUI):
- [Swift Documentation](https://docs.swift.org/swift-book/)
- [SwiftUI Tutorials](https://developer.apple.com/tutorials/swiftui)
- [Hacking with Swift](https://www.hackingwithswift.com/)
- [100 Days of SwiftUI](https://www.hackingwithswift.com/100/swiftui)

### Android (Kotlin + Compose):
- [Kotlin Documentation](https://kotlinlang.org/docs/home.html)
- [Jetpack Compose Tutorial](https://developer.android.com/jetpack/compose/tutorial)
- [Android Developers Codelabs](https://developer.android.com/courses)
- [Compose Pathway](https://developer.android.com/courses/pathways/compose)

### Web (Angular):
- [Angular Documentation](https://angular.io/docs)
- [Angular Material](https://material.angular.io/)
- [RxJS Official Guide](https://rxjs.dev/guide/overview)
- [Angular University Courses](https://angular-university.io/)

---

## ⚡️ Quick Start (TL;DR)

### Backend (Microservicios con Docker Compose)

```bash
# Clonar repositorio
git clone https://github.com/amigusto/amigusto-backend.git
cd amigusto-backend

# Levantar TODA la infraestructura de microservicios
docker-compose up -d

# Verificar que todos los servicios están corriendo
docker-compose ps

# Ver logs
docker-compose logs -f
```

**Servicios Disponibles:**
- ✅ API Gateway: http://localhost:8080
- ✅ Eureka Dashboard: http://localhost:8761
- ✅ RabbitMQ Management: http://localhost:15672 (guest/guest)
- ✅ Zipkin Tracing: http://localhost:9411
- ✅ Auth Service (Swagger): http://localhost:8081/swagger-ui.html
- ✅ Event Service (Swagger): http://localhost:8082/swagger-ui.html
- ✅ User Service (Swagger): http://localhost:8083/swagger-ui.html
- ✅ Promoter Service (Swagger): http://localhost:8084/swagger-ui.html
- ✅ Storage Service (Swagger): http://localhost:8086/swagger-ui.html

**Health Checks:**
```bash
# Verificar salud de todos los servicios
curl http://localhost:8081/actuator/health  # Auth Service
curl http://localhost:8082/actuator/health  # Event Service
curl http://localhost:8083/actuator/health  # User Service
```

### iOS

```bash
cd amigusto-ios
open AmigustoiOS.xcodeproj
# Ejecutar en Xcode (⌘R)
```

### Android

```bash
cd amigusto-android
./gradlew assembleDebug
# Abrir en Android Studio y Run
```

### Web (Angular)

```bash
cd amigusto-web
npm install

# Portal para Promotores
ng serve
# ✅ Portal: http://localhost:4200

# Panel Admin (en otra terminal)
ng serve --project admin-panel --port 4201
# ✅ Admin: http://localhost:4201
```

---

## 👥 Equipo Recomendado (MVP)

- **1 Tech Lead** (Full-Stack, Java + Angular)
- **2 Backend Developers** (Java Spring Boot)
- **1 iOS Developer** (Swift + SwiftUI)
- **1 Android Developer** (Kotlin + Jetpack Compose)
- **2 Frontend Developers** (Angular + TypeScript)
- **1 UI/UX Designer**
- **1 QA Engineer**
- **1 DevOps Engineer** (part-time)

**Total:** 9-10 personas

---

## 💰 Estimación de Costos (MVP)

### Infraestructura Mensual (Fase Inicial):

```
Backend (Railway/Render): $20
PostgreSQL (Supabase): $25
Redis (Upstash): $0 (free tier)
Storage (Cloudinary): $0 (free tier, hasta 10GB)
CDN (Cloudflare): $0 (free tier)
Web (Vercel): $0 (free tier)
Firebase (Analytics): $0 (free tier)
Apple Developer: $99/año
Google Play Developer: $25 (one-time)
Google Maps API: ~$50-100 (según uso)

Total Mes 1-3: ~$100-150/mes
```

---

## 📝 Licencia

Este proyecto es propiedad de Amigusto. Todos los derechos reservados.

---

**Última actualización:** 2025-10-26
**Versión de documentación:** 2.0 (Stack actualizado: Java/Swift/Kotlin/Angular)
**Preparado por:** Equipo de Arquitectura de Amigusto

---

Para dudas o sugerencias sobre esta documentación, crear un issue en GitHub o contactar al Tech Lead del proyecto.

**¡Bienvenido al equipo de Amigusto! 🎉**
