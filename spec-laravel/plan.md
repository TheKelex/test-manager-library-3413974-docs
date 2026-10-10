# Plan técnico — Manager Library (React + Laravel)

Este plan define cómo construir lo especificado en [`spec.md`](spec.md) respetando su
[`constitution.md`](constitution.md). Las reglas de datos viven en [`data-model.md`](data-model.md),
la arquitectura en [`architecture.md`](architecture.md) y las tareas con evidencia en
[`tasks.md`](tasks.md).

## 1. Decisiones técnicas

| # | Decisión | Elección para este reto | Por qué |
|---|---|---|---|
| D-01 | Backend runtime/framework | PHP 8.4 y Laravel 13, dependencias exactas fijadas en `composer.lock` | Compatible con el esqueleto y las dependencias actuales generadas por Composer |
| D-02 | Frontend | React, ejecutado en contenedor | Stack exigido para la aplicación de operación |
| D-03 | Backend | PHP con Laravel | Stack exigido; Laravel es el adaptador de entrada HTTP y el mecanismo de composición |
| D-04 | Persistencia | PostgreSQL 16; migraciones de Laravel son propietarias del esquema | Una sola fuente de verdad para DDL y semilla (ADR-001) |
| D-05 | Arquitectura backend | Dominio y aplicación PHP sin dependencias de Laravel; adaptadores Laravel en el borde | Mantener puertos y dependencias hacia el núcleo |
| D-06 | Concurrencia | Actualización condicional/versionada del libro dentro de una transacción, más `CHECK` de rango | Evitar prestar dos veces el último ejemplar (RNF-2, ADR-002) |
| D-07 | Fechas | Instantes UTC (`timestamptz`); el reloj es un puerto | Probar vencimientos sin esperar 14 días |
| D-08 | Reporte | Consulta agregada de PostgreSQL detrás de un puerto de lectura | Resultado estable y eficiente (ADR-004) |
| D-09 | Objetos de valor | `Isbn`, `Quantity`, `DocumentNumber` y `DateRange` en el dominio, sin tablas propias | No tienen identidad independiente |
| D-10 | Portadas | Binario en almacenamiento externo; la base guarda solo `cover_key` | Clave opaca, nunca ruta ni bytes |
| D-11 | Contraseñas | Argon2id detrás del puerto `PasswordHasher` | El dominio no recibe ni almacena la clave en claro |
| D-12 | Arranque | El backend crea el admin inicial desde `ADMIN_USERNAME` y `ADMIN_PASSWORD` | El hash debe producirse mediante el puerto de seguridad |
| D-13 | Autenticación | JWT HS256, duración de 8 horas, secreto en `JWT_SECRET` | Mantener el contrato funcional del spec |
| D-14 | Ejecución | Docker/Compose para API, frontend, base de datos y utilidades | Docker es la única vía de ejecución del proyecto |
| D-15 | Pruebas | Pruebas del dominio y aplicación con PHPUnit/fakes; integración y contrato contra PostgreSQL y API en contenedores | Aislar reglas y verificar comportamiento desplegado |

Las librerías concretas de JWT y cualquier dependencia adicional deben elegirse e incorporarse solo
si son necesarias para satisfacer estas decisiones; no se cambia el contrato para acomodar una librería.

## 2. Docker y momento de implementación

Docker **no se deja para el final**. Desde T-01 se implementa el entorno mínimo reproducible para
construir y probar el esqueleto de Laravel, verificar `GET /health` y disponer de PostgreSQL vacío.
Las imágenes o servicios se añaden cuando cada componente exista.

### T-01: alcance y decisiones de arranque

- `infra/compose.yaml` orquesta `api` y `db`; `db` usa PostgreSQL 16 y un volumen persistente.
- La API se construye desde `api/Dockerfile` y expone HTTP en el puerto 8000 del host.
- El contenedor de API instala las dependencias fijadas por `composer.lock` y sirve Laravel en
  `0.0.0.0:8000`.
- `GET /health` es público y responde exactamente `{"status":"ok"}`. No se cambia a
  `/api/v1/health`: el contrato define explícitamente `/health`.
- PostgreSQL se configura con variables de entorno. `.env` local no se versiona; `.env.example`
  documenta nombres y valores ilustrativos no secretos.
- No se ejecutan `artisan migrate`, seeders ni creación automática de tablas al arrancar. La base
  debe estar disponible, pero sin tablas de negocio hasta T-02.
- Las pruebas T-01 incluyen una prueba HTTP del endpoint y una consulta dentro del contenedor de
  PostgreSQL que confirma que no existen tablas de negocio.

