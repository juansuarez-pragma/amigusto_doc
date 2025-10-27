# Ejemplos de Código - Amigusto

## Índice

1. [Backend - Java Spring Boot](#backend---java-spring-boot)
2. [iOS - Swift + SwiftUI](#ios---swift--swiftui)
3. [Android - Kotlin + Jetpack Compose](#android---kotlin--jetpack-compose)
4. [Web - Angular](#web---angular)

---

## Backend - Java Spring Boot

### 1. Entidad JPA - Event

```java
package com.amigusto.model.entity;

import jakarta.persistence.*;
import jakarta.validation.constraints.*;
import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.AllArgsConstructor;
import org.hibernate.annotations.CreationTimestamp;
import org.hibernate.annotations.UpdateTimestamp;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.Set;
import java.util.UUID;

@Entity
@Table(name = "events", indexes = {
    @Index(name = "idx_event_status", columnList = "status"),
    @Index(name = "idx_event_start_date", columnList = "start_date")
})
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Event {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @NotBlank(message = "El título es obligatorio")
    @Size(min = 5, max = 255, message = "El título debe tener entre 5 y 255 caracteres")
    @Column(nullable = false)
    private String title;

    @NotBlank(message = "La descripción es obligatoria")
    @Size(min = 20, max = 2000, message = "La descripción debe tener entre 20 y 2000 caracteres")
    @Column(nullable = false, length = 2000)
    private String description;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "promoter_id", nullable = false)
    private Promoter promoter;

    @NotNull(message = "La fecha de inicio es obligatoria")
    @Future(message = "La fecha debe ser futura")
    @Column(name = "start_date", nullable = false)
    private LocalDateTime startDate;

    @Column(name = "end_date")
    private LocalDateTime endDate;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private EventStatus status = EventStatus.DRAFT;

    // Coordenadas geográficas
    @NotNull(message = "La latitud es obligatoria")
    @DecimalMin(value = "-90.0", message = "Latitud inválida")
    @DecimalMax(value = "90.0", message = "Latitud inválida")
    @Column(nullable = false, precision = 10, scale = 8)
    private BigDecimal lat;

    @NotNull(message = "La longitud es obligatoria")
    @DecimalMin(value = "-180.0", message = "Longitud inválida")
    @DecimalMax(value = "180.0", message = "Longitud inválida")
    @Column(nullable = false, precision = 11, scale = 8)
    private BigDecimal lng;

    @NotBlank(message = "La dirección es obligatoria")
    @Column(nullable = false)
    private String address;

    @NotBlank(message = "La ciudad es obligatoria")
    @Column(nullable = false, length = 100)
    private String city;

    @Column(name = "venue_name", length = 200)
    private String venueName;

    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(
        name = "event_gustos",
        joinColumns = @JoinColumn(name = "event_id"),
        inverseJoinColumns = @JoinColumn(name = "gusto_id")
    )
    @NotEmpty(message = "Debe seleccionar al menos un gusto")
    private Set<Gusto> gustos;

    @Column(name = "image_url")
    private String imageUrl;

    @Column(name = "is_free", nullable = false)
    private Boolean isFree = false;

    @DecimalMin(value = "0.0", message = "El precio no puede ser negativo")
    @Column(precision = 10, scale = 2)
    private BigDecimal price;

    @Column(length = 3)
    private String currency = "EUR";

    @Column(name = "external_url", length = 500)
    private String externalUrl;

    @Column(name = "max_attendees")
    private Integer maxAttendees;

    @CreationTimestamp
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @UpdateTimestamp
    @Column(name = "updated_at", nullable = false)
    private LocalDateTime updatedAt;

    // Estado de curación
    @Column(name = "reviewed_by")
    private UUID reviewedBy;

    @Column(name = "reviewed_at")
    private LocalDateTime reviewedAt;

    @Column(name = "rejection_reason", length = 500)
    private String rejectionReason;
}
```

### 2. EventStatus Enum

```java
package com.amigusto.model.entity;

public enum EventStatus {
    DRAFT,
    PENDING_REVIEW,
    APPROVED,
    REJECTED,
    CANCELLED,
    ENDED
}
```

### 3. Repository - EventRepository

```java
package com.amigusto.repository;

import com.amigusto.model.entity.Event;
import com.amigusto.model.entity.EventStatus;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;
import java.util.UUID;

@Repository
public interface EventRepository extends JpaRepository<Event, UUID> {

    // Eventos por estado
    Page<Event> findByStatus(EventStatus status, Pageable pageable);

    // Eventos por promotor
    Page<Event> findByPromoterId(UUID promoterId, Pageable pageable);

    // Búsqueda geográfica con Haversine
    @Query(value = """
        SELECT e.* FROM events e
        WHERE e.status = 'APPROVED'
        AND e.start_date > :now
        AND (
            6371 * acos(
                cos(radians(:lat))
                * cos(radians(e.lat))
                * cos(radians(e.lng) - radians(:lng))
                + sin(radians(:lat))
                * sin(radians(e.lat))
            )
        ) < :radiusKm
        ORDER BY e.start_date ASC
        """, nativeQuery = true)
    List<Event> findEventsNearby(
        @Param("lat") BigDecimal lat,
        @Param("lng") BigDecimal lng,
        @Param("radiusKm") double radiusKm,
        @Param("now") LocalDateTime now
    );

    // Eventos por ciudad y gustos
    @Query("""
        SELECT DISTINCT e FROM Event e
        JOIN e.gustos g
        WHERE e.status = 'APPROVED'
        AND e.city = :city
        AND e.startDate > :now
        AND g.id IN :gustoIds
        ORDER BY e.startDate ASC
        """)
    Page<Event> findByGustosAndCity(
        @Param("gustoIds") List<UUID> gustoIds,
        @Param("city") String city,
        @Param("now") LocalDateTime now,
        Pageable pageable
    );

    // Eventos pendientes de revisión
    @Query("""
        SELECT e FROM Event e
        WHERE e.status = 'PENDING_REVIEW'
        ORDER BY e.createdAt ASC
        """)
    Page<Event> findPendingReview(Pageable pageable);
}
```

### 4. DTO - EventResponse

```java
package com.amigusto.model.dto;

import com.amigusto.model.entity.EventStatus;
import lombok.Data;
import lombok.Builder;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.Set;
import java.util.UUID;

@Data
@Builder
public class EventResponse {
    private UUID id;
    private String title;
    private String description;
    private LocalDateTime startDate;
    private LocalDateTime endDate;
    private EventStatus status;

    private BigDecimal lat;
    private BigDecimal lng;
    private String address;
    private String city;
    private String venueName;

    private Set<GustoResponse> gustos;

    private String imageUrl;
    private Boolean isFree;
    private BigDecimal price;
    private String currency;

    private String externalUrl;
    private Integer maxAttendees;

    private PromoterSimpleResponse promoter;

    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;

    // Solo para admins
    private UUID reviewedBy;
    private LocalDateTime reviewedAt;
    private String rejectionReason;
}
```

### 5. Service - EventService

```java
package com.amigusto.service;

import com.amigusto.exception.ResourceNotFoundException;
import com.amigusto.model.dto.EventRequest;
import com.amigusto.model.dto.EventResponse;
import com.amigusto.model.entity.Event;
import com.amigusto.model.entity.EventStatus;
import com.amigusto.repository.EventRepository;
import com.amigusto.repository.GustoRepository;
import com.amigusto.repository.PromoterRepository;
import com.amigusto.mapper.EventMapper;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;
import java.util.UUID;

@Service
@RequiredArgsConstructor
@Slf4j
@Transactional
public class EventService {

    private final EventRepository eventRepository;
    private final PromoterRepository promoterRepository;
    private final GustoRepository gustoRepository;
    private final EventMapper eventMapper;

    /**
     * Crear evento (promotor)
     */
    public EventResponse createEvent(EventRequest request, UUID promoterId) {
        log.info("Creando evento: {} por promotor: {}", request.getTitle(), promoterId);

        var promoter = promoterRepository.findById(promoterId)
            .orElseThrow(() -> new ResourceNotFoundException("Promotor no encontrado"));

        var gustos = gustoRepository.findAllById(request.getGustoIds());
        if (gustos.isEmpty()) {
            throw new IllegalArgumentException("Debe seleccionar al menos un gusto válido");
        }

        Event event = eventMapper.toEntity(request);
        event.setPromoter(promoter);
        event.setGustos(gustos);
        event.setStatus(EventStatus.DRAFT);

        Event savedEvent = eventRepository.save(event);
        log.info("Evento creado con ID: {}", savedEvent.getId());

        return eventMapper.toResponse(savedEvent);
    }

    /**
     * Enviar evento a revisión
     */
    @CacheEvict(value = "pending-events", allEntries = true)
    public EventResponse submitForReview(UUID eventId, UUID promoterId) {
        log.info("Enviando evento {} a revisión", eventId);

        Event event = eventRepository.findById(eventId)
            .orElseThrow(() -> new ResourceNotFoundException("Evento no encontrado"));

        if (!event.getPromoter().getId().equals(promoterId)) {
            throw new IllegalArgumentException("No tienes permisos para este evento");
        }

        if (event.getStatus() != EventStatus.DRAFT) {
            throw new IllegalStateException("Solo eventos en DRAFT pueden enviarse a revisión");
        }

        event.setStatus(EventStatus.PENDING_REVIEW);
        Event updated = eventRepository.save(event);

        return eventMapper.toResponse(updated);
    }

    /**
     * Descubrir eventos (app móvil)
     */
    @Cacheable(value = "discover-events", key = "#lat + '-' + #lng + '-' + #gustoIds.hashCode()")
    @Transactional(readOnly = true)
    public Page<EventResponse> discoverEvents(
            BigDecimal lat,
            BigDecimal lng,
            List<UUID> gustoIds,
            String city,
            Pageable pageable) {

        log.info("Descubriendo eventos para ciudad: {}, gustos: {}", city, gustoIds.size());

        Page<Event> events = eventRepository.findByGustosAndCity(
            gustoIds,
            city,
            LocalDateTime.now(),
            pageable
        );

        return events.map(eventMapper::toResponse);
    }

    /**
     * Aprobar evento (admin)
     */
    @CacheEvict(value = {"discover-events", "pending-events"}, allEntries = true)
    public EventResponse approveEvent(UUID eventId, UUID adminId) {
        log.info("Aprobando evento {} por admin {}", eventId, adminId);

        Event event = eventRepository.findById(eventId)
            .orElseThrow(() -> new ResourceNotFoundException("Evento no encontrado"));

        if (event.getStatus() != EventStatus.PENDING_REVIEW) {
            throw new IllegalStateException("Solo eventos PENDING_REVIEW pueden aprobarse");
        }

        event.setStatus(EventStatus.APPROVED);
        event.setReviewedBy(adminId);
        event.setReviewedAt(LocalDateTime.now());

        Event updated = eventRepository.save(event);
        log.info("Evento {} aprobado exitosamente", eventId);

        return eventMapper.toResponse(updated);
    }

    /**
     * Rechazar evento (admin)
     */
    @CacheEvict(value = "pending-events", allEntries = true)
    public EventResponse rejectEvent(UUID eventId, UUID adminId, String reason) {
        log.info("Rechazando evento {} por admin {}", eventId, adminId);

        Event event = eventRepository.findById(eventId)
            .orElseThrow(() -> new ResourceNotFoundException("Evento no encontrado"));

        event.setStatus(EventStatus.REJECTED);
        event.setReviewedBy(adminId);
        event.setReviewedAt(LocalDateTime.now());
        event.setRejectionReason(reason);

        Event updated = eventRepository.save(event);

        return eventMapper.toResponse(updated);
    }

    /**
     * Obtener eventos pendientes de revisión (admin)
     */
    @Cacheable(value = "pending-events")
    @Transactional(readOnly = true)
    public Page<EventResponse> getPendingReviewEvents(Pageable pageable) {
        Page<Event> events = eventRepository.findPendingReview(pageable);
        return events.map(eventMapper::toResponse);
    }
}
```

### 6. Controller - EventController

```java
package com.amigusto.controller;

import com.amigusto.model.dto.EventRequest;
import com.amigusto.model.dto.EventResponse;
import com.amigusto.security.CurrentUser;
import com.amigusto.security.UserPrincipal;
import com.amigusto.service.EventService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.security.SecurityRequirement;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.*;

import java.math.BigDecimal;
import java.util.List;
import java.util.UUID;

@RestController
@RequestMapping("/api/v1/events")
@RequiredArgsConstructor
@Tag(name = "Events", description = "Gestión de eventos")
@SecurityRequirement(name = "bearer-jwt")
public class EventController {

    private final EventService eventService;

    @PostMapping
    @PreAuthorize("hasRole('PROMOTER')")
    @Operation(summary = "Crear evento (promotor)")
    public ResponseEntity<EventResponse> createEvent(
            @Valid @RequestBody EventRequest request,
            @CurrentUser UserPrincipal currentUser) {

        EventResponse response = eventService.createEvent(request, currentUser.getId());
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }

    @PostMapping("/{eventId}/submit-review")
    @PreAuthorize("hasRole('PROMOTER')")
    @Operation(summary = "Enviar evento a revisión")
    public ResponseEntity<EventResponse> submitForReview(
            @PathVariable UUID eventId,
            @CurrentUser UserPrincipal currentUser) {

        EventResponse response = eventService.submitForReview(eventId, currentUser.getId());
        return ResponseEntity.ok(response);
    }

    @GetMapping("/discover")
    @PreAuthorize("hasRole('CONSUMER')")
    @Operation(summary = "Descubrir eventos personalizados (app móvil)")
    public ResponseEntity<Page<EventResponse>> discoverEvents(
            @RequestParam BigDecimal lat,
            @RequestParam BigDecimal lng,
            @RequestParam List<UUID> gustoIds,
            @RequestParam String city,
            Pageable pageable) {

        Page<EventResponse> events = eventService.discoverEvents(lat, lng, gustoIds, city, pageable);
        return ResponseEntity.ok(events);
    }

    @PostMapping("/{eventId}/approve")
    @PreAuthorize("hasRole('ADMIN')")
    @Operation(summary = "Aprobar evento (admin)")
    public ResponseEntity<EventResponse> approveEvent(
            @PathVariable UUID eventId,
            @CurrentUser UserPrincipal currentUser) {

        EventResponse response = eventService.approveEvent(eventId, currentUser.getId());
        return ResponseEntity.ok(response);
    }

    @PostMapping("/{eventId}/reject")
    @PreAuthorize("hasRole('ADMIN')")
    @Operation(summary = "Rechazar evento (admin)")
    public ResponseEntity<EventResponse> rejectEvent(
            @PathVariable UUID eventId,
            @RequestParam String reason,
            @CurrentUser UserPrincipal currentUser) {

        EventResponse response = eventService.rejectEvent(eventId, currentUser.getId(), reason);
        return ResponseEntity.ok(response);
    }

    @GetMapping("/pending-review")
    @PreAuthorize("hasRole('ADMIN')")
    @Operation(summary = "Eventos pendientes de revisión (admin)")
    public ResponseEntity<Page<EventResponse>> getPendingReview(Pageable pageable) {
        Page<EventResponse> events = eventService.getPendingReviewEvents(pageable);
        return ResponseEntity.ok(events);
    }
}
```

### 7. Configuración de Seguridad - SecurityConfig

```java
package com.amigusto.security;

import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.config.annotation.authentication.configuration.AuthenticationConfiguration;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@Configuration
@EnableWebSecurity
@EnableMethodSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthFilter;
    private final JwtAuthenticationEntryPoint jwtAuthenticationEntryPoint;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .exceptionHandling(exception ->
                exception.authenticationEntryPoint(jwtAuthenticationEntryPoint))
            .authorizeHttpRequests(auth -> auth
                // Endpoints públicos
                .requestMatchers("/api/v1/auth/**").permitAll()
                .requestMatchers("/swagger-ui/**", "/v3/api-docs/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/v1/gustos").permitAll()

                // Endpoints de consumidores (app móvil)
                .requestMatchers("/api/v1/events/discover").hasRole("CONSUMER")
                .requestMatchers("/api/v1/saved-events/**").hasRole("CONSUMER")

                // Endpoints de promotores (portal web)
                .requestMatchers("/api/v1/events").hasRole("PROMOTER")
                .requestMatchers("/api/v1/events/my-events").hasRole("PROMOTER")

                // Endpoints de admin
                .requestMatchers("/api/v1/events/pending-review").hasRole("ADMIN")
                .requestMatchers("/api/v1/events/*/approve").hasRole("ADMIN")
                .requestMatchers("/api/v1/events/*/reject").hasRole("ADMIN")

                // Todos los demás requieren autenticación
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);
    }

    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();
        configuration.setAllowedOrigins(List.of(
            "http://localhost:4200",  // Angular dev
            "https://portal.amigusto.com",
            "https://admin.amigusto.com"
        ));
        configuration.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "PATCH"));
        configuration.setAllowedHeaders(List.of("*"));
        configuration.setAllowCredentials(true);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", configuration);
        return source;
    }
}
```

### 8. Microservicios - Spring Cloud

#### 8.1 Feign Client - Comunicación Síncrona entre Microservicios

```java
package com.amigusto.event.client;

import com.amigusto.event.model.dto.PromoterResponse;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;

import java.util.UUID;

/**
 * Cliente Feign para comunicación con Promoter Service
 *
 * name: Nombre del servicio registrado en Eureka
 * fallback: Clase que implementa lógica de fallback si el servicio falla
 */
@FeignClient(
    name = "PROMOTER-SERVICE",
    fallback = PromoterServiceFallback.class
)
public interface PromoterServiceClient {

    @GetMapping("/api/v1/promoters/{id}")
    PromoterResponse getPromoter(@PathVariable("id") UUID id);

    @GetMapping("/api/v1/promoters/{id}/verify-status")
    boolean isPromoterVerified(@PathVariable("id") UUID id);
}
```

**Implementación de Fallback:**

```java
package com.amigusto.event.client;

import com.amigusto.event.model.dto.PromoterResponse;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import java.util.UUID;

/**
 * Fallback para PromoterServiceClient
 * Se ejecuta cuando el servicio no está disponible
 */
@Component
@Slf4j
public class PromoterServiceFallback implements PromoterServiceClient {

    @Override
    public PromoterResponse getPromoter(UUID id) {
        log.warn("Fallback activado para getPromoter({}). Servicio no disponible", id);

        // Retornar datos por defecto o lanzar excepción custom
        return PromoterResponse.builder()
            .id(id)
            .name("Promotor no disponible")
            .verified(false)
            .build();
    }

    @Override
    public boolean isPromoterVerified(UUID id) {
        log.warn("Fallback activado para isPromoterVerified({})", id);
        // Por seguridad, retornar false si el servicio no responde
        return false;
    }
}
```

**Configuración de Feign:**

```java
package com.amigusto.event.config;

import feign.Logger;
import feign.RequestInterceptor;
import feign.codec.ErrorDecoder;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;

@Configuration
@Slf4j
public class FeignConfig {

    /**
     * Nivel de logging para requests Feign
     */
    @Bean
    public Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL; // NONE, BASIC, HEADERS, FULL
    }

    /**
     * Interceptor para propagar JWT entre microservicios
     */
    @Bean
    public RequestInterceptor requestTokenBearerInterceptor() {
        return requestTemplate -> {
            Authentication authentication = SecurityContextHolder.getContext().getAuthentication();

            if (authentication != null && authentication.getCredentials() instanceof String) {
                String token = (String) authentication.getCredentials();
                requestTemplate.header("Authorization", "Bearer " + token);
            }
        };
    }

    /**
     * Decoder personalizado para errores
     */
    @Bean
    public ErrorDecoder errorDecoder() {
        return (methodKey, response) -> {
            log.error("Error en Feign call: {} - Status: {}", methodKey, response.status());
            return new ErrorDecoder.Default().decode(methodKey, response);
        };
    }
}
```

**Uso en Service con Circuit Breaker:**

```java
package com.amigusto.event.service;

import com.amigusto.event.client.PromoterServiceClient;
import com.amigusto.event.model.dto.PromoterResponse;
import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import io.github.resilience4j.retry.annotation.Retry;
import io.github.resilience4j.timelimiter.annotation.TimeLimiter;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.time.temporal.ChronoUnit;
import java.util.UUID;
import java.util.concurrent.CompletableFuture;

@Service
@RequiredArgsConstructor
@Slf4j
public class EventService {

    private final PromoterServiceClient promoterClient;

    /**
     * Obtener promotor con Circuit Breaker, Retry y Timeout
     *
     * @CircuitBreaker: Si falla 50% de requests, abre el circuito
     * @Retry: Reintentar 3 veces con backoff exponencial
     * @TimeLimiter: Timeout de 3 segundos
     */
    @CircuitBreaker(name = "promoterService", fallbackMethod = "getPromoterFallback")
    @Retry(name = "promoterService")
    @TimeLimiter(name = "promoterService")
    public CompletableFuture<PromoterResponse> getPromoterAsync(UUID promoterId) {
        log.info("Llamando a Promoter Service para obtener promotor: {}", promoterId);

        return CompletableFuture.supplyAsync(() ->
            promoterClient.getPromoter(promoterId)
        );
    }

    /**
     * Fallback method - debe tener misma firma + Throwable
     */
    private CompletableFuture<PromoterResponse> getPromoterFallback(
            UUID promoterId,
            Throwable throwable) {

        log.error("Circuit breaker abierto para getPromoter({}): {}",
            promoterId, throwable.getMessage());

        return CompletableFuture.completedFuture(
            PromoterResponse.builder()
                .id(promoterId)
                .name("Información temporalmente no disponible")
                .verified(false)
                .build()
        );
    }
}
```

**Configuración de Resilience4j:**

```yaml
# application.yml
resilience4j:
  circuitbreaker:
    instances:
      promoterService:
        registerHealthIndicator: true
        slidingWindowSize: 10                    # Ventana de 10 requests
        failureRateThreshold: 50                 # Abre circuito si falla >50%
        waitDurationInOpenState: 10s             # Esperar 10s antes de probar
        permittedNumberOfCallsInHalfOpenState: 3 # Permitir 3 calls en half-open
        automaticTransitionFromOpenToHalfOpenEnabled: true

  retry:
    instances:
      promoterService:
        maxAttempts: 3                           # Reintentar máximo 3 veces
        waitDuration: 500ms                      # Esperar 500ms entre intentos
        enableExponentialBackoff: true           # Backoff exponencial
        exponentialBackoffMultiplier: 2          # 500ms -> 1s -> 2s
        retryExceptions:
          - org.springframework.web.client.HttpServerErrorException
          - java.io.IOException

  timelimiter:
    instances:
      promoterService:
        timeoutDuration: 3s                      # Timeout de 3 segundos
```

#### 8.2 RabbitMQ - Comunicación Asíncrona

**Configuración de RabbitMQ:**

```java
package com.amigusto.event.config;

import org.springframework.amqp.core.*;
import org.springframework.amqp.rabbit.connection.ConnectionFactory;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.amqp.support.converter.Jackson2JsonMessageConverter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RabbitMQConfig {

    // Exchange names
    public static final String EVENT_EXCHANGE = "event.events";
    public static final String USER_EXCHANGE = "user.events";

    // Queue names
    public static final String EVENT_APPROVED_QUEUE = "notification-service.event.approved";
    public static final String EVENT_CREATED_QUEUE = "promoter-service.event.created";
    public static final String EVENT_SAVED_QUEUE = "event-service.event.saved";

    // Routing keys
    public static final String EVENT_APPROVED_KEY = "event.approved";
    public static final String EVENT_CREATED_KEY = "event.created";
    public static final String EVENT_SAVED_KEY = "event.saved";

    /**
     * Topic Exchange para eventos de Event Service
     */
    @Bean
    public TopicExchange eventExchange() {
        return new TopicExchange(EVENT_EXCHANGE, true, false);
    }

    /**
     * Queue para eventos aprobados (Notification Service)
     */
    @Bean
    public Queue eventApprovedQueue() {
        return QueueBuilder.durable(EVENT_APPROVED_QUEUE)
            .withArgument("x-dead-letter-exchange", "dlx.events")
            .withArgument("x-message-ttl", 3600000) // 1 hora TTL
            .build();
    }

    /**
     * Binding de queue con exchange y routing key
     */
    @Bean
    public Binding eventApprovedBinding() {
        return BindingBuilder
            .bind(eventApprovedQueue())
            .to(eventExchange())
            .with(EVENT_APPROVED_KEY);
    }

    /**
     * Converter para serializar/deserializar mensajes como JSON
     */
    @Bean
    public Jackson2JsonMessageConverter messageConverter() {
        return new Jackson2JsonMessageConverter();
    }

    /**
     * RabbitTemplate configurado con converter JSON
     */
    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
        RabbitTemplate template = new RabbitTemplate(connectionFactory);
        template.setMessageConverter(messageConverter());
        return template;
    }
}
```

**Publisher - Publicar Eventos:**

```java
package com.amigusto.event.messaging;

import com.amigusto.event.config.RabbitMQConfig;
import com.amigusto.event.model.entity.Event;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.stereotype.Component;

import java.time.LocalDateTime;
import java.util.UUID;

@Component
@RequiredArgsConstructor
@Slf4j
public class EventPublisher {

    private final RabbitTemplate rabbitTemplate;

    /**
     * Publicar evento cuando un evento es aprobado
     */
    public void publishEventApproved(Event event) {
        EventApprovedMessage message = EventApprovedMessage.builder()
            .eventId(event.getId())
            .promoterId(event.getPromoter().getId())
            .title(event.getTitle())
            .city(event.getCity())
            .startDate(event.getStartDate())
            .approvedAt(event.getReviewedAt())
            .build();

        log.info("Publicando evento aprobado: {} a exchange: {}",
            event.getId(), RabbitMQConfig.EVENT_EXCHANGE);

        rabbitTemplate.convertAndSend(
            RabbitMQConfig.EVENT_EXCHANGE,
            RabbitMQConfig.EVENT_APPROVED_KEY,
            message
        );
    }

    /**
     * Publicar evento cuando un evento es creado
     */
    public void publishEventCreated(Event event) {
        EventCreatedMessage message = EventCreatedMessage.builder()
            .eventId(event.getId())
            .promoterId(event.getPromoter().getId())
            .title(event.getTitle())
            .createdAt(event.getCreatedAt())
            .build();

        log.info("Publicando evento creado: {}", event.getId());

        rabbitTemplate.convertAndSend(
            RabbitMQConfig.EVENT_EXCHANGE,
            RabbitMQConfig.EVENT_CREATED_KEY,
            message
        );
    }
}
```

**Message DTOs:**

```java
package com.amigusto.event.messaging;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.io.Serializable;
import java.time.LocalDateTime;
import java.util.UUID;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class EventApprovedMessage implements Serializable {
    private UUID eventId;
    private UUID promoterId;
    private String title;
    private String city;
    private LocalDateTime startDate;
    private LocalDateTime approvedAt;
}

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class EventCreatedMessage implements Serializable {
    private UUID eventId;
    private UUID promoterId;
    private String title;
    private LocalDateTime createdAt;
}
```

**Consumer - Consumir Eventos:**

```java
package com.amigusto.notification.messaging;

import com.amigusto.notification.service.NotificationService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

import java.util.UUID;

@Component
@RequiredArgsConstructor
@Slf4j
public class EventMessageConsumer {

    private final NotificationService notificationService;

    /**
     * Consumir mensajes de eventos aprobados
     *
     * @RabbitListener: Escucha la queue especificada
     * containerFactory: Configuración de contenedor de listeners
     */
    @RabbitListener(
        queues = "notification-service.event.approved",
        containerFactory = "rabbitListenerContainerFactory"
    )
    public void handleEventApproved(EventApprovedMessage message) {
        log.info("Recibido evento aprobado: {}", message.getEventId());

        try {
            // Enviar notificación al promotor
            notificationService.sendEventApprovedEmail(
                message.getPromoterId(),
                message.getEventId(),
                message.getTitle()
            );

            log.info("Notificación enviada para evento: {}", message.getEventId());

        } catch (Exception e) {
            log.error("Error procesando evento aprobado: {}", message.getEventId(), e);
            // El mensaje será enviado a DLQ automáticamente si falla
            throw e;
        }
    }

    /**
     * Consumir mensajes de eventos creados
     */
    @RabbitListener(queues = "promoter-service.event.created")
    public void handleEventCreated(EventCreatedMessage message) {
        log.info("Recibido evento creado: {}", message.getEventId());

        try {
            // Actualizar métricas del promotor
            notificationService.incrementPromoterEventCount(message.getPromoterId());

        } catch (Exception e) {
            log.error("Error procesando evento creado: {}", message.getEventId(), e);
            throw e;
        }
    }
}
```

**Configuración de Listener:**

```java
package com.amigusto.notification.config;

import org.springframework.amqp.rabbit.config.SimpleRabbitListenerContainerFactory;
import org.springframework.amqp.rabbit.connection.ConnectionFactory;
import org.springframework.amqp.support.converter.Jackson2JsonMessageConverter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RabbitListenerConfig {

    @Bean
    public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory(
            ConnectionFactory connectionFactory,
            Jackson2JsonMessageConverter messageConverter) {

        SimpleRabbitListenerContainerFactory factory =
            new SimpleRabbitListenerContainerFactory();

        factory.setConnectionFactory(connectionFactory);
        factory.setMessageConverter(messageConverter);
        factory.setConcurrentConsumers(3);      // 3 consumers concurrentes
        factory.setMaxConcurrentConsumers(10);  // Máximo 10 consumers
        factory.setPrefetchCount(10);           // Prefetch 10 mensajes
        factory.setDefaultRequeueRejected(false); // No requeue si falla (va a DLQ)

        return factory;
    }
}
```

#### 8.3 Service Discovery - Eureka Client

**Configuración de Eureka Client:**

```yaml
# bootstrap.yml (cada microservicio)
spring:
  application:
    name: event-service  # Nombre único del servicio

eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
    registerWithEureka: true
    fetchRegistry: true
    healthcheck:
      enabled: true
  instance:
    preferIpAddress: true
    leaseRenewalIntervalInSeconds: 30    # Heartbeat cada 30s
    leaseExpirationDurationInSeconds: 90 # Expira si no hay heartbeat en 90s
    metadata-map:
      version: 1.0.0
      environment: ${spring.profiles.active:dev}
```

**Enable Eureka Client:**

```java
package com.amigusto.event;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;
import org.springframework.cloud.openfeign.EnableFeignClients;

@SpringBootApplication
@EnableEurekaClient      // Habilita cliente Eureka
@EnableFeignClients      // Habilita clientes Feign
public class EventServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(EventServiceApplication.class, args);
    }
}
```

#### 8.4 Distributed Tracing - Sleuth + Zipkin

**Configuración de Sleuth:**

```yaml
# application.yml
spring:
  sleuth:
    sampler:
      probability: 1.0  # 100% en dev, 0.1 (10%) en producción
  zipkin:
    base-url: http://localhost:9411
    sender:
      type: web

logging:
  pattern:
    level: "%5p [${spring.application.name:},%X{traceId:-},%X{spanId:-}]"
```

**Dependencias Maven:**

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-sleuth-zipkin</artifactId>
</dependency>
```

**Logs con Trace ID:**

```java
package com.amigusto.event.service;

import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

@Service
@Slf4j
public class EventService {

    public void createEvent(EventRequest request) {
        // Sleuth automáticamente agrega traceId y spanId a los logs
        log.info("Creando evento: {}", request.getTitle());

        // Llamadas a otros servicios propagarán el trace ID
        promoterClient.getPromoter(request.getPromoterId());

        log.info("Evento creado exitosamente");
        // Logs se verán como:
        // INFO [event-service,abc123,def456] Creando evento: Concierto
    }
}
```

#### 8.5 API Gateway - Filtros Personalizados

**Global Filter - Logging:**

```java
package com.amigusto.gateway.filter;

import lombok.extern.slf4j.Slf4j;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.http.HttpHeaders;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.time.Instant;

@Component
@Slf4j
public class LoggingGlobalFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        long startTime = Instant.now().toEpochMilli();
        String path = exchange.getRequest().getPath().toString();
        String method = exchange.getRequest().getMethod().toString();

        log.info("Request: {} {}", method, path);

        return chain.filter(exchange)
            .then(Mono.fromRunnable(() -> {
                long duration = Instant.now().toEpochMilli() - startTime;
                int statusCode = exchange.getResponse().getStatusCode().value();

                log.info("Response: {} {} - Status: {} - Duration: {}ms",
                    method, path, statusCode, duration);
            }));
    }

    @Override
    public int getOrder() {
        return Ordered.LOWEST_PRECEDENCE; // Ejecutar al final
    }
}
```

**Gateway Filter - JWT Validation:**

```java
package com.amigusto.gateway.filter;

import com.amigusto.gateway.util.JwtUtil;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.cloud.gateway.filter.factory.AbstractGatewayFilterFactory;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

@Component
@Slf4j
public class JwtAuthenticationFilter
        extends AbstractGatewayFilterFactory<JwtAuthenticationFilter.Config> {

    private final JwtUtil jwtUtil;

    public JwtAuthenticationFilter(JwtUtil jwtUtil) {
        super(Config.class);
        this.jwtUtil = jwtUtil;
    }

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            String path = exchange.getRequest().getPath().toString();

            // Skip JWT validation para rutas públicas
            if (isPublicPath(path)) {
                return chain.filter(exchange);
            }

            // Extraer token JWT del header
            String token = extractToken(exchange);

            if (token == null || !jwtUtil.validateToken(token)) {
                log.warn("Token JWT inválido o ausente para: {}", path);
                exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
                return exchange.getResponse().setComplete();
            }

            // Agregar userId al header para downstream services
            String userId = jwtUtil.extractUserId(token);
            ServerWebExchange mutatedExchange = exchange.mutate()
                .request(r -> r.header("X-User-Id", userId))
                .build();

            log.info("Request autenticado para userId: {}", userId);

            return chain.filter(mutatedExchange);
        };
    }

    private String extractToken(ServerWebExchange exchange) {
        String authHeader = exchange.getRequest()
            .getHeaders()
            .getFirst(HttpHeaders.AUTHORIZATION);

        if (authHeader != null && authHeader.startsWith("Bearer ")) {
            return authHeader.substring(7);
        }
        return null;
    }

    private boolean isPublicPath(String path) {
        return path.startsWith("/api/v1/auth") ||
               path.startsWith("/api/v1/gustos") ||
               path.contains("/swagger") ||
               path.contains("/actuator/health");
    }

    public static class Config {
        // Configuración del filtro si es necesaria
    }
}
```

**Rate Limiting Filter:**

```java
package com.amigusto.gateway.filter;

