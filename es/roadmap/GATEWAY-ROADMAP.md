---
title: Roadmap del servicio Gateway
lastUpdate: 2026-11-11
lang: es
category: planificacion
status: active
---

# Roadmap del servicio Gateway

Estado de las funcionalidades de `services/gateway/` (`@salamandradevs/gateway`). Las rutas
en `refs` son relativas a `services/gateway/`; los documentos de plan bajo `docs/plans/`
son relativos a la raíz del espacio de trabajo.

## Funcionalidades de GATEWAY

Servicio de borde en Fastify que termina la API key de tenant, mide el pool de peticiones
de cada tenant por plan en Redis, posee el registro de API keys y su administración, y hace
de proxy hacia los servicios upstream por prefijo de ruta, afirmando el tenant con
`x-tenant-id`. Capas: `api/` (plugins de transporte y de política HTTP), `application/`
(política), `storage/` (el registro de API keys) y `runtime/` (la raíz de composición), con
almacenamiento neutral al backend y claves de firma consumidos desde `@salamandradevs/core`.

- [implementado] GATEWAY-001 Arquitectura del servicio: raíz de composición, plugins de
  política HTTP encapsulados, enlace del contexto de ejecución y mapeo de errores
  clasificados a problemas RFC 9457. (refs: `docs/plans/261106-gateway-plan.md`,
  `src/runtime/`, `src/api/plugins/`)
- [implementado] GATEWAY-002 Terminación de la API key de tenant en el borde: el plugin de
  API key verifica la cabecera `x-api-key`, establece `x-tenant-id` a partir de la clave
  verificada, y el límite de confianza elimina cualquier `x-tenant-id` o `x-api-key`
  suministrado por el cliente antes de reenviar. (refs: `src/api/plugins/api-key.ts`,
  `src/api/plugins/security.ts`, `src/api/plugins/proxy.ts`)
- [implementado] GATEWAY-003 Registro y administración de API keys: emitir, listar y revocar
  claves de tenant mediante `ApiKeysService`, todo en el tenant por defecto donde vive el
  registro. Rutas: `POST`, `GET /gateway/v1/apikeys`,
  `DELETE /gateway/v1/apikeys/:recordId`. (refs: `src/application/api-keys-service/`,
  `src/api/routes/v1/apikeys/`, `src/storage/apikeys/`)
- [implementado] GATEWAY-004 API keys firmadas autocontenidas: una clave es
  `slm_<payload>.<signature>` que lleva `{ v: 2, kid, jti, tid, plan, exp }`, firmada con
  ES256 mediante `ISigningKeyProvider`; `verify` es un cálculo local sobre la clave firmada
  y un conjunto de revocaciones en memoria, por lo que no hace ninguna lectura de base de
  datos. El conjunto se reconstruye desde el registro cada
  `gateway.apiKeys.revocationRefreshSecs` y se reemplaza por completo, y una réplica que
  revoca rechaza la clave de inmediato. (refs: `src/application/api-keys-service/`,
  `src/runtime/api-keys.ts`, `README.md`)
- [implementado] GATEWAY-005 Valores seguros de transporte por defecto y límite de tasa por
  tenant: cabeceras de seguridad de Helmet, CORS cerrado y niveles de límite de tasa
  respaldados por Redis, con un nivel `public` por IP del cliente y una política
  `{ max, timeWindow }` por plan (`platform`, `community`, `client`) contados bajo claves
  con prefijo de nivel para que cada réplica comparta un solo conteo. Cada ruta debe
  declarar exactamente un nivel base o el arranque falla; un límite excedido responde un 429
  clasificado con `retry-after`, y una caída de Redis falla en abierto hacia los niveles
  propios de cada upstream. (refs: `src/api/plugins/security.ts`,
  `src/api/routes/v1/hooks/rate-limit.ts`, `src/runtime/redis.ts`)
- [implementado] GATEWAY-006 Proxy inverso hacia upstreams: una ruta comodín con clave por
  cada upstream configurado (`ALL /<prefix>/*`) y una ruta explícita por cada ruta pública,
  reenviadas mediante `@fastify/reply-from`. Las rutas públicas cuentan contra el nivel por
  IP; el gateway añade la IP del cliente a `x-forwarded-for` y reenvía `x-request-id`.
  (refs: `src/api/plugins/proxy.ts`, `src/config/schema.ts`)
