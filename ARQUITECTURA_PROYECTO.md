# Arquitectura de Proyecto - Amigusto
## Estructura de Directorios y Organización del Monorepo

---

## 1. ESTRUCTURA GENERAL DEL MONOREPO

Recomendación: Usar **Turborepo** o **Nx** para gestionar el monorepo.

```
amigusto/
├── apps/
│   ├── mobile/              # App móvil React Native
│   ├── web-portal/          # Portal web B2B (Next.js)
│   ├── web-admin/           # Panel admin (Next.js)
│   └── api/                 # Backend API (Node.js + Express)
│
├── packages/
│   ├── ui/                  # Componentes UI compartidos
│   ├── types/               # TypeScript types compartidos
│   ├── config/              # Configs compartidas (ESLint, Prettier, TS)
│   ├── utils/               # Utilidades compartidas
│   └── database/            # Prisma schema y cliente
│
├── docs/                    # Documentación del proyecto
├── scripts/                 # Scripts de automatización
├── .github/                 # GitHub Actions workflows
├── docker-compose.yml       # Para desarrollo local
├── turbo.json               # Config de Turborepo
├── package.json             # Root package.json
└── README.md
```

---

## 2. ESTRUCTURA DETALLADA POR APP

### 2.1 App Móvil (`/apps/mobile`)

```
apps/mobile/
├── src/
│   ├── app/                    # Entry point
│   │   └── index.tsx
│   │
│   ├── screens/                # Pantallas principales
│   │   ├── auth/
│   │   │   ├── LoginScreen.tsx
│   │   │   └── RegisterScreen.tsx
│   │   ├── onboarding/
│   │   │   ├── WelcomeScreen.tsx
│   │   │   └── SelectGustosScreen.tsx
│   │   ├── discover/
│   │   │   ├── DiscoverScreen.tsx
│   │   │   ├── EventDetailScreen.tsx
│   │   │   └── FilterScreen.tsx
│   │   ├── plans/
│   │   │   └── MyPlansScreen.tsx
│   │   └── profile/
│   │       ├── ProfileScreen.tsx
│   │       └── EditGustosScreen.tsx
│   │
│   ├── components/             # Componentes reutilizables
│   │   ├── cards/
│   │   │   ├── EventCard.tsx
│   │   │   └── EventCardSkeleton.tsx
│   │   ├── buttons/
│   │   │   ├── SaveButton.tsx
│   │   │   └── ShareButton.tsx
│   │   ├── inputs/
│   │   │   └── SearchInput.tsx
│   │   ├── lists/
│   │   │   ├── EventList.tsx
│   │   │   └── EmptyState.tsx
│   │   ├── pickers/
│   │   │   ├── GustoChip.tsx
│   │   │   └── GustoSelector.tsx
│   │   └── common/
│   │       ├── Header.tsx
│   │       ├── LoadingSpinner.tsx
│   │       └── ErrorView.tsx
│   │
│   ├── navigation/             # Configuración de navegación
│   │   ├── RootNavigator.tsx
│   │   ├── MainTabNavigator.tsx
│   │   ├── AuthStackNavigator.tsx
│   │   └── linking.ts          # Deep linking config
│   │
│   ├── services/               # Servicios de API
│   │   ├── api/
│   │   │   ├── client.ts       # Axios instance
│   │   │   ├── events.ts       # Events API calls
│   │   │   ├── auth.ts         # Auth API calls
│   │   │   ├── gustos.ts       # Gustos API calls
│   │   │   └── users.ts        # Users API calls
│   │   ├── storage/
│   │   │   └── secureStore.ts  # Local storage wrapper
│   │   └── location/
│   │       └── geolocation.ts  # Location services
│   │
│   ├── hooks/                  # Custom React hooks
│   │   ├── useEvents.ts
│   │   ├── useSavedEvents.ts
│   │   ├── useLocation.ts
│   │   ├── useAuth.ts
│   │   └── useGustos.ts
│   │
│   ├── stores/                 # State management (Zustand)
│   │   ├── authStore.ts
│   │   ├── userStore.ts
│   │   ├── eventsStore.ts
│   │   └── locationStore.ts
│   │
│   ├── types/                  # TypeScript types
│   │   ├── models.ts           # API models
│   │   ├── navigation.ts       # Navigation types
│   │   └── api.ts              # API response types
│   │
│   ├── utils/                  # Utilidades
│   │   ├── constants.ts        # Constantes (colores, etc)
│   │   ├── formatters.ts       # Date, price formatters
│   │   ├── validators.ts       # Form validators
│   │   └── analytics.ts        # Analytics helpers
│   │
│   ├── theme/                  # Tema y estilos
│   │   ├── colors.ts
│   │   ├── typography.ts
│   │   ├── spacing.ts
│   │   └── theme.ts
│   │
│   └── assets/                 # Assets estáticos
│       ├── images/
│       ├── fonts/
│       └── icons/
│
├── app.json                    # Expo config
├── eas.json                    # EAS Build config
├── babel.config.js
├── tsconfig.json
└── package.json
```