import io.github.bucket4j.Bandwidth;
import io.github.bucket4j.Bucket;
import io.github.bucket4j.Bucket4j;
import io.github.bucket4j.Refill;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.cloud.gateway.filter.factory.AbstractGatewayFilterFactory;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;

import java.time.Duration;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@Component
@Slf4j
public class RateLimitingFilter
        extends AbstractGatewayFilterFactory<RateLimitingFilter.Config> {

    private final Map<String, Bucket> cache = new ConcurrentHashMap<>();

    public RateLimitingFilter() {
        super(Config.class);
    }

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            String clientId = exchange.getRequest()
                .getHeaders()
                .getFirst("X-User-Id");

            if (clientId == null) {
                clientId = exchange.getRequest()
                    .getRemoteAddress()
                    .getAddress()
                    .getHostAddress();
            }

            Bucket bucket = resolveBucket(clientId, config);

            if (bucket.tryConsume(1)) {
                return chain.filter(exchange);
            } else {
                log.warn("Rate limit excedido para: {}", clientId);
                exchange.getResponse().setStatusCode(HttpStatus.TOO_MANY_REQUESTS);
                return exchange.getResponse().setComplete();
            }
        };
    }

    private Bucket resolveBucket(String clientId, Config config) {
        return cache.computeIfAbsent(clientId, key -> {
            Bandwidth limit = Bandwidth.classic(
                config.getCapacity(),
                Refill.intervally(
                    config.getCapacity(),
                    Duration.ofMinutes(config.getRefillMinutes())
                )
            );
            return Bucket4j.builder().addLimit(limit).build();
        });
    }

    public static class Config {
        private long capacity = 100;        // 100 requests
        private long refillMinutes = 1;     // por minuto

        public long getCapacity() { return capacity; }
        public void setCapacity(long capacity) { this.capacity = capacity; }

        public long getRefillMinutes() { return refillMinutes; }
        public void setRefillMinutes(long refillMinutes) {
            this.refillMinutes = refillMinutes;
        }
    }
}
```

**Configuración de Routes en API Gateway:**

```yaml
# application.yml (API Gateway)
spring:
  cloud:
    gateway:
      routes:
        # Event Service
        - id: event-service
          uri: lb://EVENT-SERVICE  # Load balanced via Eureka
          predicates:
            - Path=/api/v1/events/**
          filters:
            - name: JwtAuthenticationFilter
            - name: RateLimitingFilter
              args:
                capacity: 100
                refillMinutes: 1
            - RewritePath=/api/v1/events/(?<segment>.*), /${segment}

        # Auth Service (público)
        - id: auth-service
          uri: lb://AUTH-SERVICE
          predicates:
            - Path=/api/v1/auth/**
          filters:
            - RewritePath=/api/v1/auth/(?<segment>.*), /${segment}

        # User Service
        - id: user-service
          uri: lb://USER-SERVICE
          predicates:
            - Path=/api/v1/users/**,/api/v1/saved-events/**
          filters:
            - name: JwtAuthenticationFilter
            - name: RateLimitingFilter
              args:
                capacity: 200
                refillMinutes: 1

      # Global CORS Configuration
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins:
              - "http://localhost:4200"
              - "https://app.amigusto.com"
            allowedMethods:
              - GET
              - POST
              - PUT
              - DELETE
              - PATCH
            allowedHeaders: "*"
            allowCredentials: true
```

#### 8.6 Config Server - Configuración Centralizada

**Config Server Application:**

```java
package com.amigusto.configserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.config.server.EnableConfigServer;

@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

**Configuración de Config Server:**

```yaml
# application.yml (Config Server)
server:
  port: 8888

spring:
  application:
    name: config-server
  cloud:
    config:
      server:
        git:
          uri: https://github.com/amigusto/config-repo
          default-label: main
          clone-on-start: true
          search-paths:
            - '{application}'
          # Autenticación si es repo privado
          username: ${GIT_USERNAME}
          password: ${GIT_PASSWORD}

eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
```

**Estructura del repositorio de configuración:**

```
config-repo/
├── application.yml              # Configuración común a todos
├── application-dev.yml          # Común para dev
├── application-prod.yml         # Común para prod
├── event-service/
│   ├── event-service.yml        # Config específico de event-service
│   ├── event-service-dev.yml
│   └── event-service-prod.yml
├── auth-service/
│   ├── auth-service.yml
│   ├── auth-service-dev.yml
│   └── auth-service-prod.yml
└── api-gateway/
    ├── api-gateway.yml
    ├── api-gateway-dev.yml
    └── api-gateway-prod.yml
```

**Ejemplo de configuración compartida (application.yml):**

```yaml
# application.yml (en repositorio Git)
spring:
  datasource:
    driver-class-name: org.postgresql.Driver
    hikari:
      maximum-pool-size: 10
      minimum-idle: 5
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
    properties:
      hibernate:
        format_sql: true

  rabbitmq:
    host: ${RABBITMQ_HOST:localhost}
    port: ${RABBITMQ_PORT:5672}
    username: ${RABBITMQ_USERNAME:guest}
    password: ${RABBITMQ_PASSWORD:guest}

  redis:
    host: ${REDIS_HOST:localhost}
    port: ${REDIS_PORT:6379}

logging:
  level:
    root: INFO
    com.amigusto: DEBUG
```

**Client Configuration (bootstrap.yml en cada microservicio):**

```yaml
# bootstrap.yml (en cada microservicio)
spring:
  application:
    name: event-service
  cloud:
    config:
      uri: http://localhost:8888
      fail-fast: true
      retry:
        max-attempts: 6
        initial-interval: 1000
        multiplier: 1.5
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:dev}
```

**Refresh Configuration Endpoint:**

```java
package com.amigusto.event.controller;

import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RefreshScope  // Permite refrescar configuración sin reiniciar
public class ConfigTestController {

    @Value("${app.feature.newFeatureEnabled:false}")
    private boolean newFeatureEnabled;

    @GetMapping("/config/status")
    public String getConfigStatus() {
        return "New feature enabled: " + newFeatureEnabled;
    }
}
```

**Refrescar configuración sin reiniciar:**

```bash
# POST request para refrescar configuración
curl -X POST http://localhost:8082/actuator/refresh
```

---

## iOS - Swift + SwiftUI

### 1. Modelo - Event

```swift
import Foundation

struct Event: Identifiable, Codable, Equatable {
    let id: UUID
    let title: String
    let description: String
    let startDate: Date
    let endDate: Date?
    let status: EventStatus

    let lat: Double
    let lng: Double
    let address: String
    let city: String
    let venueName: String?

    let gustos: [Gusto]

    let imageUrl: String?
    let isFree: Bool
    let price: Double?
    let currency: String?

    let externalUrl: String?
    let maxAttendees: Int?

    let promoter: PromoterSimple

    let createdAt: Date
    let updatedAt: Date

    enum CodingKeys: String, CodingKey {
        case id, title, description
        case startDate = "start_date"
        case endDate = "end_date"
        case status, lat, lng, address, city
        case venueName = "venue_name"
        case gustos
        case imageUrl = "image_url"
        case isFree = "is_free"
        case price, currency
        case externalUrl = "external_url"
        case maxAttendees = "max_attendees"
        case promoter
        case createdAt = "created_at"
        case updatedAt = "updated_at"
    }
}

enum EventStatus: String, Codable {
    case draft = "DRAFT"
    case pendingReview = "PENDING_REVIEW"
    case approved = "APPROVED"
    case rejected = "REJECTED"
    case cancelled = "CANCELLED"
    case ended = "ENDED"
}

struct Gusto: Identifiable, Codable, Hashable {
    let id: UUID
    let name: String
    let icon: String
    let category: String
}

struct PromoterSimple: Codable, Equatable {
    let id: UUID
    let name: String
    let logoUrl: String?

    enum CodingKeys: String, CodingKey {
        case id, name
        case logoUrl = "logo_url"
    }
}
```

### 2. APIService - Networking

```swift
import Foundation
import Combine

enum APIError: Error {
    case invalidURL
    case requestFailed(Error)
    case invalidResponse
    case unauthorized
    case serverError(String)
    case decodingError(Error)
}

class APIService {
    static let shared = APIService()

    private let baseURL = "https://api.amigusto.com/api/v1"
    private let decoder: JSONDecoder = {
        let decoder = JSONDecoder()
        decoder.dateDecodingStrategy = .iso8601
        return decoder
    }()

    private init() {}

    // MARK: - Generic Request

    private func request<T: Decodable>(
        endpoint: String,
        method: String = "GET",
        body: Encodable? = nil,
        requiresAuth: Bool = true
    ) -> AnyPublisher<T, APIError> {

        guard let url = URL(string: "\(baseURL)\(endpoint)") else {
            return Fail(error: APIError.invalidURL).eraseToAnyPublisher()
        }

        var request = URLRequest(url: url)
        request.httpMethod = method
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")

        if requiresAuth, let token = KeychainManager.shared.getToken() {
            request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        }

        if let body = body {
            request.httpBody = try? JSONEncoder().encode(body)
        }

        return URLSession.shared.dataTaskPublisher(for: request)
            .tryMap { data, response -> Data in
                guard let httpResponse = response as? HTTPURLResponse else {
                    throw APIError.invalidResponse
                }

                switch httpResponse.statusCode {
                case 200...299:
                    return data
                case 401:
                    throw APIError.unauthorized
                default:
                    let errorMessage = String(data: data, encoding: .utf8) ?? "Unknown error"
                    throw APIError.serverError(errorMessage)
                }
            }
            .decode(type: T.self, decoder: decoder)
            .mapError { error in
                if let apiError = error as? APIError {
                    return apiError
                } else if error is DecodingError {
                    return APIError.decodingError(error)
                } else {
                    return APIError.requestFailed(error)
                }
            }
            .receive(on: DispatchQueue.main)
            .eraseToAnyPublisher()
    }

    // MARK: - Events

    func discoverEvents(
        lat: Double,
        lng: Double,
        gustoIds: [UUID],
        city: String,
        page: Int = 0,
        size: Int = 20
    ) -> AnyPublisher<PageResponse<Event>, APIError> {

        var components = URLComponents(string: "\(baseURL)/events/discover")!
        components.queryItems = [
            URLQueryItem(name: "lat", value: "\(lat)"),
            URLQueryItem(name: "lng", value: "\(lng)"),
            URLQueryItem(name: "city", value: city),
            URLQueryItem(name: "page", value: "\(page)"),
            URLQueryItem(name: "size", value: "\(size)")
        ]

        gustoIds.forEach { id in
            components.queryItems?.append(URLQueryItem(name: "gustoIds", value: id.uuidString))
        }

        guard let url = components.url else {
            return Fail(error: APIError.invalidURL).eraseToAnyPublisher()
        }

        var request = URLRequest(url: url)
        if let token = KeychainManager.shared.getToken() {
            request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        }

        return URLSession.shared.dataTaskPublisher(for: request)
            .tryMap { $0.data }
            .decode(type: PageResponse<Event>.self, decoder: decoder)
            .mapError { error in
                if let apiError = error as? APIError {
                    return apiError
                } else {
                    return APIError.decodingError(error)
                }
            }
            .receive(on: DispatchQueue.main)
            .eraseToAnyPublisher()
    }

    func getEventDetails(id: UUID) -> AnyPublisher<Event, APIError> {
        return request(endpoint: "/events/\(id.uuidString)")
    }

    func saveEvent(id: UUID) -> AnyPublisher<Void, APIError> {
        return request(
            endpoint: "/saved-events",
            method: "POST",
            body: ["eventId": id.uuidString]
        )
    }
}

struct PageResponse<T: Decodable>: Decodable {
    let content: [T]
    let pageable: Pageable
    let totalElements: Int
    let totalPages: Int
    let last: Bool
    let first: Bool
    let size: Int
    let number: Int

    struct Pageable: Decodable {
        let pageNumber: Int
        let pageSize: Int
    }
}
```

### 3. ViewModel - EventsViewModel

```swift
import Foundation
import Combine
import CoreLocation

@MainActor
class EventsViewModel: ObservableObject {
    @Published var events: [Event] = []
    @Published var isLoading = false
    @Published var errorMessage: String?
    @Published var hasMorePages = true

    private var currentPage = 0
    private let pageSize = 20
    private var cancellables = Set<AnyCancellable>()

    private let apiService = APIService.shared
    private let locationManager: CLLocationManager

    init(locationManager: CLLocationManager = CLLocationManager()) {
        self.locationManager = locationManager
        self.locationManager.requestWhenInUseAuthorization()
    }

    func loadEvents(gustoIds: [UUID], city: String, refresh: Bool = false) {
        guard !isLoading else { return }

        if refresh {
            currentPage = 0
            events = []
            hasMorePages = true
        }

        guard hasMorePages else { return }

        isLoading = true
        errorMessage = nil

        guard let location = locationManager.location else {
            errorMessage = "No se pudo obtener tu ubicación"
            isLoading = false
            return
        }

        apiService.discoverEvents(
            lat: location.coordinate.latitude,
            lng: location.coordinate.longitude,
            gustoIds: gustoIds,
            city: city,
            page: currentPage,
            size: pageSize
        )
        .sink(
            receiveCompletion: { [weak self] completion in
                self?.isLoading = false

                if case .failure(let error) = completion {
                    self?.handleError(error)
                }
            },
            receiveValue: { [weak self] response in
                guard let self = self else { return }

                if refresh {
                    self.events = response.content
                } else {
                    self.events.append(contentsOf: response.content)
                }

                self.currentPage += 1
                self.hasMorePages = !response.last
            }
        )
        .store(in: &cancellables)
    }

    func saveEvent(_ event: Event) {
        apiService.saveEvent(id: event.id)
            .sink(
                receiveCompletion: { [weak self] completion in
                    if case .failure(let error) = completion {
                        self?.handleError(error)
                    }
                },
                receiveValue: { [weak self] _ in
                    print("Evento guardado: \(event.title)")
                    // Opcional: Mostrar toast/alert de éxito
                }
            )
            .store(in: &cancellables)
    }

    private func handleError(_ error: APIError) {
        switch error {
        case .unauthorized:
            errorMessage = "Sesión expirada. Por favor inicia sesión nuevamente."
        case .serverError(let message):
            errorMessage = message
        case .decodingError:
            errorMessage = "Error al procesar la respuesta del servidor"
        default:
            errorMessage = "Ha ocurrido un error. Inténtalo de nuevo."
        }
    }
}
```

### 4. Vista - DiscoverView

```swift
import SwiftUI
import MapKit

struct DiscoverView: View {
    @StateObject private var viewModel = EventsViewModel()
    @State private var selectedGustoIds: [UUID] = []
    @State private var selectedCity: String = "Madrid"

    var body: some View {
        NavigationView {
            ZStack {
                if viewModel.isLoading && viewModel.events.isEmpty {
                    ProgressView("Cargando eventos...")
                } else if let errorMessage = viewModel.errorMessage {
                    ErrorView(message: errorMessage) {
                        viewModel.loadEvents(
                            gustoIds: selectedGustoIds,
                            city: selectedCity,
                            refresh: true
                        )
                    }
                } else {
                    ScrollView {
                        LazyVStack(spacing: 16) {
                            ForEach(viewModel.events) { event in
                                NavigationLink(destination: EventDetailView(event: event)) {
                                    EventCard(event: event) {
                                        viewModel.saveEvent(event)
                                    }
                                }
                                .buttonStyle(PlainButtonStyle())

                                // Infinite scroll
                                if event == viewModel.events.last && viewModel.hasMorePages {
                                    ProgressView()
                                        .onAppear {
                                            viewModel.loadEvents(
                                                gustoIds: selectedGustoIds,
                                                city: selectedCity
                                            )
                                        }
                                }
                            }
                        }
                        .padding()
                    }
                    .refreshable {
                        viewModel.loadEvents(
                            gustoIds: selectedGustoIds,
                            city: selectedCity,
                            refresh: true
                        )
                    }
                }
            }
            .navigationTitle("Descubrir")
            .toolbar {
                ToolbarItem(placement: .navigationBarTrailing) {
                    Button(action: {
                        // Abrir filtros
                    }) {
                        Image(systemName: "slider.horizontal.3")
                    }
                }
            }
            .onAppear {
                if viewModel.events.isEmpty {
                    loadUserGustosAndFetchEvents()
                }
            }
        }
    }

    private func loadUserGustosAndFetchEvents() {
        // Cargar gustos del usuario desde UserDefaults o API
        if let storedGustoIds = UserDefaults.standard.array(forKey: "user_gusto_ids") as? [String] {
            selectedGustoIds = storedGustoIds.compactMap { UUID(uuidString: $0) }
        }

        viewModel.loadEvents(gustoIds: selectedGustoIds, city: selectedCity)
    }
}

struct EventCard: View {
    let event: Event
    let onSave: () -> Void

    var body: some View {
        VStack(alignment: .leading, spacing: 8) {
            // Imagen
            if let imageUrl = event.imageUrl, let url = URL(string: imageUrl) {
                AsyncImage(url: url) { phase in
                    switch phase {
                    case .empty:
                        Rectangle()
                            .fill(Color.gray.opacity(0.3))
                            .frame(height: 200)
                            .overlay(ProgressView())
                    case .success(let image):
                        image
                            .resizable()
                            .aspectRatio(contentMode: .fill)
                            .frame(height: 200)
                            .clipped()
                    case .failure:
                        Rectangle()
                            .fill(Color.gray.opacity(0.3))
                            .frame(height: 200)
                            .overlay(Image(systemName: "photo"))
                    @unknown default:
                        EmptyView()
                    }
                }
                .cornerRadius(12)
            }

            // Título
            Text(event.title)
                .font(.headline)
                .foregroundColor(.primary)
                .lineLimit(2)

            // Fecha y ubicación
            HStack(spacing: 4) {
                Image(systemName: "calendar")
                    .font(.caption)
                Text(event.startDate, style: .date)
                    .font(.caption)

                Spacer()

                Image(systemName: "mappin.circle")
                    .font(.caption)
                Text(event.city)
                    .font(.caption)
            }
            .foregroundColor(.secondary)

            // Gustos
            ScrollView(.horizontal, showsIndicators: false) {
                HStack(spacing: 8) {
                    ForEach(event.gustos) { gusto in
                        GustoChip(gusto: gusto)
                    }
                }
            }

            // Precio y botón guardar
            HStack {
                if event.isFree {
                    Text("GRATIS")
                        .font(.caption)
                        .fontWeight(.bold)
                        .foregroundColor(.green)
                        .padding(.horizontal, 8)
                        .padding(.vertical, 4)
                        .background(Color.green.opacity(0.2))
                        .cornerRadius(4)
                } else if let price = event.price {
                    Text("\(price, specifier: "%.2f") €")
                        .font(.caption)
                        .fontWeight(.semibold)
                }

                Spacer()

                Button(action: onSave) {
                    Image(systemName: "bookmark")
                        .font(.title3)
                        .foregroundColor(.blue)
                }
            }
        }
        .padding()
        .background(Color(.systemBackground))
        .cornerRadius(16)
        .shadow(color: .black.opacity(0.1), radius: 8, x: 0, y: 4)
    }
}

struct GustoChip: View {
    let gusto: Gusto

    var body: some View {
        HStack(spacing: 4) {
            Text(gusto.icon)
                .font(.caption)
            Text(gusto.name)
                .font(.caption)
                .fontWeight(.medium)
        }
        .padding(.horizontal, 8)
        .padding(.vertical, 4)
        .background(Color.blue.opacity(0.1))
        .foregroundColor(.blue)
        .cornerRadius(8)
    }
}

struct ErrorView: View {
    let message: String
    let onRetry: () -> Void

    var body: some View {
        VStack(spacing: 16) {
            Image(systemName: "exclamationmark.triangle")
                .font(.system(size: 48))
                .foregroundColor(.orange)

            Text(message)
                .font(.body)
                .multilineTextAlignment(.center)
                .foregroundColor(.secondary)

            Button("Reintentar", action: onRetry)
                .buttonStyle(.borderedProminent)
        }
        .padding()
    }
}
```

---

## Android - Kotlin + Jetpack Compose

### 1. Modelo - Event

```kotlin
package com.amigusto.android.domain.model

import java.time.LocalDateTime
import java.util.UUID

data class Event(
    val id: UUID,
    val title: String,
    val description: String,
    val startDate: LocalDateTime,
    val endDate: LocalDateTime?,
    val status: EventStatus,

    val lat: Double,
    val lng: Double,
    val address: String,
    val city: String,
    val venueName: String?,

    val gustos: List<Gusto>,

    val imageUrl: String?,
    val isFree: Boolean,
    val price: Double?,
    val currency: String?,

    val externalUrl: String?,
    val maxAttendees: Int?,

    val promoter: PromoterSimple,

    val createdAt: LocalDateTime,
    val updatedAt: LocalDateTime
)

enum class EventStatus {
    DRAFT,
    PENDING_REVIEW,
    APPROVED,
    REJECTED,
    CANCELLED,
    ENDED
}

data class Gusto(
    val id: UUID,
    val name: String,
    val icon: String,
    val category: String
)

data class PromoterSimple(
    val id: UUID,
    val name: String,
    val logoUrl: String?
)
```

### 2. API Service - Retrofit

```kotlin
package com.amigusto.android.data.remote.api

import com.amigusto.android.data.remote.dto.EventResponseDto
import com.amigusto.android.data.remote.dto.PageResponse
import retrofit2.http.*
import java.util.UUID

interface ApiService {

    @GET("events/discover")
    suspend fun discoverEvents(
        @Query("lat") lat: Double,
        @Query("lng") lng: Double,
        @Query("gustoIds") gustoIds: List<UUID>,
        @Query("city") city: String,
        @Query("page") page: Int = 0,
        @Query("size") size: Int = 20
    ): PageResponse<EventResponseDto>

    @GET("events/{id}")
    suspend fun getEventDetails(
        @Path("id") id: UUID
    ): EventResponseDto

    @POST("saved-events")
    suspend fun saveEvent(
        @Body request: SaveEventRequest
    )

    @DELETE("saved-events/{id}")
    suspend fun unsaveEvent(
        @Path("id") id: UUID
    )

    @GET("saved-events")
    suspend fun getSavedEvents(
        @Query("page") page: Int = 0,
        @Query("size") size: Int = 20
    ): PageResponse<EventResponseDto>
}

data class SaveEventRequest(
    val eventId: UUID
)
```

### 3. Repository Implementation

```kotlin
package com.amigusto.android.data.repository

import com.amigusto.android.data.local.dao.EventDao
import com.amigusto.android.data.remote.api.ApiService
import com.amigusto.android.data.remote.api.SaveEventRequest
import com.amigusto.android.domain.model.Event
import com.amigusto.android.domain.repository.EventRepository
import com.amigusto.android.data.mapper.toEvent
import com.amigusto.android.data.mapper.toEntity
import com.amigusto.android.util.Resource
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.flow
import retrofit2.HttpException
import java.io.IOException
import java.util.UUID
import javax.inject.Inject

class EventRepositoryImpl @Inject constructor(
    private val apiService: ApiService,
    private val eventDao: EventDao
) : EventRepository {

    override fun discoverEvents(
        lat: Double,
        lng: Double,
        gustoIds: List<UUID>,
        city: String,
        page: Int
    ): Flow<Resource<List<Event>>> = flow {
        emit(Resource.Loading())

        // Intentar obtener datos de caché local primero
        val cachedEvents = eventDao.getAllEvents().map { it.toEvent() }
        if (cachedEvents.isNotEmpty() && page == 0) {
            emit(Resource.Loading(data = cachedEvents))
        }

        try {
            val response = apiService.discoverEvents(
                lat = lat,
                lng = lng,
                gustoIds = gustoIds,
                city = city,
                page = page
            )

            val events = response.content.map { it.toEvent() }

            // Guardar en caché si es la primera página
            if (page == 0) {
                eventDao.deleteAllEvents()
                eventDao.insertEvents(events.map { it.toEntity() })
            }

            emit(Resource.Success(events))

        } catch (e: HttpException) {
            emit(Resource.Error(
                message = "Error del servidor: ${e.code()}",
                data = cachedEvents.takeIf { it.isNotEmpty() }
            ))
        } catch (e: IOException) {
            emit(Resource.Error(
                message = "Sin conexión a internet",
                data = cachedEvents.takeIf { it.isNotEmpty() }
            ))
        } catch (e: Exception) {
            emit(Resource.Error(
                message = "Error inesperado: ${e.localizedMessage}",
                data = cachedEvents.takeIf { it.isNotEmpty() }
            ))
        }
    }

    override fun getEventDetails(id: UUID): Flow<Resource<Event>> = flow {
        emit(Resource.Loading())

        try {
            val eventDto = apiService.getEventDetails(id)
            emit(Resource.Success(eventDto.toEvent()))
        } catch (e: HttpException) {
            emit(Resource.Error("Error al cargar el evento"))
        } catch (e: IOException) {
            emit(Resource.Error("Sin conexión a internet"))
        }
    }

    override suspend fun saveEvent(eventId: UUID): Result<Unit> {
        return try {
            apiService.saveEvent(SaveEventRequest(eventId))
            eventDao.markEventAsSaved(eventId)
            Result.success(Unit)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }

    override suspend fun unsaveEvent(eventId: UUID): Result<Unit> {
        return try {
            apiService.unsaveEvent(eventId)
            eventDao.markEventAsUnsaved(eventId)
            Result.success(Unit)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}

sealed class Resource<T>(val data: T? = null, val message: String? = null) {
    class Success<T>(data: T) : Resource<T>(data)
    class Error<T>(message: String, data: T? = null) : Resource<T>(data, message)
    class Loading<T>(data: T? = null) : Resource<T>(data)
}
```

### 4. ViewModel - EventsViewModel

```kotlin
package com.amigusto.android.presentation.discover

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.amigusto.android.domain.model.Event
import com.amigusto.android.domain.repository.EventRepository
import com.amigusto.android.data.repository.Resource
import dagger.hilt.android.lifecycle.HiltViewModel
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.update
import kotlinx.coroutines.launch
import java.util.UUID
import javax.inject.Inject

@HiltViewModel
class EventsViewModel @Inject constructor(
    private val eventRepository: EventRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow(EventsUiState())
    val uiState: StateFlow<EventsUiState> = _uiState.asStateFlow()

    fun loadEvents(
        lat: Double,
        lng: Double,
        gustoIds: List<UUID>,
        city: String,
        refresh: Boolean = false
    ) {
        if (_uiState.value.isLoading) return

        val page = if (refresh) 0 else _uiState.value.currentPage

        viewModelScope.launch {
            eventRepository.discoverEvents(lat, lng, gustoIds, city, page)
                .collect { resource ->
                    when (resource) {
                        is Resource.Loading -> {
                            _uiState.update { it.copy(
                                isLoading = true,
                                events = resource.data ?: it.events
                            ) }
                        }

                        is Resource.Success -> {
                            val newEvents = resource.data ?: emptyList()
                            _uiState.update { currentState ->
                                currentState.copy(
                                    isLoading = false,
                                    events = if (refresh) newEvents
                                             else currentState.events + newEvents,
                                    errorMessage = null,
                                    currentPage = page + 1,
                                    hasMorePages = newEvents.isNotEmpty()
                                )
                            }
                        }

                        is Resource.Error -> {
                            _uiState.update { it.copy(
                                isLoading = false,
                                errorMessage = resource.message,
                                events = resource.data ?: it.events
                            ) }
                        }
                    }
                }
        }
    }

    fun saveEvent(eventId: UUID) {
        viewModelScope.launch {
            eventRepository.saveEvent(eventId)
                .onSuccess {
                    // Opcional: Mostrar snackbar de éxito
                }
                .onFailure { error ->
                    _uiState.update { it.copy(
                        errorMessage = "Error al guardar evento: ${error.message}"
                    ) }
                }
        }
    }

    fun clearError() {
        _uiState.update { it.copy(errorMessage = null) }
    }
}

data class EventsUiState(
    val events: List<Event> = emptyList(),
    val isLoading: Boolean = false,
    val errorMessage: String? = null,
    val currentPage: Int = 0,
    val hasMorePages: Boolean = true
)
```

### 5. Screen - DiscoverScreen (Jetpack Compose)

```kotlin
package com.amigusto.android.presentation.discover

import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.foundation.lazy.rememberLazyListState
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.layout.ContentScale
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.unit.dp
import androidx.hilt.navigation.compose.hiltViewModel
import coil.compose.AsyncImage
import coil.request.ImageRequest
import com.amigusto.android.domain.model.Event
import com.amigusto.android.domain.model.Gusto
import com.google.accompanist.swiperefresh.SwipeRefresh
import com.google.accompanist.swiperefresh.rememberSwipeRefreshState
import java.time.format.DateTimeFormatter
import java.util.UUID

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun DiscoverScreen(
    viewModel: EventsViewModel = hiltViewModel(),
    onEventClick: (UUID) -> Unit,
    lat: Double,
    lng: Double,
    gustoIds: List<UUID>,
    city: String
) {
    val uiState by viewModel.uiState.collectAsState()
    val listState = rememberLazyListState()
    val snackbarHostState = remember { SnackbarHostState() }

    // Cargar eventos al iniciar
    LaunchedEffect(Unit) {
        if (uiState.events.isEmpty()) {
            viewModel.loadEvents(lat, lng, gustoIds, city)
        }
    }

    // Mostrar error en Snackbar
    LaunchedEffect(uiState.errorMessage) {
        uiState.errorMessage?.let { message ->
            snackbarHostState.showSnackbar(message)
            viewModel.clearError()
        }
    }

    // Infinite scroll
    val shouldLoadMore = remember {
        derivedStateOf {
            val lastVisibleItem = listState.layoutInfo.visibleItemsInfo.lastOrNull()
            lastVisibleItem != null &&
            lastVisibleItem.index >= uiState.events.size - 5 &&
            !uiState.isLoading &&
            uiState.hasMorePages
        }
    }

    LaunchedEffect(shouldLoadMore.value) {
        if (shouldLoadMore.value) {
            viewModel.loadEvents(lat, lng, gustoIds, city)
        }
    }

    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("Descubrir") },
                actions = {
                    IconButton(onClick = { /* Abrir filtros */ }) {
                        Icon(Icons.Default.Tune, contentDescription = "Filtros")
                    }
                }
            )
        },
        snackbarHost = { SnackbarHost(snackbarHostState) }
    ) { paddingValues ->
        SwipeRefresh(
            state = rememberSwipeRefreshState(uiState.isLoading && uiState.events.isEmpty()),
            onRefresh = { viewModel.loadEvents(lat, lng, gustoIds, city, refresh = true) },
            modifier = Modifier.padding(paddingValues)
        ) {
            if (uiState.events.isEmpty() && !uiState.isLoading) {
                EmptyState()
            } else {
                LazyColumn(
                    state = listState,
                    contentPadding = PaddingValues(16.dp),
                    verticalArrangement = Arrangement.spacedBy(16.dp)
                ) {
                    items(
                        items = uiState.events,
                        key = { it.id }
                    ) { event ->
                        EventCard(
                            event = event,
                            onEventClick = { onEventClick(event.id) },
                            onSaveClick = { viewModel.saveEvent(event.id) }
                        )
                    }

                    if (uiState.isLoading && uiState.events.isNotEmpty()) {
                        item {
                            Box(
                                modifier = Modifier.fillMaxWidth(),
                                contentAlignment = Alignment.Center
                            ) {
                                CircularProgressIndicator(modifier = Modifier.padding(16.dp))
                            }
                        }
                    }
                }
            }
        }
    }
}

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun EventCard(
    event: Event,
    onEventClick: () -> Unit,
    onSaveClick: () -> Unit
) {
    Card(
        onClick = onEventClick,
        modifier = Modifier.fillMaxWidth(),
        elevation = CardDefaults.cardElevation(defaultElevation = 4.dp)
    ) {
        Column {
            // Imagen
            event.imageUrl?.let { imageUrl ->
                AsyncImage(
                    model = ImageRequest.Builder(LocalContext.current)
                        .data(imageUrl)
                        .crossfade(true)
                        .build(),
                    contentDescription = event.title,
                    modifier = Modifier
                        .fillMaxWidth()
                        .height(200.dp),
                    contentScale = ContentScale.Crop
                )
            }

            Column(modifier = Modifier.padding(16.dp)) {
                // Título
                Text(
                    text = event.title,
                    style = MaterialTheme.typography.titleLarge,
                    maxLines = 2
                )

                Spacer(modifier = Modifier.height(8.dp))

                // Fecha y ubicación
                Row(
                    modifier = Modifier.fillMaxWidth(),
                    horizontalArrangement = Arrangement.SpaceBetween
                ) {
                    Row(verticalAlignment = Alignment.CenterVertically) {
                        Icon(
                            imageVector = Icons.Default.CalendarToday,
                            contentDescription = null,
                            modifier = Modifier.size(16.dp)
                        )
                        Spacer(modifier = Modifier.width(4.dp))
                        Text(
                            text = event.startDate.format(
                                DateTimeFormatter.ofPattern("dd MMM yyyy")
                            ),
                            style = MaterialTheme.typography.bodySmall
                        )
                    }

                    Row(verticalAlignment = Alignment.CenterVertically) {
                        Icon(
                            imageVector = Icons.Default.Place,
                            contentDescription = null,
                            modifier = Modifier.size(16.dp)
                        )
                        Spacer(modifier = Modifier.width(4.dp))
                        Text(
                            text = event.city,
                            style = MaterialTheme.typography.bodySmall
                        )
                    }
                }

                Spacer(modifier = Modifier.height(8.dp))

                // Gustos
                Row(
                    horizontalArrangement = Arrangement.spacedBy(8.dp),
                    modifier = Modifier.fillMaxWidth()
                ) {
                    event.gustos.take(3).forEach { gusto ->
                        GustoChip(gusto = gusto)
                    }
                    if (event.gustos.size > 3) {
                        Text(
                            text = "+${event.gustos.size - 3}",
                            style = MaterialTheme.typography.bodySmall,
                            modifier = Modifier.align(Alignment.CenterVertically)
                        )
                    }
                }

                Spacer(modifier = Modifier.height(12.dp))

                // Precio y botón guardar
                Row(
                    modifier = Modifier.fillMaxWidth(),
                    horizontalArrangement = Arrangement.SpaceBetween,
                    verticalAlignment = Alignment.CenterVertically
                ) {
                    if (event.isFree) {
                        SuggestionChip(
                            onClick = {},
                            label = { Text("GRATIS") },
                            colors = SuggestionChipDefaults.suggestionChipColors(
                                containerColor = MaterialTheme.colorScheme.primaryContainer
                            )
                        )
                    } else {
                        event.price?.let { price ->
                            Text(
                                text = "%.2f €".format(price),
                                style = MaterialTheme.typography.titleMedium
                            )
                        }
                    }

                    IconButton(onClick = onSaveClick) {
                        Icon(
                            imageVector = Icons.Default.BookmarkBorder,
                            contentDescription = "Guardar evento"
                        )
                    }
                }
            }
        }
    }
}

