# Ejemplos de Código - Amigusto
## Implementaciones de Referencia para Componentes Principales

---

## 1. BACKEND - API EJEMPLOS

### 1.1 Configuración de Express App

```typescript
// apps/api/src/app.ts
import express, { Application } from 'express';
import helmet from 'helmet';
import cors from 'cors';
import { errorMiddleware } from './common/middleware/error.middleware';
import { logger } from './common/utils/logger';
import { authRoutes } from './modules/auth/auth.routes';
import { eventsRoutes } from './modules/events/events.routes';
import { gustosRoutes } from './modules/gustos/gustos.routes';
import { promotersRoutes } from './modules/promoters/promoters.routes';
import { adminRoutes } from './modules/admin/admin.routes';

export const createApp = (): Application => {
  const app = express();

  // Middleware de seguridad
  app.use(helmet());
  app.use(cors({
    origin: process.env.ALLOWED_ORIGINS?.split(',') || '*',
    credentials: true,
  }));

  // Parsers
  app.use(express.json({ limit: '10mb' }));
  app.use(express.urlencoded({ extended: true }));

  // Logging
  app.use((req, res, next) => {
    logger.info(`${req.method} ${req.path}`);
    next();
  });

  // Health check
  app.get('/health', (req, res) => {
    res.json({ status: 'ok', timestamp: new Date().toISOString() });
  });

  // Routes
  app.use('/api/v1/auth', authRoutes);
  app.use('/api/v1/events', eventsRoutes);
  app.use('/api/v1/gustos', gustosRoutes);
  app.use('/api/v1/promoter', promotersRoutes);
  app.use('/api/v1/admin', adminRoutes);

  // Error handling (debe ser el último)
  app.use(errorMiddleware);

  return app;
};
```

### 1.2 Middleware de Autenticación

```typescript
// apps/api/src/common/middleware/auth.middleware.ts
import { Request, Response, NextFunction } from 'express';
import jwt from 'jsonwebtoken';
import { UnauthorizedError } from '../utils/errors';

export interface JWTPayload {
  userId: string;
  email: string;
  role: 'CONSUMER' | 'PROMOTER' | 'ADMIN';
  promoterId?: string;
}

declare global {
  namespace Express {
    interface Request {
      user?: JWTPayload;
    }
  }
}

export const requireAuth = (
  req: Request,
  res: Response,
  next: NextFunction
) => {
  try {
    const authHeader = req.headers.authorization;

    if (!authHeader || !authHeader.startsWith('Bearer ')) {
      throw new UnauthorizedError('No token provided');
    }

    const token = authHeader.substring(7);
    const decoded = jwt.verify(
      token,
      process.env.JWT_SECRET!
    ) as JWTPayload;

    req.user = decoded;
    next();
  } catch (error) {
    next(new UnauthorizedError('Invalid token'));
  }
};

export const requireRole = (...roles: JWTPayload['role'][]) => {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!req.user) {
      return next(new UnauthorizedError('Not authenticated'));
    }

    if (!roles.includes(req.user.role)) {
      return next(new UnauthorizedError('Insufficient permissions'));
    }

    next();
  };
};
```

### 1.3 Service de Eventos (con filtrado geoespacial)

