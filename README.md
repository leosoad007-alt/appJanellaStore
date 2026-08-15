# Janella Store

Aplicación móvil de punto de venta (POS) construida en **Flutter**, diseñada para operar **100% offline**: inventario, ventas, créditos a clientes y compras a proveedores, todo persistido en una base de datos SQLite local. Sin backend, sin login, sin conexión a internet requerida.

> App ID Android: `com.janellastore.janella_store` · Versión: `1.0.0+1` · SDK Dart: `^3.9.0`

---

## 📊 Estado del proyecto

**Fase actual: MVP funcional avanzado**, en iteración activa sobre flujos de crédito y reportes.

| Módulo | Estado | Detalle |
|---|---|---|
| Catálogo de productos | ✅ Completo | CRUD, stock, precio de venta, búsqueda |
| Punto de venta (POS) / Carrito | ✅ Completo | Ventas en efectivo y a crédito, validación de stock |
| Clientes | ✅ Completo | CRUD, búsqueda avanzada, integración con contactos del teléfono |
| Créditos y abonos | ✅ Completo | Registro de abonos, distribución automática, eliminación de abonos con recálculo de saldo, pestañas pendiente/saldado |
| Estado de cuenta | ✅ Completo | Timeline de cargos/abonos con saldo corrido y filtro por fecha |
| Anulación de ventas | ✅ Completo | Restaura stock y elimina crédito/abonos asociados |
| Ingresos de mercadería | ✅ Completo | Registro de compras por proveedor, historial y anulación con reversión de stock |
| Kardex | ✅ Completo | Trazabilidad de movimientos de stock |
| Reportes | ✅ Completo | Ventas del día, deuda activa, inversión total, gráfico de productos más vendidos (fl_chart) |
| Proveedores | ✅ Completo | CRUD básico |
| Respaldo/restauración de BD | ✅ Completo | Exportación e importación del archivo SQLite vía `share_plus` / `file_picker` |
| Auto-seeder | ✅ Completo | Carga de datos de ejemplo (60 productos) en primer arranque |
| Pruebas automatizadas | ⚠️ Pendiente | Solo el `widget_test.dart` por defecto de Flutter; sin cobertura de repositorios/lógica de negocio |
| CI/CD | ⚠️ No configurado | Sin workflows de GitHub Actions |
| Distribución iOS/Desktop | ⚠️ Sin publicar | Carpetas `ios/`, `macos/`, `linux/`, `windows/` generadas por Flutter pero sin builds firmados de release |

**20 pantallas**, **7 repositorios de datos**, **9 tablas** relacionales y **4 servicios de dominio** implementados sobre ~8.300 líneas de código Dart (`lib/`).

---

## 🔐 Auditoría de credenciales y datos sensibles

Se revisó el código fuente completo, el historial de commits (`git log --all -p`) y los archivos rastreados por git en busca de API keys, tokens, contraseñas, cadenas de conexión, certificados y credenciales de firma.

**Resultado: no se encontraron credenciales sensibles expuestas.**

| Verificación | Resultado |
|---|---|
| Archivos `.env` / secretos rastreados en git | No existen |
| API keys, tokens, `client_secret`, `password=` en código o historial | No se encontraron coincidencias |
| Backends externos (Firebase, Supabase, REST APIs propias) | No aplica — la app no tiene capa de red ni SDKs de terceros con claves |
| `android/app/build.gradle.kts` — firma de release | Usa `signingConfigs.debug`; **no hay keystore ni contraseñas de firma en el repositorio** |
| `android/gradle.properties` | Solo configuración de memoria de Gradle, sin credenciales |
| `google-services.json` / `GoogleService-Info.plist` / `*.jks` / `*.p12` / `*.pem` | No existen en el repositorio |
| Base de datos | 100% local (SQLite vía Drift), sin credenciales de conexión porque no hay servidor de BD |

**Observaciones de seguridad (no son secretos filtrados, pero conviene tenerlas presentes):**

