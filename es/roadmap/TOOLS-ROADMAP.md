---
title: Roadmap de la librería Tools
lastUpdate: 2026-11-18
lang: es
category: planificacion
status: active
---

# Roadmap de la librería Tools

Estado de las funcionalidades de `tools/` (`@salamandradevs/tools`). Las rutas en `refs`
son relativas a `tools/`; los documentos de plan bajo `docs/plans/` son relativos a la raíz
del espacio de trabajo.

## Funcionalidades de TOOLS

Pequeños helpers compartidos sin política de aplicación. Exporta
`@salamandradevs/tools/shared` (seguro para el navegador) y `@salamandradevs/tools/node`
(solo Node).

- [implementado] TOOLS-001 `CustomError` y nombres de error. (refs: `src/shared/errors.ts`)
- [implementado] TOOLS-003 Guardas de tipo. (refs: `src/shared/typeguards.ts`)
- [implementado] TOOLS-004 `zodToMongoSchema`. (refs: `src/shared/transform.ts`)
- [implementado] TOOLS-006 Entrada de firma de JWT y comparación de hash con tiempo
  constante. (refs: `src/node/crypto.ts`)
- [implementado] TOOLS-007 `loadEnvVariables`: lee de `process.env` los campos
  `.meta({ env })` de un esquema de configuración, valida cada uno con su propio esquema de
  campo y devuelve los valores resueltos sin tocar el objeto de configuración, lanzando
  `ENV_VAR_ERROR` ante una variable ausente o mal formada. (refs: `src/node/environment.ts`)
- [propuesto] TOOLS-005 Conversión real de `zodToMySqlSchema`: la función actual es un
  marcador de posición. (refs: `docs/plans/zod-to-mysql-schema.md`)
- [implementado] TOOLS-008 Nombre de error `RATE_LIMIT_EXCEEDED`: añade la clasificación que
  el limitador de tasa de auth usa para su respuesta 429. (refs: `src/shared/errors.ts`,
  `docs/plans/261031-rate-limit-plan.md`)
- [propuesto] TOOLS-009 Herramientas de mantenimiento de datasets: helpers programáticos para
  podar registros de base de datos y de Redis (como registros de API key expirados y
  entradas de caché obsoletas), invocados explícitamente por el módulo cron de cada servicio
  en lugar de por políticas del lado del almacén como los índices TTL. Se añaden cuando los
  servicios se ejecuten juntos. (refs: pendiente; consumido por los módulos cron de cada
  servicio)
