# Tareas — Manager Library (React + Laravel)

Cada tarea termina cuando su evidencia existe y se puede ejecutar mediante Docker, según
[Artículo XIII](constitution.md). Estado inicial: **pendiente**. Se conserva el orden funcional del
spec base y se adaptan las evidencias técnicas al stack PHP/Laravel y React.

## Estado y evidencia de ejecución

- **T-01 — completada:** API Laravel y PostgreSQL 16 levantan con Docker Compose; `GET /health`
  respondió HTTP 200 con `{"status":"ok"}`; `php artisan test --display-warnings` pasó (2 pruebas,
  3 aserciones); la consulta a `pg_tables` confirmó cero tablas en `public` antes de T-02.
- Docker engine y Compose se usaron para generar el esqueleto Laravel; no se ejecutaron migraciones.
- **T-02 — completada:** `php artisan migrate --seed --force` crea las seis tablas en `library` y
  siembra las cinco categorías; rollback y reaplicación pasan; `migrate:status` detecta la migración;
  PostgreSQL informa 30 columnas, cero valores `DEFAULT`, restricciones e índices coincidentes con
  el modelo. Repetir `CategorySeeder` conserva cinco filas.
- **T-03 a T-10 — completadas:** dominio PHP puro para ISBN, cantidades, fechas, libros, préstamos,
  socios y usuarios; puertos de salida y casos de uso de autenticación, catálogo, portadas, préstamos,
  devoluciones y vencidos. En el contenedor API, `php artisan test --display-warnings` pasó con
  **30 pruebas y 104 aserciones**, y `./vendor/bin/pint --test app/Domain app/Application
  tests/Fakes tests/Unit tests/Feature/ApplicationUseCasesTest.php` aprobó el formato. Los
  adaptadores de infraestructura y HTTP permanecen para sus tareas posteriores.
- **T-11 a T-20 — completadas:** reporte PostgreSQL y adaptadores persistentes, seguridad JWT/Argon2id,
  almacenamiento privado de portadas, API Laravel, admin inicial, restricciones CHECK, límites
  arquitectónicos y Compose con React. En Docker, `php artisan test --display-warnings` pasó con
  **52 pruebas y 488 aserciones**, `./vendor/bin/pint --test` aprobó **131 archivos** y
  `npm run build` generó el frontend. La prueba HTTP concurrente confirmó una respuesta 201 y otra
  409 al competir por la última copia; API, app y DB quedaron saludables. Una base temporal nueva
  tuvo cero tablas funcionales antes de migrar y se eliminó al terminar la verificación. Sin cambio
  al contrato API ni a la arquitectura hexagonal.
- Las tareas T-21 en adelante continúan pendientes.