@Composable
fun GustoChip(gusto: Gusto) {
    SuggestionChip(
        onClick = {},
        label = {
            Row(
                verticalAlignment = Alignment.CenterVertically,
                horizontalArrangement = Arrangement.spacedBy(4.dp)
            ) {
                Text(gusto.icon)
                Text(gusto.name, style = MaterialTheme.typography.labelSmall)
            }
        }
    )
}

@Composable
fun EmptyState() {
    Box(
        modifier = Modifier.fillMaxSize(),
        contentAlignment = Alignment.Center
    ) {
        Column(
            horizontalAlignment = Alignment.CenterHorizontally,
            verticalArrangement = Arrangement.spacedBy(8.dp)
        ) {
            Icon(
                imageVector = Icons.Default.EventBusy,
                contentDescription = null,
                modifier = Modifier.size(64.dp),
                tint = MaterialTheme.colorScheme.outline
            )
            Text(
                text = "No hay eventos disponibles",
                style = MaterialTheme.typography.bodyLarge
            )
        }
    }
}
```

---

## Web - Angular

### 1. Modelo - Event Interface

```typescript
// src/app/core/models/event.model.ts
export interface Event {
  id: string;
  title: string;
  description: string;
  startDate: string;
  endDate?: string;
  status: EventStatus;