**Ejemplo de estructura de un screen:**

```typescript
// src/screens/discover/DiscoverScreen.tsx
import React from 'react';
import { View, FlatList } from 'react-native';
import { useEvents } from '@/hooks/useEvents';
import { useUserStore } from '@/stores/userStore';
import { EventCard } from '@/components/cards/EventCard';
import { EmptyState } from '@/components/lists/EmptyState';

export const DiscoverScreen = () => {
  const { selectedGustos, city } = useUserStore();
  const { data, isLoading, fetchNextPage } = useEvents({
    gustos: selectedGustos,
    city,
  });

  return (
    <View>
      <FlatList
        data={data?.pages.flatMap(p => p.events)}
        renderItem={({ item }) => <EventCard event={item} />}
        onEndReached={() => fetchNextPage()}
        ListEmptyComponent={<EmptyState />}
      />
    </View>
  );
};
```

---

### 2.2 Portal Web B2B (`/apps/web-portal`)

```
apps/web-portal/
├── app/                        # Next.js App Router
│   ├── (auth)/                 # Route group para auth
│   │   ├── login/
│   │   │   └── page.tsx
│   │   ├── registro/
│   │   │   └── page.tsx
│   │   └── layout.tsx
│   │
│   ├── (dashboard)/            # Route group protegido
│   │   ├── dashboard/
│   │   │   ├── page.tsx        # Lista de eventos
│   │   │   └── loading.tsx
│   │   ├── eventos/
│   │   │   ├── nuevo/
│   │   │   │   └── page.tsx    # Crear evento
│   │   │   └── [id]/
│   │   │       ├── page.tsx    # Ver evento
│   │   │       └── editar/
│   │   │           └── page.tsx
│   │   ├── perfil/
│   │   │   └── page.tsx
│   │   └── layout.tsx          # Layout con sidebar
│   │
│   ├── (marketing)/            # Landing pages públicas
│   │   ├── page.tsx            # Home
│   │   ├── por-que-amigusto/
│   │   │   └── page.tsx
│   │   ├── preguntas-frecuentes/
│   │   │   └── page.tsx
│   │   └── layout.tsx
│   │
│   ├── api/                    # API Routes (si se necesita)
│   │   ├── auth/
│   │   │   └── [...nextauth]/
│   │   │       └── route.ts
│   │   └── upload/
│   │       └── route.ts        # Upload de imágenes
│   │
│   ├── layout.tsx              # Root layout
│   ├── globals.css
│   └── error.tsx
│
├── components/                 # Componentes React
│   ├── forms/
│   │   ├── EventForm/
│   │   │   ├── EventForm.tsx
│   │   │   ├── BasicInfoStep.tsx
│   │   │   ├── LocationStep.tsx
│   │   │   ├── DateTimeStep.tsx
│   │   │   ├── PricingStep.tsx
│   │   │   └── GustosStep.tsx
│   │   ├── ImageUpload.tsx
│   │   └── LocationAutocomplete.tsx
│   │
│   ├── layouts/
│   │   ├── DashboardLayout.tsx
│   │   ├── Sidebar.tsx
│   │   └── Header.tsx
│   │
│   ├── tables/
│   │   ├── EventsTable.tsx
│   │   └── EventRow.tsx
│   │
│   ├── ui/                     # shadcn/ui components
│   │   ├── button.tsx
│   │   ├── input.tsx
│   │   ├── select.tsx
│   │   ├── dialog.tsx
│   │   └── ...
│   │
│   └── common/
│       ├── EventStatusBadge.tsx
│       ├── LoadingSpinner.tsx
│       └── ErrorBoundary.tsx
│
├── lib/                        # Librerías y utilidades
│   ├── api.ts                  # Axios client
│   ├── auth.ts                 # NextAuth config
│   ├── validations.ts          # Zod schemas
│   ├── utils.ts                # Helpers generales
│   └── constants.ts
│
├── hooks/                      # Custom hooks
│   ├── useEvents.ts
│   ├── useUpload.ts
│   └── useForm.ts
│
├── types/                      # TypeScript types
│   └── index.d.ts
│
├── public/                     # Assets estáticos
│   ├── images/
│   └── icons/
│
├── next.config.js
├── tailwind.config.ts
├── tsconfig.json
└── package.json
```

