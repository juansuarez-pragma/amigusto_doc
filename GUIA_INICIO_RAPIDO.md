# Guía de Inicio Rápido - Amigusto
## Setup Inicial y Primeros Pasos para el Equipo de Desarrollo

---

## 1. PREREQUISITES (REQUISITOS PREVIOS)

Antes de comenzar, asegúrate de tener instalado:

### 1.1 Software Requerido por Rol

#### Para Backend Developers (Java Spring Boot):

```bash
# Java JDK 17 o superior
java -version  # Debe ser >= 17

# Maven (o Gradle)
mvn -version   # Maven 3.9+
# O
gradle -version # Gradle 8+

# PostgreSQL Client
psql --version

# Git
git --version

# Docker (para BD local)
docker --version
docker-compose --version

# IDE recomendado
# - IntelliJ IDEA (Community o Ultimate)
# - Eclipse con Spring Tools
# - VS Code con Extension Pack for Java
```

#### Para iOS Developers (Swift):

```bash
# Mac con macOS Ventura o superior (REQUERIDO)
sw_vers

# Xcode 15 o superior
xcodebuild -version

# CocoaPods (opcional, si no usas SPM)
pod --version

# Homebrew (package manager)
brew --version

# Git
git --version
```

#### Para Android Developers (Kotlin):

```bash
# Java JDK 17
java -version

# Android Studio Iguana o superior
# Descargar desde: https://developer.android.com/studio

# Android SDK (incluido con Android Studio)
# SDK Platform-Tools
adb --version

# Gradle (incluido con Android Studio)
gradle -version

# Git
git --version
```

#### Para Frontend Developers (Angular):

```bash
# Node.js 18+ LTS
node --version  # Debe ser >= 18.x
npm --version   # Debe ser >= 10.x

# Angular CLI
ng version      # Debe ser >= 17.x
# Si no está instalado:
npm install -g @angular/cli

# Git
git --version

# IDE recomendado
# - VS Code con Angular Language Service
# - WebStorm
```

### 1.2 Cuentas y API Keys Necesarias:

- [ ] Cuenta de GitHub (para repositorios)
- [ ] Cuenta de Vercel/Netlify (para deploys de Angular)
- [ ] Apple Developer Account ($99/año) - para iOS
- [ ] Google Play Developer Account ($25 one-time) - para Android
- [ ] Google Cloud Console (para Maps API)
- [ ] Cloudinary o AWS S3 (para storage de imágenes)
- [ ] SendGrid API Key (para emails)
- [ ] Firebase Account (para analytics móvil)

---

## 2. SETUP DEL BACKEND (JAVA SPRING BOOT)

### 2.1 Crear Proyecto Spring Boot

**Opción 1: Usando Spring Initializr (Recomendado)**

