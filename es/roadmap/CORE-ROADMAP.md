---
title: Roadmap de la librería Core
lastUpdate: 2026-10-25
lang: es
category: planificacion
status: active
---

# Roadmap de la librería Core

Estado de las funcionalidades de `core/` (`@salamandradevs/core`). Las rutas en `refs` son
relativas a `core/`.

## Funcionalidades de CORE

Base compartida de registros y base de datos: contratos neutrales al backend, adaptadores,
manejo de registros con conciencia de tenant y contexto de ejecución de la petición.

- [implementado] CORE-001 Contratos de base de datos: ciclo de vida de la conexión, CRUD y
  contratos de dataset. (refs: `src/database/contracts.ts`)
- [implementado] CORE-002 Adaptador de MongoDB: conexión, CRUD, aprovisionamiento de
  datasets. (refs: `src/database/mongo/`)
- [implementado] CORE-003 Adaptador de MySQL: conexión, CRUD, aprovisionamiento de
  datasets. (refs: `src/database/mysql/`)
- [implementado] CORE-004 Aprovisionador de datasets: crea datasets e índices a partir de
  definiciones basadas en Zod. (refs: `src/database/dataset-provisioner.ts`)
- [implementado] CORE-005 `RecordsHandler`: alcance de tenant, `recordId`, metadatos de
  auditoría y paginación por cursor determinista ordenada por `createdAt` y luego
  `recordId`, ambos en orden descendente. (refs: `src/records/records-handler.ts`)
- [implementado] CORE-006 Contexto de ejecución: AsyncLocalStorage que transporta el actor,
  el tenant y el id de petición. (refs: `src/utils/context.ts`)
- [implementado] CORE-008 Proveedor de base de datos y adaptadores ligados al backend:
  `DatabaseProvider` (`IDatabaseProvider`) posee la selección de backend y el ciclo de vida
  de la conexión, exponiendo un `DatabaseHandle` discriminado, y `DatabaseAdapter`
  selecciona el backend una sola vez para `provision` las definiciones de dataset (mediante
  `IDatasetProvisioner`) y `createCrud` los adaptadores `ICrud` por nombre.
  (refs: `src/infra/database/database-provider.ts`,
  `src/infra/database/database-adapter.ts`)
- [propuesto] CORE-007 Conversión de esquema de MySQL: depende de
  [TOOLS-005](./TOOLS-ROADMAP.md#funcionalidades-de-tools).
</content>