**Ejemplo de formulario de evento:**

```typescript
// app/(dashboard)/eventos/nuevo/page.tsx
'use client';

import { EventForm } from '@/components/forms/EventForm/EventForm';
import { useRouter } from 'next/navigation';
import { createEvent } from '@/lib/api';

export default function NuevoEventoPage() {
  const router = useRouter();

  const handleSubmit = async (data: EventFormData) => {
    try {
      const event = await createEvent(data);
      router.push(`/dashboard?created=${event.id}`);
    } catch (error) {
      console.error(error);
    }
  };

  return (
    <div className="container mx-auto py-8">
      <h1 className="text-3xl font-bold mb-6">Crear Nuevo Evento</h1>
      <EventForm onSubmit={handleSubmit} />
    </div>
  );
}
```

---

### 2.3 Panel Admin (`/apps/web-admin`)

```
apps/web-admin/
├── app/
│   ├── (auth)/
│   │   └── login/
│   │       └── page.tsx
│   │
│   ├── (admin)/                # Layout protegido (solo admins)
│   │   ├── dashboard/
│   │   │   └── page.tsx        # Métricas y stats
│   │   ├── queue/              # Cola de aprobación
│   │   │   ├── page.tsx
│   │   │   └── loading.tsx
│   │   ├── review/
│   │   │   └── [eventId]/
│   │   │       └── page.tsx    # Página de revisión
│   │   ├── approved/
│   │   │   └── page.tsx
│   │   ├── rejected/
│   │   │   └── page.tsx
│   │   ├── promoters/
│   │   │   ├── page.tsx
│   │   │   └── [id]/
│   │   │       └── page.tsx
│   │   └── layout.tsx
│   │
│   └── layout.tsx
│
├── components/
│   ├── review/
│   │   ├── EventReviewCard.tsx
│   │   ├── ApprovalActions.tsx
│   │   ├── RejectionModal.tsx
│   │   └── EditGustosPanel.tsx
│   ├── tables/
│   │   ├── QueueTable.tsx      # Tabla de eventos pendientes
│   │   └── PromotersTable.tsx
│   ├── stats/
│   │   ├── MetricsCard.tsx
│   │   └── ApprovalChart.tsx
│   └── ui/
│       └── ...
│
├── lib/
│   ├── adminApi.ts             # API calls específicas de admin
│   └── permissions.ts          # Lógica de permisos
│
└── ...
```

---

### 2.4 Backend API (`/apps/api`)

