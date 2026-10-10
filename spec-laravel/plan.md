# Plan técnico — Manager Library (React + Laravel)

Este plan define cómo construir lo especificado en [`spec.md`](spec.md) respetando su
[`constitution.md`](constitution.md). Las reglas de datos viven en [`data-model.md`](data-model.md),
la arquitectura en [`architecture.md`](architecture.md) y las tareas con evidencia en
[`tasks.md`](tasks.md).

## 1. Decisiones técnicas

| # | Decisión | Elección para este reto | Por qué |
|---|---|---|---|
| D-01 | Frontend | React, ejecutado en contenedor | Stack exigido para la aplicación de operación |
| D-02 | Backend | PHP con Laravel | Stack exigido; Laravel es el adaptador de entrada HTTP y el mecanismo de composición |
| D-03 | Persistencia | PostgreSQL 16; migraciones de Laravel son propietarias del esquema | Una sola fuente de verdad para DDL y semilla (ADR-001) |
| D-04 | Arquitectura backend | Dominio y aplicación PHP sin dependencias de Laravel; adaptadores Laravel en el borde | Mantener puertos y dependencias hacia el núcleo |
| D-05 | Concurrencia | Actualización condicional/versionada del libro dentro de una transacción, más `CHECK` de rango | Evitar prestar dos veces el último ejemplar (RNF-2, ADR-002) |
| D-06 | Fechas | Instantes UTC (`timestamptz`); el reloj es un puerto | Probar vencimientos sin esperar 14 días |
| D-07 | Reporte | Consulta agregada de PostgreSQL detrás de un puerto de lectura | Resultado estable y eficiente (ADR-004) |
| D-08 | Objetos de valor | `Isbn`, `Quantity`, `DocumentNumber` y `DateRange` en el dominio, sin tablas propias | No tienen identidad independiente |
| D-09 | Portadas | Binario en almacenamiento externo; la base guarda solo `cover_key` | Clave opaca, nunca ruta ni bytes |
| D-10 | Contraseñas | Argon2id detrás del puerto `PasswordHasher` | El dominio no recibe ni almacena la clave en claro |
| D-11 | Arranque | El backend crea el admin inicial desde `ADMIN_USERNAME` y `ADMIN_PASSWORD` | El hash debe producirse mediante el puerto de seguridad |
| D-12 | Autenticación | JWT HS256, duración de 8 horas, secreto en `JWT_SECRET` | Mantener el contrato funcional del spec |
| D-13 | Ejecución | Docker/Compose para API, frontend, base de datos y utilidades | Docker es la única vía de ejecución del proyecto |
| D-14 | Pruebas | Pruebas del dominio y aplicación con PHPUnit/fakes; integración y contrato contra PostgreSQL y API en contenedores | Aislar reglas y verificar comportamiento desplegado |

Las librerías concretas de JWT y cualquier dependencia adicional deben elegirse e incorporarse solo
si son necesarias para satisfacer estas decisiones; no se cambia el contrato para acomodar una librería.

## 2. Docker y momento de implementación

Docker **no se deja para el final**. Desde T-01 se implementa el entorno mínimo reproducible para
construir y probar el esqueleto de Laravel, verificar `GET /health` y disponer de PostgreSQL vacío.
Las imágenes o servicios se añaden cuando cada componente exista.

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

## 5. Orden de construcción

Documentar y validar el diseño → esqueleto Laravel y PostgreSQL vacío en Docker → migración inicial
y categorías → dominio → casos de uso con fakes → adaptadores de persistencia, seguridad y archivos
→ API y pruebas de contrato/concurrencia → composición y alta inicial del admin → Compose de núcleo
con esqueleto React → frontend funcional, sembrador y sitio público en contenedores. El detalle y las dependencias están en
[`tasks.md`](tasks.md).
