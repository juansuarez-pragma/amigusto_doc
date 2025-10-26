# Arquitectura de Proyecto - Amigusto
## Estructura de Repositorios y Organización del Código

---

## 1. ESTRUCTURA GENERAL DE REPOSITORIOS

### 1.1 Enfoque Recomendado: 4 Repositorios Separados

```
amigusto-backend/        (Java Spring Boot)
amigusto-ios/           (Swift + SwiftUI)
amigusto-android/       (Kotlin + Jetpack Compose)
amigusto-web/           (Angular - Portal + Admin)
```

**Justificación:**
- ✅ Cada equipo trabaja independientemente
- ✅ CI/CD específico por plataforma
- ✅ Control de versiones independiente
- ✅ Releases independientes
- ✅ Permisos granulares por repositorio

### 1.2 Alternativa: Monorepo (No Recomendado para este Stack)

Si prefieres un monorepo, sería más complejo porque mezcla tecnologías muy diferentes (Java, Swift, Kotlin, TypeScript).

---

## 2. ESTRUCTURA DEL BACKEND (JAVA SPRING BOOT)

### 2.1 Estructura Completa

```
amigusto-backend/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/
│   │   │       └── amigusto/
│   │   │           ├── AmigustoApplication.java
│   │   │           │
│   │   │           ├── config/
│   │   │           │   ├── SecurityConfig.java
│   │   │           │   ├── JwtConfig.java
│   │   │           │   ├── RedisConfig.java
│   │   │           │   ├── S3Config.java
│   │   │           │   ├── CorsConfig.java
│   │   │           │   └── OpenApiConfig.java
│   │   │           │
│   │   │           ├── controller/
│   │   │           │   ├── AuthController.java
│   │   │           │   ├── EventController.java
│   │   │           │   ├── GustoController.java
│   │   │           │   ├── UserController.java
│   │   │           │   ├── PromoterController.java
│   │   │           │   └── AdminController.java
│   │   │           │
│   │   │           ├── service/
│   │   │           │   ├── AuthService.java
│   │   │           │   ├── EventService.java
│   │   │           │   ├── GustoService.java
│   │   │           │   ├── UserService.java
│   │   │           │   ├── PromoterService.java
│   │   │           │   ├── EmailService.java
│   │   │           │   └── StorageService.java
│   │   │           │
│   │   │           ├── repository/
│   │   │           │   ├── UserRepository.java
│   │   │           │   ├── EventRepository.java
│   │   │           │   ├── GustoRepository.java
│   │   │           │   ├── PromoterRepository.java
│   │   │           │   ├── SavedEventRepository.java
│   │   │           │   └── CityRepository.java
│   │   │           │
│   │   │           ├── model/
│   │   │           │   ├── entity/
│   │   │           │   │   ├── User.java
│   │   │           │   │   ├── Promoter.java
│   │   │           │   │   ├── Event.java
│   │   │           │   │   ├── Gusto.java
│   │   │           │   │   ├── SavedEvent.java
│   │   │           │   │   └── City.java
│   │   │           │   │
│   │   │           │   ├── dto/
│   │   │           │   │   ├── request/
│   │   │           │   │   │   ├── LoginRequest.java
│   │   │           │   │   │   ├── RegisterRequest.java
│   │   │           │   │   │   ├── CreateEventRequest.java
│   │   │           │   │   │   ├── UpdateEventRequest.java
│   │   │           │   │   │   └── EventFilterRequest.java
│   │   │           │   │   └── response/
│   │   │           │   │       ├── AuthResponse.java
│   │   │           │   │       ├── EventResponse.java
│   │   │           │   │       ├── EventDetailResponse.java
│   │   │           │   │       ├── GustoResponse.java
│   │   │           │   │       ├── PageResponse.java
│   │   │           │   │       └── ApiResponse.java
│   │   │           │   │
│   │   │           │   └── enums/
│   │   │           │       ├── UserRole.java
│   │   │           │       ├── EventStatus.java
│   │   │           │       ├── EventType.java
│   │   │           │       └── PromoterStatus.java
│   │   │           │
│   │   │           ├── security/
│   │   │           │   ├── JwtTokenProvider.java
│   │   │           │   ├── JwtAuthenticationFilter.java
│   │   │           │   ├── JwtAuthenticationEntryPoint.java
│   │   │           │   ├── UserDetailsServiceImpl.java
│   │   │           │   └── CustomUserDetails.java
│   │   │           │
│   │   │           ├── exception/
│   │   │           │   ├── GlobalExceptionHandler.java
│   │   │           │   ├── ResourceNotFoundException.java
│   │   │           │   ├── UnauthorizedException.java
│   │   │           │   ├── BadRequestException.java
│   │   │           │   └── BusinessException.java
│   │   │           │
│   │   │           └── util/
│   │   │               ├── DateUtil.java
│   │   │               ├── GeoUtil.java
│   │   │               ├── SlugUtil.java
│   │   │               └── ValidationUtil.java
│   │   │
│   │   └── resources/
│   │       ├── application.yml
│   │       ├── application-dev.yml
│   │       ├── application-prod.yml
│   │       ├── db/
│   │       │   └── migration/
│   │       │       ├── V1__create_initial_schema.sql
│   │       │       └── V2__seed_gustos_and_cities.sql
│   │       └── templates/
│   │           └── email/
│   │               ├── event-approved.html
│   │               ├── event-rejected.html
│   │               └── welcome.html
│   │
│   └── test/
│       └── java/
│           └── com/
│               └── amigusto/
│                   ├── controller/
│                   │   ├── AuthControllerTest.java
│                   │   ├── EventControllerTest.java
│                   │   └── AdminControllerTest.java
│                   ├── service/
│                   │   ├── EventServiceTest.java
│                   │   ├── AuthServiceTest.java
│                   │   └── UserServiceTest.java
│                   └── repository/
│                       └── EventRepositoryTest.java
│
├── docker-compose.yml
├── Dockerfile
├── pom.xml (o build.gradle.kts)
├── .gitignore
├── README.md
└── .env.example
```

