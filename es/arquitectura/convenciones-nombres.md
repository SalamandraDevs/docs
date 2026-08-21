---
title: Convenciones de nombres
owner: OdairTrujillo
lastUpdate: 2026-08-21
category: arquitectura
lang: es
status: publicado
---

## Propósito

Definir el único estándar global de nomenclatura para ingeniería y DevOps de slmdevs.

Este documento es la autoridad para nombres y rutas. Cuando el código u otros documentos difieran, siga primero este documento y luego alinee la implementación.

## Ingeniería

### Servicios

- **Identificadores canónicos de servicio:** nombre de una sola palabra en minúsculas.
- **Identificadores definidos:** `auth`, `profile`, `gateway`.
- **Ejemplos:** `messages`, `students`.
- **Directorio del repositorio:** `services/{service}/`, donde `{service}` coincide con el identificador canónico.
- **Nombre del paquete:** `@slmdevs/{service}` (por ejemplo, `@slmdevs/auth`, `@slmdevs/profile`).

### APIs

- **Formato:** `/{service}/v{N}/...` con segmentos de ruta en minúsculas.
- **Segmento de servicio:** identificador canónico del servicio.
- **Segmento de versión:** `v1`, `v2`, etc.
- **Ejemplos:** `api.slmdevs.com/profile/v1/person`, `/auth/v1/user`.
- **Segmentos:** palabras únicas en minúsculas; sin mayúsculas, mezcla de mayúsculas y minúsculas ni kebab-case.

### Entidades

- **Tipo de registro de entidad:** singular en PascalCase (`User`, `Person`, `Org`, `OutboxEvent`).
- **Clase de política (módulo del servicio):** plural en PascalCase (`Users`, `Persons`, `Orgs`, `OutboxEvents`).
- **Campos:** camelCase en tipos TypeScript y esquemas Zod (`tenantId`, `entityId`, `createdBy`, `updatedAt`).
- **Enumeraciones de estado:** nombre de tipo en PascalCase, valores literales en camelCase y exportación Zod `{entity}StatusSchema`.
- **Nunca:** campos en PascalCase o snake_case en tipos de aplicación o esquemas Zod de entidades.

### Módulos de entidades

Cada agregado se ubica en `{service}/src/entities/{plural}/`:

| Archivo | Nomenclatura |
| --- | --- |
| `{plural}.ts` | archivo de clase de política (`users.ts` → `Users`) |
| `schema.ts` | esquemas Zod de entidad y solicitudes |
| `dataset.ts` | `MongoDatasetDef` e índices |
| `types.ts` | registros, comandos y consultas |
| `index.ts` | reexportaciones del barrel |

**Carpeta `{plural}`:** plural en minúsculas o kebab-case que coincida con la colección Mongo o el plural del dominio (`users`, `persons`, `orgs`, `outbox-events`).

**Agregado Organization:** `Org` es el token registrado de la entidad, no una abreviatura técnica. Nombres canónicos: registro `Org`, clase de política `Orgs`, directorio de módulo `orgs/`, colección Mongo `orgs` y sujeto de evento `org`. Use `Organization` / `organizations` solo en prosa cuando sea más claro.

### Eventos (NATS)

- **Formato:** `{service}.{subject}.{action}` — tokens en minúsculas separados por puntos.
- **Sujeto:** sustantivo del agregado o dominio, singular y en minúsculas (`user`, `person`, `org`, `outbox`).
- **Acción:** verbo en pasado (`created`, `updated`, `deleted`, `published`).
- **Ejemplos:** `auth.user.created`, `profile.person.updated`, `profile.org.created`.
- **Nunca:** barras en nombres de eventos, identificadores variables en el sujeto (colóquelos en el payload o los headers) ni tokens en mayúsculas.

### Convenciones de símbolos (por agregado)

| Tipo de símbolo | Patrón | Ejemplo |
| --- | --- | --- |
| Tipo de registro de entidad | `{Entity}` | `Person` |
| Clase de política | plural `{Entities}` | `Persons` |
| Esquema Zod de entidad | `{entity}Schema` | `personSchema` |
| Esquema Zod de estado | `{entity}StatusSchema` | `personStatusSchema` |
| Esquema Zod de solicitud de creación | `create{Entity}RequestSchema` | `createPersonRequestSchema` |
| DTO de solicitud de creación | `Create{Entity}Request` | `CreatePersonRequest` |
| Comando de creación | `Create{Entity}Command` | `CreatePersonCommand` |
| Esquema Zod de solicitud de actualización | `update{Entity}RequestSchema` | `updatePersonRequestSchema` |
| DTO de solicitud de actualización | `Update{Entity}Request` | `UpdatePersonRequest` |
| Comando de actualización | `Update{Entity}Command` | `UpdatePersonCommand` |
| DTO de respuesta | `{Entity}Response` | `PersonResponse` |
| Definición de dataset | `{entity}DatasetDef` | `personDatasetDef`, `orgDatasetDef` |

Segundo ejemplo de agregado (`Org`): registro `Org`, política `Orgs`, esquemas `orgSchema` / `orgStatusSchema`, solicitudes `CreateOrgRequest`, comandos `CreateOrgCommand` y respuestas `OrgResponse`.

### Alias de importación TypeScript (servicios)

| Alias | Se asigna a |
| --- | --- |
| `#root/*` | `src/*` |
| `#config/*` | `src/config/*` |
| `#entities/*` | `src/entities/*` |

Use la extensión `.js` en especificadores de importación relativos y de paquetes (NodeNext).

### Biblioteca core (`@slmdevs/core`)