1. **Sin autenticación en la app.** Es una decisión de diseño ("Sin Login"), pero implica que cualquier persona con acceso físico al dispositivo puede ver y modificar ventas, créditos y stock. Si el negocio requiere trazabilidad por usuario/cajero, se necesitará un módulo de auth local (PIN/huella) — hoy no existe.
2. **Respaldo de base de datos sin cifrar.** `BackupService` exporta el `.sqlite` en texto plano y lo comparte vía el selector nativo (`share_plus`); cualquier app con la que se comparta el archivo tiene acceso íntegro a clientes, deudas y ventas. Si se maneja información sensible de clientes, valorar cifrar el archivo antes de compartirlo.
3. **Antes de compilar un release firmado**, el keystore de producción y sus contraseñas deberán vivir **fuera del repositorio** (variables de entorno / `key.properties` ignorado en `.gitignore`, o un gestor de secretos de CI). Verificar esto explícitamente cuando se configure la firma de release, ya que hoy el proyecto solo firma en modo debug.

No se requiere ninguna acción correctiva inmediata sobre el repositorio actual: no hay nada que rotar ni revocar.

---

## 🏗 Arquitectura

La aplicación sigue una arquitectura por capas simple (sin backend), orientada a mantenibilidad sobre un dominio pequeño-mediano:

```
┌─────────────────────────────────────────────────────────┐
│  Presentación (lib/screens, lib/widgets)                 │
│  - Widgets Flutter + Material 3                          │
│  - Consumen estado vía flutter_riverpod (ConsumerWidget)  │
└───────────────────────────┬────────────────────────────-─┘
                             │
┌────────────────────────────▼───────────────────────────-─┐
│  Estado (lib/providers)                                   │
│  - providers.dart: exposición de repos/servicios como     │
│    providers de Riverpod                                  │
│  - cart_provider.dart: estado del carrito de compras      │
└───────────────────────────┬────────────────────────────-─┘
                             │
┌────────────────────────────▼───────────────────────────-─┐
│  Dominio / Servicios (lib/services)                        │
│  - auto_seeder_service, backup_service,                    │
│    contacts_service, reportes_service                      │
└───────────────────────────┬────────────────────────────-─┘
                             │
┌────────────────────────────▼───────────────────────────-─┐
│  Acceso a datos (lib/data/repositories)                    │
│  - Un repositorio por agregado: clientes, productos,        │
│    proveedores, ventas, créditos, abonos, ingresos           │
└───────────────────────────┬────────────────────────────-─┘
                             │
┌────────────────────────────▼───────────────────────────-─┐
│  Persistencia (lib/data/database)                           │
│  - Drift (ORM type-safe sobre SQLite)                        │
│  - drift_flutter + sqlite3_flutter_libs                      │
│  - Código generado vía build_runner (database.g.dart)        │
└───────────────────────────────────────────────────────────┘
```

**Decisiones clave de arquitectura:**

- **Offline-first real**: no hay capa HTTP ni sincronización remota; toda la lógica de negocio (stock, créditos, saldos) se resuelve localmente y de forma transaccional en Drift.
- **Riverpod como única fuente de estado**: los repositorios y servicios se inyectan como providers (`lib/providers/providers.dart`), evitando singletons globales y facilitando testing (aunque los tests aún no se han escrito).
- **GoRouter declarativo** (`lib/router.dart`): navegación por rutas nombradas con parámetros (`/clientes/:id/creditos`, `/productos/:id`, etc.), en vez de `Navigator.push` imperativo.
- **Modelo relacional normalizado en Drift**: 9 tablas con claves foráneas y `onDelete: cascade` donde corresponde (p. ej. detalles de venta/ingreso, abonos de crédito), evitando lógica de integridad duplicada en Dart.
- **Servicios como capa de infraestructura transversal**, separados de los repositorios: `BackupService` (export/import de archivo SQLite), `ContactsService` (integración con la agenda del teléfono), `AutoSeederService` (datos de demostración en primer arranque), `ReportesService` (agregaciones para el dashboard).