### 2.2 Convenciones de Código Java

**Naming:**
- Clases: PascalCase (`EventService`, `UserController`)
- Métodos: camelCase (`getEventById`, `saveEvent`)
- Constantes: UPPER_SNAKE_CASE (`MAX_FILE_SIZE`, `JWT_SECRET_KEY`)
- Packages: lowercase (`com.amigusto.service`)

**Anotaciones Spring:**
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

---

## 3. ESTRUCTURA DE iOS (SWIFT + SWIFTUI)

### 3.1 Estructura Completa

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
│   │   │   ├── User.swift
│   │   │   ├── Promoter.swift
│   │   │   └── City.swift
│   │   │
│   │   ├── Networking/
│   │   │   ├── APIService.swift
│   │   │   ├── Endpoints.swift
│   │   │   ├── NetworkError.swift
│   │   │   └── RequestBuilder.swift
│   │   │
│   │   └── Persistence/
│   │       ├── CoreDataStack.swift
│   │       ├── KeychainManager.swift
│   │       └── UserDefaultsManager.swift
│   │
│   ├── Features/
│   │   ├── Auth/
│   │   │   ├── Views/
│   │   │   │   ├── LoginView.swift
│   │   │   │   └── RegisterView.swift
│   │   │   └── ViewModels/
│   │   │       └── AuthViewModel.swift
│   │   │
│   │   ├── Onboarding/
│   │   │   ├── Views/
│   │   │   │   ├── WelcomeView.swift
│   │   │   │   ├── GustoSelectionView.swift
│   │   │   │   └── LocationPermissionView.swift
│   │   │   └── ViewModels/
│   │   │       └── OnboardingViewModel.swift
│   │   │
│   │   ├── Discover/
│   │   │   ├── Views/
│   │   │   │   ├── DiscoverView.swift
│   │   │   │   ├── EventCard.swift
│   │   │   │   ├── EventDetailView.swift
│   │   │   │   └── FilterSheet.swift
│   │   │   └── ViewModels/
│   │   │       ├── DiscoverViewModel.swift
│   │   │       └── EventDetailViewModel.swift
│   │   │
│   │   └── MyPlans/
│   │       ├── Views/
│   │       │   ├── MyPlansView.swift
│   │       │   └── SavedEventCard.swift
│   │       └── ViewModels/
│   │           └── MyPlansViewModel.swift
│   │
│   ├── Shared/
│   │   ├── Components/
│   │   │   ├── GustoChip.swift
│   │   │   ├── LoadingView.swift
│   │   │   ├── EmptyStateView.swift
│   │   │   └── ErrorView.swift
│   │   │
│   │   ├── Extensions/
│   │   │   ├── View+Extensions.swift
│   │   │   ├── String+Extensions.swift
│   │   │   ├── Date+Extensions.swift
│   │   │   └── Color+Extensions.swift
│   │   │
│   │   └── Modifiers/
│   │       ├── CardModifier.swift
│   │       └── ShimmerModifier.swift
│   │
│   ├── Services/
│   │   ├── EventService.swift
│   │   ├── AuthService.swift
│   │   ├── UserService.swift
│   │   ├── LocationService.swift
│   │   └── AnalyticsService.swift
│   │
│   ├── Utilities/
│   │   ├── Constants.swift
│   │   ├── Configuration.swift
│   │   ├── Formatters.swift
│   │   └── Logger.swift
│   │
│   ├── Resources/
│   │   ├── Assets.xcassets/
│   │   ├── Colors.xcassets/
│   │   ├── Localizable.strings
│   │   └── Info.plist
│   │
│   └── Navigation/
│       ├── MainTabView.swift
│       ├── NavigationCoordinator.swift
│       └── DeepLinkHandler.swift
│
├── AmigustoiOSTests/
│   ├── ViewModelTests/
│   │   ├── DiscoverViewModelTests.swift
│   │   └── AuthViewModelTests.swift
│   ├── ServiceTests/
│   │   └── EventServiceTests.swift
│   └── MockData/
│       └── MockEventData.swift
│
├── AmigustoiOSUITests/
│   └── DiscoverFlowTests.swift
│
├── AmigustoiOS.xcodeproj
└── Podfile (o Package.swift si usas SPM)
```

### 3.2 Convenciones de Código Swift

**Naming:**
- Tipos: PascalCase (`EventViewModel`, `DiscoverView`)
- Funciones/Variables: camelCase (`fetchEvents`, `selectedGustos`)
- Protocolos: PascalCase con sufijo "able" o "ing" (`Codable`, `ObservableObject`)

**SwiftUI Views:**
```swift
struct DiscoverView: View {
    @StateObject private var viewModel = DiscoverViewModel()

