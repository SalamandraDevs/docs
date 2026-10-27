---
title: Roadmap del servicio Auth
lastUpdate: 2026-10-25
lang: es
category: planificacion
status: active
---

# Roadmap del servicio Auth

Estado de las funcionalidades de `services/auth/` (`@salamandradevs/auth`). Las rutas en
`refs` son relativas a `services/auth/`.

## Funcionalidades de AUTH

API REST en Fastify que emite y verifica sesiones mediante credenciales de contraseña y
proveedores OAuth. Capas: `api/` (transporte), `application/` (política), `records/`
(persistencia), `infrastructure/` (claves de OpenBao, proveedores OAuth).

- [implementado] AUTH-001 Arquitectura del servicio: raíz de composición, plugins de ruta
  encapsulados, enlace del contexto de ejecución, OpenAPI. (refs: `docs/auth-plan.md`,
  `src/runtime/`)
- [implementado] AUTH-002 Inicio de sesión con contraseña: verificación con Argon2 en el
  tenant por defecto. Ruta: `POST /v1/session`. (refs: `src/application/passwords-service/`)
- [implementado] AUTH-003 Flujo OAuth de código de autorización: proveedores Google y
  GitHub. Rutas: `POST` y `GET /v1/oauth/:provider/authorize`,
  `GET /v1/oauth/:provider/callback`. (refs: `src/application/oauth-service/`,
  `src/infra/oauth/`)
- [implementado] AUTH-004 Sesiones: emite, lee, termina y refresca con tokens de refresco
  opacos y rotativos almacenados como hashes HMAC. Rutas: `GET`, `DELETE /v1/session`,
  `POST /v1/session/refresh`. (refs: `docs/session-plan.md`,
  `src/application/sessions-service/`)
- [implementado] AUTH-005 Tokens de acceso: emisión y verificación de JWT ES256 hacia una
  concesión de sesión. (refs: `src/application/tokens-service/`)
- [implementado] AUTH-006 Claves de firma durables: firma con OpenBao Transit contra un
  ciclo de vida de clave propiedad del operador, con JWKS en
  `GET /.well-known/jwks.json`. (refs: AUTH-023, `docs/signing-plan.md`,
  `services/auth/src/infra/keys/`, `docs/en/howto/openbao/`)
- [implementado] AUTH-007 Usuarios: crear, registrar, leer, actualizar, eliminar y buscar
  mediante `UsersService`. Rutas bajo `/v1/users`. (refs: `docs/users-service-plan.md`,
  `docs/users-service-completion.md`)
- [implementado] AUTH-008 Establecimiento de contraseña del administrador:
  `POST /v1/password/set`. (refs: `src/application/passwords-service/`)
- [implementado] AUTH-009 Plugin de paginación por cursor: decodificación de la petición y
  `nextCursor` en la respuesta. (refs: `docs/entity-cursor-plugin.md`,
  `src/api/plugins/record-cursor-codec.ts`)
- [implementado] AUTH-010 Salud y OpenAPI: `GET /health`, `GET /openapi.json`, UI en
  `/docs`. (refs: `src/api/routes/health/`, `src/api/plugins/openapi.ts`)
- [implementado] AUTH-018 Autorización a nivel de recurso para las autoactualizaciones de
  usuario: los propietarios pueden actualizar solo username/email, los administradores
  pueden actualizar todos los campos. (refs: `src/api/routes/v1/users/hooks.ts`)
- [implementado] AUTH-019 Cierre de brechas de IDOR y autorización en la eliminación de
  usuarios: `DELETE /v1/users/:recordId` restringido a administradores, protección del
  administrador bootstrap configurado frente a su eliminación, configuración obligatoria de
  `bootstrapAdmin` y cascada de revocación de tokens de refresco ante la eliminación de un
  usuario y el cambio de privilegios. (refs: `docs/idorGapsClosing.md`,
  `src/api/routes/v1/users/routes.ts`, `src/application/users-service/`,
  `src/application/sessions-service/`)