1. Ir a [https://start.spring.io/](https://start.spring.io/)
2. Configurar:
   - **Project:** Maven o Gradle
   - **Language:** Java
   - **Spring Boot:** 3.2.x (última stable)
   - **Java:** 17
   - **Packaging:** Jar
   - **Group:** com.amigusto
   - **Artifact:** amigusto-backend
   - **Name:** Amigusto Backend
   - **Package name:** com.amigusto

3. Agregar dependencias:
   ```
   - Spring Web
   - Spring Data JPA
   - Spring Security
   - PostgreSQL Driver
   - Spring Data Redis
   - Validation
   - Lombok
   - Spring Boot DevTools
   - Spring Boot Actuator
   - Springdoc OpenAPI (para Swagger)
   ```

4. Click "Generate" y descomprimir el proyecto

**Opción 2: Usando IntelliJ IDEA**

1. File → New → Project
2. Seleccionar "Spring Initializr"
3. Configurar igual que arriba

### 2.2 Estructura Inicial del Proyecto

```bash
cd amigusto-backend

# Estructura generada:
amigusto-backend/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/amigusto/
│   │   │       └── AmigustoApplication.java
│   │   └── resources/
│   │       ├── application.yml
│   │       └── application-dev.yml
│   └── test/
├── pom.xml (o build.gradle)
└── README.md
```

### 2.3 Configurar application.yml

```yaml
# src/main/resources/application.yml
spring:
  application:
    name: amigusto-backend

  profiles:
    active: dev

---
# src/main/resources/application-dev.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/amigusto_dev
    username: amigusto
    password: dev_password
    driver-class-name: org.postgresql.Driver

  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        dialect: org.hibernate.dialect.PostgreSQLDialect

  data:
    redis:
      host: localhost
      port: 6379

server:
  port: 8080

# JWT Configuration
jwt:
  secret: your-secret-key-change-in-production
  expiration: 900000 # 15 minutes
  refresh-expiration: 604800000 # 7 days

# Springdoc OpenAPI
springdoc:
  api-docs:
    path: /api-docs
  swagger-ui:
    path: /swagger-ui.html
```

### 2.4 Agregar Dependencias Adicionales (pom.xml)

```xml
<!-- JWT -->
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.3</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.3</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.3</version>
    <scope>runtime</scope>
</dependency>

<!-- Springdoc OpenAPI -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.3.0</version>
</dependency>

<!-- AWS S3 (opcional) -->
<dependency>
    <groupId>com.amazonaws</groupId>
    <artifactId>aws-java-sdk-s3</artifactId>
    <version>1.12.565</version>
</dependency>
```

### 2.5 Ejecutar el Proyecto

```bash
# Con Maven
./mvnw spring-boot:run

# Con Gradle
./gradlew bootRun

# O desde IntelliJ: Click derecho en AmigustoApplication.java → Run
```

**Verificar:**
- API: http://localhost:8080
- Swagger UI: http://localhost:8080/swagger-ui.html
- Actuator: http://localhost:8080/actuator/health

---

## 3. SETUP DE POSTGRESQL + REDIS (DOCKER)

### 3.1 Crear docker-compose.yml

```yaml
# docker-compose.yml (en la raíz del proyecto backend)
version: '3.8'

services:
  postgres:
    image: postgres:16
    container_name: amigusto-postgres
    environment:
      POSTGRES_DB: amigusto_dev
      POSTGRES_USER: amigusto
      POSTGRES_PASSWORD: dev_password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - amigusto-network

  redis:
    image: redis:7-alpine
    container_name: amigusto-redis
    ports:
      - "6379:6379"
    networks:
      - amigusto-network

  # Opcional: PgAdmin para gestión visual
  pgadmin:
    image: dpage/pgadmin4
    container_name: amigusto-pgadmin
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@amigusto.com
      PGADMIN_DEFAULT_PASSWORD: admin
    ports:
      - "5050:80"
    networks:
      - amigusto-network

volumes:
  postgres_data:

networks:
  amigusto-network:
    driver: bridge
```

### 3.2 Iniciar Servicios

```bash
# Iniciar PostgreSQL y Redis
docker-compose up -d

# Ver logs
docker-compose logs -f

# Detener servicios
docker-compose down

# Detener y eliminar volúmenes (¡cuidado! Borra los datos)
docker-compose down -v
```

### 3.3 Ejecutar Scripts SQL Iniciales

```bash
# Conectar a PostgreSQL
docker exec -it amigusto-postgres psql -U amigusto -d amigusto_dev

# O usar cliente local
psql -h localhost -U amigusto -d amigusto_dev

# Ejecutar script de schema (ver PLAN_TECNICO_AMIGUSTO.md sección 3.1)
\i path/to/initial_schema.sql
```

---

## 4. SETUP DE iOS (SWIFT + SWIFTUI)

### 4.1 Crear Proyecto en Xcode

1. Abrir Xcode
2. File → New → Project
3. Seleccionar "App" template
4. Configurar:
   - **Product Name:** AmigustoiOS
   - **Team:** Tu Apple Developer Team
   - **Organization Identifier:** com.amigusto
   - **Bundle Identifier:** com.amigusto.AmigustoiOS
   - **Interface:** SwiftUI
   - **Language:** Swift
   - **Storage:** None (usaremos CoreData después)

### 4.2 Configurar Swift Package Manager (SPM)

File → Add Package Dependencies:

```
// Agregar estas librerías:
Alamofire: https://github.com/Alamofire/Alamofire
SDWebImageSwiftUI: https://github.com/SDWebImage/SDWebImageSwiftUI
```

### 4.3 Estructura de Carpetas Recomendada

Crear grupos (folders) en Xcode:

```
AmigustoiOS/
├── App/
│   └── AmigustoiOSApp.swift
├── Models/
│   ├── Event.swift
│   ├── Gusto.swift
│   └── User.swift
├── ViewModels/
│   ├── EventsViewModel.swift
│   └── OnboardingViewModel.swift
├── Views/
│   ├── Onboarding/
│   ├── Discover/
│   └── MyPlans/
├── Services/
│   ├── APIService.swift
│   └── LocationService.swift
├── Utilities/
│   ├── Constants.swift
│   └── Extensions/
└── Resources/
    └── Assets.xcassets
```

### 4.4 Configurar Info.plist

Agregar permisos necesarios:

```xml
<key>NSLocationWhenInUseUsageDescription</key>
<string>Necesitamos tu ubicación para mostrarte eventos cerca de ti</string>

<key>NSLocationAlwaysAndWhenInUseUsageDescription</key>
<string>Necesitamos tu ubicación para mostrarte eventos cerca de ti</string>
```

### 4.5 Configurar Esquema de URL (Deep Linking)

1. Target → Info → URL Types
2. Agregar:
   - **Identifier:** com.amigusto
   - **URL Schemes:** amigusto

### 4.6 Ejecutar Proyecto

- Seleccionar simulador (iPhone 15 Pro)
- Presionar ⌘R o click en Play
- O dispositivo físico (requiere Apple Developer Account)

---

## 5. SETUP DE ANDROID (KOTLIN + JETPACK COMPOSE)

### 5.1 Crear Proyecto en Android Studio

1. Abrir Android Studio
2. File → New → New Project
3. Seleccionar "Empty Activity" (Compose)
4. Configurar:
   - **Name:** AmigustoAndroid
   - **Package name:** com.amigusto
   - **Save location:** /path/to/your/projects
   - **Language:** Kotlin
   - **Minimum SDK:** API 24 (Android 7.0) - cubre ~95% de dispositivos
   - **Build configuration language:** Kotlin DSL

### 5.2 Configurar build.gradle.kts (Module: app)

```kotlin
plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.android)
    alias(libs.plugins.kotlin.kapt)
    alias(libs.plugins.hilt.android)
}

android {
    namespace = "com.amigusto"
    compileSdk = 34

    defaultConfig {
        applicationId = "com.amigusto"
        minSdk = 24
        targetSdk = 34
        versionCode = 1
        versionName = "1.0.0"

        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
        vectorDrawables {
            useSupportLibrary = true
        }
    }

    buildTypes {
        release {
            isMinifyEnabled = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }

    kotlinOptions {
        jvmTarget = "17"
    }

    buildFeatures {
        compose = true
    }

    composeOptions {
        kotlinCompilerExtensionVersion = "1.5.3"
    }

    packaging {
        resources {
            excludes += "/META-INF/{AL2.0,LGPL2.1}"
        }
    }
}

dependencies {
    // Core
    implementation(libs.androidx.core.ktx)
    implementation(libs.androidx.lifecycle.runtime.ktx)
    implementation(libs.androidx.activity.compose)

    // Compose
    implementation(platform(libs.androidx.compose.bom))
    implementation(libs.androidx.compose.ui)
    implementation(libs.androidx.compose.ui.graphics)
    implementation(libs.androidx.compose.ui.tooling.preview)
    implementation(libs.androidx.compose.material3)

    // Navigation
    implementation(libs.androidx.navigation.compose)

    // Hilt (Dependency Injection)
    implementation(libs.hilt.android)
    kapt(libs.hilt.android.compiler)
    implementation(libs.androidx.hilt.navigation.compose)

    // Retrofit (Networking)
    implementation(libs.retrofit)
    implementation(libs.retrofit.converter.gson)
    implementation(libs.okhttp)
    implementation(libs.okhttp.logging.interceptor)

    // Room (Database)
    implementation(libs.androidx.room.runtime)
    implementation(libs.androidx.room.ktx)
    kapt(libs.androidx.room.compiler)

    // DataStore
    implementation(libs.androidx.datastore.preferences)

    // Coil (Image Loading)
    implementation(libs.coil.compose)

    // Google Maps
    implementation(libs.play.services.maps)
    implementation(libs.play.services.location)
    implementation(libs.maps.compose)

    // Firebase
    implementation(platform(libs.firebase.bom))
    implementation(libs.firebase.analytics)

    // Testing
    testImplementation(libs.junit)
    androidTestImplementation(libs.androidx.junit)
    androidTestImplementation(libs.androidx.espresso.core)
    androidTestImplementation(platform(libs.androidx.compose.bom))
    androidTestImplementation(libs.androidx.compose.ui.test.junit4)
    debugImplementation(libs.androidx.compose.ui.tooling)
    debugImplementation(libs.androidx.compose.ui.test.manifest)
}
```

### 5.3 Configurar libs.versions.toml

```toml
[versions]
agp = "8.2.0"
kotlin = "1.9.10"
compose-bom = "2024.02.00"
retrofit = "2.9.0"
okhttp = "4.12.0"
hilt = "2.48"
room = "2.6.1"

[libraries]
androidx-core-ktx = { group = "androidx.core", name = "core-ktx", version = "1.12.0" }
androidx-lifecycle-runtime-ktx = { group = "androidx.lifecycle", name = "lifecycle-runtime-ktx", version = "2.7.0" }
androidx-activity-compose = { group = "androidx.activity", name = "activity-compose", version = "1.8.2" }
androidx-compose-bom = { group = "androidx.compose", name = "compose-bom", version.ref = "compose-bom" }
androidx-compose-ui = { group = "androidx.compose.ui", name = "ui" }
androidx-compose-ui-graphics = { group = "androidx.compose.ui", name = "ui-graphics" }
androidx-compose-ui-tooling-preview = { group = "androidx.compose.ui", name = "ui-tooling-preview" }
androidx-compose-material3 = { group = "androidx.compose.material3", name = "material3" }
androidx-navigation-compose = { group = "androidx.navigation", name = "navigation-compose", version = "2.7.6" }

# Hilt
hilt-android = { group = "com.google.dagger", name = "hilt-android", version.ref = "hilt" }
hilt-android-compiler = { group = "com.google.dagger", name = "hilt-android-compiler", version.ref = "hilt" }
androidx-hilt-navigation-compose = { group = "androidx.hilt", name = "hilt-navigation-compose", version = "1.1.0" }

# Retrofit
retrofit = { group = "com.squareup.retrofit2", name = "retrofit", version.ref = "retrofit" }
retrofit-converter-gson = { group = "com.squareup.retrofit2", name = "converter-gson", version.ref = "retrofit" }
okhttp = { group = "com.squareup.okhttp3", name = "okhttp", version.ref = "okhttp" }
okhttp-logging-interceptor = { group = "com.squareup.okhttp3", name = "logging-interceptor", version.ref = "okhttp" }

# Room
androidx-room-runtime = { group = "androidx.room", name = "room-runtime", version.ref = "room" }
androidx-room-ktx = { group = "androidx.room", name = "room-ktx", version.ref = "room" }
androidx-room-compiler = { group = "androidx.room", name = "room-compiler", version.ref = "room" }

# DataStore
androidx-datastore-preferences = { group = "androidx.datastore", name = "datastore-preferences", version = "1.0.0" }

# Coil
coil-compose = { group = "io.coil-kt", name = "coil-compose", version = "2.5.0" }

# Maps
play-services-maps = { group = "com.google.android.gms", name = "play-services-maps", version = "18.2.0" }
play-services-location = { group = "com.google.android.gms", name = "play-services-location", version = "21.1.0" }
maps-compose = { group = "com.google.maps.android", name = "maps-compose", version = "4.3.0" }

# Firebase
firebase-bom = { group = "com.google.firebase", name = "firebase-bom", version = "32.7.0" }
firebase-analytics = { group = "com.google.firebase", name = "firebase-analytics" }

# Testing
junit = { group = "junit", name = "junit", version = "4.13.2" }
androidx-junit = { group = "androidx.test.ext", name = "junit", version = "1.1.5" }
androidx-espresso-core = { group = "androidx.test.espresso", name = "espresso-core", version = "3.5.1" }
androidx-compose-ui-test-junit4 = { group = "androidx.compose.ui", name = "ui-test-junit4" }
androidx-compose-ui-tooling = { group = "androidx.compose.ui", name = "ui-tooling" }
androidx-compose-ui-test-manifest = { group = "androidx.compose.ui", name = "ui-test-manifest" }

[plugins]
android-application = { id = "com.android.application", version.ref = "agp" }
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
kotlin-kapt = { id = "org.jetbrains.kotlin.kapt", version.ref = "kotlin" }
hilt-android = { id = "com.google.dagger.hilt.android", version.ref = "hilt" }
```

### 5.4 Configurar AndroidManifest.xml

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <!-- Permisos -->
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />

    <application
        android:name=".AmigustoApplication"
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.AmigustoAndroid"
        android:usesCleartextTraffic="true">

        <!-- Google Maps API Key -->
        <meta-data
            android:name="com.google.android.geo.API_KEY"
            android:value="${MAPS_API_KEY}" />

        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:theme="@style/Theme.AmigustoAndroid">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>

</manifest>
```

### 5.5 Ejecutar Proyecto

```bash
# Desde terminal
./gradlew assembleDebug

# O en Android Studio:
# 1. Seleccionar dispositivo/emulador
# 2. Click en Run (▶️) o Shift+F10
```

---

## 6. SETUP DE WEB (ANGULAR)

### 6.1 Crear Proyecto Angular

```bash
# Instalar Angular CLI globalmente
npm install -g @angular/cli@17

# Crear proyecto
ng new amigusto-web

# Responder preguntas:
? Would you like to add Angular routing? Yes
? Which stylesheet format would you like to use? SCSS
? Do you want to enable Server-Side Rendering (SSR)? No

cd amigusto-web
```

### 6.2 Instalar Angular Material

```bash
ng add @angular/material

# Responder:
? Choose a prebuilt theme name: Indigo/Pink
? Set up global Angular Material typography styles? Yes
? Include the Angular animations module? Yes
```

### 6.3 Instalar Dependencias Adicionales

```bash
# RxJS (ya viene con Angular)
# Pero actualizar a última versión
npm install rxjs@latest

# Google Maps
npm install @angular/google-maps

# Date handling
npm install date-fns

# Charts (para panel admin)
npm install ng2-charts chart.js

# Rich Text Editor
npm install ngx-quill quill

# File Upload
npm install ngx-dropzone

# State Management (opcional)
npm install @ngrx/store @ngrx/effects @ngrx/entity @ngrx/store-devtools
```

### 6.4 Estructura de Carpetas (Generar con CLI)

```bash
# Generar módulos core y shared
ng generate module core
ng generate module shared

# Generar servicios
ng generate service core/services/api
ng generate service core/services/auth
ng generate service core/services/event

# Generar guards
ng generate guard core/guards/auth

# Generar interceptors
ng generate interceptor core/interceptors/auth
ng generate interceptor core/interceptors/error

# Generar componentes de features
ng generate component features/auth/login --standalone
ng generate component features/auth/register --standalone
ng generate component features/dashboard/dashboard --standalone
ng generate component features/events/create-event --standalone

# Generar componentes shared
ng generate component shared/components/event-status-badge --standalone
ng generate component shared/components/gusto-selector --standalone
```

### 6.5 Configurar environment files

```typescript
// src/environments/environment.ts
export const environment = {
  production: false,
  apiUrl: 'http://localhost:8080/api/v1',
  googleMapsApiKey: 'YOUR_GOOGLE_MAPS_API_KEY'
};

// src/environments/environment.prod.ts
export const environment = {
  production: true,
  apiUrl: 'https://api.amigusto.com/v1',
  googleMapsApiKey: 'YOUR_PRODUCTION_GOOGLE_MAPS_API_KEY'
};
```

### 6.6 Configurar angular.json para múltiples apps (Portal + Admin)

Si quieres tener Portal y Admin en el mismo workspace:

```bash
# Generar segunda aplicación
ng generate application admin-panel

# Esto crea:
# projects/admin-panel/
```

### 6.7 Ejecutar Proyecto

```bash
# Portal (app principal)
ng serve
# ✅ http://localhost:4200

# Admin (si creaste segunda app)
ng serve admin-panel --port 4201
# ✅ http://localhost:4201

# Con proxy para evitar CORS
ng serve --proxy-config proxy.conf.json
```

**Crear proxy.conf.json:**

```json
{
  "/api": {
    "target": "http://localhost:8080",
    "secure": false,
    "changeOrigin": true
  }
}
```

---

## 7. FLUJO DE TRABAJO DIARIO

### 7.1 Iniciar Desarrollo Local (Backend Developer)

```bash
# Terminal 1: Iniciar Docker (BD y Redis)
cd amigusto-backend
docker-compose up -d

# Terminal 2: Iniciar Spring Boot
./mvnw spring-boot:run
# O desde IntelliJ: Run AmigustoApplication

# Verificar:
# ✅ API: http://localhost:8080
# ✅ Swagger: http://localhost:8080/swagger-ui.html
# ✅ H2 Console (si usas): http://localhost:8080/h2-console
```

### 7.2 Iniciar Desarrollo Local (iOS Developer)

```bash
cd amigusto-ios
open AmigustoiOS.xcodeproj

# En Xcode:
# 1. Seleccionar scheme: AmigustoiOS
# 2. Seleccionar simulator: iPhone 15 Pro
# 3. Presionar ⌘R
```

### 7.3 Iniciar Desarrollo Local (Android Developer)

```bash
cd amigusto-android

# Opción 1: Desde Android Studio
# File → Open → amigusto-android
# Click Run

# Opción 2: Desde terminal
./gradlew installDebug
adb shell am start -n com.amigusto/.MainActivity
```

### 7.4 Iniciar Desarrollo Local (Angular Developer)

```bash
cd amigusto-web
npm start
# O
ng serve --open

# ✅ http://localhost:4200
```

---

## 8. COMANDOS ÚTILES POR PLATAFORMA

### 8.1 Backend (Spring Boot)

```bash
# Build
./mvnw clean install

# Run tests
./mvnw test

# Run specific test
./mvnw test -Dtest=EventServiceTest

# Generate JAR
./mvnw package

# Run JAR
java -jar target/amigusto-backend-1.0.0.jar

# Format code
./mvnw spotless:apply

# Check dependencies
./mvnw dependency:tree
```

### 8.2 iOS (Swift)

```bash
# Build desde terminal
xcodebuild -scheme AmigustoiOS -destination 'platform=iOS Simulator,name=iPhone 15 Pro' build

# Run tests
xcodebuild test -scheme AmigustoiOS -destination 'platform=iOS Simulator,name=iPhone 15 Pro'

# Format code con SwiftFormat (si instalado)
swiftformat .

# Linting
swiftlint

# Clean build folder
xcodebuild clean
```

### 8.3 Android (Kotlin)

```bash
# Build
./gradlew build

# Run tests
./gradlew test

# Run instrumented tests
./gradlew connectedAndroidTest

# Lint
./gradlew lint

# Format code (si usas ktlint)
./gradlew ktlintFormat

# Generate APK
./gradlew assembleDebug

# Generate signed APK
./gradlew assembleRelease

# Install on device
./gradlew installDebug
```

### 8.4 Angular

```bash
# Build
ng build

# Build production
ng build --configuration production

# Run tests
ng test

# E2E tests (si configuraste Cypress)
ng e2e

# Lint
ng lint

# Format con Prettier
npx prettier --write "src/**/*.{ts,html,scss}"

# Analyze bundle size
ng build --stats-json
npx webpack-bundle-analyzer dist/amigusto-web/stats.json
```

---

## 9. TROUBLESHOOTING COMÚN

### 9.1 Backend (Spring Boot)

**Error: "Cannot connect to PostgreSQL"**

```bash
# Verificar que Docker esté corriendo
docker ps

# Reiniciar contenedor
docker-compose restart postgres

# Ver logs
docker-compose logs postgres
```

**Error: "Port 8080 already in use"**

```bash
# Encontrar proceso
lsof -ti:8080

# Matar proceso
kill -9 $(lsof -ti:8080)

# O cambiar puerto en application.yml
server:
  port: 8081
```

**Error: "Bean creation failed"**

- Verificar que todas las dependencias estén en pom.xml
- Limpiar y rebuild: `./mvnw clean install`
- Verificar configuración de Spring en application.yml

### 9.2 iOS

**Error: "Command PhaseScriptExecution failed"**

```bash
# Limpiar build folder
Product → Clean Build Folder (⇧⌘K)

# O desde terminal
xcodebuild clean
```

**Error: "No such module 'Alamofire'"**

- File → Packages → Resolve Package Versions
- O eliminar Package.resolved y resolver de nuevo

**Error: Simulator not appearing**

```bash
# Reiniciar CoreSimulator service
xcrun simctl shutdown all
sudo killall -9 com.apple.CoreSimulator.CoreSimulatorService
```

### 9.3 Android

**Error: "SDK location not found"**

Crear `local.properties`:
```properties
sdk.dir=/Users/YOUR_USER/Library/Android/sdk
```

**Error: "Gradle sync failed"**

```bash
# Limpiar cache de Gradle
./gradlew clean
./gradlew --stop

# Invalidate caches en Android Studio
File → Invalidate Caches / Restart
```

**Error: "ADB not found"**

```bash
# Agregar a PATH
export ANDROID_HOME=$HOME/Library/Android/sdk
export PATH=$PATH:$ANDROID_HOME/platform-tools
```

### 9.4 Angular

**Error: "Module not found"**

```bash
# Limpiar node_modules
rm -rf node_modules package-lock.json
npm install
```

**Error: "Port 4200 already in use"**

```bash
# Usar otro puerto
ng serve --port 4201

# O matar proceso
lsof -ti:4200 | xargs kill -9
```

**Error: Compilation errors**

```bash
# Limpiar cache de Angular
rm -rf .angular
ng serve
```

---

## 10. TESTING

### 10.1 Backend (JUnit 5)

```java
// Ejemplo de test
@SpringBootTest
@AutoConfigureMockMvc
class EventControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void shouldReturnEvents() throws Exception {
        mockMvc.perform(get("/api/v1/events")
                .param("city", "Bogotá"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.data").isArray());
    }
}
```

### 10.2 iOS (XCTest)

```swift
// Ejemplo de test
import XCTest
@testable import AmigustoiOS

final class EventServiceTests: XCTestCase {

    func testFetchEvents() async throws {
        let service = EventService()
        let events = try await service.fetchEvents(city: "Bogotá")
        XCTAssertGreaterThan(events.count, 0)
    }
}
```

### 10.3 Android (JUnit + Espresso)

```kotlin
// Ejemplo de test unitario
class EventViewModelTest {
    @Test
    fun `fetch events should update state`() = runTest {
        val viewModel = EventViewModel(fakeRepository)
        viewModel.fetchEvents("Bogotá")

        val state = viewModel.eventsState.value
        assertTrue(state is EventState.Success)
    }
}

// Ejemplo de test UI
@RunWith(AndroidJUnit4::class)
class DiscoverScreenTest {
    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun clickOnEvent_opensDetail() {
        composeTestRule.onNodeWithTag("event_card_0").performClick()
        composeTestRule.onNodeWithTag("event_detail").assertIsDisplayed()
    }
}
```

### 10.4 Angular (Jasmine + Karma)

```typescript
// Ejemplo de test
describe('EventService', () => {
  let service: EventService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [EventService]
    });
    service = TestBed.inject(EventService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  it('should fetch events', () => {
    const mockEvents = [{ id: '1', title: 'Test Event' }];

    service.getEvents('Bogotá').subscribe(events => {
      expect(events.length).toBe(1);
      expect(events[0].title).toBe('Test Event');
    });

    const req = httpMock.expectOne('http://localhost:8080/api/v1/events?city=Bogotá');
    expect(req.request.method).toBe('GET');
    req.flush({ data: mockEvents });
  });
});
```

---

## 11. CHECKLIST ANTES DEL PRIMER COMMIT

### Backend (Java):
- [ ] `.env` y `application-dev.yml` están en `.gitignore`
- [ ] Código compila sin errores
- [ ] Tests pasan (`./mvnw test`)
- [ ] Código formateado (Checkstyle/SpotBugs)
- [ ] No hay credenciales hardcodeadas

### iOS:
- [ ] Proyecto compila en Xcode
- [ ] Tests pasan (⌘U)
- [ ] SwiftLint pasa (sin warnings críticos)
- [ ] No hay API keys en código (usar Configuration)

### Android:
- [ ] `local.properties` está en `.gitignore`
- [ ] Proyecto compila (`./gradlew build`)
- [ ] Tests pasan (`./gradlew test`)
- [ ] ktlint pasa
- [ ] No hay API keys en código (usar BuildConfig)

### Angular:
- [ ] `environment.ts` con datos de dev (no producción)
- [ ] Proyecto compila (`ng build`)
- [ ] Tests pasan (`ng test`)
- [ ] ESLint pasa (`ng lint`)
- [ ] No hay `console.log` innecesarios

---

## 12. RECURSOS ADICIONALES

### Documentación Oficial:
- [Spring Boot Docs](https://docs.spring.io/spring-boot/docs/current/reference/html/)
- [Swift Docs](https://docs.swift.org/swift-book/)
- [Android Developers](https://developer.android.com/)
- [Angular Docs](https://angular.io/docs)

### Tools:
- [Postman](https://www.postman.com/) - Testing de APIs
- [Insomnia](https://insomnia.rest/) - Alternativa a Postman
- [DBeaver](https://dbeaver.io/) - Cliente universal de BD
- [Proxyman](https://proxyman.io/) - Debugger HTTP para iOS/Android

---

¡Listo! Con esta guía el equipo debería poder configurar su entorno de desarrollo para cualquier parte del stack de Amigusto.
