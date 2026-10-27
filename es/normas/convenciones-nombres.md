---
title: Convenciones de nombres
lastUpdate: 2026-10-24
lang: es
category: arquitectura
status: published
owner: Odair Trujillo
---

## Propósito

Definir el único estándar global de nombres para la ingeniería y el DevOps de SalamandraDevs.

Este documento es la autoridad para nombres y rutas. Cuando el código u otros documentos
difieran, siga primero este documento y luego alinee la implementación.

## IDs de documento

Los documentos de la casa y de arquitectura dan a cada sección un ID estable, `XX-NN`: un
prefijo de documento de dos letras y un número de dos dígitos (`GP-05`). Una sección con ID
usa el encabezado `### XX-NN Título`. Nunca reutilice ni renumere un ID; añada secciones
nuevas con el siguiente número libre. Prefijos: `SP` principios SOLID, `DP` patrones de
diseño, `GP` buenas prácticas de desarrollo, `CS` estilo de código, `DS` estilo de
documentos.

## Ingeniería

### Servicios

- **Ids de servicio:** nombre de una sola palabra en minúsculas.
- **Ids definidos:** `auth`.
- **Directorio del repositorio:** `services/{service}/` donde `{service}` equivale al id del
  servicio.
- **Nombre del paquete:** `@salamandradevs/{service}` (por ejemplo `@salamandradevs/auth`).

### Estructura de servicios

Cada árbol de código fuente de servicio bajo `services/{service}/src/` se divide en estas
secciones principales:

- `application/`; servicios de caso de uso y los contratos que publican o consumen.
  Requerido.
  - `{plural}-service/` por caso de uso (`users-service/`, `sessions-service/`), que
    contiene `{plural}-service.ts`, `schema.ts`, `types.ts` e `index.ts`.
  - `contracts.ts` en la raíz de la sección para los contratos de aplicación
    (`ISessionsService`, `ISigningKeyProvider`, `IOAuthProvider`).
  - `config/` para el esquema y los tipos de configuración de la aplicación.
- `config/`; carga y valida el archivo de configuración del servicio (`config.ts`,
  `schema.ts`, `types.ts`). Los archivos de despliegue mismos viven en
  `services/{service}/config/` (`devel.json`, `prod.json`). Requerido.
- `infra/`; adaptadores para sistemas externos, una carpeta por preocupación (`keys/`,
  `oauth/`), con una subcarpeta por proveedor (`keys/jose/`, `keys/openbao/`,
  `oauth/google/`). Solo cuando el servicio integra sistemas externos.
- `runtime/`; la raíz de composición. `runtime.ts` expone `createRuntime`, y cada paso de
  arranque vive en un archivo nombrado por su tema (`database.ts`, `keys.ts`, `oauth.ts`,
  `services.ts`, `admin.ts`). Requerido.