  lat: number;
  lng: number;
  address: string;
  city: string;
  venueName?: string;

  gustos: Gusto[];

  imageUrl?: string;
  isFree: boolean;
  price?: number;
  currency?: string;

  externalUrl?: string;
  maxAttendees?: number;

  promoter: PromoterSimple;

  createdAt: string;
  updatedAt: string;
}

export enum EventStatus {
  DRAFT = 'DRAFT',
  PENDING_REVIEW = 'PENDING_REVIEW',
  APPROVED = 'APPROVED',
  REJECTED = 'REJECTED',
  CANCELLED = 'CANCELLED',
  ENDED = 'ENDED'
}

export interface Gusto {
  id: string;
  name: string;
  icon: string;
  category: string;
}

export interface PromoterSimple {
  id: string;
  name: string;
  logoUrl?: string;
}

export interface PageResponse<T> {
  content: T[];
  pageable: {
    pageNumber: number;
    pageSize: number;
  };
  totalElements: number;
  totalPages: number;
  last: boolean;
  first: boolean;
  size: number;
  number: number;
}
```

### 2. Service - EventService

```typescript
// src/app/core/services/event.service.ts
import { Injectable } from '@angular/core';
import { HttpClient, HttpParams } from '@angular/common/http';
import { Observable } from 'rxjs';
import { environment } from '../../../environments/environment';
import { Event, EventStatus, PageResponse } from '../models/event.model';

