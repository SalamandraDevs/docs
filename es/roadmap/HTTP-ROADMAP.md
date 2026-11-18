---
title: Roadmap de la librería HTTP
lastUpdate: 2026-11-18
lang: es
category: planificacion
status: active
---

# Roadmap de la librería HTTP

Estado de las funcionalidades de `http/` (`@salamandradevs/http`). Las rutas en `refs` son
relativas a `http/`; los documentos de plan bajo `docs/plans/` son relativos a la raíz del
espacio de trabajo.

## Funcionalidades de HTTP

Transporte Fastify compartido: los plugins, los tipos de hook, los esquemas de transporte y
el cliente saliente que `auth` y `gateway` registran y consumen.

- [implementado] HTTP-001 Plugin de errores (`http-errors`): mapea los nombres de
  `CustomError` a respuestas de problema RFC 9457 para que cada límite reporte un cuerpo
  estable y documentado. (refs: `src/plugins/errors.ts`, `src/schemas/problems.ts`,
  `docs/plans/261112-http-package-plan.md`)
- [implementado] HTTP-002 Plugin de seguridad (`http-security`): valores seguros de
  transporte por defecto y una guarda `onRoute` que exige exactamente un nivel de límite de
  tasa por ruta, con contadores en el `options.store` inyectado (en memoria por defecto,
  Redis cuando se comparte). (refs: `src/plugins/security.ts`, `src/schemas/rate-limit.ts`,
  `docs/plans/261112-http-package-plan.md`)
- [implementado] HTTP-003 Plugin de autenticación (`http-authentication`): verifica el token
  bearer y decora `authenticateRequest` y `request.sessionGrant`. (refs:
  `src/plugins/authentication.ts`, `docs/plans/261112-http-package-plan.md`)
- [implementado] HTTP-004 Plugin de autorización (`http-authorization`): decora
  `requireAdminRole` y `requireDefaultTenantSession`. (refs:
  `src/plugins/authorization.ts`, `docs/plans/261112-http-package-plan.md`)
- [implementado] HTTP-005 Plugin de contexto de ejecución (`http-execution-context`):
  enlaza el `ExecutionContext` por petición al alcance asíncrono y decora
  `bindExecutionContext`. (refs: `src/plugins/execution-context.ts`,
  `docs/plans/261112-http-package-plan.md`)
- [implementado] HTTP-006 Plugin de códec de cursor de registros
  (`http-record-cursor-codec`): decora `request.decodeCursor` y `reply.encodeCursor` para
  cursores de paginación opacos. (refs: `src/plugins/record-cursor-codec.ts`,
  `docs/plans/261112-http-package-plan.md`)
- [implementado] HTTP-007 Esquemas de transporte: `problemDetailsSchema`,
  `problemResponses`, `paginationQuerySchema`, `createPageSchema`, `recordParamsSchema` y
  `rateLimitPolicySchema`, con sus tipos inferidos. (refs: `src/schemas/`,
  `docs/plans/261112-http-package-plan.md`)
- [implementado] HTTP-008 Tipos de arreglo de hooks: `PreHandlerHooks` y `OnRequestHooks`,
  que los compositores de servicio especializan en sus cadenas de ruta. (refs: `src/hooks/`,
  `docs/plans/261112-http-package-plan.md`)
- [implementado] HTTP-009 Cliente saliente `makeApiRequest`: el helper de fetch compartido
  para llamadas de servicio a servicio, que reenvía el id de petición enlazado como
  `x-request-id` cuando quien llama pasa su proveedor `executionCtx`, y parsea las
  respuestas de problema contra `problemDetailsSchema`.
  Trasladado aquí desde `tools` (antes TOOLS-002). (refs: `src/client/`,
  `docs/plans/261112-http-package-plan.md`)
- [propuesto] HTTP-010 Suite de pruebas en la costura de contrato: el paquete se publica
  actualmente sin pruebas (decisión 14), de modo que los plugins de transporte, los
  esquemas y el cliente se ejercitan solo a través de sus consumidores. Añadir una suite de
  pruebas del paquete cuando aterrice la estrategia de pruebas de la plataforma. (refs:
  `docs/plans/261029-tests-suite-plan.md`, `docs/plans/261112-http-package-plan.md`)