```
apps/api/
├── src/
│   ├── modules/                # Módulos por dominio
│   │   ├── auth/
│   │   │   ├── auth.controller.ts
│   │   │   ├── auth.service.ts
│   │   │   ├── auth.routes.ts
│   │   │   ├── auth.middleware.ts
│   │   │   ├── auth.schemas.ts      # Zod schemas
│   │   │   └── auth.types.ts
│   │   │
│   │   ├── events/
│   │   │   ├── events.controller.ts
│   │   │   ├── events.service.ts
│   │   │   ├── events.repository.ts # Queries a BD
│   │   │   ├── events.routes.ts
│   │   │   ├── events.schemas.ts
│   │   │   └── events.types.ts
│   │   │
│   │   ├── gustos/
│   │   │   ├── gustos.controller.ts
│   │   │   ├── gustos.service.ts
│   │   │   ├── gustos.routes.ts
│   │   │   └── gustos.schemas.ts
│   │   │
│   │   ├── users/
│   │   │   ├── users.controller.ts
│   │   │   ├── users.service.ts
│   │   │   ├── users.repository.ts
│   │   │   ├── users.routes.ts
│   │   │   └── users.schemas.ts
│   │   │
│   │   ├── promoters/
│   │   │   ├── promoters.controller.ts
│   │   │   ├── promoters.service.ts
│   │   │   ├── promoters.routes.ts
│   │   │   └── promoters.schemas.ts
│   │   │
│   │   ├── admin/
│   │   │   ├── admin.controller.ts
│   │   │   ├── admin.service.ts
│   │   │   ├── admin.routes.ts
│   │   │   └── admin.middleware.ts
│   │   │
│   │   └── notifications/
│   │       ├── notifications.service.ts
│   │       ├── email.service.ts
│   │       └── templates/
│   │           ├── event-approved.hbs
│   │           └── event-rejected.hbs
│   │
│   ├── common/                 # Código compartido
│   │   ├── middleware/
│   │   │   ├── auth.middleware.ts
│   │   │   ├── error.middleware.ts
│   │   │   ├── validation.middleware.ts
│   │   │   ├── ratelimit.middleware.ts
│   │   │   └── cors.middleware.ts
│   │   │
│   │   ├── utils/
│   │   │   ├── logger.ts
│   │   │   ├── response.ts
│   │   │   ├── errors.ts
│   │   │   ├── jwt.ts
│   │   │   └── hash.ts
│   │   │
│   │   ├── types/
│   │   │   ├── express.d.ts    # Extend Express types
│   │   │   └── index.ts
│   │   │
│   │   └── constants/
│   │       ├── errors.ts
│   │       └── roles.ts
│   │
│   ├── config/                 # Configuraciones
│   │   ├── database.ts         # Prisma client
│   │   ├── redis.ts            # Redis client
│   │   ├── s3.ts               # S3/Cloudinary config
│   │   ├── email.ts            # Nodemailer config
│   │   └── env.ts              # Validación de env vars (Zod)
│   │
│   ├── jobs/                   # Background jobs (BullMQ)
│   │   ├── queues/
│   │   │   ├── email.queue.ts
│   │   │   └── event.queue.ts
│   │   └── workers/
│   │       ├── email.worker.ts
│   │       └── event.worker.ts # Auto-marcar eventos pasados
│   │
│   ├── app.ts                  # Express app setup
│   └── server.ts               # Entry point
│
├── prisma/
│   ├── schema.prisma           # Ver PLAN_TECNICO_AMIGUSTO.md
│   ├── seed.ts                 # Seed inicial
│   └── migrations/
│
├── tests/                      # Tests
│   ├── unit/
│   ├── integration/
│   └── e2e/
│
├── .env.example
├── Dockerfile
├── docker-compose.yml
├── tsconfig.json
└── package.json
```

**Ejemplo de módulo completo (Events):**

```typescript
// src/modules/events/events.controller.ts
import { Request, Response, NextFunction } from 'express';
import { EventsService } from './events.service';
import { createEventSchema, updateEventSchema } from './events.schemas';

export class EventsController {
  constructor(private eventsService: EventsService) {}

  async getEvents(req: Request, res: Response, next: NextFunction) {
    try {
      const filters = req.query; // Validar con Zod
      const events = await this.eventsService.findEvents(filters);
      res.json({ data: events });
    } catch (error) {
      next(error);
    }
  }

  async createEvent(req: Request, res: Response, next: NextFunction) {
    try {
      const data = createEventSchema.parse(req.body);
      const event = await this.eventsService.createEvent(data, req.user!.id);
      res.status(201).json({ data: event });
    } catch (error) {
      next(error);
    }
  }

  // ... más métodos
}
```

---

## 3. PACKAGES COMPARTIDOS

