# Especificación adaptada — Manager Library (React + Laravel)

Esta carpeta adapta el spec base de Manager Library al stack del reto. No reemplaza ni redefine sus
historias, reglas de negocio, contrato HTTP o modelo de datos: esos requisitos se conservan y sus
documentos se reproducen aquí para que esta adaptación pueda guiar la construcción.

## Fuente y alcance de la adaptación

La fuente funcional es [`../spec-python/`](../spec-python/), que describe el mismo producto que
[`../spec-.net/`](../spec-.net/). Se conserva su intención de negocio y se sustituyen únicamente las
decisiones tecnológicas por React, PHP con Laravel y PostgreSQL.

| Documento | Para qué sirve |
|---|---|
| [`constitution.md`](constitution.md) | Principios obligatorios y verificaciones para PHP/Laravel, React y Docker |
| [`spec.md`](spec.md) | Alcance, actores, historias y reglas de negocio |
| [`plan.md`](plan.md) | Decisiones tecnológicas y secuencia de construcción |
| [`architecture.md`](architecture.md) | Arquitectura hexagonal del backend y estructura del frontend |
| [`arquitectura-panoramica.md`](arquitectura-panoramica.md) | Vista de componentes y responsabilidades de los repositorios |
| [`data-model.md`](data-model.md) | Esquema y restricciones de PostgreSQL |
| [`api-contract.md`](api-contract.md) | Contrato HTTP que debe conservar Laravel |
| [`tasks.md`](tasks.md) | Backlog ordenado por dependencias y evidencia |
| [`adr/`](adr/) | Decisiones de arquitectura y sus consecuencias |

## Regla SDD y cambios

Antes de implementar una tarea se revisan y actualizan su requisito, decisión técnica, evidencia y
dependencias en esta carpeta. Si la implementación obliga a cambiar un requisito, se registra primero
la decisión y se mantiene explícita la diferencia frente al spec de origen.

Cada commit que incluya estructura o código debe incluir primero la documentación correspondiente,
conforme a la instrucción del proyecto. Esta regla no cambia el orden de construcción: Docker se
documenta desde esta fase y se usa desde el esqueleto; T-20 integra el núcleo API/frontend/base de
datos y las tareas del sembrador y el sitio público incorporan sus contenedores al entorno.

## Cómo usar el spec

Los archivos `spec.md`, `api-contract.md` y `data-model.md` mantienen los requisitos funcionales,
HTTP y de datos del spec base. Las adaptaciones de este stack se definen en `constitution.md`,
`plan.md`, `architecture.md`, `arquitectura-panoramica.md` y `tasks.md`. Ante cualquier cambio, se
actualizan juntos los documentos afectados antes del código.