- [implementado] AUTH-021 Extracción de la selección de backend de base de datos:
  `bootstrapService` se conecta mediante `DatabaseProvider` y usa `DatabaseAdapter` para
  aprovisionar datasets y construir adaptadores CRUD, sin dejar ninguna rama de backend en
  el bootstrap.
  (refs: [CORE-008](./CORE-ROADMAP.md#funcionalidades-de-core), `services/auth/src/runtime/`)
- [implementado] AUTH-023 Ciclo de vida de la clave de firma propiedad del operador: el
  proveedor de OpenBao se reduce a las cuatro operaciones de `ISigningKeyProvider` y hace
  exactamente dos llamadas, una lectura KV del documento de ciclo de vida y una firma
  Transit. Desaparecen la autorrenovación, el preflight de arranque, el reporte del tiempo
  de vida del token, la creación de claves, el motor de ciclo de vida que cubría
  activación, rotación y limpieza, la destrucción de claves y toda la vía de escritura,
  junto con `rotationIntervalSecs` y `verificationOverlapSecs`. El proveedor pasa de 1238 a
  642 líneas. El token de runtime es de larga duración en lugar de periódico, y la política
  ahora es `read` sobre la ruta de estado más `update` sobre `transit/sign/*`, con
  `transit/keys/*` ausente por completo, de modo que un servicio comprometido no puede ni
  destruir material de clave ni reescribir su propio estado. El aprovisionamiento, la
  rotación y la destrucción son procedimientos del operador. (refs:
  `docs/en/howto/openbao/README.md`, `docs/signing-plan.md`,
  `services/auth/src/infra/keys/openbao/`)
- [implementado] AUTH-024 Configuración originada en el entorno: cada valor respaldado por
  el entorno (`BAO_ADDR`, `BAO_TOKEN`, los secretos de cliente OAuth, `JWT_HASH_SECRET`,
  `BOOTSTRAP_ADMIN_PASSWORD`, la passphrase del archivo de clave jose) se declara en su
  esquema con `.meta({ env })`, se transporta en `config/local.json` como marcador de
  posición y lo resuelve `loadEnvVariables`
  ([TOOLS-007](./TOOLS-ROADMAP.md#funcionalidades-de-tools)) en el límite de consumo. El
  objeto de configuración conserva el marcador de posición, de modo que un secreto nunca
  entra en el `request.server.config` decorado. En `pnpm start` el proceso carga `.env`
  mediante el `--env-file` de Node, de modo que las variables declaradas llegan a
  `process.env` antes de que la raíz de composición las lea.
  (refs: [TOOLS-007](./TOOLS-ROADMAP.md#funcionalidades-de-tools),
  `src/application/config/schema.ts`, `src/infra/keys/`, `src/infra/oauth/config/schema.ts`,
  `src/runtime/`, `config/devel.json`, `config/prod.json`, `package.json`)
- [implementado] AUTH-027 Contenerización con Docker: imagen multietapa `node:22-alpine`
  (pnpm fijado con corepack, `--frozen-lockfile`, `pnpm prune --prod`, usuario no root,
  prebuilds musl de argon2 para que Alpine no necesite toolchain), con `docker-compose.yml`
  (producción, baja la imagen de GHCR, monta `config/prod.json`) y `docker-compose.dev.yml`
  (build local, monta `config/devel.json` y el keyfile jose). El contenedor selecciona la
  configuración por `NODE_ENV` y toma los secretos como variables de entorno en lugar de un
  archivo `.env`. (refs: `services/auth/Dockerfile`, `services/auth/.dockerignore`,
  `services/auth/docker-compose.yml`, `services/auth/docker-compose.dev.yml`, INFRA-001)
- [implementado] AUTH-028 CI/CD con GitHub Actions: `build-and-push` construye y publica
  `ghcr.io/<owner>/auth` en la rama de producción y en los tags de versión, y `deploy` se
  ejecuta en un runner autoalojado con una comprobación de preparación con fallo rápido
  consciente del proveedor; lee `signingKeys.provider.name` y `dbServerUri` de
  `config/prod.json`, siempre sondea MongoDB, y sondea OpenBao solo cuando el proveedor es
  `openbao`, antes de `docker compose pull && up -d`. Una costura comentada dispara primero
  el pipeline de infraestructura. (refs: `services/auth/.github/workflows/deploy.yml`,
  INFRA-001)
- [implementado] AUTH-022 Selección de clave de firma neutral al backend:
  `bootstrapSigningKeys` devuelve el `OpenBaoSigningKeyProvider` concreto, `runtime/types.ts`
  tipa el servicio compuesto como esa clase, y el apagado llama a `stopTokenRenewal()` por
  su nombre, de modo que la raíz de composición sigue sabiendo qué backend construyó.
  Requiere cerrar antes la brecha del contrato: el bootstrap y el apagado usan `preflight`,
  `describeTokenLifetime`, `startTokenRenewal` y `stopTokenRenewal`, ninguno de los cuales
  declara `ISigningKeyProvider` y todos los cuales son centrados en el token.
  Reemplazarlos por una comprobación de preparación que devuelva advertencias del operador,
  y por trabajo en segundo plano plegado dentro de `initialize` más una operación de apagado
  equivalente, es independiente de cualquier segundo proveedor; la unión discriminada `type`
  y la factoría de proveedores permanecen bloqueadas por la regla del plan de firma contra
  añadir cualquiera de las dos antes de que exista una segunda implementación. (refs:
  `docs/signing-plan.md`, `services/auth/src/runtime/keys.ts`,
  `services/auth/src/runtime/types.ts`, `services/auth/src/application/contracts.ts`)
- [planificado] AUTH-011 Restablecimiento de contraseña: `POST /v1/password/reset` existe
  pero siempre falla con `ARCHITECTURE_NOT_IMPLEMENTED`.
  (refs: `src/application/passwords-service/`)
- [propuesto] AUTH-012 Vía con tiempo constante para usuario desconocido: reducir la señal
  de enumeración de usuarios. (refs: NOTA en `passwords-service.ts`)
- [propuesto] AUTH-013 PKCE de OAuth para el flujo de código de autorización:
  `code_verifier` por intento y `code_challenge` S256, lo que requiere un almacén durable de
  autorizaciones pendientes y una capacidad PKCE por proveedor (Google soportado; GitHub por
  verificar). (refs: `src/application/oauth-service/`, TODO en
  `src/application/contracts.ts`)
- [propuesto] AUTH-014 Inicio de sesión multi-tenant y selección de tenant: hoy el inicio de
  sesión es de un único tenant por defecto. (refs: `docs/session-plan.md`)
- [propuesto] AUTH-016 Verificación de correo electrónico.
- [propuesto] AUTH-017 Eventos de ciclo de vida del usuario como `auth.user.created`.
  (refs: `docs/es/normas/convenciones-nombres.md`)
- [propuesto] AUTH-020 Refresco de tokens de proveedor OAuth: persistir y refrescar los
  tokens de acceso del proveedor para llamadas posteriores a la API del proveedor en nombre
  del usuario. Aplazado mientras se prueba el flujo de inicio de sesión OAuth; hoy el inicio
  de sesión descarta los tokens del proveedor tras resolver la identidad. (refs: TODO en
  `src/application/contracts.ts`)
- [propuesto] AUTH-025 Estrategia de suite de pruebas, niveles unitario e integración:
  Vitest con alias de subruta, fakes en memoria compartidos para `ICrud`,
  `ISigningKeyProvider` e `IOAuthProvider`, y `app.inject` de Fastify sobre `buildApi`, de
  modo que el bucle rápido se ejecute sin MongoDB, OpenBao ni un proveedor OAuth real.
  (refs: `docs/tests-suite.md`, AUTH-001)
- [propuesto] AUTH-026 Arnés de pruebas de extremo a extremo: la imagen Docker construida
  ejercitada por HTTP contra MongoDB, JoseSigningKeyProvider y un proveedor OIDC simulado
  mediante docker-compose en CI, cubriendo los adaptadores concretos y el bootstrap de
  `createRuntime` que el nivel de integración omite. Se construye después de AUTH-025.
  (refs: `docs/tests-suite.md`, AUTH-025)
- [cancelado] AUTH-015 Rotación programada de la clave de firma: rechazada. AUTH-023 movió
  todo el ciclo de vida de la clave al operador, así que la rotación es un procedimiento por
  CLI de OpenBao y el servicio auth no tiene ninguna rotación que programar. (refs:
  `docs/en/howto/openbao/README.md`)
</content>