| # | Tarea | Depende de | Evidencia |
|---|---|---|---|
| T-01 | Esqueleto Laravel, Docker mínimo, PostgreSQL vacío y `GET /health` | — | `infra/compose.yaml` levanta API y PostgreSQL; `GET /health` devuelve exactamente `{"status":"ok"}` y PostgreSQL no tiene tablas de negocio; prueba HTTP ejecutable dentro del contenedor |
| T-02 | Migración inicial: seis tablas, FK, índices únicos y cinco categorías | T-01 | Migraciones/seeders crean lo documentado; consultas de `data-model.md` §10 coinciden; ninguna columna con `DEFAULT` |
| T-03 | Dominio: `Isbn`, `Quantity`, `DateRange` | T-01 | **Completada:** pruebas unitarias de ISBN de 13 dígitos, cantidad > 0 y rango de fechas válido; dominio PHP puro |
| T-04 | Dominio: `Book` (retirar, reponer y baja lógica) | T-03 | **Completada:** pruebas unitarias de disponibilidad, ajuste de total y baja con préstamos |
| T-05 | Dominio: `Loan` y `LoanItem` (alta, devolución, estado calculado, 14 días y máximo 3) | T-04 | **Completada:** reloj controlado prueba estados `open`, `overdue`, `closed`, plazo, máximo y doble devolución |
| T-06 | Dominio: `Member` y `User` | T-03 | **Completada:** pruebas de nombre/documento, normalización de username y roles permitidos |
| T-07 | Puertos y casos de uso de autenticación con fakes | T-06 | **Completada:** login con error indistinguible, token/rol/vigencia y alta de bibliotecario con hash y rol permitido |
| T-08 | Puertos y casos de uso de libros con fakes | T-04 | **Completada:** alta/búsqueda/consulta/edición/baja; ISBN único activo y validación de portada JPG/PNG hasta 2 MB |
| T-09 | Caso de uso de préstamo con fakes | T-05, T-08 | **Completada:** límite del socio, vencidos, duplicados, disponibilidad y una sola escritura atómica |
| T-10 | Casos de uso de devolución y vencidos | T-09 | **Completada:** reposición única, rechazo de segunda devolución y vencidos estrictos ordenados por atraso |
| T-11 | Caso de uso y puerto del reporte de más prestados | T-09 | **Completada:** consulta PostgreSQL con rango inclusivo, snapshots congelados, orden contractual y rechazo de rango invertido |
| T-12 | Adaptador de persistencia para libros, socios y usuarios | T-02, T-08 | **Completada:** repositorios/mapeos verificados contra PostgreSQL real en pruebas de integración |
| T-13 | Adaptador de persistencia de préstamos y control de concurrencia | T-12, T-09 | **Completada:** solicitudes HTTP concurrentes por la última copia producen una 201 y una 409; escritura/reposición atómicas |
| T-14 | Adaptador de seguridad: hash y JWT | T-07 | **Completada:** Argon2id; JWT `sub`/`role` con vigencia de ocho horas; pruebas de firma y claims inválidos |
| T-15 | Adaptador de almacenamiento de portadas | T-08 | **Completada:** imagen válida de 2 MiB aceptada; exceso y contenido falso rechazados; almacenamiento privado y clave opaca |
| T-16 | Adaptador HTTP Laravel y contrato completo | T-10, T-11, T-14, T-15 | **Completada:** pruebas HTTP de rutas, roles, validación, serialización, errores y códigos del `api-contract.md` |
| T-17 | Composición Laravel y admin inicial desde entorno | T-16 | **Completada:** provider como composition root; comando de admin desde entorno probado como idempotente; login y contraseña no expuesta |
| T-18 | Añadir a PostgreSQL `ck_book_total_positive` y `ck_user_role` | T-12 | **Completada:** PostgreSQL rechaza escrituras inválidas; migración reversible y `data-model.md` §4 reflejan ambas reglas |
| T-19 | Pruebas de arquitectura hexagonal | T-17 | **Completada:** PHPUnit detecta dependencias prohibidas y bindings fuera del Service Provider |
| T-20 | Integrar Docker Compose para API, esqueleto frontend y PostgreSQL vacío | T-17 | **Completada:** API, React y DB saludables; build React aprobado; nueva base verificada con cero tablas antes de migrar |
| T-21 | Sembrador de demostración en contenedor que usa la API | T-17 | ≥ 40 libros, 10 socios, 15 préstamos y 3 vencidos; API devuelve los vencidos y reporte con filas |
| T-22 | Frontend React en contenedor: login, catálogo, socios, prestar, devolver, vencidos y reporte | T-16, T-20 | Recorrido manual de punta a punta usando frontend y API en contenedores |
| T-23 | Sitio público estático en contenedor | — | Se sirve en contenedor sin depender de la API; correcto en móvil y escritorio |

## Definición de terminado

Llegar a T-17 con evidencia significa que el núcleo de negocio y la API están construidos. T-18 a
T-23 completan endurecimiento, integración y superficies de presentación. La evaluación prioriza
cumplimiento del spec y evidencia verificable, no el número de tareas marcadas como terminadas.
