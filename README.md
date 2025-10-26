# Amigusto - Documentación Técnica
## Motor de Descubrimiento de Eventos Hiper-Personalizado

> **"Cero ruido, solo tus intereses"**

---

## Índice de Documentación

Este repositorio contiene toda la documentación técnica necesaria para implementar el MVP de Amigusto. A continuación, encontrarás una guía de qué documento revisar según tu necesidad.

---

## 📚 Documentos Principales

### 1. [PLAN_TECNICO_AMIGUSTO.md](./PLAN_TECNICO_AMIGUSTO.md)
**Documento maestro con la arquitectura completa del sistema**

📋 **Contenido:**
- Arquitectura general del sistema (diagramas de alto nivel)
- Stack tecnológico completo:
  - **Backend**: Java 17 + Spring Boot 3.2
  - **iOS**: Swift 5.9 + SwiftUI
  - **Android**: Kotlin 1.9 + Jetpack Compose
  - **Web**: Angular 17 + Material
- Diseño completo de base de datos (SQL + JPA)
- Arquitectura de API REST (endpoints, DTOs, Spring Security)
- Plan de implementación por fases (8 fases, 23 semanas)
- Consideraciones de seguridad, escalabilidad y costos
- KPIs y métricas de éxito

👥 **Para quién:**
- Tech Leads
- Arquitectos de Software
- Product Managers
- Stakeholders técnicos

---

### 2. [GUIA_INICIO_RAPIDO.md](./GUIA_INICIO_RAPIDO.md)
**Setup inicial paso a paso para comenzar a desarrollar**

📋 **Contenido:**
- Prerequisites (Java, Xcode, Android Studio, Node.js)
- Setup de Spring Boot con Maven/Gradle
- Setup de proyectos iOS (Xcode) y Android (Android Studio)
- Setup de proyectos Angular (Portal + Admin)
- Configuración de PostgreSQL + Redis
- Docker Compose para desarrollo local
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

## 🛠️ Stack Tecnológico (Resumen)

| Componente | Tecnología |
|------------|------------|
| **Backend API** | Java 17 + Spring Boot 3.2 |
| **Base de Datos** | PostgreSQL 16 + PostGIS |
| **ORM** | Spring Data JPA + Hibernate |
| **Cache** | Redis 7 + Spring Data Redis |
| **App iOS** | Swift 5.9 + SwiftUI |
| **App Android** | Kotlin 1.9 + Jetpack Compose |
| **Portal Web B2B** | Angular 17 + Material |
| **Panel Admin** | Angular 17 + Material |
| **Build (Backend)** | Maven 3.9 / Gradle 8 |
| **Build (iOS)** | Xcode 15 + SPM |
| **Build (Android)** | Gradle 8.2 + Kotlin DSL |
| **Build (Web)** | npm + Angular CLI |
| **Testing (Backend)** | JUnit 5 + Mockito |
| **Testing (iOS)** | XCTest + XCUITest |
| **Testing (Android)** | JUnit + Espresso |
| **Testing (Web)** | Jasmine + Karma + Cypress |
| **API Docs** | Springdoc OpenAPI (Swagger) |
| **Security** | Spring Security + JWT |

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

```bash
# Backend (Spring Boot)
cd amigusto-backend
./mvnw spring-boot:run
# ✅ API: http://localhost:8080
# ✅ Swagger UI: http://localhost:8080/swagger-ui.html

# iOS
cd amigusto-ios
open AmigustoiOS.xcodeproj
# Ejecutar en Xcode (⌘R)

# Android
cd amigusto-android
./gradlew assembleDebug
# Abrir en Android Studio y Run

# Web (Angular)
cd amigusto-web
npm install
ng serve
# ✅ Portal: http://localhost:4200
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