    var body: some View {
        // UI here
    }
}
```

---

## 4. ESTRUCTURA DE ANDROID (KOTLIN + JETPACK COMPOSE)

### 4.1 Estructura Completa

```
AmigustoAndroid/
├── app/
│   ├── src/
│   │   ├── main/
│   │   │   ├── java/
│   │   │   │   └── com/
│   │   │   │       └── amigusto/
│   │   │   │           ├── AmigustoApplication.kt
│   │   │   │           │
│   │   │   │           ├── di/
│   │   │   │           │   ├── AppModule.kt
│   │   │   │           │   ├── NetworkModule.kt
│   │   │   │           │   ├── DatabaseModule.kt
│   │   │   │           │   └── RepositoryModule.kt
│   │   │   │           │
│   │   │   │           ├── data/
│   │   │   │           │   ├── remote/
│   │   │   │           │   │   ├── api/
│   │   │   │           │   │   │   ├── ApiService.kt
│   │   │   │           │   │   │   ├── AuthApi.kt
│   │   │   │           │   │   │   └── EventApi.kt
│   │   │   │           │   │   ├── dto/
│   │   │   │           │   │   │   ├── EventDto.kt
│   │   │   │           │   │   │   ├── GustoDto.kt
│   │   │   │           │   │   │   └── AuthDto.kt
│   │   │   │           │   │   └── interceptor/
│   │   │   │           │   │       ├── AuthInterceptor.kt
│   │   │   │           │   │       └── ErrorInterceptor.kt
│   │   │   │           │   │
│   │   │   │           │   ├── local/
│   │   │   │           │   │   ├── database/
│   │   │   │           │   │   │   ├── AppDatabase.kt
│   │   │   │           │   │   │   └── Converters.kt
│   │   │   │           │   │   ├── dao/
│   │   │   │           │   │   │   ├── EventDao.kt
│   │   │   │           │   │   │   └── UserDao.kt
│   │   │   │           │   │   ├── entity/
│   │   │   │           │   │   │   ├── EventEntity.kt
│   │   │   │           │   │   │   └── UserEntity.kt
│   │   │   │           │   │   └── preferences/
│   │   │   │           │   │       └── PreferencesManager.kt
│   │   │   │           │   │
│   │   │   │           │   ├── repository/
│   │   │   │           │   │   ├── EventRepository.kt
│   │   │   │           │   │   ├── EventRepositoryImpl.kt
│   │   │   │           │   │   ├── AuthRepository.kt
│   │   │   │           │   │   └── AuthRepositoryImpl.kt
│   │   │   │           │   │
│   │   │   │           │   └── mapper/
│   │   │   │           │       ├── EventMapper.kt
│   │   │   │           │       └── GustoMapper.kt
│   │   │   │           │
│   │   │   │           ├── domain/
│   │   │   │           │   ├── model/
│   │   │   │           │   │   ├── Event.kt
│   │   │   │           │   │   ├── Gusto.kt
│   │   │   │           │   │   ├── User.kt
│   │   │   │           │   │   └── EventStatus.kt
│   │   │   │           │   │
│   │   │   │           │   ├── usecase/
│   │   │   │           │   │   ├── GetEventsUseCase.kt
│   │   │   │           │   │   ├── SaveEventUseCase.kt
│   │   │   │           │   │   ├── GetSavedEventsUseCase.kt
│   │   │   │           │   │   └── LoginUseCase.kt
│   │   │   │           │   │
│   │   │   │           │   └── repository/
│   │   │   │           │       ├── IEventRepository.kt
│   │   │   │           │       └── IAuthRepository.kt
│   │   │   │           │
│   │   │   │           └── presentation/
│   │   │   │               ├── MainActivity.kt
│   │   │   │               ├── navigation/
│   │   │   │               │   ├── NavGraph.kt
│   │   │   │               │   ├── Screen.kt
│   │   │   │               │   └── NavigationExtensions.kt
│   │   │   │               │
│   │   │   │               ├── theme/
│   │   │   │               │   ├── Color.kt
│   │   │   │               │   ├── Type.kt
│   │   │   │               │   ├── Theme.kt
│   │   │   │               │   └── Shape.kt
│   │   │   │               │
│   │   │   │               ├── components/
│   │   │   │               │   ├── EventCard.kt
│   │   │   │               │   ├── GustoChip.kt
│   │   │   │               │   ├── LoadingIndicator.kt
│   │   │   │               │   └── ErrorMessage.kt
│   │   │   │               │
│   │   │   │               ├── onboarding/
│   │   │   │               │   ├── OnboardingScreen.kt
│   │   │   │               │   ├── OnboardingViewModel.kt
│   │   │   │               │   └── GustoSelectionScreen.kt
│   │   │   │               │
│   │   │   │               ├── discover/
│   │   │   │               │   ├── DiscoverScreen.kt
│   │   │   │               │   ├── DiscoverViewModel.kt
│   │   │   │               │   ├── EventDetailScreen.kt
│   │   │   │               │   └── EventDetailViewModel.kt
│   │   │   │               │
│   │   │   │               ├── myplans/
│   │   │   │               │   ├── MyPlansScreen.kt
│   │   │   │               │   └── MyPlansViewModel.kt
│   │   │   │               │
│   │   │   │               └── util/
│   │   │   │                   ├── Constants.kt
│   │   │   │                   ├── Extensions.kt
│   │   │   │                   └── DateFormatter.kt
│   │   │   │
│   │   │   ├── res/
│   │   │   │   ├── values/
│   │   │   │   │   ├── strings.xml
│   │   │   │   │   ├── colors.xml
│   │   │   │   │   └── themes.xml
│   │   │   │   ├── drawable/
│   │   │   │   └── mipmap/
│   │   │   │
│   │   │   └── AndroidManifest.xml
│   │   │
│   │   ├── test/
│   │   │   └── java/
│   │   │       └── com/amigusto/
│   │   │           ├── viewmodel/
│   │   │           │   └── DiscoverViewModelTest.kt
│   │   │           ├── repository/
│   │   │           │   └── EventRepositoryTest.kt
│   │   │           └── usecase/
│   │   │               └── GetEventsUseCaseTest.kt
│   │   │
│   │   └── androidTest/
│   │       └── java/
│   │           └── com/amigusto/
│   │               └── ui/
│   │                   └── DiscoverScreenTest.kt
│   │
│   ├── build.gradle.kts
│   └── proguard-rules.pro
│
├── gradle/
│   └── libs.versions.toml
├── build.gradle.kts
├── settings.gradle.kts
└── gradle.properties
```

### 4.2 Convenciones de Código Kotlin

**Naming:**
- Clases/Interfaces: PascalCase (`EventRepository`, `DiscoverViewModel`)
- Funciones/Variables: camelCase (`fetchEvents`, `savedEvents`)
- Constantes: UPPER_SNAKE_CASE (`API_BASE_URL`, `MAX_RETRY_COUNT`)
- Packages: lowercase (`com.amigusto.data.repository`)

**Composables:**
```kotlin
@Composable
fun DiscoverScreen(
    viewModel: DiscoverViewModel = hiltViewModel()
) {
    val uiState by viewModel.uiState.collectAsState()

    // UI here
}
```

---

## 5. ESTRUCTURA DE WEB (ANGULAR)

### 5.1 Estructura Completa - Portal B2B + Admin

```
amigusto-web/
├── projects/
│   ├── portal/                    # App principal (Portal B2B)
│   │   ├── src/
│   │   │   ├── app/
│   │   │   │   ├── core/
│   │   │   │   │   ├── services/
│   │   │   │   │   │   ├── api.service.ts
│   │   │   │   │   │   ├── auth.service.ts
│   │   │   │   │   │   ├── event.service.ts
│   │   │   │   │   │   ├── gusto.service.ts
│   │   │   │   │   │   └── storage.service.ts
│   │   │   │   │   │
│   │   │   │   │   ├── guards/
│   │   │   │   │   │   ├── auth.guard.ts
│   │   │   │   │   │   └── promoter.guard.ts
│   │   │   │   │   │
│   │   │   │   │   ├── interceptors/
│   │   │   │   │   │   ├── auth.interceptor.ts
│   │   │   │   │   │   ├── error.interceptor.ts
│   │   │   │   │   │   └── loading.interceptor.ts
│   │   │   │   │   │
│   │   │   │   │   └── models/
│   │   │   │   │       ├── event.model.ts
│   │   │   │   │       ├── gusto.model.ts
│   │   │   │   │       ├── user.model.ts
│   │   │   │   │       └── api-response.model.ts
│   │   │   │   │
│   │   │   │   ├── features/
│   │   │   │   │   ├── auth/
│   │   │   │   │   │   ├── login/
│   │   │   │   │   │   │   ├── login.component.ts
│   │   │   │   │   │   │   ├── login.component.html
│   │   │   │   │   │   │   └── login.component.scss
│   │   │   │   │   │   └── register/
│   │   │   │   │   │       ├── register.component.ts
│   │   │   │   │   │       ├── register.component.html
│   │   │   │   │   │       └── register.component.scss
│   │   │   │   │   │
│   │   │   │   │   ├── dashboard/
│   │   │   │   │   │   ├── dashboard.component.ts
│   │   │   │   │   │   ├── dashboard.component.html
│   │   │   │   │   │   ├── dashboard.component.scss
│   │   │   │   │   │   └── components/
│   │   │   │   │   │       └── events-list/
│   │   │   │   │   │           ├── events-list.component.ts
│   │   │   │   │   │           ├── events-list.component.html
│   │   │   │   │   │           └── events-list.component.scss
│   │   │   │   │   │
│   │   │   │   │   ├── events/
│   │   │   │   │   │   ├── create-event/
│   │   │   │   │   │   │   ├── create-event.component.ts
│   │   │   │   │   │   │   ├── create-event.component.html
│   │   │   │   │   │   │   └── create-event.component.scss
│   │   │   │   │   │   ├── edit-event/
│   │   │   │   │   │   │   ├── edit-event.component.ts
│   │   │   │   │   │   │   ├── edit-event.component.html
│   │   │   │   │   │   │   └── edit-event.component.scss
│   │   │   │   │   │   └── components/
│   │   │   │   │   │       └── event-form/
│   │   │   │   │   │           ├── event-form.component.ts
│   │   │   │   │   │           ├── event-form.component.html
│   │   │   │   │   │           └── event-form.component.scss
│   │   │   │   │   │
│   │   │   │   │   └── profile/
│   │   │   │   │       ├── profile.component.ts
│   │   │   │   │       ├── profile.component.html
│   │   │   │   │       └── profile.component.scss
│   │   │   │   │
│   │   │   │   ├── shared/
│   │   │   │   │   ├── components/
│   │   │   │   │   │   ├── event-status-badge/
│   │   │   │   │   │   │   ├── event-status-badge.component.ts
│   │   │   │   │   │   │   ├── event-status-badge.component.html
│   │   │   │   │   │   │   └── event-status-badge.component.scss
│   │   │   │   │   │   ├── gusto-selector/
│   │   │   │   │   │   │   ├── gusto-selector.component.ts
│   │   │   │   │   │   │   ├── gusto-selector.component.html
│   │   │   │   │   │   │   └── gusto-selector.component.scss
│   │   │   │   │   │   ├── image-uploader/
│   │   │   │   │   │   │   ├── image-uploader.component.ts
│   │   │   │   │   │   │   ├── image-uploader.component.html
│   │   │   │   │   │   │   └── image-uploader.component.scss
│   │   │   │   │   │   └── location-autocomplete/
│   │   │   │   │   │       ├── location-autocomplete.component.ts
│   │   │   │   │   │       ├── location-autocomplete.component.html
│   │   │   │   │   │       └── location-autocomplete.component.scss
│   │   │   │   │   │
│   │   │   │   │   ├── directives/
│   │   │   │   │   │   └── highlight.directive.ts
│   │   │   │   │   │
│   │   │   │   │   └── pipes/
│   │   │   │   │       ├── date-format.pipe.ts
│   │   │   │   │       ├── price-format.pipe.ts
│   │   │   │   │       └── truncate.pipe.ts
│   │   │   │   │
│   │   │   │   ├── app.component.ts
│   │   │   │   ├── app.component.html
│   │   │   │   ├── app.component.scss
│   │   │   │   ├── app.config.ts
│   │   │   │   └── app.routes.ts
│   │   │   │
│   │   │   ├── assets/
│   │   │   │   ├── images/
│   │   │   │   ├── icons/
│   │   │   │   └── i18n/
│   │   │   │
│   │   │   ├── environments/
│   │   │   │   ├── environment.ts
│   │   │   │   └── environment.prod.ts
│   │   │   │
│   │   │   ├── styles/
│   │   │   │   ├── _variables.scss
│   │   │   │   ├── _mixins.scss
│   │   │   │   └── styles.scss
│   │   │   │
│   │   │   ├── index.html
│   │   │   └── main.ts
│   │   │
│   │   └── project.json
│   │
│   └── admin-panel/              # App de administración
│       ├── src/
│       │   ├── app/
│       │   │   ├── core/
│       │   │   │   └── (similar a portal)
│       │   │   │
│       │   │   ├── features/
│       │   │   │   ├── queue/
│       │   │   │   │   ├── queue.component.ts
│       │   │   │   │   └── queue.component.html
│       │   │   │   ├── review/
│       │   │   │   │   ├── review.component.ts
│       │   │   │   │   └── review.component.html
│       │   │   │   ├── promoters/
│       │   │   │   │   ├── promoters.component.ts
│       │   │   │   │   └── promoters.component.html
│       │   │   │   └── dashboard/
│       │   │   │       ├── dashboard.component.ts
│       │   │   │       └── dashboard.component.html
│       │   │   │
│       │   │   └── shared/
│       │   │       └── (componentes compartidos)
│       │   │
│       │   └── (igual estructura que portal)
│       │
│       └── project.json
│
├── angular.json
├── package.json
├── tsconfig.json
└── README.md
```

### 5.2 Convenciones de Código Angular

**Naming:**
- Components: kebab-case files, PascalCase classes (`event-card.component.ts`, `EventCardComponent`)
- Services: kebab-case files con sufijo `.service.ts` (`auth.service.ts`, `AuthService`)
- Interfaces: PascalCase con prefijo "I" opcional (`IEvent` o `Event`)
- Enums: PascalCase (`EventStatus`)

**Component Structure:**
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

---

## 6. GITIGNORE RECOMENDADO POR PROYECTO

### 6.1 Backend (Java Spring Boot)

```gitignore
# Maven
target/
pom.xml.tag
pom.xml.releaseBackup
pom.xml.versionsBackup

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