Este es un entorno inicial de desarrollo y verificación, no un despliegue de producción. Las
necesidades de servidor, endurecimiento de imagen y servicios frontend se documentan al integrar
Compose en T-20.

### T-02: migración y semilla

- El esquema funcional es `library`; la conexión PostgreSQL usa `search_path=library,public`.
- Una migración inicial crea el esquema y las seis tablas descritas en `data-model.md`, con los
  nombres, tipos, nulabilidad, claves foráneas e índices especificados.
- Los UUID los proporciona la aplicación; ninguna columna funcional usa `DEFAULT`, timestamps
  automáticos o columnas de auditoría.
- El historial técnico de Laravel se declara como `public.migrations` de forma explícita; las
  seis tablas funcionales se crean en `library`. Esto permite que `migrate:status` encuentre su
  historial aunque la conexión use `library` como esquema actual.
- Los índices parciales específicos de PostgreSQL se declaran explícitamente en la migración.
- `DatabaseSeeder` invoca solo `CategorySeeder`, que inserta las cinco categorías con sus UUID
  fijos. `php artisan migrate --seed` se ejecuta explícitamente; Compose no migra ni siembra al
  arrancar.
- La tabla técnica `public.migrations` es el registro de Laravel y no forma parte de las seis
  entidades. El esquema `library` se crea desde la migración, nunca desde `infra`.
- Se implementan solo las restricciones marcadas como motor para esta tarea. `ck_book_total_positive`
  y `ck_user_role` siguen pendientes de T-18, como indica el modelo.

T-20 integra el núcleo compuesto por API, frontend y PostgreSQL después de que exista el esqueleto
React. Las tareas de `tool` y `page` deben ejecutarse también dentro de contenedores y añadir sus
servicios a Compose cuando existan; no se espera hasta el final para comenzar a trabajar con Docker.

La documentación de Docker se define ahora: responsabilidades de imágenes, redes, volúmenes,
variables de entorno, salud y comando de arranque. La implementación concreta evoluciona junto con
las tareas T-01 y T-20. El motor debe iniciar vacío; las tablas y categorías se crean con migraciones
Laravel ejecutadas explícitamente, no con scripts de inicialización de PostgreSQL.

| Componente | Responsabilidad Docker | Cuándo se verifica |
|---|---|---|
| `infra` | Compose, red, volúmenes, health checks y PostgreSQL vacío | T-01 y T-20 |
| `api` | Imagen PHP/Laravel; ejecución de aplicación y pruebas | Desde T-01 |
| `app` | Imagen de React servida por un servidor web del contenedor | Esqueleto en T-20; flujo funcional en T-22 |
| `tool` | Contenedor de ejecución puntual que siembra por la API | T-21 |
| `page` | Contenedor que sirve el sitio estático | T-23 |

## 3. Configuración por entorno

| Variable | Obligatoria | Uso |
|---|---|---|
| `DATABASE_URL` o variables `DB_*` equivalentes | Sí | Conexión a PostgreSQL del servicio de base de datos |
| `JWT_SECRET` | Sí | Firma de tokens; nunca se versiona |
| `ADMIN_USERNAME` / `ADMIN_PASSWORD` | Sí | Alta inicial del administrador; nunca se versionan |
| `COVERS_DIR` | No | Ubicación del almacenamiento de portadas |
| `CORS_ORIGIN` | Sí | Origen permitido para el frontend React |
| `DB_SCHEMA_PATH` | No | `library,public`, para resolver el esquema funcional del modelo |

Los valores de desarrollo se suministran mediante `.env` ignorado por Git y `.env.example` sin
secretos. En contenedores, la API se conecta al nombre del servicio de PostgreSQL, no a `localhost`.

## 4. Estrategia de pruebas

| Nivel | Qué prueba | Ejecución / dependencia |
|---|---|---|
| Dominio | Invariantes: máximo 3, plazo de 14 días, copias disponibles dentro del rango | PHPUnit; objetos PHP puros, sin Laravel ni base |
| Aplicación | Casos de uso: prestar, devolver, autenticación y reporte | PHPUnit con fakes de los puertos |
| Integración | Repositorios, transacciones y consulta de reporte | PHPUnit contra PostgreSQL real en contenedor |
| Contrato | Rutas, autorización, cuerpo de error y códigos HTTP | API levantada en contenedor |
| Arquitectura | Independencia del dominio, dirección de dependencias y composición única | Pruebas automatizadas sobre namespaces/importaciones y bindings |
| Concurrencia | Dos préstamos simultáneos del último ejemplar | Prueba concurrente contra PostgreSQL en contenedor |
| Frontend | Flujos de usuario y contrato del cliente HTTP | Pruebas del proyecto React dentro de su contenedor |
| E2E | Login, préstamo, devolución y reporte | Stack Compose integrado |