@Injectable({
  providedIn: 'root'
})
export class EventService {
  private readonly apiUrl = `${environment.apiBaseUrl}/events`;

  constructor(private http: HttpClient) {}

  /**
   * Crear evento (promotor)
   */
  createEvent(request: CreateEventRequest): Observable<Event> {
    return this.http.post<Event>(this.apiUrl, request);
  }

  /**
   * Obtener eventos del promotor
   */
  getMyEvents(page: number = 0, size: number = 20): Observable<PageResponse<Event>> {
    const params = new HttpParams()
      .set('page', page.toString())
      .set('size', size.toString());

    return this.http.get<PageResponse<Event>>(`${this.apiUrl}/my-events`, { params });
  }

  /**
   * Obtener detalles de un evento
   */
  getEventDetails(id: string): Observable<Event> {
    return this.http.get<Event>(`${this.apiUrl}/${id}`);
  }

  /**
   * Actualizar evento
   */
  updateEvent(id: string, request: UpdateEventRequest): Observable<Event> {
    return this.http.put<Event>(`${this.apiUrl}/${id}`, request);
  }

  /**
   * Enviar evento a revisión
   */
  submitForReview(id: string): Observable<Event> {
    return this.http.post<Event>(`${this.apiUrl}/${id}/submit-review`, {});
  }

