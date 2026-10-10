# Tareas — Manager Library (React + Laravel)

Cada tarea termina cuando su evidencia existe y se puede ejecutar mediante Docker, según
[Artículo XIII](constitution.md). Estado inicial: **pendiente**. Se conserva el orden funcional del
spec base y se adaptan las evidencias técnicas al stack PHP/Laravel y React.

## Estado y evidencia de ejecución

- **T-01 — completada:** API Laravel y PostgreSQL 16 levantan con Docker Compose; `GET /health`
  respondió HTTP 200 con `{"status":"ok"}`; `php artisan test --display-warnings` pasó (2 pruebas,
  3 aserciones); la consulta a `pg_tables` confirmó cero tablas en `public` antes de T-02.
- Docker engine y Compose se usaron para generar el esqueleto Laravel; no se ejecutaron migraciones.
- Las tareas T-02 en adelante continúan pendientes.

| # | Tarea | Depende de | Evidencia |
|---|---|---|---|
| T-01 | Esqueleto Laravel, Docker mínimo, PostgreSQL vacío y `GET /health` | — | `infra/compose.yaml` levanta API y PostgreSQL; `GET /health` devuelve exactamente `{"status":"ok"}` y PostgreSQL no tiene tablas de negocio; prueba HTTP ejecutable dentro del contenedor |
| T-02 | Migración inicial: seis tablas, FK, índices únicos y cinco categorías | T-01 | Migraciones/seeders crean lo documentado; consultas de `data-model.md` §10 coinciden; ninguna columna con `DEFAULT` |
| T-03 | Dominio: `Isbn`, `Quantity`, `DateRange` | T-01 | PHPUnit prueba ISBN de 13 dígitos, cantidad > 0 y fin ≥ inicio, sin framework ni base |
| T-04 | Dominio: `Book` (retirar, reponer y baja lógica) | T-03 | PHPUnit demuestra que disponibles nunca quedan < 0 ni > total |
| T-05 | Dominio: `Loan` y `LoanItem` (alta, devolución, estado calculado, 14 días y máximo 3) | T-04 | PHPUnit con reloj falso prueba `open`, `overdue` y `closed` |
| T-06 | Dominio: `Member` y `User` | T-03 | Pruebas de normalización (`"Ana "` → `ana`) y rol; endpoint rechazará `admin` como rol solicitado |
| T-07 | Puertos y casos de uso de autenticación con fakes | T-06 | HU-01 y HU-02 pasan con PHPUnit y fakes |
| T-08 | Puertos y casos de uso de libros con fakes | T-04 | HU-03 y HU-04 pasan con PHPUnit y fakes |
| T-09 | Caso de uso de préstamo con fakes | T-05, T-08 | HU-06 completa: límite, vencidos, duplicados y falta de ejemplares |
| T-10 | Casos de uso de devolución y vencidos | T-09 | HU-07 y HU-09; devolver dos veces produce `ITEM_ALREADY_RETURNED` |
| T-11 | Caso de uso y puerto del reporte de más prestados | T-09 | HU-10; el valor congelado se conserva tras renombrar (ADR-004) |
| T-12 | Adaptador de persistencia para libros, socios y usuarios | T-02, T-08 | Pruebas de integración contra PostgreSQL real en contenedor |
| T-13 | Adaptador de persistencia de préstamos y control de concurrencia | T-12, T-09 | Dos solicitudes concurrentes por el último ejemplar: una 201 y otra 409 (RNF-2) |
| T-14 | Adaptador de seguridad: hash y JWT | T-07 | La clave nunca aparece en logs ni respuestas (RNF-3); token válido por 8 horas |
| T-15 | Adaptador de almacenamiento de portadas | T-08 | PNG de 2 MB aceptado; 3 MB → 422; sin portada → `cover_key` nulo |
| T-16 | Adaptador HTTP Laravel y contrato completo | T-10, T-11, T-14, T-15 | Pruebas de contrato recorren rutas, roles, cuerpo único de error y códigos de `api-contract.md` |
| T-17 | Composición Laravel y admin inicial desde entorno | T-16 | Base vacía, migraciones ejecutadas y login inicial exitoso desde API |
| T-18 | Añadir a PostgreSQL `ck_book_total_positive` y `ck_user_role` | T-12 | Escrituras inválidas fallan en PostgreSQL; `data-model.md` §4 se actualiza |
| T-19 | Pruebas de arquitectura hexagonal | T-17 | Pruebas automatizadas detectan dependencias prohibidas y composición fuera del Service Provider |
| T-20 | Integrar Docker Compose para API, esqueleto frontend y PostgreSQL vacío | T-17 | Un comando levanta los contenedores del núcleo; base sin tablas antes de migrar; health checks disponibles |
| T-21 | Sembrador de demostración en contenedor que usa la API | T-17 | ≥ 40 libros, 10 socios, 15 préstamos y 3 vencidos; API devuelve los vencidos y reporte con filas |
| T-22 | Frontend React en contenedor: login, catálogo, socios, prestar, devolver, vencidos y reporte | T-16, T-20 | Recorrido manual de punta a punta usando frontend y API en contenedores |
| T-23 | Sitio público estático en contenedor | — | Se sirve en contenedor sin depender de la API; correcto en móvil y escritorio |

## Definición de terminado

Llegar a T-17 con evidencia significa que el núcleo de negocio y la API están construidos. T-18 a
T-23 completan endurecimiento, integración y superficies de presentación. La evaluación prioriza
cumplimiento del spec y evidencia verificable, no el número de tareas marcadas como terminadas.