- `storage/`; módulos de registro y la colección de datasets (`datasets.ts`). Solo cuando el
  servicio persiste registros. Véase [Módulos de registro](#módulos-de-registro).

El punto de entrada del proceso es `src/index.ts`.

### Alias de importación de TypeScript (servicios)

| Alias | Mapea a |
| --- | --- |
| `#root/*` | `src/*` |
| `#api/*` | `src/api/*` |
| `#application/*` | `src/application/*` |
| `#config/*` | `src/config/*` |
| `#infra/*` | `src/infra/*` |
| `#storage/*` | `src/storage/*` |

- Declare cada alias dos veces: en `imports` de `package.json` (mapeando a `./dist/...`) y en
  `paths` de `tsconfig.json` (mapeando a `src/...`).
- Use la extensión `.js` en los especificadores de import relativos y de paquete (NodeNext).

### APIs

- **Formato:** `/{service}/v{N}/{resource}/...` con segmentos de ruta en minúsculas.
- **Segmento de servicio:** id del servicio, configurado como `http.apiPrefix` (`/auth`).
- **Segmento de versión:** `v1`, `v2`, etc.
- **Ejemplos:** `/auth/v1/users`, `/auth/v1/users/:recordId`, `/auth/v1/session`,
  `/auth/v1/session/refresh`, `/auth/v1/oauth/:provider/callback`.
- **Segmentos:** palabras sueltas en minúsculas; sin mayúsculas, mayúsculas y minúsculas
  mezcladas ni kebab-case.
- **Parámetros de ruta:** camelCase coincidiendo con el campo del registro (`:recordId`,
  `:provider`).
- **Rutas sin versión:** los endpoints a nivel de servicio quedan fuera del segmento de
  versión (`/health`, `/openapi.json`, `/docs`, `/.well-known/jwks.json`).

### Registros

- **Tipo de registro:** PascalCase singular con un sufijo `Record` (`UserRecord`,
  `SessionRecord`).
- **Clase de manejador de registros:** PascalCase singular con un sufijo `RecordsHandler`,
  que extiende el `RecordsHandler` de core (`UserRecordsHandler`, `SessionRecordsHandler`).
- **Nombre del manejador:** el literal `handlerName` equivale al nombre del dataset
  (`'users'`, `'sessions'`).
- **Campos:** camelCase en los tipos TypeScript y los esquemas Zod (`tenantId`, `recordId`,
  `handlerName`, `createdBy`, `updatedAt`).
- **Enums de estado:** literales de valor de una sola palabra en minúsculas (`active`,
  `suspended`, `terminated`), exportados como `{resource}RecordStatusSchema` con el tipo
  inferido `{Resource}RecordStatus`.
- **Nunca:** nombres de campo en PascalCase; nombres de campo en snake_case en los tipos de
  aplicación o los esquemas Zod de registro.

### Módulos de registro

Cada módulo de registro bajo `{service}/src/storage/{plural}/`:

| Archivo | Nombre |
| --- | --- |
| `{singular}-records-handler.ts` | clase de manejador de registros (`user-records-handler.ts` → `UserRecordsHandler`) |
| `schema.ts` | esquemas Zod de registro y de estado |
| `datasetdef.ts` | entradas de `DatasetDef` + `IndexDef`, siempre incluyendo `recordIndexDef` |
| `types.ts` | tipos inferidos de los esquemas de registro |
| `index.ts` | re-exports de barril |

**Carpeta `{plural}`:** plural en minúsculas coincidiendo con el nombre del dataset
(`users`, `sessions`).

### Convenciones de símbolos (por registro)

| Tipo de símbolo | Patrón | Ejemplo |
| --- | --- | --- |
| Tipo de registro | `{Resource}Record` | `UserRecord` |
| Esquema Zod de registro | `{resource}RecordSchema` | `userRecordSchema` |
| Esquema Zod de estado de registro | `{resource}RecordStatusSchema` | `userRecordStatusSchema` |
| Tipo de estado de registro | `{Resource}RecordStatus` | `UserRecordStatus` |
| Clase de manejador de registros | `{Resource}RecordsHandler` | `UserRecordsHandler` |
| Definición de dataset | `{resource}DatasetDef` | `userDatasetDef` |
| Clase de servicio de aplicación | `{Resources}Service` | `UsersService` |
| Contrato de servicio de aplicación | `I{Resources}Service` | `ISessionsService` |
| Esquema Zod de entrada de aplicación | `{action}{Resource}InputSchema` | `createUserInputSchema` |
| Tipo de entrada de aplicación | `{Action}{Resource}Input` | `CreateUserInput` |
| Esquema Zod de vista de aplicación | `{resource}ViewSchema` | `userViewSchema` |
| Tipo de vista de aplicación | `{Resource}View` | `UserView` |
| Esquema Zod de cuerpo de petición | `{action}{Resource}BodySchema` | `createUserBodySchema` |
| Tipo de cuerpo de petición | `{Action}{Resource}Body` | `CreateUserBody` |
| Esquema Zod de query de listado | `{resource}ListQuerySchema` | `userListQuerySchema` |
| Tipo de query de listado | `{Resource}ListQuery` | `UserListQuery` |
| Esquema Zod de respuesta | `{resource}ResponseSchema` | `userResponseSchema` |
| Tipo de respuesta | `{Resource}Response` | `UserResponse` |
| Esquema Zod de respuesta de listado | `{resource}ListResponseSchema` | `userListResponseSchema` |
| Mapa de respuestas de problema | `{resource}ProblemResponses` | `userProblemResponses` |
| Definición de ruta | `{Action}{Resource}Request` | `UpdateUserRequest` |

- Las operaciones de búsqueda sobre una colección usan el recurso en plural
  (`findUsersInputSchema`, `FindUsersInput`).
- Una definición de ruta es el genérico de ruta de Fastify (`{ Params; Body }`) compartido
  por la ruta, sus hooks y su controlador; no es un tipo de payload.
- Los esquemas de parámetros de ruta compartidos entre grupos de rutas usan el nombre del
  campo del registro (`recordParamsSchema`, `RecordParams`).

### Contratos

- Las interfaces llevan un prefijo `I` seguido de un nombre de capacidad o rol (`ICrud`,
  `IDataset`, `IDatabaseProvider`, `ISigningKeyProvider`, `IOAuthProvider`).
- Las implementaciones se nombran por su proveedor y rol (`MongoCrud`, `MySqlDataset`,
  `JoseSigningKeyProvider`, `GoogleOAuthProvider`).

### Librería core (`@salamandradevs/core`)

Formas establecidas (las palabras completas siguen siendo válidas bajo
[Abreviaturas](#abreviaturas)):

- **`Db`** para los tipos de conexión a base de datos (`IDbConnection`, `MongoDbConnection`,
  `MySqlDbConnection`).
- **`Dataset` no abreviado** para los contratos y clases de dataset (`IDataset`,
  `DatasetProvisioner`, `MongoDataset`).
- **`Def`** para los registros de definición (`DatasetDef`, `IndexDef`, `recordIndexDef`,
  `{resource}DatasetDef`).
- **Archivos de implementación:** `{backend}-{name}.ts` en kebab-case dentro de la carpeta
  del backend (`mongo/mongo-crud.ts`, `mongo/mongo-db-connection.ts`, `mysql/mysql-crud.ts`,
  `mysql/mysql-dataset.ts`).
- **Capa de registros:** `storage/` contiene `RecordsHandler`, `BaseRecord`,
  `baseRecordSchema` y `recordIndexDef`.

### Nombres de dataset e índice

Definidos en el `datasetdef.ts` de cada módulo de registro; las mismas definiciones
aprovisionan colecciones de MongoDB y tablas de MySQL.

| Ítem | Convención | Ejemplos |
| --- | --- | --- |
| `DatasetDef.name` | nombre de dataset en plural y minúsculas | `users`, `sessions` |
| `IndexDef.name` | nombre descriptivo en snake_case | `tenant_record`, `tenant_username`, `tenant_email`, `tenant_created_record` |
| `IndexDef.fields[].name` | nombres de campo en camelCase coincidiendo con los campos del registro | `{ name: 'tenantId', direction: 'asc' }` |

### Abreviaturas

Las abreviaturas técnicas son opcionales: prefiera la palabra completa cuando el nombre se
mantenga claro. Solo se permiten las abreviaciones establecidas de la industria, y solo
cuando mantienen legible un símbolo compuesto.

**Conteo de palabras**

Cuente solo las palabras completas al decidir si un nombre es lo bastante largo para
abreviar. Prefiera acortar cuando el compuesto completo exceda **dos palabras contadas**;
los términos y sufijos convencionales independientes (`Id`, `Config`, `Crud`) están exentos.

Los acrónimos estándar **no** cuentan como palabras: `HTTP`, `API`, `JSON`, `SQL`, `URI`,
`JWT`, `JWK`, `OAuth`, `CRUD` y formas similares de la industria. Renderícelos según el
casing del identificador (`IOAuthProvider`, `MySqlCrud`).

Ejemplos:

- `Mongo` + `Database` + `Connection` → tres palabras → `MongoDbConnection`
- `user` + `Dataset` + `Definition` → tres palabras → `userDatasetDef`
- `OAuth` + `Provider` → una palabra contada (`Provider`) → mantenga `IOAuthProvider`; no
  abrevie más

**Cuándo abreviar**

- Abrevie solo cuando la ortografía completa sea innecesariamente larga; la forma deletreada
  sigue siendo válida (`MongoDatabaseConnection`, `userDatasetDefinition`).
- Acorte **como máximo una** palabra listada por nombre compuesto.
- No abrevie nombres cortos o ya claros (`Dataset`, `User`, `Session`, `Connection`).
- No invente acortamientos específicos del proyecto (`Usr`, `Prov`, `Cfg`).

**Abreviaturas consecutivas**

Dos abreviaturas no deben aparecer una tras otra en el mismo símbolo.

- Permitido: `MongoDbConnection` (`Db` solamente), `IndexDef` (`Def` solamente)
- No permitido: `DbCrud`, `CrudDef`, `DbDef`

**Abreviaturas permitidas**

| Palabra completa | Abreviatura | Ejemplo |
| --- | --- | --- |
| Database | `Db` | `IDbConnection`, `MongoDbConnection` |
| Definition | `Def` | `DatasetDef`, `userDatasetDef` |
| Identifier | `Id` | `recordId`, `tenantId`, `actorId` (sufijo de campo) |
| Configuration | `Config` | `config/`, `ServiceConfig` |
| Context | `Ctx` | `getExecutionCtx`, `runWithExecutionCtx` |
| Create, Read, Update, Delete | `Crud` | `ICrud`, `MongoCrud` |

**Nunca**

- Dos o más abreviaturas listadas en un símbolo, especialmente adyacentes (`DbCrud`).
- Truncamientos ad hoc no listados arriba.
- Abreviaturas en los ids de servicio o en los segmentos de ruta de API.

## DevOps

### Ramas

Ramas de flujo: `devel`, `main`, `staging`, `production`.

| Rama | Rol |
| --- | --- |
| `devel` | desarrollo continuo; por delante de `main` |
| `main` | estado de desarrollo congelado; correcciones y funcionalidades menores; candidata a staging |
| `staging` | objetivo de despliegue en infraestructura de staging (puede compartir infra de prod); dominio no productivo |
| `production` | objetivo de despliegue en infraestructura de producción; dominio principal |

El trabajo de funcionalidad y de corrección usa nombres `{featureBranch}` o `{fixBranch}`
elegidos según la convención del equipo.

- **Nombres de ramas de flujo:** solo palabras sueltas en minúsculas; sin mayúsculas,
  separadores ni casing mezclado.
</content>