# Application properties
application-dev.yml
application-local.yml
*.env

# Logs
logs/
*.log

# OS
.DS_Store
Thumbs.db
```

### 6.2 iOS (Swift)

```gitignore
# Xcode
xcuserdata/
*.xcodeproj/*
!*.xcodeproj/project.pbxproj
!*.xcworkspace/contents.xcworkspacedata

# SPM
.swiftpm/
.build/

# CocoaPods
Pods/
*.podspec

# Build
DerivedData/
build/

# Configuration
Configuration.plist

# OS
.DS_Store
```

### 6.3 Android (Kotlin)

```gitignore
# Gradle
.gradle/
build/
local.properties

# Android Studio
.idea/
*.iml
.externalNativeBuild
.cxx

# Keystore
*.jks
*.keystore

# Build
app/release/

# OS
.DS_Store
```

### 6.4 Web (Angular)

```gitignore
# Dependencies
node_modules/
npm-debug.log
yarn-error.log

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

## 7. CI/CD PIPELINE SUGERIDO

### 7.1 Backend (GitHub Actions)

```yaml
# .github/workflows/backend-ci.yml
name: Backend CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Build with Maven
        run: ./mvnw clean install

      - name: Run tests
        run: ./mvnw test

      - name: Build Docker image
        run: docker build -t amigusto-backend .
```

### 7.2 iOS (Xcode Cloud o Fastlane)

```ruby
# Fastfile
lane :test do
  run_tests(scheme: "AmigustoiOS")
end

lane :beta do
  build_app(scheme: "AmigustoiOS")
  upload_to_testflight
end
```

### 7.3 Android (GitHub Actions)

```yaml
# .github/workflows/android-ci.yml
name: Android CI

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'

      - name: Build with Gradle
        run: ./gradlew build

      - name: Run tests
        run: ./gradlew test
```

### 7.4 Web (GitHub Actions)

```yaml
# .github/workflows/web-ci.yml
name: Web CI/CD

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Test
        run: npm test

      - name: Build
        run: npm run build
```

---

Esta arquitectura está 100% alineada con el stack actualizado: **Java Spring Boot, iOS nativo (Swift), Android nativo (Kotlin), y Angular**. No hay referencias a tecnologías obsoletas.