Formas establecidas (las palabras completas siguen siendo válidas según [Abreviaturas](#abreviaturas)):

- **`Db`** para tipos de conexión a bases de datos (`IDbConnection`, `MongoDbConnection`).
- **`Dataset` sin abreviar** para la clase abstracta y sus métodos (`Dataset`, `ensureDataset()`).
- **`Def`** para registros de definición (`MongoDatasetDef`, `MongoIndexDef`, `{entity}DatasetDef`).
- **Archivos de implementación Mongo:** `mongo-{name}.ts` en kebab-case (`mongo-crud.ts`, `mongo-db-connection.ts`).
- **Archivos de implementación MySQL:** `{name}.ts` sin prefijo adicional cuando la carpeta ya es `mysql/` (`mysqlcrud.ts`, `mysqldataset.ts`).

### Nombres de datasets e índices Mongo

Definidos en el `dataset.ts` de cada entidad:

| Elemento | Convención | Ejemplos |
| --- | --- | --- |
| `MongoDatasetDef.name` | nombre de colección plural en minúsculas | `users`, `persons`, `orgs`, `events_outbox`, `contractors` |
| `MongoIndexDef.name` | nombre descriptivo en snake_case | `tenant_username`, `tenant_email`, `tenant_person` |
| Claves de índice | campos en camelCase que coincidan con los de la entidad | `{ tenantId: 1, username: 1 }` |

### Abreviaturas

Las abreviaturas técnicas son opcionales: prefiera la palabra completa cuando el nombre siga siendo claro. Solo se permiten formas abreviadas establecidas en la industria y únicamente cuando mantienen legible un símbolo compuesto. Los tokens de entidad registrados, como `Org`, son nombres canónicos y quedan fuera de estas reglas.

**Conteo de palabras**

Cuente solo palabras completas al decidir si un nombre es suficientemente largo para abreviarlo. Prefiera abreviar cuando el compuesto completo exceda **dos palabras contadas**; los términos independientes y sufijos convencionales (`Id`, `Config`, `Crud`) están exentos.

Los acrónimos estándar **no** cuentan como palabras: `HTTP`, `API`, `JSON`, `BSON`, `SQL`, `NATS`, `URI`, `DTO`, `CRUD` y formas similares de la industria. Represéntelos según el casing del identificador (`HttpApiClient`).

Ejemplos:

- `Mongo` + `Database` + `Connection` → tres palabras → `MongoDbConnection`
- `person` + `Dataset` + `Definition` → tres palabras → `personDatasetDef`
- `HTTP` + `API` + `Client` → una palabra contada (`Client`) → conserve `HttpApiClient`; no abrevie más

**Cuándo abreviar**

- Abrevie solo cuando la escritura completa sea innecesariamente larga; la forma completa sigue siendo válida (`MongoDatabaseConnection`, `personDatasetDefinition`).
- Acorte **como máximo una** palabra listada por nombre compuesto.
- No abrevie nombres cortos o ya claros (`Dataset`, `Person`, `User`, `Connection`).
- No invente formas abreviadas propias del proyecto (`Pers`, `Prov`, `Cfg`).

**Abreviaturas consecutivas**

Dos abreviaturas no deben aparecer juntas en el mismo símbolo.

- Permitido: `MongoDbConnection` (solo `Db`), `MongoIndexDef` (solo `Def`; escriba `Index` completo cuando también use `Def`)
- No permitido: `MongoDbIdxDef`, `DbCrud`, `CrudDto`, `IdxDef`

**Abreviaturas permitidas**

| Palabra completa | Abreviatura | Ejemplo |
| --- | --- | --- |
| Database | `Db` | `IDbConnection`, `MongoDbConnection` |
| Definition | `Def` | `MongoDatasetDef`, `personDatasetDef` |
| Index | `Idx` | `searchResultIdx` |
| Identifier | `Id` | `entityId`, `tenantId`, `eventId` (sufijo de campo) |
| Configuration | `Config` | `config/`, `ServiceConfig` |
| Create, Read, Update, Delete | `Crud` | `ICrud`, `MongoCrud` |
| Data Transfer Object | `DTO` | rol de DTO; use nombres semánticos como `CreatePersonRequest` |

**Nunca**

- Dos o más abreviaturas listadas en un símbolo, especialmente si son adyacentes (`MongoDbIdxDef`).
- Truncamientos ad hoc no listados arriba.
- Abreviaturas en identificadores de servicios o segmentos de rutas API.
- Tokens de entidad ad hoc; `Org` es el único token de entidad abreviado registrado.

### Archivos y código fuente

- **Directorios y nombres de archivo:** minúsculas; los segmentos de varias palabras usan kebab-case cuando sea necesario.
- **Clases, interfaces y tipos TypeScript:** PascalCase.
- **Funciones, constantes y variables:** camelCase.
- **Nunca:** directorios en `camelCase` o `PascalCase`, ni espacios en rutas.

## DevOps

### Ramas

Ramas de flujo: `devel`, `main`, `staging`, `production`.

| Rama | Función |
| --- | --- |
| `devel` | desarrollo continuo; adelantada respecto a `main` |
| `main` | estado de desarrollo congelado; correcciones y funcionalidades menores; candidata para staging |
| `staging` | destino de despliegue en infraestructura de staging (puede compartir infraestructura de producción); dominio no productivo |
| `production` | destino de despliegue en infraestructura de producción; dominio principal |

El trabajo de funcionalidades y correcciones usa nombres `{featureBranch}` o `{fixBranch}` elegidos según la convención del equipo.

- **Nombres de ramas de flujo:** palabras únicas en minúsculas; sin mayúsculas, separadores ni mezcla de mayúsculas y minúsculas.