```typescript
// apps/api/src/modules/events/events.service.ts
import { prisma } from '@amigusto/database';
import { Prisma } from '@prisma/client';

export interface EventFilters {
  city: string;
  gustos?: string[];
  isFree?: boolean;
  startDate?: string;
  endDate?: string;
  lat?: number;
  lng?: number;
  radius?: number; // en km
  page?: number;
  limit?: number;
}

export class EventsService {
  async findEvents(filters: EventFilters) {
    const {
      city,
      gustos = [],
      isFree,
      startDate,
      endDate,
      lat,
      lng,
      radius = 50,
      page = 1,
      limit = 20,
    } = filters;

    const skip = (page - 1) * limit;

    // Construir el WHERE clause dinámicamente
    const where: Prisma.EventWhereInput = {
      status: 'APPROVED',
      city,
      ...(isFree !== undefined && { isFree }),
      ...(startDate && { startDate: { gte: new Date(startDate) } }),
      ...(endDate && { endDate: { lte: new Date(endDate) } }),
      ...(gustos.length > 0 && {
        gustos: {
          some: {
            id: { in: gustos },
          },
        },
      }),
    };

    // Si hay lat/lng, filtrar por distancia
    let events;
    if (lat && lng) {
      // Query raw SQL para cálculo de distancia (Haversine)
      events = await prisma.$queryRaw<any[]>`
        SELECT e.*,
          (
            6371 * acos(
              cos(radians(${lat})) * cos(radians(e.lat)) *
              cos(radians(e.lng) - radians(${lng})) +
              sin(radians(${lat})) * sin(radians(e.lat))
            )
          ) AS distance
        FROM "Event" e
        WHERE
          e.status = 'APPROVED'
          AND e.city = ${city}
          AND (
            6371 * acos(
              cos(radians(${lat})) * cos(radians(e.lat)) *
              cos(radians(e.lng) - radians(${lng})) +
              sin(radians(${lat})) * sin(radians(e.lat))
            )
          ) <= ${radius}
        ORDER BY e."startDate" ASC
        LIMIT ${limit}
        OFFSET ${skip}
      `;
    } else {
      // Query normal sin geolocalización
      events = await prisma.event.findMany({
        where,
        include: {
          gustos: true,
          promoter: {
            select: {
              organizationName: true,
            },
          },
        },
        orderBy: {
          startDate: 'asc',
        },
        skip,
        take: limit,
      });
    }

    const totalCount = await prisma.event.count({ where });

    return {
      events,
      pagination: {
        page,
        limit,
        totalPages: Math.ceil(totalCount / limit),
        totalCount,
        hasNextPage: page * limit < totalCount,
        hasPreviousPage: page > 1,
      },
    };
  }

  async createEvent(data: CreateEventDTO, promoterId: string) {
    const event = await prisma.event.create({
      data: {
        ...data,
        promoterId,
        status: 'DRAFT',
        gustos: {
          connect: data.gustoIds.map((id) => ({ id })),
        },
      },
      include: {
        gustos: true,
      },
    });

    return event;
  }

  async updateEventStatus(
    eventId: string,
    status: 'APPROVED' | 'REJECTED',
    adminId: string,
    rejectionReason?: string
  ) {
    const event = await prisma.event.update({
      where: { id: eventId },
      data: {
        status,
        reviewedBy: adminId,
        reviewedAt: new Date(),
        ...(status === 'APPROVED' && { publishedAt: new Date() }),
        ...(status === 'REJECTED' && { rejectionReason }),
      },
    });

    // Enviar notificación al promotor
    // await this.notificationService.notifyPromoter(event, status);

    return event;
  }

  async incrementSaveCount(eventId: string) {
    return prisma.event.update({
      where: { id: eventId },
      data: {
        saveCount: {
          increment: 1,
        },
      },
    });
  }
}
```

### 1.4 Controller de Eventos

```typescript
// apps/api/src/modules/events/events.controller.ts
import { Request, Response, NextFunction } from 'express';
import { EventsService } from './events.service';
import { eventFiltersSchema, createEventSchema } from './events.schemas';

export class EventsController {
  constructor(private eventsService: EventsService) {}

  getEvents = async (req: Request, res: Response, next: NextFunction) => {
    try {
      const filters = eventFiltersSchema.parse(req.query);
      const result = await this.eventsService.findEvents(filters);

      res.json({
        success: true,
        data: result.events,
        pagination: result.pagination,
      });
    } catch (error) {
      next(error);
    }
  };

  getEventById = async (req: Request, res: Response, next: NextFunction) => {
    try {
      const { id } = req.params;
      const event = await this.eventsService.findById(id);

      if (!event) {
        return res.status(404).json({
          success: false,
          error: 'Event not found',
        });
      }

      res.json({ success: true, data: event });
    } catch (error) {
      next(error);
    }
  };

  createEvent = async (req: Request, res: Response, next: NextFunction) => {
    try {
      const data = createEventSchema.parse(req.body);
      const promoterId = req.user!.promoterId;

      if (!promoterId) {
        return res.status(403).json({
          success: false,
          error: 'Only promoters can create events',
        });
      }

      const event = await this.eventsService.createEvent(data, promoterId);

      res.status(201).json({ success: true, data: event });
    } catch (error) {
      next(error);
    }
  };

  saveEvent = async (req: Request, res: Response, next: NextFunction) => {
    try {
      const { id } = req.params;
      const userId = req.user!.userId;

      await this.eventsService.saveEvent(userId, id);

      res.json({ success: true, message: 'Event saved' });
    } catch (error) {
      next(error);
    }
  };
}
```