  /**
   * Cancelar evento
   */
  cancelEvent(id: string): Observable<Event> {
    return this.http.post<Event>(`${this.apiUrl}/${id}/cancel`, {});
  }

  // ADMIN endpoints

  /**
   * Obtener eventos pendientes de revisión
   */
  getPendingReviewEvents(page: number = 0, size: number = 20): Observable<PageResponse<Event>> {
    const params = new HttpParams()
      .set('page', page.toString())
      .set('size', size.toString());

    return this.http.get<PageResponse<Event>>(`${this.apiUrl}/pending-review`, { params });
  }

  /**
   * Aprobar evento
   */
  approveEvent(id: string): Observable<Event> {
    return this.http.post<Event>(`${this.apiUrl}/${id}/approve`, {});
  }

  /**
   * Rechazar evento
   */
  rejectEvent(id: string, reason: string): Observable<Event> {
    const params = new HttpParams().set('reason', reason);
    return this.http.post<Event>(`${this.apiUrl}/${id}/reject`, {}, { params });
  }
}

export interface CreateEventRequest {
  title: string;
  description: string;
  startDate: string;
  endDate?: string;

  lat: number;
  lng: number;
  address: string;
  city: string;
  venueName?: string;

  gustoIds: string[];

  imageUrl?: string;
  isFree: boolean;
  price?: number;
  currency?: string;