### Modelo de datos (Drift / SQLite)

```
Clientes ──< Ventas ──< VentasDetalle >── Productos
   │            │
   │            └──< Creditos ──< Abonos
   │
   └── (referenciado también en Creditos)

Proveedores ──< IngresosMercaderia ──< IngresosDetalle >── Productos
                                                              │
                                                        AjustesStock
```

- `Clientes`, `Productos`, `Proveedores`: catálogos base.
- `Ventas` / `VentasDetalle`: cabecera-detalle de cada venta; `esCredito` determina si genera un registro en `Creditos`.
- `Creditos` / `Abonos`: saldo vivo por crédito, actualizado en cada abono o anulación.
- `IngresosMercaderia` / `IngresosDetalle`: compras a proveedores, impactan stock de `Productos`.
- `AjustesStock`: bitácora de movimientos de inventario (entradas/salidas) para el Kardex.

---

## 🛠 Stack tecnológico

| Categoría | Tecnología | Versión |
|---|---|---|
| Framework | Flutter | SDK Dart `^3.9.0` |
| Base de datos local | Drift (ORM) + `sqlite3_flutter_libs` | `^2.14.0` |
| Manejo de estado | Riverpod (`flutter_riverpod`) | `^2.4.0` |
| Navegación | GoRouter | `^12.0.0` |
| Gráficos/reportes | `fl_chart` | `^0.65.0` |
| Preferencias locales | `shared_preferences` | `^2.2.0` |
| Contactos del dispositivo | `flutter_contacts` + `permission_handler` | `^1.1.9+2` / `^12.0.1` |
| Compartir/exportar archivos | `share_plus` + `file_picker` | `^12.0.1` / `^10.3.7` |
| Internacionalización/moneda | `intl` (Soles peruanos, `S/`) | `^0.18.0` |
| Codegen de BD | `drift_dev` + `build_runner` | `^2.14.0` / `^2.4.0` |
| Lint | `flutter_lints` | `^5.0.0` |
| Plataformas objetivo | Android (principal), iOS/macOS/Linux/Windows/Web (scaffolding generado, sin builds de release) | — |

---

## 📦 Instalación y ejecución

### Requisitos previos

- Flutter SDK 3.9.0 o superior
- Dart SDK (incluido con Flutter)
- Android SDK (para compilar/ejecutar en Android)

### Pasos

```bash
# 1. Clonar el repositorio
git clone <url-del-repositorio>
cd appJanellaStore

# 2. Instalar dependencias
flutter pub get

# 3. Generar código de Drift (tablas, queries)
dart run build_runner build --delete-conflicting-outputs

# 4. Ejecutar la aplicación
flutter run
```

### Comandos útiles de desarrollo

```bash
flutter analyze                                  # Análisis estático
flutter test                                     # Ejecutar pruebas
dart run build_runner watch -d                   # Regenerar código de Drift en vivo
flutter build apk --release                      # Compilar APK de producción
```

> Ver `DESARROLLO.md` para la guía extendida de emuladores, compilación y distribución del APK.

---

## 🎯 Funcionalidades

- ✅ **100% Offline** — no requiere conexión a internet
- ✅ **Sin login** — acceso directo a la aplicación
- ✅ **Gestión de productos** — catálogo completo con stock y precio de venta
- ✅ **Gestión de clientes** — registro, búsqueda avanzada e integración con contactos del teléfono
- ✅ **Carrito y punto de venta** — ventas con múltiples productos, en efectivo o a crédito
- ✅ **Anulación de ventas** — restaura stock y revierte crédito/abonos asociados
- ✅ **Créditos y abonos** — historial detallado por cliente, pestañas de pendientes/saldados, eliminación individual de abonos con recálculo de saldo
- ✅ **Estado de cuenta del cliente** — timeline de cargos y abonos con saldo corrido, filtrable por fecha
- ✅ **Ingresos de mercadería** — registro de compras por proveedor, historial y anulación con reversión de stock
- ✅ **Kardex** — trazabilidad de movimientos de inventario
- ✅ **Reportes visuales** — ventas del día, deuda activa, inversión total, gráfico de productos más vendidos
- ✅ **Proveedores** — gestión básica
- ✅ **Respaldo y restauración** — exportación/importación del archivo de base de datos
- ✅ **Auto-seeder** — carga de datos de ejemplo en el primer arranque