### 1.5 Validación con Zod

```typescript
// apps/api/src/modules/events/events.schemas.ts
import { z } from 'zod';

export const eventFiltersSchema = z.object({
  city: z.string(),
  gustos: z.string().optional().transform((val) => val?.split(',') || []),
  isFree: z.string().optional().transform((val) => val === 'true'),
  startDate: z.string().datetime().optional(),
  endDate: z.string().datetime().optional(),
  lat: z.string().optional().transform(Number),
  lng: z.string().optional().transform(Number),
  radius: z.string().optional().transform(Number),
  page: z.string().optional().transform((val) => Number(val) || 1),
  limit: z.string().optional().transform((val) => Number(val) || 20),
});

export const createEventSchema = z.object({
  title: z.string().min(3).max(200),
  description: z.string().min(10).max(5000),
  imageUrl: z.string().url(),
  imageGallery: z.array(z.string().url()).optional(),

  eventType: z.enum(['SINGLE', 'MULTI_DAY', 'RECURRING']),
  startDate: z.string().datetime(),
  endDate: z.string().datetime().optional(),
  startTime: z.string().regex(/^\d{2}:\d{2}$/).optional(),
  endTime: z.string().regex(/^\d{2}:\d{2}$/).optional(),

  venueName: z.string().min(2),
  venueAddress: z.string().min(5),
  city: z.string(),
  lat: z.number().min(-90).max(90),
  lng: z.number().min(-180).max(180),

  isFree: z.boolean(),
  price: z.number().positive().optional(),
  ticketUrl: z.string().url().optional(),

  gustoIds: z.array(z.string()).min(1).max(10),

  metadata: z.record(z.any()).optional(),
});

export type CreateEventDTO = z.infer<typeof createEventSchema>;
export type EventFilters = z.infer<typeof eventFiltersSchema>;
```

---

## 2. REACT NATIVE - APP MÓVIL

### 2.1 Custom Hook para Eventos

```typescript
// apps/mobile/src/hooks/useEvents.ts
import { useInfiniteQuery } from '@tanstack/react-query';
import { eventService } from '@/services/api/events';
import { useUserStore } from '@/stores/userStore';

export interface UseEventsOptions {
  city?: string;
  gustos?: string[];
  isFree?: boolean;
}

export const useEvents = (options?: UseEventsOptions) => {
  const { selectedGustos, city: userCity } = useUserStore();

  const city = options?.city || userCity;
  const gustos = options?.gustos || selectedGustos;

  return useInfiniteQuery({
    queryKey: ['events', city, gustos, options?.isFree],
    queryFn: ({ pageParam = 1 }) =>
      eventService.getEvents({
        city: city!,
        gustos,
        isFree: options?.isFree,
        page: pageParam,
        limit: 20,
      }),
    getNextPageParam: (lastPage) =>
      lastPage.pagination.hasNextPage
        ? lastPage.pagination.page + 1
        : undefined,
    enabled: !!city && gustos.length > 0,
  });
};
```

### 2.2 Componente EventCard

