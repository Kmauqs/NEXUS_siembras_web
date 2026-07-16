# NEXUS_siembras
Aplicación de control agropecuario para pequeños productores

- **Desarrollador:** NEXUS CREATIO
- **Package Android:** `com.nexuscreatio.nexus_siembras`
- **Versión:** 0.1.0+1
- **Fase actual:** 3e-7 — detalle de zonas calientes en mapa (siguiente en cola)

## Alcance funcional

- Catálogo de plantas/variedades con condiciones edafoclimáticas óptimas.
- Registro de cultivos por predio y lote con georreferenciación GNSS.
- Modelo de etapas fenológicas: siembra directa vs. germinador → trasplante → fenología.
- Cronograma en tres vistas: Gantt, calendario y actividades registradas.
- Registro de tareas completadas con acumulación de HH e insumos consumidos del inventario.
- Compras por año fiscal con adjuntos por proveedor.
- Análisis fisicoquímicos de suelo por lote y condiciones edafoclimáticas por predio.
- **Multi-usuario:** un mismo predio puede tener propietario + colaboradores con roles `trabajador` o `consultor`, con permisos diferenciados por RLS de Postgres.
- **Contribución comunitaria** (opt-in): compartir reportes de patologías anonimizados al catálogo global.
- Notificaciones locales de eventos próximos y vencidos.
- Reportes PDF por cultivo.

## Stack técnico

- **Framework:** Flutter 3.22+ / Dart 3.4+
- **Estado:** Riverpod (`flutter_riverpod ^2`)
- **Router:** `go_router`
- **BD local:** Drift 2.x sobre SQLite (schema v8) — offline-first
- **Sync remoto:** Supabase (Postgres + Auth + Storage + RLS)
- **Auth:** email/password vía `supabase_flutter ^2.16`
- **Permisos:** `permission_handler ^11.3`
- **GNSS:** `geolocator`
- **Notificaciones:** `flutter_local_notifications`
- **PDF:** `pdf` + `printing`
- **Env:** `flutter_dotenv`

## Requisitos previos

1. Flutter SDK 3.22 o superior — `flutter --version`
2. Android Studio con SDK 36 y device/emulador
3. Cuenta gratuita en [Supabase](https://supabase.com/dashboard) (opcional — la app funciona en modo local sin ella)
4. VS Code o Android Studio con extensiones Dart + Flutter

## Estructura del proyecto

```
nexus_siembras/
├── android/                # Config Android (compileSdk 36, permisos, firma)
├── web/                    # Manifest PWA e íconos
├── windows/                # Config Windows desktop
├── lib/
│   ├── main.dart           # Entry point (init dotenv, Supabase, notifs)
│   ├── app.dart            # MaterialApp + gating de onboarding
│   ├── router.dart         # Rutas go_router
│   ├── core/
│   │   ├── theme/          # Material y accesible
│   │   ├── units/          # Catálogo y conversiones de unidades
│   │   ├── widgets/        # AppShell, SyncBadge, UnitDropdown…
│   │   └── reports/        # Builder de PDFs por cultivo
│   ├── data/
│   │   ├── database/       # Schema Drift (v8), migraciones, DAOs
│   │   ├── repositories/   # CultivoRepository, PlantaRepository…
│   │   └── seed/           # Catálogo inicial (plantas, patologías, países)
│   ├── features/
│   │   ├── onboarding/     # 6 pasos: login → preferencias → predio → ubicación → permisos → consentimiento
│   │   ├── auth/           # Login y perfil de cuenta
│   │   ├── dashboard/      # KPIs, alertas, HH, compras del año
│   │   ├── crops/          # Ver/agregar cultivos, registrar tareas
│   │   ├── plants/         # Catálogo de variedades
│   │   ├── predios/        # Admin de predios + colaboradores
│   │   ├── inventory/      # Inventario por predio
│   │   ├── purchases/      # Compras por año fiscal
│   │   ├── schedule/       # Gantt / Calendario / Actividades
│   │   ├── soil/           # Análisis fisicoquímicos
│   │   └── settings/       # Configuración general
│   ├── services/           # SupabaseService, SyncService, NotificationService…
│   └── state/              # Providers Riverpod (auth_state, data_state, app_state)
├── supabase/               # Schemas SQL + scripts de diagnóstico
├── assets/                 # imágenes, animaciones, .env
├── test/
├── pubspec.yaml
├── .env.example
└── README.md
```

## Modelo de datos

**Local (Drift, schema v8).** Tablas principales: `predios`, `lotes`, `cultivos`, `plantas`, `plantaFotos`, `inventarios`, `compras`, `proveedores`, `analisisSuelo`, `condicionesPredio`, `eventosCultivo`, `tareasCompletadas`, `patologias`, `cultivoPatologias`, `predioColaboradores`, `patologiasReportadas`, `configs`, `syncMappings`, `syncTables`.

**Remoto (Postgres + RLS).** Espejo de las tablas anteriores más `predio_shares` y funciones `SECURITY DEFINER`:

- `rol_en_predio(predio_id)` → `'propietario' | 'trabajador' | 'consultor' | NULL`
- `puede_ver_predio(predio_id)` — cualquier rol
- `puede_editar_predio(predio_id)` — propietario o trabajador
- `es_propietario_predio(predio_id)` — solo propietario

Reglas de acceso resumidas:

| Recurso | Propietario | Trabajador | Consultor |
|---|---|---|---|
| Predio, lotes | R/W | R | R |
| Cultivos, eventos, tareas | R/W | R/W | R |
| Inventario | R/W | R/W (consumo) | R |
| Compras | R/W | — | — |
| Análisis suelo, condiciones | R/W | R | R |
| Colaboradores | R/W | — | — |

## Notas de desarrollo

- **Offline-first.** Todas las mutaciones escriben primero a Drift local. El `SyncService` reconcilia con Supabase respetando `updated_at` (last-write-wins).
- **Modo local.** Sin `.env` la app funciona 100% local; el usuario ve el aviso en la pantalla de Cuenta.
- **Migraciones.** Cada bump de `schemaVersion` en `database.dart` requiere una rama `onUpgrade`. La migración v7→v8 añadió `ownerUserId` a `predios`. Al subir schema, correr `dart run build_runner build --delete-conflicting-outputs` antes de compilar.
- **Reset total.** El botón en Cuenta borra la BD local en transacción **antes** de `signOut()` — si se invierte, `signOut` desmonta el widget y corta la ejecución.
- **Android SDK.** Forzado a compileSdk 36 en `android/build.gradle.kts` para compatibilidad con `file_picker`. Kotlin incremental deshabilitado en Windows para evitar errores de caché.
- **Core library desugaring** habilitado en `android/app/build.gradle.kts` para `flutter_local_notifications 17.x`.

## Licencia

Propietario — NEXUS CREATIO. Todos los derechos reservados.

Prohibida la reproducción, distribución, modificación o venta de este código fuente 
sin el permiso expreso y por escrito del titular de los derechos de autor.

La versión web de esta aplicación está destinada únicamente a fines de demostración 
y evaluación. Su publicación, clonación o redistribución en plataformas de terceros 
(incluyendo, de forma enunciativa pero no limitativa, Google Play Store) está 
estrictamente prohibida a menos que se otorgue una licencia comercial oficial.