### 3.1 `/packages/types`

```
packages/types/
├── src/
│   ├── models/
│   │   ├── User.ts
│   │   ├── Event.ts
│   │   ├── Gusto.ts
│   │   └── Promoter.ts
│   ├── api/
│   │   ├── requests.ts
│   │   └── responses.ts
│   └── index.ts
├── tsconfig.json
└── package.json
```

**Ejemplo:**

```typescript
// packages/types/src/models/Event.ts
export interface Event {
  id: string;
  title: string;
  description: string;
  imageUrl: string;
  startDate: string;
  endDate?: string;
  venueName: string;
  venueAddress: string;
  city: string;
  lat: number;
  lng: number;
  isFree: boolean;
  price?: number;
  gustos: Gusto[];
  status: EventStatus;
  // ... más campos
}

export type EventStatus =
  | 'DRAFT'
  | 'PENDING_REVIEW'
  | 'APPROVED'
  | 'REJECTED'
  | 'CANCELLED'
  | 'ENDED';
```

### 3.2 `/packages/ui`

```
packages/ui/
├── src/
│   ├── components/
│   │   ├── Button.tsx          # Componente genérico
│   │   ├── Input.tsx
│   │   └── GustoChip.tsx       # Específico de Amigusto
│   ├── styles/
│   │   └── theme.ts
│   └── index.ts
└── package.json
```

### 3.3 `/packages/database`

```
packages/database/
├── prisma/
│   ├── schema.prisma
│   └── migrations/
├── src/
│   ├── client.ts               # Export Prisma client
│   └── seed.ts
└── package.json
```

**Uso en apps:**

```typescript
// En apps/api/src/config/database.ts
import { prisma } from '@amigusto/database';

export { prisma };
```

---

## 4. CONVENCIONES DE CÓDIGO

### 4.1 Naming Conventions

**TypeScript/JavaScript:**
- **Archivos:** kebab-case (`event-card.tsx`)
- **Componentes:** PascalCase (`EventCard`)
- **Funciones:** camelCase (`getEventById`)
- **Constantes:** UPPER_SNAKE_CASE (`API_BASE_URL`)
- **Interfaces:** PascalCase con "I" opcional (`IEvent` o `Event`)
- **Types:** PascalCase (`EventStatus`)

**Base de Datos (Prisma):**
- **Modelos:** PascalCase singular (`Event`, `User`)
- **Campos:** camelCase (`startDate`, `isFree`)
- **Enums:** PascalCase (`EventStatus`)

**API Endpoints:**
- RESTful naming: `/events`, `/events/:id`
- Plural para colecciones
- Kebab-case para multi-palabra: `/saved-events`

### 4.2 Estructura de Imports

```typescript
// 1. React y librerías externas
import React, { useState } from 'react';
import { View, Text } from 'react-native';
import axios from 'axios';

// 2. Types y constantes
import type { Event } from '@amigusto/types';
import { API_BASE_URL } from '@/utils/constants';

// 3. Servicios y hooks
import { useEvents } from '@/hooks/useEvents';
import { eventService } from '@/services/api/events';

// 4. Componentes
import { EventCard } from '@/components/cards/EventCard';
import { Button } from '@amigusto/ui';

// 5. Estilos
import styles from './styles.module.css';
```

### 4.3 Comentarios y Documentación

**JSDoc para funciones públicas:**

```typescript
/**
 * Obtiene eventos filtrados por gustos y ciudad
 * @param filters - Filtros de búsqueda
 * @param filters.gustos - IDs de gustos seleccionados
 * @param filters.city - Ciudad del usuario
 * @returns Promise con array de eventos
 */
export async function getEvents(filters: EventFilters): Promise<Event[]> {
  // ...
}
```

**TODO comments:**

```typescript
// TODO: Implementar cache en Redis
// FIXME: Este query es lento, optimizar con índice
// NOTE: Este endpoint será deprecado en v2
```

---

## 5. CONFIGURACIÓN DE DESARROLLO

### 5.1 Docker Compose para Desarrollo Local

