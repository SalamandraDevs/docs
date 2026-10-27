---
title: Roadmap de la librería Tools
lastUpdate: 2026-10-25
lang: es
category: planificacion
status: active
---

# Roadmap de la librería Tools

Estado de las funcionalidades de `tools/` (`@salamandradevs/tools`). Las rutas en `refs`
son relativas a `tools/`.

## Funcionalidades de TOOLS

Pequeños helpers compartidos sin política de aplicación. Exporta
`@salamandradevs/tools/shared` (seguro para el navegador) y `@salamandradevs/tools/node`
(solo Node).

- [implementado] TOOLS-001 `CustomError` y nombres de error. (refs: `src/shared/errors.ts`)
- [implementado] TOOLS-002 Helper de fetch `makeApiRequest`. (refs: `src/shared/requests.ts`)
- [implementado] TOOLS-003 Guardas de tipo. (refs: `src/shared/typeguards.ts`)
- [implementado] TOOLS-004 `zodToMongoSchema`. (refs: `src/shared/transform.ts`)
- [implementado] TOOLS-006 Entrada de firma de JWT y comparación de hash con tiempo
  constante. (refs: `src/node/crypto.ts`)
- [implementado] TOOLS-007 `loadEnvVariables`: lee de `process.env` los campos
  `.meta({ env })` de un esquema de configuración, valida cada uno con su propio esquema de
  campo y devuelve los valores resueltos sin tocar el objeto de configuración, lanzando
  `ENV_VAR_ERROR` ante una variable ausente o mal formada. (refs: `src/node/environment.ts`)
- [propuesto] TOOLS-005 Conversión real de `zodToMySqlSchema`: la función actual es un
  marcador de posición. (refs: `docs/zod-to-mysql-schema.md`)
</content>