Los comandos de construcción, pruebas y ejecución se documentan para correr mediante Docker, de
acuerdo con la restricción del proyecto.

## 5. Pruebas y entregables de dominio/aplicación (T-03–T-10)

- Los tipos del dominio residen en `api/app/Domain/`; sus pruebas no cargan Laravel ni necesitan
  conexión PostgreSQL.
- Los puertos y servicios residen en `api/app/Application/`. Cada operación exterior se sustituye
  por un fake en PHPUnit; no se escriben adaptadores de base o HTTP en esta fase.
- Los casos de uso de préstamo/devolución entregan los cambios al puerto como una operación
  atómica. Las pruebas de fake verifican que errores de validación no invocan la escritura.
- El calendario de préstamos usa un `Clock` inyectado. Se toma un único instante por operación, se
  guarda UTC, se calcula vencimiento a 14 días y se considera vencido solo cuando `due_at < now`.
- La unicidad de ISBN/nombre de usuario/documento consulta el puerto antes de guardar, sin sustituir
  la restricción última del motor que ya está documentada.
- Resultado de búsqueda y excepciones son tipos de aplicación sin clases Laravel ni códigos HTTP.

## 6. Orden de construcción

Documentar y validar el diseño → esqueleto Laravel y PostgreSQL vacío en Docker → migración inicial
y categorías → dominio → casos de uso con fakes → adaptadores de persistencia, seguridad y archivos
→ API y pruebas de contrato/concurrencia → composición y alta inicial del admin → Compose de núcleo
con esqueleto React → frontend funcional, sembrador y sitio público en contenedores. El detalle y las
dependencias están en [`tasks.md`](tasks.md).

## 7. Alcance de construcción T-11–T-20

Estas tareas completan el reporte y el backend utilizable, y dejan el esqueleto React integrado. No
modifican el contrato de [`api-contract.md`](api-contract.md) ni las fronteras de
[`architecture.md`](architecture.md).

| Tarea | Decisión de implementación | Verificación Docker |
|---|---|---|
| T-11 | Puerto de lectura del reporte; consulta agregada con fechas inclusivas por `loaned_at` y snapshots congelados de `loan_item` | Pruebas del caso de uso y consulta PostgreSQL para rango inválido, desempate, inclusividad y renombre de catálogo |
| T-12 | Adaptadores Eloquent para libros, categorías, socios y usuarios; conversión explícita a entidades/DTOs de aplicación | Pruebas de integración contra el servicio PostgreSQL migrado |
| T-13 | Escrituras atómicas de préstamo/devolución; bloqueo del socio y libros en orden estable, revalidación bajo transacción y control de última copia | Prueba de concurrencia contra PostgreSQL real: una operación exitosa y otra conflicto al competir por la última copia |
| T-14 | Hash Argon2id y JWT HS256 con claims `sub`/`role`, secreto por entorno y 8 horas | Pruebas de hash/token y análisis de respuestas/logs sin secretos |
| T-15 | Disco local privado con clave opaca y validación por contenido de JPEG/PNG hasta 2 MiB | Aceptar 2 MiB, rechazar >2 MiB y contenido falso; referencia nula al quitar portada |
| T-16 | Adaptadores Laravel para cada ruta del contrato, autenticación/autorización por rol, validación y error uniforme | Pruebas HTTP de rutas, códigos, permisos, serialización y cuerpo de error frente al contrato |
| T-17 | Proveedor de composición único; aprovisionamiento idempotente de admin desde variables de entorno | En DB limpia, migración explícita, bootstrap seguro del admin y login HTTP correcto |
| T-18 | Migración reversible con `ck_book_total_positive` y `ck_user_role` | Inserciones inválidas fallan en PostgreSQL; consultas de `data-model.md` reflejan ambas reglas |
| T-19 | Pruebas que bloquean dependencias Laravel/Adapters desde Domain/Application y bindings fuera del composition root | PHPUnit ejecutado en contenedor |
| T-20 | Esqueleto React con Vite y servidor estático de desarrollo en contenedor; PostgreSQL y API en Compose | Arranque Compose, health checks disponibles y cero tablas de negocio antes de migración |

Compose no migra, no siembra y no crea el administrador al arrancar. Las operaciones de
configuración de datos permanecen explícitas. Las variables sensibles se suministran en `.env`
local ignorado y nunca se incorporan al frontend.
