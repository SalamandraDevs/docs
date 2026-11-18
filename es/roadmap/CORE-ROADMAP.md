---
title: Roadmap de la librería Core
lastUpdate: 2026-11-18
lang: es
category: planificacion
status: active
---

# Roadmap de la librería Core

Estado de las funcionalidades de `core/` (`@salamandradevs/core`). Las rutas en `refs` son
relativas a `core/`; los documentos de plan bajo `docs/plans/` son relativos a la raíz del
espacio de trabajo.

## Funcionalidades de CORE

Base compartida de registros y base de datos: contratos neutrales al backend, adaptadores,
manejo de registros con conciencia de tenant y contexto de ejecución de la petición.

- [implementado] CORE-001 Contratos de base de datos: ciclo de vida de la conexión, CRUD y
  contratos de dataset. (refs: `src/infra/database/contracts.ts`)
- [implementado] CORE-002 Adaptador de MongoDB: conexión, CRUD, aprovisionamiento de
  datasets. (refs: `src/infra/database/mongo/`)
- [implementado] CORE-003 Adaptador de MySQL: conexión, CRUD, aprovisionamiento de
  datasets. (refs: `src/infra/database/mysql/`)
- [implementado] CORE-004 Aprovisionador de datasets: crea datasets e índices a partir de
  definiciones basadas en Zod. (refs: `src/infra/database/dataset-provisioner.ts`)
- [implementado] CORE-005 `RecordsHandler`: alcance de tenant, `recordId`, metadatos de
  auditoría y paginación por cursor determinista ordenada por `createdAt` y luego
  `recordId`, ambos en orden descendente. (refs: `src/storage/records-handler.ts`)
- [implementado] CORE-006 Contexto de ejecución: el contrato `IExecutionCtxProvider` y la
  implementación `ExecutionCtxProvider` sobre un `AsyncLocalStorage` propio del servicio,
  que transporta el actor, el tenant y el id de petición; inyectado en `RecordsHandler`,
  sin instancia a nivel de módulo. (refs: `src/runtime/`)
- [implementado] CORE-008 Proveedor de base de datos y adaptadores ligados al backend:
  `DatabaseProvider` (`IDatabaseProvider`) posee la selección de backend y el ciclo de vida
  de la conexión, exponiendo un `DatabaseHandle` discriminado, y `DatabaseAdapter`
  selecciona el backend una sola vez para `provision` las definiciones de dataset (mediante
  `IDatasetProvisioner`) y `createCrud` los adaptadores `ICrud` por nombre.
  (refs: `src/infra/database/database-provider.ts`,
  `src/infra/database/database-adapter.ts`)
- [propuesto] CORE-007 Conversión de esquema de MySQL: depende de
  [TOOLS-005](./TOOLS-ROADMAP.md#funcionalidades-de-tools).
- [implementado] CORE-009 Proveedores de claves: `ISigningKeyProvider`,
  `IVerificationKeyProvider` e `IKeysProvider`; los proveedores Jose, OpenBao y de JWKS
  remoto; los esquemas del ciclo de vida de firma; y `createKeysProvider`. Trasladados desde
  `services/auth` para que los servicios compartan una sola implementación, cada uno con sus
  propias claves. (refs: `src/keys/`, `docs/plans/261106-gateway-plan.md`)
- [implementado] CORE-010 Verificación de token de acceso: `IAccessTokenVerifier`,
  `AccessTokenVerifier` y `accessTokenClaimsSchema`, para que el emisor y cada verificador
  compartan una sola definición de claims y un solo flujo de verificación.
  (refs: `src/tokens/`, `docs/plans/261106-gateway-plan.md`)
- [implementado] CORE-011 Búsqueda find-all entre tenants sin paginar: `ICrud.findAll` y
  `RecordsHandler.findAll` devuelven todos los registros coincidentes sin paginación,
  evitando deliberadamente el alcance de tenant y el contexto de ejecución para escaneos de
  todo el proceso, como la reconstrucción de revocaciones del gateway; `FindAllOptions`
  lleva un tope `number | null`, un orden opcional y límites `greaterThan`/`lessThan`
  neutrales al backend traducidos por cada adaptador, y `assertFindAllFilter` rechaza un
  escaneo sin filtro.
  (refs: `src/infra/database/contracts.ts`, `src/infra/database/mongo/mongo-crud.ts`,
  `src/infra/database/mysql/mysql-crud.ts`, `src/storage/records-handler.ts`,
  `src/infra/database/utils.ts`)
- [implementado] CORE-012 Verificación de API key: `IApiKeyVerifier`, `ApiKeyVerifier`,
  `apiKeyPlanSchema`, `apiKeyVerifyConfigSchema` y `apiKeyPayloadSchema`, para que el emisor
  del gateway y cada verificador compartan un solo formato de clave, esquema de payload y
  flujo de verificación; el verificador lee el prefijo y el tope de longitud desde la
  configuración y no toma ninguna decisión de revocación.
  (refs: `src/tokens/api-key-verifier.ts`,
  `docs/plans/261111-api-key-verification-plan.md`)