```typescript
// apps/mobile/src/components/cards/EventCard.tsx
import React, { useState } from 'react';
import {
  View,
  Text,
  Image,
  TouchableOpacity,
  StyleSheet,
  Pressable,
} from 'react-native';
import { Event } from '@amigusto/types';
import { Ionicons } from '@expo/vector-icons';
import { useNavigation } from '@react-navigation/native';
import { useSaveEvent } from '@/hooks/useSaveEvent';
import { formatDate } from '@/utils/formatters';
import * as Haptics from 'expo-haptics';
import * as Sharing from 'expo-sharing';

interface EventCardProps {
  event: Event;
  isSaved?: boolean;
}

export const EventCard: React.FC<EventCardProps> = ({ event, isSaved = false }) => {
  const navigation = useNavigation();
  const { saveEvent, unsaveEvent, isSaving } = useSaveEvent();
  const [saved, setSaved] = useState(isSaved);

  const handlePress = () => {
    navigation.navigate('EventDetail', { eventId: event.id });
  };

  const handleSave = async () => {
    Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Medium);

    if (saved) {
      await unsaveEvent(event.id);
      setSaved(false);
    } else {
      await saveEvent(event.id);
      setSaved(true);
    }
  };

  const handleShare = async () => {
    const shareUrl = `https://amigusto.com/eventos/${event.id}`;

    await Sharing.shareAsync(shareUrl, {
      dialogTitle: `Mira este evento: ${event.title}`,
    });
  };

  return (
    <Pressable onPress={handlePress} style={styles.container}>
      {/* Imagen */}
      <Image source={{ uri: event.imageUrl }} style={styles.image} />

      {/* Badge de GRATIS */}
      {event.isFree && (
        <View style={styles.freeBadge}>
          <Text style={styles.freeBadgeText}>GRATIS</Text>
        </View>
      )}

      {/* Contenido */}
      <View style={styles.content}>
        {/* Título */}
        <Text style={styles.title} numberOfLines={2}>
          {event.title}
        </Text>

        {/* Fecha y Lugar */}
        <View style={styles.info}>
          <Ionicons name="calendar-outline" size={16} color="#666" />
          <Text style={styles.infoText}>
            {formatDate(event.startDate)}
          </Text>
        </View>

        <View style={styles.info}>
          <Ionicons name="location-outline" size={16} color="#666" />
          <Text style={styles.infoText} numberOfLines={1}>
            {event.venueName}
          </Text>
        </View>

        {/* Gustos (tags) */}
        <View style={styles.gustos}>
          {event.gustos.slice(0, 3).map((gusto) => (
            <View key={gusto.id} style={styles.gustoChip}>
              <Text style={styles.gustoText}>{gusto.icon} {gusto.name}</Text>
            </View>
          ))}
        </View>

        {/* Acciones */}
        <View style={styles.actions}>
          <TouchableOpacity
            onPress={handleSave}
            disabled={isSaving}
            style={styles.actionButton}
          >
            <Ionicons
              name={saved ? 'bookmark' : 'bookmark-outline'}
              size={24}
              color={saved ? '#FF6B6B' : '#333'}
            />
            <Text style={styles.actionText}>
              {saved ? 'Guardado' : 'Guardar'}
            </Text>
          </TouchableOpacity>

          <TouchableOpacity onPress={handleShare} style={styles.actionButton}>
            <Ionicons name="share-social-outline" size={24} color="#333" />
            <Text style={styles.actionText}>Compartir</Text>
          </TouchableOpacity>
        </View>
      </View>
    </Pressable>
  );
};