  externalUrl?: string;
  maxAttendees?: number;
}

export interface UpdateEventRequest extends Partial<CreateEventRequest> {}
```

### 3. Component - Create Event Form

```typescript
// src/app/features/events/components/create-event/create-event.component.ts
import { Component, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FormBuilder, FormGroup, Validators, ReactiveFormsModule } from '@angular/forms';
import { Router } from '@angular/router';
import { MatFormFieldModule } from '@angular/material/form-field';
import { MatInputModule } from '@angular/material/input';
import { MatButtonModule } from '@angular/material/button';
import { MatDatepickerModule } from '@angular/material/datepicker';
import { MatSelectModule } from '@angular/material/select';
import { MatCheckboxModule } from '@angular/material/checkbox';
import { MatSnackBar, MatSnackBarModule } from '@angular/material/snack-bar';
import { EventService } from '../../../../core/services/event.service';
import { GustoService } from '../../../../core/services/gusto.service';
import { Gusto } from '../../../../core/models/event.model';

@Component({
  selector: 'app-create-event',
  standalone: true,
  imports: [
    CommonModule,
    ReactiveFormsModule,
    MatFormFieldModule,
    MatInputModule,
    MatButtonModule,
    MatDatepickerModule,
    MatSelectModule,
    MatCheckboxModule,
    MatSnackBarModule
  ],
  templateUrl: './create-event.component.html',
  styleUrls: ['./create-event.component.scss']
})
export class CreateEventComponent implements OnInit {
  eventForm!: FormGroup;
  gustos: Gusto[] = [];
  isSubmitting = false;

  constructor(
    private fb: FormBuilder,
    private eventService: EventService,
    private gustoService: GustoService,
    private router: Router,
    private snackBar: MatSnackBar
  ) {
    this.initForm();
  }

  ngOnInit(): void {
    this.loadGustos();
  }

