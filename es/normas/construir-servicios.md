---
title: Construir servicios
lastUpdate: 2026-11-18
lang: es
category: guia
status: published
---

## Propósito

Reglas de la casa para construir servicios Fastify en la plataforma de Salamandra Devs: cómo
un servicio aplica las [Buenas prácticas de desarrollo][gp] (`GP`) generales a su capa de
transporte, hooks de ruta, endpoints de salud, correlación de peticiones y respuestas de
error. Estas reglas aplican a cada contribuyente, humano o agente, que trabaje en `services/`
o en un paquete que entregue plugins de Fastify.

Cada regla nombra la práctica GP o el patrón DP que aplica. Esa práctica o patrón enuncia el
principio de toda la plataforma; la regla BS enuncia cómo un servicio Fastify lo cumple.
Cuando ambos parezcan discrepar, la práctica o el patrón gana y la regla BS se corrige.

BS-05 a BS-08 eran CS-09 a CS-12 en [Estilo de código](./estilo-codigo.md) hasta el
2026-11-13; los IDs CS están retirados y nunca se reutilizan.

Cada sección de regla lleva un ID estable (`BS-NN`) para su citación; los IDs siguen
[IDs de documento](./convenciones-nombres.md#ids-de-documento). Cada sección termina con una
línea **Comprobación:**, una condición que quien revisa o un agente puede verificar en el
repositorio. Los ejemplos son extractos recortados que comienzan con un comentario que nombra
su archivo fuente; los contraejemplos se marcan con `Evitar` y son hipotéticos.

## Transporte

### BS-01 Valores seguros de transporte por defecto

Los servicios HTTP habilitan headers de seguridad, CORS explícito, límites de tasa y
atributos de cookie seguros. Cada ruta debe declarar su nivel de límite de tasa, y el
servicio se niega a arrancar cuando una ruta no declara ninguno, de modo que una ruta nueva
nunca queda sin límite por accidente; una ruta queda sin límite solo cuando declara
explícitamente el nivel exento. Los niveles cuentan por la identidad más estrecha conocida en
cada punto: el gateway mide el tráfico anónimo por IP del cliente (rutas públicas y peticiones
cuya API key falló) y el tráfico con clave por el tenant que nombra su API key; cada servicio
detrás del gateway mide el tráfico público por IP del cliente, el subject autenticado una vez
verificado un token, y la cuenta nombrada en la petición en las rutas de credenciales. Los
valores de los niveles provienen de la configuración, nunca de números literales en el
código. Detrás de un proxy, el servicio confía solo en las direcciones del proxy, de modo que
los límites cuentan clientes reales. El plugin de seguridad compartido de
`@salamandradevs/http` impone la guarda del nivel base en cada servicio detrás del gateway; el
propio plugin de API key del gateway impone su nivel base, que también admite el hook de API
key.

```typescript
// services/auth/src/api/routes/v1/session/routes.ts
onRequest: tenantHooks(fastify),
preHandler: accountRateLimitHooks(fastify, 'username'),

// services/auth/src/api/routes/health/routes.ts
config: { tenant: false },
onRequest: exemptRateLimitHooks(fastify),
```

Una ruta sin nivel detiene el arranque con un `CONFIG_ERROR` clasificado ([GP-08][gp-08]), y
un límite excedido responde con un 429 clasificado con un header `retry-after`
([BS-04][bs-04]). La cookie de refresco lleva el prefijo `__Host-` con `secure`, `httpOnly` y
`sameSite`.

```typescript
// Evitar: una ruta sin nivel, un límite fijado en el código, y un proxy en el que se confía
// para cualquier llamador.
fastify.get('/', { preHandler: protectionHooks(fastify) }, readCurrentSession);
config: { rateLimit: { max: 5, timeWindow: '1 minute' } },
Fastify({ trustProxy: true });
```

Aplica: [GP-25][gp-25]; [GP-10][gp-10]; [GP-14][gp-14].

**Comprobación:** cada ruta declara exactamente un nivel base (público o exento), impuesto al
arrancar; en el gateway cada ruta con clave declara el hook de API key como su nivel base, y
cada ruta de servicio con alcance de tenant declara el hook de tenant, negándose a arrancar en
caso contrario; ninguna ruta declara valores de límite en el código; las rutas de
credenciales añaden un límite por cuenta; `trustProxy` nunca es `true`.

## Operaciones

### BS-02 Liveness y readiness

Un servicio expone una comprobación de liveness (el proceso responde) y una comprobación de
readiness (sus servicios de respaldo requeridos son alcanzables), y la definición del
contenedor las usa. Sin readiness, el tráfico alcanza una instancia que no puede servirlo.

```typescript
// services/auth/src/api/routes/health/controller.ts
const response: HealthResponse = {
  service: 'auth',
  status: 'ok',
  timestamp: new Date().toISOString()
};
```

```yaml
# Evitar: una definición de contenedor sin comprobación de salud, de modo que una instancia
# rota permanece en rotación hasta que alguien lo nota.
services:
  auth:
    image: ghcr.io/salamandradevs/auth:<sha>
```

Aplica: [GP-15][gp-15]; [GP-03][gp-03].

**Comprobación:** existe un endpoint de readiness y el Dockerfile o el archivo compose
declara un `HEALTHCHECK` o `healthcheck`.

### BS-03 Correlación de peticiones

Cada petición lleva un id de correlación que se acepta de quien llama en el header
`x-request-id` cuando está presente, se adjunta a cada línea de log y se enlaza en el contexto
de ejecución para el código posterior. Fastify rellena `request.id` a partir de ese header
mediante la opción `requestIdHeader`, y el plugin de contexto de ejecución lo copia en
`ExecutionContext.request.requestId`.

```typescript
// http/src/plugins/execution-context.ts
const executionContext: ExecutionContext = {
  authenticated: { actorId: request.sessionGrant.actorId, /* ... */ },
  request: { requestId: request.id }
};

options.executionCtx.run(executionContext, done);
```

```typescript
// Evitar: una llamada a otro servicio de la plataforma que descarta el id de correlación.
await fetch(`${profilesUrl}/v1/persons/${recordId}`);
```

Una llamada a otro servicio de la plataforma reenvía el id a través del cliente compartido
`makeApiRequest` de `@salamandradevs/http`. La opción `executionCtx` se omite por defecto, de
modo que una llamada a un tercero nunca lleva el id de la plataforma; una llamada a un par de
la plataforma pasa el proveedor de contexto de ejecución del servicio, cuyo id de petición
enlazado se envía como `x-request-id`.

```typescript
// Preferir: reenviar el id de correlación a un par de la plataforma.
await makeApiRequest(profilesUrl, `/v1/persons/${recordId}`, { method: 'GET' }, {
  executionCtx
});
```

Aplica: [GP-18][gp-18]; [DP-09][dp-09].

**Comprobación:** la instancia de Fastify establece `requestIdHeader: 'x-request-id'`, y las
llamadas salientes a otros servicios de la plataforma reenvían `x-request-id`.

### BS-04 Respuestas de problema

El transporte mapea cada `ErrorName` a una respuesta de problema RFC 9457 con una URI de
`type` de problema estable. Cada URI es un slug resuelto contra una única base construida a
partir del origen de API configurado, de modo que ningún host se escribe en el código.

```typescript
// services/auth/src/api/plugins/index.ts
problemTypeBase: new URL('/problems/', options.config.publicUrls.api)

// http/src/plugins/errors.ts
} else if (error.name === 'RESOURCE_NOT_FOUND') {
  status = 404;
  slug = 'resource-not-found';
// ...
type: new URL(slug, options.problemTypeBase).toString(),
```

```typescript
// Evitar: un type de problema con un host fijado en el código.
type = 'https://api.example.com/problems/resource-not-found';
```

Aplica: [GP-20][gp-20]; [GP-10][gp-10].

**Comprobación:** cada URI de `type` de problema se resuelve contra una única base construida
a partir de la configuración, y ningún archivo fuente contiene un host de type de problema.

## Hooks de ruta

Cómo los servicios de API RESTful componen los hooks de ruta de Fastify: dónde viven los
hooks compartidos, cuándo un grupo de rutas puede definir los suyos, y cómo los hooks locales
de ruta permanecen tipados sin aserciones. Los arreglos de hooks son cadenas de
responsabilidad ([DP-06][dp-06]).

### BS-05 Hooks compartidos

- Los hooks de ruta transversales o reutilizables deben publicarse como decoraciones de plugin
  de Fastify. Esto incluye autenticación, enlace del contexto de ejecución, hooks de
  tasa/tenant/seguridad y hooks de autorización comunes. No exporte funciones helper sueltas
  desde los módulos de hooks compartidos ni las pase directamente como hooks de ciclo de vida
  de Fastify.
- Use `api/routes/v1/hooks/` como la fuente preferida para la composición de hooks compartidos
  de v1. Estos archivos deberían componer los hooks decorados de Fastify en arreglos ordenados
  para las políticas de ruta compartidas.
- Un compositor compartido puede recibir parámetros cuando construye un hook a partir de una
  decoración de plugin, como `fastify.rateLimitHook(policy)`. Cada llamada crea una nueva
  instancia del hook, que es como una política configurada obtiene su propio estado por ruta.
  Los parámetros configuran el hook decorado; nunca lo reemplazan por una función suelta.

```ts
// api/routes/v1/hooks/authenticate.ts
export function protectionHooks(fastify: FastifyInstance): PreHandlerHooks {
  return [
    fastify.authenticateRequest,
    fastify.authenticatedRateLimit,
    fastify.bindExecutionContext
  ];
}

// api/routes/v1/hooks/rate-limit.ts
export function accountRateLimitHooks(
  fastify: FastifyInstance,
  field: 'username' | 'email'
): PreHandlerHooks {
  const policy = fastify.config.http.rateLimit.credential.perAccount;

  return [fastify.rateLimitHook(policy /* , keyGenerator desde request.ip + field */)];
}
```

Aplica: [DP-06][dp-06]; [GP-02][gp-02].

**Comprobación:** ningún módulo de hooks compartidos exporta una función suelta que una ruta
pase directamente como hook de ciclo de vida; los compositores en `api/routes/v1/hooks/`
devuelven arreglos de decoraciones `fastify.*`.

### BS-06 Hooks locales de ruta

- En `api/routes/v1/[slug]/`, añada un `hooks.ts` local solo cuando la ruta necesite una
  composición de hooks específica del slug o una política de ruta muy local. Los archivos de
  hooks locales pueden componer hooks compartidos de v1 con hooks decorados de Fastify, y
  pueden definir hooks locales de ruta privados solo cuando la política no se comparte fuera de
  ese grupo de rutas.

```ts
// api/routes/v1/users/hooks.ts
function requireUserOwnerOrAdmin(request: FastifyRequest): void {
  // Política local de ruta que no comparten otros grupos de rutas.
}

export function userOwnerProtectionHooks(fastify: FastifyInstance): PreHandlerHooks {
  return [...protectionHooks(fastify), requireUserOwnerOrAdmin];
}
```

- Cuando una política local de ruta se vuelve útil fuera de un grupo de rutas, promuévala a una
  decoración de plugin de Fastify y compóngala a través de `api/routes/v1/hooks/` en lugar de
  importarla como un helper compartido.

Aplica: [DP-06][dp-06].

**Comprobación:** cada hook definido en un `hooks.ts` local de ruta se usa solo por su propio
grupo de rutas.

### BS-07 Controladores

- Los controladores deben asumir que los hooks declarados ya establecieron las precondiciones
  de la ruta. No repita las comprobaciones de autenticación, contexto de ejecución, tenant o
  autorización común en los controladores, salvo que el controlador esté manejando un estado de
  ejecución distinto que los hooks no puedan establecer.

Aplica: [DP-06][dp-06].

**Comprobación:** ningún controlador vuelve a comprobar autenticación, contexto de ejecución,
tenant o roles que los hooks de su ruta ya establecen.

### BS-08 Hooks locales de ruta tipados

- Un hook local de ruta que necesita acceso tipado a partes de la petición específicas de la
  ruta (`params`, `body`, `querystring`, `headers`) se tipa mediante un tipo de definición de
  ruta, nunca mediante augmentación de `FastifyRequest` ni mediante aserciones. La augmentación
  se reserva para decoraciones invariantes de ruta como `sessionGrant` o los codecs de cursor.
- Tipe el parámetro del hook como `FastifyRequest<TRouteDef>`, componga el arreglo de hooks a
  través del helper compartido `PreHandlerHooks<TRouteDef>` de `@salamandradevs/http`, y
  reutilice la misma definición de ruta en el registro de la ruta y en el controlador para que
  el contrato de la petición tenga una sola fuente de verdad. Defina `TRouteDef` en
  `api/routes/v1/[slug]/types.ts`.
- Mantenga los compositores compartidos e invariantes de ruta sobre el genérico por defecto
  `PreHandlerHooks` y ponga la especialización en el compositor local de ruta; un hook de
  genérico por defecto compone dentro de un arreglo especializado, nunca al revés.
- Pista: si un hook necesita aserción de `request.params`, `body` o `query` (`as ...`) para
  tipar, el enfoque de definición de ruta no se aplicó; introduzca una definición de ruta y
  pásela por el compositor en lugar de aseverar.

```ts
// http/src/hooks/types.ts
export type PreHandlerHooks<
  TRouteDef extends RouteGenericInterface = RouteGenericInterface
> = preHandlerHookHandler<
  RawServerDefault,
  RawRequestDefaultExpression,
  RawReplyDefaultExpression,
  TRouteDef
>[];

// api/routes/v1/users/hooks.ts
function authorizeUserUpdate(request: FastifyRequest<UpdateUserRequest>): void {
  const targetRecordId: string = request.params.recordId; // inferido, sin aserción
}

export function userUpdateProtectionHooks(
  fastify: FastifyInstance
): PreHandlerHooks<UpdateUserRequest> {
  return [...protectionHooks(fastify), authorizeUserUpdate];
}
```

Aplica: [DP-06][dp-06]; [GP-04][gp-04].

**Comprobación:** ningún hook ni controlador asevera `request.params`, `body` o `query` con
`as`; las partes de la petición específicas de la ruta se tipan mediante una definición de
ruta en el `types.ts` del slug.

## Referencias

- Nottingham, Mark, Erik Wilde, y Sanjay Dalal.
  [RFC 9457: Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457). IETF,
  2023.

[bs-04]: #bs-04-respuestas-de-problema
[dp-06]: ../arquitectura/patrones-diseno.md#dp-06-chain-of-responsibility
[dp-09]: ../arquitectura/patrones-diseno.md#dp-09-ambient-context
[gp]: ../arquitectura/buenas-practicas-desarrollo.md
[gp-02]: ../arquitectura/buenas-practicas-desarrollo.md#gp-02-estructura-de-servicio-en-capas
[gp-03]: ../arquitectura/buenas-practicas-desarrollo.md#gp-03-contratos-cableado-en-la-raíz-de-composición-y-backends-configurados
[gp-04]: ../arquitectura/buenas-practicas-desarrollo.md#gp-04-esquemas-como-única-fuente-de-verdad
[gp-08]: ../arquitectura/buenas-practicas-desarrollo.md#gp-08-configuración-validada-fallo-rápido
[gp-10]: ../arquitectura/buenas-practicas-desarrollo.md#gp-10-la-configuración-varía-por-despliegue-no-por-build
[gp-14]: ../arquitectura/buenas-practicas-desarrollo.md#gp-14-procesos-sin-estado
[gp-15]: ../arquitectura/buenas-practicas-desarrollo.md#gp-15-descartabilidad-arranque-rápido-parada-ordenada
[gp-18]: ../arquitectura/buenas-practicas-desarrollo.md#gp-18-logs-como-flujo-de-eventos
[gp-20]: ../arquitectura/buenas-practicas-desarrollo.md#gp-20-errores-clasificados
[gp-25]: ../arquitectura/buenas-practicas-desarrollo.md#gp-25-seguro-por-defecto-y-mínimo-privilegio