const styles = StyleSheet.create({
  container: {
    backgroundColor: '#fff',
    borderRadius: 12,
    marginBottom: 16,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
    elevation: 3,
    overflow: 'hidden',
  },
  image: {
    width: '100%',
    height: 200,
    resizeMode: 'cover',
  },
  freeBadge: {
    position: 'absolute',
    top: 12,
    right: 12,
    backgroundColor: '#4CAF50',
    paddingHorizontal: 12,
    paddingVertical: 6,
    borderRadius: 20,
  },
  freeBadgeText: {
    color: '#fff',
    fontWeight: 'bold',
    fontSize: 12,
  },
  content: {
    padding: 16,
  },
  title: {
    fontSize: 18,
    fontWeight: 'bold',
    color: '#333',
    marginBottom: 8,
  },
  info: {
    flexDirection: 'row',
    alignItems: 'center',
    marginBottom: 6,
  },
  infoText: {
    fontSize: 14,
    color: '#666',
    marginLeft: 8,
    flex: 1,
  },
  gustos: {
    flexDirection: 'row',
    flexWrap: 'wrap',
    marginTop: 8,
    marginBottom: 12,
  },
  gustoChip: {
    backgroundColor: '#F5F5F5',
    paddingHorizontal: 10,
    paddingVertical: 4,
    borderRadius: 12,
    marginRight: 6,
    marginBottom: 6,
  },
  gustoText: {
    fontSize: 12,
    color: '#555',
  },
  actions: {
    flexDirection: 'row',
    justifyContent: 'space-around',
    borderTopWidth: 1,
    borderTopColor: '#E0E0E0',
    paddingTop: 12,
  },
  actionButton: {
    flexDirection: 'row',
    alignItems: 'center',
  },
  actionText: {
    fontSize: 14,
    color: '#333',
    marginLeft: 6,
  },
});
```

### 2.3 Screen de Descubrimiento (Feed)

```typescript
// apps/mobile/src/screens/discover/DiscoverScreen.tsx
import React from 'react';
import {
  View,
  FlatList,
  StyleSheet,
  RefreshControl,
  ActivityIndicator,
} from 'react-native';
import { EventCard } from '@/components/cards/EventCard';
import { EmptyState } from '@/components/lists/EmptyState';
import { useEvents } from '@/hooks/useEvents';
import { useSavedEventsIds } from '@/hooks/useSavedEvents';

export const DiscoverScreen = () => {
  const {
    data,
    isLoading,
    isError,
    refetch,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
  } = useEvents();

  const { savedIds } = useSavedEventsIds();

  const events = data?.pages.flatMap((page) => page.events) || [];

  const handleLoadMore = () => {
    if (hasNextPage && !isFetchingNextPage) {
      fetchNextPage();
    }
  };

  const renderFooter = () => {
    if (!isFetchingNextPage) return null;
    return (
      <View style={styles.footer}>
        <ActivityIndicator size="large" color="#FF6B6B" />
      </View>
    );
  };

  if (isLoading) {
    return (
      <View style={styles.centered}>
        <ActivityIndicator size="large" color="#FF6B6B" />
      </View>
    );
  }

  if (isError) {
    return (
      <EmptyState
        icon="alert-circle-outline"
        title="Error al cargar eventos"
        message="Por favor, intenta de nuevo"
        actionLabel="Reintentar"
        onAction={refetch}
      />
    );
  }

  return (
    <View style={styles.container}>
      <FlatList
        data={events}
        keyExtractor={(item) => item.id}
        renderItem={({ item }) => (
          <EventCard event={item} isSaved={savedIds.includes(item.id)} />
        )}
        contentContainerStyle={styles.list}
        onEndReached={handleLoadMore}
        onEndReachedThreshold={0.5}
        ListFooterComponent={renderFooter}
        ListEmptyComponent={
          <EmptyState
            icon="calendar-outline"
            title="No hay eventos"
            message="Aún no hay eventos en tu ciudad que coincidan con tus gustos"
          />
        }
        refreshControl={
          <RefreshControl refreshing={false} onRefresh={refetch} />
        }
      />
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#F9F9F9',
  },
  list: {
    padding: 16,
  },
  centered: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
  },
  footer: {
    paddingVertical: 20,
  },
});
```

### 2.4 Zustand Store para Usuario

```typescript
// apps/mobile/src/stores/userStore.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import AsyncStorage from '@react-native-async-storage/async-storage';

interface UserState {
  userId: string | null;
  email: string | null;
  name: string | null;

  selectedGustos: string[];
  city: string | null;
  lat: number | null;
  lng: number | null;

  setUser: (user: Partial<UserState>) => void;
  setGustos: (gustos: string[]) => void;
  setLocation: (city: string, lat: number, lng: number) => void;
  clearUser: () => void;
}