- [implementado] GATEWAY-007 Autorización de administrador mediante JWKS: las tres rutas de
  administración de claves requieren un token bearer de administrador, que el gateway
  verifica como cliente JWKS contra el conjunto de claves publicado por auth usando el
  `AccessTokenVerifier` de core, sin poseer claves de firma de token propias. (refs:
  [CORE-010](./CORE-ROADMAP.md#funcionalidades-de-core),
  `src/api/plugins/authentication.ts`, `src/api/plugins/authorization.ts`,
  `src/runtime/tokens.ts`)
- [implementado] GATEWAY-008 Almacenamiento y claves de firma neutrales al backend:
  `DatabaseProvider` y `DatabaseAdapter` conectan y aprovisionan el registro, y
  `createKeysProvider` selecciona el proveedor de clave de firma nombrado por
  `SIGNING_KEY_PROVIDER` (`openbao` o `jose`), cada uno desde `@salamandradevs/core`.
  (refs: [CORE-008](./CORE-ROADMAP.md#funcionalidades-de-core),
  [CORE-009](./CORE-ROADMAP.md#funcionalidades-de-core), `src/runtime/database.ts`,
  `src/runtime/keys.ts`)
- [implementado] GATEWAY-009 Semilla de arranque de la clave de plataforma: con
  `GENERATE_PLATFORM_API_KEY` establecido, el gateway emite o vuelve a firmar la clave
  `platform` del tenant por defecto en su propio contexto de tenant por defecto y la
  registra en `warn`, de modo que un borde por lo demás con clave pueda acuñar su primera
  credencial. La semilla es idempotente en el registro, reutilizando un único registro entre
  ejecuciones. (refs: `src/runtime/api-keys.ts`)
- [implementado] GATEWAY-010 Plugin de paginación por cursor: decodificación de la petición
  y `nextCursor` en la respuesta sobre el códec de cursor de registros compartido.
  (refs: `src/api/plugins/record-cursor-codec.ts`)
- [implementado] GATEWAY-011 Comprobación de salud: liveness en `GET /health` fuera del
  segmento de versión. (refs: `src/api/routes/health/`)
- [implementado] GATEWAY-012 Configuración originada en el entorno: cada valor respaldado por
  el entorno (`DB_SERVER_URI`, `REDIS_URL`, `SIGNING_KEY_PROVIDER`,
  `JOSE_KEYFILE_PASSPHRASE`, `BAO_ADDR`, `BAO_TOKEN`, los niveles de log) se declara en su
  esquema con `.meta({ env })`, se transporta en `config/config.json` como marcador de
  posición y lo resuelve `loadEnvVariables` en el límite de consumo, de modo que un secreto
  nunca entra en el objeto de configuración decorado. (refs:
  [TOOLS-007](./TOOLS-ROADMAP.md#funcionalidades-de-tools), `src/config/schema.ts`,
  `src/api/config/schema.ts`)
- [implementado] GATEWAY-013 Contenerización con Docker y CI/CD: una imagen multietapa
  `node:22-alpine` se ejecuta como usuario no root, y el workflow `build-and-deploy` ejecuta
  `pnpm check` y `pnpm lint` como puerta antes de construir y publicar la imagen por SHA de
  commit. (refs: `Dockerfile`, `docker-compose.yml`, `docker-compose.dev.yml`,
  `.github/workflows/deploy.yml`)
- [implementado] GATEWAY-014 Verificación de API key compartida en core: el verificador y el
  formato de clave se trasladaron a `@salamandradevs/core`, de modo que otros servicios
  pueden verificar claves de tenant. `ApiKeysService` ahora firma mediante
  `ISigningKeyProvider` en un `sign` privado y verifica mediante el `ApiKeyVerifier` de
  core, y `apiKeysConfigSchema` extiende el `apiKeyVerifyConfigSchema` de core, de modo que
  `prefix` y `maxLength` vienen de `gateway.apiKeys`. (refs:
  `docs/plans/261111-api-key-verification-plan.md`,
  [CORE-012](./CORE-ROADMAP.md#funcionalidades-de-core),
  `src/application/api-keys-service/`, `src/application/config/schema.ts`,
  `src/runtime/api-keys.ts`)