  private initForm(): void {
    this.eventForm = this.fb.group({
      title: ['', [Validators.required, Validators.minLength(5), Validators.maxLength(255)]],
      description: ['', [Validators.required, Validators.minLength(20), Validators.maxLength(2000)]],
      startDate: ['', Validators.required],
      endDate: [''],

      lat: ['', [Validators.required, Validators.min(-90), Validators.max(90)]],
      lng: ['', [Validators.required, Validators.min(-180), Validators.max(180)]],
      address: ['', Validators.required],
      city: ['', Validators.required],
      venueName: [''],

      gustoIds: [[], [Validators.required, Validators.minLength(1)]],

      imageUrl: ['', Validators.pattern('https?://.+')],
      isFree: [false],
      price: [{ value: '', disabled: true }],
      currency: [{ value: 'EUR', disabled: true }],

      externalUrl: ['', Validators.pattern('https?://.+')],
      maxAttendees: ['', Validators.min(1)]
    });

    // Controlar habilitación de precio
    this.eventForm.get('isFree')?.valueChanges.subscribe(isFree => {
      if (isFree) {
        this.eventForm.get('price')?.disable();
        this.eventForm.get('currency')?.disable();
        this.eventForm.patchValue({ price: null });
      } else {
        this.eventForm.get('price')?.enable();
        this.eventForm.get('currency')?.enable();
      }
    });
  }

  private loadGustos(): void {
    this.gustoService.getAllGustos().subscribe({
      next: (gustos) => this.gustos = gustos,
      error: (error) => {
        console.error('Error loading gustos:', error);
        this.snackBar.open('Error al cargar categorías', 'Cerrar', { duration: 3000 });
      }
    });
  }

  onSubmit(): void {
    if (this.eventForm.invalid) {
      this.eventForm.markAllAsTouched();
      return;
    }

    this.isSubmitting = true;

    const formValue = {
      ...this.eventForm.getRawValue(),
      startDate: this.formatDate(this.eventForm.value.startDate),
      endDate: this.eventForm.value.endDate ? this.formatDate(this.eventForm.value.endDate) : undefined
    };

    this.eventService.createEvent(formValue).subscribe({
      next: (event) => {
        this.snackBar.open('Evento creado exitosamente', 'Cerrar', { duration: 3000 });
        this.router.navigate(['/dashboard/events', event.id]);
      },
      error: (error) => {
        console.error('Error creating event:', error);
        this.snackBar.open(
          error.error?.message || 'Error al crear evento',
          'Cerrar',
          { duration: 5000 }
        );
        this.isSubmitting = false;
      }
    });
  }

  private formatDate(date: Date): string {
    return date.toISOString();
  }

  getErrorMessage(fieldName: string): string {
    const control = this.eventForm.get(fieldName);
    if (!control?.errors) return '';

    if (control.errors['required']) return 'Este campo es obligatorio';
    if (control.errors['minlength']) return `Mínimo ${control.errors['minlength'].requiredLength} caracteres`;
    if (control.errors['maxlength']) return `Máximo ${control.errors['maxlength'].requiredLength} caracteres`;
    if (control.errors['min']) return `Valor mínimo: ${control.errors['min'].min}`;
    if (control.errors['max']) return `Valor máximo: ${control.errors['max'].max}`;
    if (control.errors['pattern']) return 'Formato inválido';

    return 'Error de validación';
  }
}
```

### 4. Template - Create Event Form

```html
<!-- src/app/features/events/components/create-event/create-event.component.html -->
<div class="create-event-container">
  <h1>Crear Nuevo Evento</h1>

  <form [formGroup]="eventForm" (ngSubmit)="onSubmit()" class="event-form">

    <!-- Información básica -->
    <section class="form-section">
      <h2>Información Básica</h2>

      <mat-form-field appearance="outline" class="full-width">
        <mat-label>Título del Evento</mat-label>
        <input matInput formControlName="title" placeholder="Ej: Concierto de Jazz en el Parque">
        <mat-error>{{ getErrorMessage('title') }}</mat-error>
      </mat-form-field>

      <mat-form-field appearance="outline" class="full-width">
        <mat-label>Descripción</mat-label>
        <textarea
          matInput
          formControlName="description"
          rows="5"
          placeholder="Describe tu evento con detalle...">
        </textarea>
        <mat-hint align="end">
          {{ eventForm.get('description')?.value?.length || 0 }} / 2000
        </mat-hint>
        <mat-error>{{ getErrorMessage('description') }}</mat-error>
      </mat-form-field>

      <div class="form-row">
        <mat-form-field appearance="outline" class="half-width">
          <mat-label>Fecha de Inicio</mat-label>
          <input matInput [matDatepicker]="startPicker" formControlName="startDate">
          <mat-datepicker-toggle matSuffix [for]="startPicker"></mat-datepicker-toggle>
          <mat-datepicker #startPicker></mat-datepicker>
          <mat-error>{{ getErrorMessage('startDate') }}</mat-error>
        </mat-form-field>

        <mat-form-field appearance="outline" class="half-width">
          <mat-label>Fecha de Fin (Opcional)</mat-label>
          <input matInput [matDatepicker]="endPicker" formControlName="endDate">
          <mat-datepicker-toggle matSuffix [for]="endPicker"></mat-datepicker-toggle>
          <mat-datepicker #endPicker></mat-datepicker>
        </mat-form-field>
      </div>
    </section>

    <!-- Ubicación -->
    <section class="form-section">
      <h2>Ubicación</h2>

      <mat-form-field appearance="outline" class="full-width">
        <mat-label>Dirección</mat-label>
        <input matInput formControlName="address" placeholder="Calle Principal 123">
        <mat-error>{{ getErrorMessage('address') }}</mat-error>
      </mat-form-field>

      <div class="form-row">
        <mat-form-field appearance="outline" class="half-width">
          <mat-label>Ciudad</mat-label>
          <input matInput formControlName="city" placeholder="Madrid">
          <mat-error>{{ getErrorMessage('city') }}</mat-error>
        </mat-form-field>

        <mat-form-field appearance="outline" class="half-width">
          <mat-label>Nombre del Lugar (Opcional)</mat-label>
          <input matInput formControlName="venueName" placeholder="Teatro Nacional">
        </mat-form-field>
      </div>

      <div class="form-row">
        <mat-form-field appearance="outline" class="half-width">
          <mat-label>Latitud</mat-label>
          <input matInput type="number" formControlName="lat" placeholder="40.4168">
          <mat-error>{{ getErrorMessage('lat') }}</mat-error>
        </mat-form-field>

        <mat-form-field appearance="outline" class="half-width">
          <mat-label>Longitud</mat-label>
          <input matInput type="number" formControlName="lng" placeholder="-3.7038">
          <mat-error>{{ getErrorMessage('lng') }}</mat-error>
        </mat-form-field>
      </div>
    </section>

    <!-- Categorías (Gustos) -->
    <section class="form-section">
      <h2>Categorías</h2>

      <mat-form-field appearance="outline" class="full-width">
        <mat-label>Selecciona Categorías</mat-label>
        <mat-select formControlName="gustoIds" multiple>
          @for (gusto of gustos; track gusto.id) {
            <mat-option [value]="gusto.id">
              {{ gusto.icon }} {{ gusto.name }}
            </mat-option>
          }
        </mat-select>
        <mat-error>{{ getErrorMessage('gustoIds') }}</mat-error>
      </mat-form-field>
    </section>

    <!-- Precio -->
    <section class="form-section">
      <h2>Precio</h2>

      <mat-checkbox formControlName="isFree" color="primary">
        Evento Gratuito
      </mat-checkbox>

      <div class="form-row" style="margin-top: 16px;">
        <mat-form-field appearance="outline" class="half-width">
          <mat-label>Precio</mat-label>
          <input matInput type="number" formControlName="price" placeholder="0.00">
          <span matSuffix>€</span>
          <mat-error>{{ getErrorMessage('price') }}</mat-error>
        </mat-form-field>

        <mat-form-field appearance="outline" class="half-width">
          <mat-label>Moneda</mat-label>
          <mat-select formControlName="currency">
            <mat-option value="EUR">EUR (€)</mat-option>
            <mat-option value="USD">USD ($)</mat-option>
            <mat-option value="GBP">GBP (£)</mat-option>
          </mat-select>
        </mat-form-field>
      </div>
    </section>

    <!-- Información adicional -->
    <section class="form-section">
      <h2>Información Adicional</h2>

      <mat-form-field appearance="outline" class="full-width">
        <mat-label>URL de Imagen</mat-label>
        <input matInput formControlName="imageUrl" placeholder="https://ejemplo.com/imagen.jpg">
        <mat-error>{{ getErrorMessage('imageUrl') }}</mat-error>
      </mat-form-field>

      <mat-form-field appearance="outline" class="full-width">
        <mat-label>URL Externa (Tickets, Más Info)</mat-label>
        <input matInput formControlName="externalUrl" placeholder="https://tickets.ejemplo.com">
        <mat-error>{{ getErrorMessage('externalUrl') }}</mat-error>
      </mat-form-field>

      <mat-form-field appearance="outline" class="full-width">
        <mat-label>Capacidad Máxima (Opcional)</mat-label>
        <input matInput type="number" formControlName="maxAttendees" placeholder="100">
        <mat-error>{{ getErrorMessage('maxAttendees') }}</mat-error>
      </mat-form-field>
    </section>

    <!-- Botones -->
    <div class="form-actions">
      <button
        mat-raised-button
        color="primary"
        type="submit"
        [disabled]="eventForm.invalid || isSubmitting">
        {{ isSubmitting ? 'Creando...' : 'Crear Evento' }}
      </button>

      <button
        mat-button
        type="button"
        [routerLink]="['/dashboard']"
        [disabled]="isSubmitting">
        Cancelar
      </button>
    </div>
  </form>
</div>
```

### 5. Styles - Create Event Form

```scss
// src/app/features/events/components/create-event/create-event.component.scss
.create-event-container {
  max-width: 900px;
  margin: 0 auto;
  padding: 24px;

  h1 {
    font-size: 32px;
    font-weight: 500;
    margin-bottom: 32px;
    color: #1976d2;
  }
}

.event-form {
  .form-section {
    margin-bottom: 32px;
    padding: 24px;
    background: #f9f9f9;
    border-radius: 8px;

    h2 {
      font-size: 20px;
      font-weight: 500;
      margin-bottom: 16px;
      color: #333;
    }
  }

  .full-width {
    width: 100%;
  }

  .form-row {
    display: flex;
    gap: 16px;
    margin-bottom: 16px;

    .half-width {
      flex: 1;
    }
  }

  mat-form-field {
    margin-bottom: 16px;
  }

  .form-actions {
    display: flex;
    gap: 16px;
    justify-content: flex-end;
    margin-top: 32px;

    button {
      min-width: 120px;
    }
  }
}
```

---

## 🎯 Conclusión

Este documento proporciona ejemplos completos y funcionales para cada capa del stack tecnológico de Amigusto:

- **Backend**: Entidades JPA, Repositories, Services, Controllers con Spring Boot
- **iOS**: Models, APIService, ViewModels, SwiftUI Views
- **Android**: Models, Retrofit API, Repository, ViewModels, Jetpack Compose Screens
- **Web**: TypeScript Models, Services, Reactive Forms con Angular Material

Todos los ejemplos siguen las mejores prácticas de cada tecnología y están listos para ser adaptados e integrados en el proyecto real.

---

**Última actualización:** 2025-10-26