export const useUserStore = create<UserState>()(
  persist(
    (set) => ({
      userId: null,
      email: null,
      name: null,
      selectedGustos: [],
      city: null,
      lat: null,
      lng: null,

      setUser: (user) => set((state) => ({ ...state, ...user })),

      setGustos: (gustos) => set({ selectedGustos: gustos }),

      setLocation: (city, lat, lng) => set({ city, lat, lng }),

      clearUser: () =>
        set({
          userId: null,
          email: null,
          name: null,
          selectedGustos: [],
          city: null,
          lat: null,
          lng: null,
        }),
    }),
    {
      name: 'user-storage',
      storage: createJSONStorage(() => AsyncStorage),
    }
  )
);
```

---

## 3. NEXT.JS - PORTAL WEB B2B

### 3.1 Formulario de Crear Evento

```typescript
// apps/web-portal/components/forms/EventForm/EventForm.tsx
'use client';

import React from 'react';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Textarea } from '@/components/ui/textarea';
import { Switch } from '@/components/ui/switch';
import { ImageUpload } from './ImageUpload';
import { GustoSelector } from './GustoSelector';
import { LocationAutocomplete } from './LocationAutocomplete';
import { DateTimePicker } from './DateTimePicker';

const eventSchema = z.object({
  title: z.string().min(3, 'Mínimo 3 caracteres').max(200),
  description: z.string().min(10, 'Mínimo 10 caracteres').max(5000),
  imageUrl: z.string().url('Debe ser una URL válida'),

  startDate: z.date(),
  endDate: z.date().optional(),
  startTime: z.string().regex(/^\d{2}:\d{2}$/),
  endTime: z.string().regex(/^\d{2}:\d{2}$/).optional(),

  venueName: z.string().min(2),
  venueAddress: z.string().min(5),
  city: z.string(),
  lat: z.number(),
  lng: z.number(),

  isFree: z.boolean(),
  price: z.number().positive().optional(),
  ticketUrl: z.string().url().optional(),

  gustoIds: z.array(z.string()).min(1, 'Selecciona al menos un gusto'),
});

type EventFormData = z.infer<typeof eventSchema>;

interface EventFormProps {
  initialData?: Partial<EventFormData>;
  onSubmit: (data: EventFormData) => void | Promise<void>;
  submitLabel?: string;
}