## 📂 Estructura del proyecto

```
lib/
├── constants/
│   └── app_constants.dart          # Formato de moneda (Soles) y constantes globales
├── data/
│   ├── database/
│   │   ├── database.dart           # Definición de tablas Drift (9 tablas)
│   │   └── database.g.dart         # Código generado por build_runner
│   ├── repositories/                # Un repositorio por agregado de negocio
│   │   ├── clientes_repository.dart
│   │   ├── productos_repository.dart
│   │   ├── proveedores_repository.dart
│   │   ├── ingresos_repository.dart
│   │   ├── ventas_repository.dart
│   │   ├── creditos_repository.dart
│   │   └── abonos_repository.dart
│   └── seeders/
│       └── database_seeder.dart     # Datos de ejemplo (60 productos)
├── providers/
│   ├── providers.dart               # Providers de Riverpod (DB, repos, servicios)
│   └── cart_provider.dart           # Estado del carrito de compras
├── services/
│   ├── auto_seeder_service.dart     # Seeder automático en primer arranque
│   ├── backup_service.dart          # Exportar/restaurar base de datos
│   ├── contacts_service.dart        # Integración con contactos del dispositivo
│   └── reportes_service.dart        # Agregaciones para el dashboard de reportes
├── screens/                          # 20 pantallas de la aplicación
├── widgets/
│   └── product_card.dart
├── router.dart                       # Rutas declarativas (GoRouter)
└── main.dart                         # Entry point, tema Material 3, auto-seeder al arrancar
```

## 🗄 Estructura de la base de datos

| Tabla | Propósito |
|---|---|
| `clientes` | Información de clientes |
| `productos` | Catálogo de productos con stock y precio |
| `proveedores` | Proveedores de mercadería |
| `ingresos_mercaderia` | Cabecera de compras |
| `ingresos_detalle` | Líneas de cada compra |
| `ventas` | Cabecera de ventas (efectivo o crédito) |
| `ventas_detalle` | Líneas de cada venta |
| `creditos` | Créditos activos vinculados a una venta |
| `abonos` | Pagos aplicados a un crédito |
| `ajustes_stock` | Bitácora de entradas/salidas de inventario (Kardex) |

## 🎨 Diseño

Material Design 3, tema claro con `seedColor` morado, tarjetas con elevación y bordes redondeados, gráficos interactivos (`fl_chart`) e indicadores visuales de stock y badges de carrito.

## 🐛 Solución de problemas

```bash
flutter clean
flutter pub get
dart run build_runner build --delete-conflicting-outputs
```

Si la base de datos local presenta problemas de corrupción, desinstala y reinstala la aplicación (no hay migración automática de esquema más allá de lo definido en Drift).

## 🗺 Próximos pasos sugeridos

1. Cobertura de pruebas unitarias sobre repositorios y lógica de créditos/stock (hoy sin cobertura real).
2. Definir estrategia de firma de release para Android (keystore fuera del repo) antes de publicar un APK/AAB firmado.
3. Evaluar autenticación local (PIN/biometría) si el negocio necesita restringir el acceso al dispositivo.
4. Cifrado opcional del archivo de respaldo exportado.
5. Pipeline de CI (análisis estático + tests) en GitHub Actions.

## 📄 Licencia

Proyecto privado, desarrollado para uso específico de Janella Store.