```yaml
# docker-compose.yml (en root)
version: '3.8'

services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: amigusto
      POSTGRES_PASSWORD: dev_password
      POSTGRES_DB: amigusto_dev
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  api:
    build: ./apps/api
    ports:
      - "3001:3001"
    environment:
      DATABASE_URL: postgresql://amigusto:dev_password@postgres:5432/amigusto_dev
      REDIS_URL: redis://redis:6379
    depends_on:
      - postgres
      - redis
    volumes:
      - ./apps/api:/app
      - /app/node_modules

volumes:
  postgres_data:
```

### 5.2 Scripts de package.json (Root)

```json
{
  "scripts": {
    "dev": "turbo run dev",
    "dev:api": "turbo run dev --filter=api",
    "dev:web": "turbo run dev --filter=web-portal",
    "dev:mobile": "turbo run dev --filter=mobile",
    "build": "turbo run build",
    "test": "turbo run test",
    "lint": "turbo run lint",
    "format": "prettier --write \"**/*.{ts,tsx,js,jsx,json,md}\"",
    "db:studio": "cd packages/database && npx prisma studio",
    "db:migrate": "cd packages/database && npx prisma migrate dev",
    "db:seed": "cd packages/database && npx prisma db seed"
  }
}
```

---

## 6. FLUJO DE TRABAJO GIT

### 6.1 Branching Strategy

```
main (producción)
  ↑
  └── staging (pre-producción)
        ↑
        └── develop (desarrollo)
              ↑
              ├── feature/nombre-feature
              ├── bugfix/nombre-bug
              └── hotfix/nombre-hotfix
```

### 6.2 Convención de Commits (Conventional Commits)

```
tipo(scope): descripción

[cuerpo opcional]

[footer opcional]
```

**Tipos:**
- `feat`: Nueva feature
- `fix`: Bug fix
- `docs`: Documentación
- `style`: Cambios de formato (no afectan código)
- `refactor`: Refactorización
- `test`: Añadir o modificar tests
- `chore`: Tareas de mantenimiento

**Ejemplos:**

```bash
git commit -m "feat(mobile): add event card component"
git commit -m "fix(api): resolve authentication bug"
git commit -m "docs(readme): update setup instructions"
```

---

## 7. VARIABLES DE ENTORNO

### 7.1 Backend API (`.env.example`)

```bash
# Server
NODE_ENV=development
PORT=3001
API_BASE_URL=http://localhost:3001

# Database
DATABASE_URL=postgresql://user:password@localhost:5432/amigusto_dev

# Redis
REDIS_URL=redis://localhost:6379

# JWT
JWT_SECRET=your-super-secret-jwt-key
JWT_REFRESH_SECRET=your-super-secret-refresh-key
JWT_EXPIRATION=15m
JWT_REFRESH_EXPIRATION=7d

# AWS S3 (or Cloudinary)
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_S3_BUCKET=amigusto-images
AWS_REGION=us-east-1

# Cloudinary (alternativa)
CLOUDINARY_CLOUD_NAME=
CLOUDINARY_API_KEY=
CLOUDINARY_API_SECRET=

# Google Maps
GOOGLE_MAPS_API_KEY=

# Email (SendGrid)
SENDGRID_API_KEY=
FROM_EMAIL=noreply@amigusto.com

# Sentry
SENTRY_DSN=

# Rate Limiting
RATE_LIMIT_WINDOW_MS=60000
RATE_LIMIT_MAX_REQUESTS=100
```

### 7.2 Next.js Apps (`.env.local`)

```bash
# API
NEXT_PUBLIC_API_URL=http://localhost:3001

# NextAuth
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=your-nextauth-secret

# Google Maps
NEXT_PUBLIC_GOOGLE_MAPS_API_KEY=

# Analytics
NEXT_PUBLIC_GA_ID=
```

### 7.3 React Native (`.env`)

```bash
# API
API_URL=http://localhost:3001

# Expo
EXPO_PROJECT_ID=

# Firebase
FIREBASE_API_KEY=
FIREBASE_PROJECT_ID=
```

---

Este documento proporciona una visión completa de cómo organizar el proyecto Amigusto desde una perspectiva de arquitectura de código. La estructura propuesta es escalable y sigue las mejores prácticas de la industria.