export const EventForm: React.FC<EventFormProps> = ({
  initialData,
  onSubmit,
  submitLabel = 'Enviar a Revisión',
}) => {
  const {
    register,
    handleSubmit,
    watch,
    setValue,
    formState: { errors, isSubmitting },
  } = useForm<EventFormData>({
    resolver: zodResolver(eventSchema),
    defaultValues: initialData,
  });

  const isFree = watch('isFree');

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-6">
      {/* Información Básica */}
      <section>
        <h2 className="text-2xl font-semibold mb-4">Información Básica</h2>

        <div className="space-y-4">
          <div>
            <label className="block text-sm font-medium mb-2">
              Nombre del Evento
            </label>
            <Input
              {...register('title')}
              placeholder="ej. Taller de Cerámica para Principiantes"
            />
            {errors.title && (
              <p className="text-red-500 text-sm mt-1">{errors.title.message}</p>
            )}
          </div>

          <div>
            <label className="block text-sm font-medium mb-2">
              Descripción
            </label>
            <Textarea
              {...register('description')}
              rows={6}
              placeholder="Describe tu evento de manera atractiva..."
            />
            {errors.description && (
              <p className="text-red-500 text-sm mt-1">
                {errors.description.message}
              </p>
            )}
          </div>

          <div>
            <label className="block text-sm font-medium mb-2">
              Imagen Principal
            </label>
            <p className="text-sm text-gray-500 mb-2">
              📸 Tip: Sube fotos de calidad, NO flyers con texto
            </p>
            <ImageUpload
              value={watch('imageUrl')}
              onChange={(url) => setValue('imageUrl', url)}
            />
            {errors.imageUrl && (
              <p className="text-red-500 text-sm mt-1">
                {errors.imageUrl.message}
              </p>
            )}
          </div>
        </div>
      </section>

      {/* Fecha y Hora */}
      <section>
        <h2 className="text-2xl font-semibold mb-4">Fecha y Hora</h2>

        <DateTimePicker
          startDate={watch('startDate')}
          startTime={watch('startTime')}
          onStartDateChange={(date) => setValue('startDate', date)}
          onStartTimeChange={(time) => setValue('startTime', time)}
        />
      </section>

      {/* Ubicación */}
      <section>
        <h2 className="text-2xl font-semibold mb-4">Ubicación</h2>

        <LocationAutocomplete
          onSelect={(place) => {
            setValue('venueName', place.name);
            setValue('venueAddress', place.address);
            setValue('city', place.city);
            setValue('lat', place.lat);
            setValue('lng', place.lng);
          }}
        />

        {errors.venueAddress && (
          <p className="text-red-500 text-sm mt-1">
            {errors.venueAddress.message}
          </p>
        )}
      </section>

      {/* Precio */}
      <section>
        <h2 className="text-2xl font-semibold mb-4">Costo</h2>

        <div className="flex items-center space-x-3 mb-4">
          <Switch
            checked={isFree}
            onCheckedChange={(checked) => setValue('isFree', checked)}
          />
          <label className="text-lg font-medium">
            Este evento es GRATIS
          </label>
        </div>

        {!isFree && (
          <div className="space-y-4">
            <div>
              <label className="block text-sm font-medium mb-2">Precio (COP)</label>
              <Input
                type="number"
                {...register('price', { valueAsNumber: true })}
                placeholder="25000"
              />
            </div>

            <div>
              <label className="block text-sm font-medium mb-2">
                Link para comprar tickets (opcional)
              </label>
              <Input
                {...register('ticketUrl')}
                placeholder="https://tuboleta.com/..."
              />
            </div>
          </div>
        )}
      </section>

      {/* Gustos */}
      <section>
        <h2 className="text-2xl font-semibold mb-4">
          ¿Qué tipo de evento es? (Gustos)
        </h2>
        <p className="text-sm text-gray-600 mb-4">
          Selecciona los gustos que mejor describan tu evento. Esto ayuda a que
          llegue a las personas correctas.
        </p>

        <GustoSelector
          selectedIds={watch('gustoIds') || []}
          onChange={(ids) => setValue('gustoIds', ids)}
        />

        {errors.gustoIds && (
          <p className="text-red-500 text-sm mt-1">
            {errors.gustoIds.message}
          </p>
        )}
      </section>

      {/* Submit */}
      <div className="flex justify-end space-x-4">
        <Button type="button" variant="outline">
          Guardar Borrador
        </Button>
        <Button type="submit" disabled={isSubmitting}>
          {isSubmitting ? 'Enviando...' : submitLabel}
        </Button>
      </div>
    </form>
  );
};
```

### 3.2 Componente GustoSelector

```typescript
// apps/web-portal/components/forms/GustoSelector.tsx
'use client';

import React from 'react';
import { useQuery } from '@tanstack/react-query';
import { gustosApi } from '@/lib/api';

interface GustoSelectorProps {
  selectedIds: string[];
  onChange: (ids: string[]) => void;
}

export const GustoSelector: React.FC<GustoSelectorProps> = ({
  selectedIds,
  onChange,
}) => {
  const { data: gustos, isLoading } = useQuery({
    queryKey: ['gustos'],
    queryFn: gustosApi.getAll,
  });

  const toggleGusto = (id: string) => {
    if (selectedIds.includes(id)) {
      onChange(selectedIds.filter((gid) => gid !== id));
    } else {
      onChange([...selectedIds, id]);
    }
  };

  if (isLoading) {
    return <div>Cargando gustos...</div>;
  }

  return (
    <div className="grid grid-cols-2 md:grid-cols-3 lg:grid-cols-4 gap-3">
      {gustos?.map((gusto) => {
        const isSelected = selectedIds.includes(gusto.id);

        return (
          <button
            key={gusto.id}
            type="button"
            onClick={() => toggleGusto(gusto.id)}
            className={`
              p-4 rounded-lg border-2 transition-all
              ${
                isSelected
                  ? 'border-blue-500 bg-blue-50'
                  : 'border-gray-200 hover:border-gray-300'
              }
            `}
          >
            <div className="text-3xl mb-2">{gusto.icon}</div>
            <div className="text-sm font-medium">{gusto.name}</div>
          </button>
        );
      })}
    </div>
  );
};
```

### 3.3 API Client

```typescript
// apps/web-portal/lib/api.ts
import axios from 'axios';

