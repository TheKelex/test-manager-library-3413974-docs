# Constitución — Manager Library (React + Laravel)

Principios innegociables del proyecto. Cada artículo incluye una forma de comprobarlo. La solución
debe respetarlos además de las historias y reglas de [`spec.md`](spec.md).

---

## Artículo I — El dominio no depende de frameworks

`api/app/Domain/` no importa Laravel, Eloquent, HTTP, base de datos ni paquetes de terceros. Contiene
reglas y tipos PHP puros.

**Verificación:** pruebas del dominio ejecutadas en contenedor sin levantar Laravel ni conectar a la
base; una prueba de arquitectura detecta dependencias prohibidas.

## Artículo II — El núcleo declara los puertos

La aplicación define los puertos de repositorio, reloj, hash, token y almacenamiento que necesita.
Sus firmas no exponen tipos Laravel, Eloquent, SQL ni HTTP. Los tests de aplicación sustituyen los
puertos por fakes.

**Verificación:** pruebas de aplicación ejecutables con fakes; prueba de arquitectura impide que
`Application` dependa de `Adapters`.

## Artículo III — Un único punto de composición

Un Service Provider dedicado en `api/app/Bootstrap/Composition/` enlaza los puertos con sus
adaptadores. Controladores y casos de uso no instancian infraestructura.

**Verificación:** prueba de arquitectura comprueba el registro de bindings y que las dependencias
concretas no se creen fuera del punto de composición.

## Artículo IV — SOLID es verificable

| Principio | Aplicación concreta | Verificación |
|---|---|---|
| SRP | Un caso de uso representa una operación del negocio | Cada clase tiene una razón principal para cambiar |
| OCP | Cambiar el almacenamiento de portadas no modifica dominio ni aplicación | Se sustituye el adaptador mediante el binding |
| LSP | Fakes y adaptadores cumplen los mismos puertos | Pruebas compartidas para implementaciones |
| ISP | Puertos pequeños, como `Clock` y `CoverStorage` | Cada adaptador implementa solo operaciones necesarias |
| DIP | Los puertos viven en la aplicación que los consume | La dirección de dependencias se prueba |

## Artículo V — Las migraciones Laravel son propietarias del esquema

Todo DDL, índices, restricciones y semilla fija de categorías vive en las migraciones/seeders de
`api`. `infra` puede levantar PostgreSQL vacío, pero no contiene `CREATE TABLE` ni DDL alternativo.
Ver [ADR-001](adr/adr-001-propiedad-del-esquema.md).

**Verificación:** PostgreSQL inicia sin tablas y las migraciones producen el esquema descrito en
[`data-model.md`](data-model.md).

## Artículo VI — La base impone las invariantes que puede expresar

Las restricciones `CHECK`, `UNIQUE` y claves foráneas expresables en PostgreSQL se declaran en
migraciones. Las reglas que solo proteja el dominio se marcan así en el modelo de datos, sin
prometer una barrera inexistente.

**Verificación:** consultas de [`data-model.md`](data-model.md) §10 coinciden con el esquema real.

## Artículo VII — Lo derivado se calcula

El estado del préstamo (`open`, `overdue`, `closed`) y los días de retraso no se almacenan.

**Verificación:** `data-model.md` no define columnas para esos valores.

## Artículo VIII — Los datos congelados no siguen al catálogo

La línea conserva título, ISBN y categoría del instante del préstamo. El reporte no consulta esos
valores del catálogo actual.

**Verificación:** una prueba cambia el nombre del libro después del préstamo y comprueba que el
reporte mantiene el valor congelado (ADR-004).

## Artículo IX — Los secretos no se versionan ni se registran

Contraseñas, hashes precalculados y claves JWT no se guardan en el repositorio ni en logs. El `.env`
real está ignorado; solo se versiona `.env.example` sin valores secretos.

**Verificación:** revisión de archivos y logs; ninguna respuesta HTTP incluye `password` ni
`password_hash`.

## Artículo X — Gana el motor

Si PostgreSQL y `data-model.md` discrepan, se corrige la documentación antes de aceptar el cambio.

## Artículo XI — Idioma y contrato

Código, nombres de tablas/campos y rutas están en inglés. Documentación y mensajes al usuario están
en español. Las rutas, campos, estados y errores HTTP respetan `api-contract.md`.

## Artículo XII — Docker es la única vía de ejecución

El proyecto se construye, prueba y ejecuta mediante Docker/Compose. No se consideran instrucciones
de ejecución local como una alternativa válida.

**Verificación:** README de `infra` documenta el comando de arranque; las pruebas y el recorrido
funcional se ejecutan contra los servicios en contenedores.

## Artículo XIII — Nada se da por hecho sin evidencia

Una tarea de [`tasks.md`](tasks.md) se considera terminada cuando su evidencia está disponible y se
puede ejecutar dentro del flujo Docker.