const API_BASE_URL = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:3001';

export const apiClient = axios.create({
  baseURL: `${API_BASE_URL}/api/v1`,
  headers: {
    'Content-Type': 'application/json',
  },
});

// Interceptor para añadir JWT token
apiClient.interceptors.request.use((config) => {
  const token = localStorage.getItem('authToken');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// API methods
export const eventsApi = {
  getAll: async (filters?: any) => {
    const { data } = await apiClient.get('/events', { params: filters });
    return data.data;
  },

  getById: async (id: string) => {
    const { data } = await apiClient.get(`/events/${id}`);
    return data.data;
  },

  create: async (eventData: any) => {
    const { data } = await apiClient.post('/promoter/events', eventData);
    return data.data;
  },

  update: async (id: string, eventData: any) => {
    const { data } = await apiClient.put(`/promoter/events/${id}`, eventData);
    return data.data;
  },

  submitForReview: async (id: string) => {
    const { data } = await apiClient.post(`/promoter/events/${id}/submit-review`);
    return data;
  },
};

export const gustosApi = {
  getAll: async () => {
    const { data } = await apiClient.get('/gustos');
    return data.data;
  },
};

export const authApi = {
  login: async (email: string, password: string) => {
    const { data } = await apiClient.post('/auth/login', { email, password });
    return data;
  },

  register: async (userData: any) => {
    const { data } = await apiClient.post('/auth/register/promoter', userData);
    return data;
  },
};
```

---

## 4. UTILIDADES Y HELPERS

### 4.1 Formatters

```typescript
// packages/utils/src/formatters.ts
import { format, parseISO } from 'date-fns';
import { es } from 'date-fns/locale';

export const formatDate = (dateString: string): string => {
  const date = parseISO(dateString);
  return format(date, "EEEE, d 'de' MMMM", { locale: es });
  // Output: "Sábado, 15 de junio"
};

export const formatTime = (timeString: string): string => {
  return timeString; // Already in "HH:mm" format
};

export const formatPrice = (price: number, currency = 'COP'): string => {
  return new Intl.NumberFormat('es-CO', {
    style: 'currency',
    currency,
    minimumFractionDigits: 0,
  }).format(price);
  // Output: "$25.000"
};

export const formatDistance = (distanceKm: number): string => {
  if (distanceKm < 1) {
    return `${Math.round(distanceKm * 1000)}m`;
  }
  return `${distanceKm.toFixed(1)}km`;
};
```

### 4.2 Logger

```typescript
// apps/api/src/common/utils/logger.ts
import winston from 'winston';

const levels = {
  error: 0,
  warn: 1,
  info: 2,
  debug: 3,
};

const colors = {
  error: 'red',
  warn: 'yellow',
  info: 'green',
  debug: 'blue',
};

winston.addColors(colors);

const format = winston.format.combine(
  winston.format.timestamp({ format: 'YYYY-MM-DD HH:mm:ss' }),
  winston.format.colorize({ all: true }),
  winston.format.printf(
    (info) => `${info.timestamp} ${info.level}: ${info.message}`
  )
);

const transports = [
  new winston.transports.Console(),
  new winston.transports.File({
    filename: 'logs/error.log',
    level: 'error',
  }),
  new winston.transports.File({ filename: 'logs/all.log' }),
];

export const logger = winston.createLogger({
  level: process.env.NODE_ENV === 'development' ? 'debug' : 'info',
  levels,
  format,
  transports,
});
```

---

Este documento proporciona ejemplos concretos de código que el equipo puede usar como punto de partida para implementar los componentes principales de cada plataforma en Amigusto.
