---
title: Buenas prácticas de desarrollo
lastUpdate: 2026-10-25
lang: es
category: arquitectura
status: published
---

## Introducción

Una **buena práctica** es una regla de ingeniería que previene una clase conocida de fallo:
builds que no pueden reproducirse, secretos que se filtran, procesos que pierden estado o
incidentes que nadie puede rastrear. Esta guía cubre las prácticas que mantienen los
paquetes y servicios de Salamandra Devs modulares, reproducibles, desplegables, observables
y seguros. Aplican a `core/`, `tools/` y a todos los servicios bajo `services/`.

Cada práctica lleva un ID estable (`GP-NN`) para su citación; los IDs siguen
[IDs de documento](../normas/convenciones-nombres.md#ids-de-documento). Dónde el código
aún no cumple una práctica se registra en la
[Revisión de Arquitectura](../../architecture-review.md), no en esta guía.

## Terminología usada en esta guía

- **Artefacto**: la salida inmutable de un build, como una imagen de contenedor o una
  versión de npm publicada.
- **Release**: un artefacto combinado con la configuración de un despliegue.
- **Manifiesto de despliegue**: el archivo que ejecuta un release (`docker-compose.yml`, un
  job de despliegue de CI).
- **Servicio de respaldo**: un recurso que el proceso alcanza por la red, como MongoDB,
  OpenBao o un proveedor OAuth.
- **Raíz de composición**: el código de arranque que construye los objetos concretos y los
  inyecta (`services/auth/src/runtime/`); véase
  [DP-08 Composition Root](./patrones-diseno.md#dp-08-composition-root).
- **Puerta**: una comprobación automatizada que debe pasar antes de que un cambio pueda
  construirse, publicarse o desplegarse.
- **Debe / debería**: "debe" es obligatorio; "debería" permite una excepción documentada.

## Mensajes de commit semánticos

Los asuntos de los commits usan la convención de prefijo semántico: una etiqueta `<tipo>`,
un `(<alcance>)` opcional y un asunto breve en imperativo. Cuando un commit resuelve una
brecha revisada, nombre los IDs involucrados en el asunto o el cuerpo, por ejemplo `GP-09`.

Formato: `<tipo>(<alcance>): <asunto>`; el alcance es opcional.

Los tipos siguen la convención de [Semantic Commit Messages][scm]; `fix`, `feat`, `chore` y
`docs` son los que más usan los repositorios:

- **feat**: una funcionalidad nueva para el usuario, no una funcionalidad nueva para un
  script de build.
- **fix**: una corrección de bug para el usuario, no una corrección a un script de build.
- **docs**: cambios a la documentación.
- **style**: formato, puntos y comas faltantes y similares; sin cambio de código de
  producción.
- **refactor**: refactorización de código de producción, como renombrar una variable.
- **test**: añadir o refactorizar pruebas; sin cambio de código de producción.
- **chore**: actualizar herramientas de build, tareas o dependencias; sin cambio de código
  de producción.

[scm]: https://gist.github.com/joshbuchea/6f47e86d2510bce28f8e7f42ae84c716

## Ejemplos de la plataforma

Los ejemplos son extractos recortados de `core/`, `tools/` y `services/auth/`, incluyendo
sus `package.json`, Dockerfile, configuración y archivos de workflow. Cada uno comienza con
un comentario que nombra su archivo fuente. Los contraejemplos se marcan con `Evitar` y son
hipotéticos: muestran lo que rompería la práctica, no código que exista. Donde el código
real sí rompe una práctica, el caso se lista en la
[Revisión de Arquitectura](../../architecture-review.md) en su lugar.

Cada sección termina con una línea **Comprobación:**, una condición que quien revisa o un
agente puede verificar en el repositorio.

### GP-01 Un repositorio por paquete o servicio

Cada librería y cada servicio desplegable vive en su propio repositorio con su propio build,
versión y release, y el código compartido se consume como un paquete versionado. Las rutas
relativas compartidas acoplan los ciclos de release y rompen cualquier build fuera del
layout de carpetas original.

```json
// services/auth/package.json
"dependencies": {
  "@salamandradevs/core": "^0.1.0",
  "@salamandradevs/tools": "^0.1.0"
}
```

```json
// Evitar: resuelve solo dentro de un layout de carpetas y rompe la CI en otro lugar.
"dependencies": {
  "@salamandradevs/core": "file:../../core"
}
```

Relacionado: GP-05, GP-12; [Publicación de paquetes](../infra/empaquetado.md).

**Comprobación:** ninguna dependencia de `package.json` usa `file:` ni `link:`.

### GP-02 Estructura de servicio en capas

Un servicio separa transporte, casos de uso, persistencia, adaptadores externos,
configuración y cableado en secciones distintas
([Estructura de servicios](../normas/convenciones-nombres.md#estructura-de-servicios)), y
las dependencias apuntan hacia adentro: el transporte depende de los contratos y tipos de
aplicación, nunca del almacenamiento ni de los adaptadores de proveedor. Las capas mezcladas
hacen que un cambio en una preocupación se propague por todas.

```typescript
// services/auth/src/api/routes/v1/users/controller.ts
import type { RecordParams } from '#api/schemas/types.js';
import type { UserView } from '#application/users-service/types.js';
```

```typescript
// Evitar: un controlador que habla con el almacenamiento se salta el caso de uso y su política.
import { UserRecordsHandler } from '#storage/users/index.js';
```

Relacionado: GP-03;
[SP-05](./principios-solid.md#sp-05-principio-de-inversión-de-dependencias-dip).

**Comprobación:** los archivos bajo `src/api/` no importan ningún módulo de `#storage/` ni
`#infra/`.

### GP-03 Contratos en los límites, cableado en la raíz de composición

El código que cruza un límite depende de un contrato, y las clases concretas se crean solo
en la raíz de composición o en una factoría que esta llama. Construir colaboradores dentro
del código de política oculta dependencias y bloquea la sustitución y las pruebas.

```typescript
// services/auth/src/runtime/services.ts
const tokens: ITokenService = new TokensService(config.auth, deps.signingKeys);
const sessions: ISessionsService = new SessionsService(
  deps.sessionRecordsHandler,
  tokens,
  config.auth,
  hashSecret
);
```

```typescript
// Evitar: el servicio construye su propia dependencia y su configuración.
export class SessionsService {
  private readonly tokens: ITokenService = new TokensService(loadConfig().auth, /* ... */);
}
```

Relacionado: [DP-08](./patrones-diseno.md#dp-08-composition-root),
[SP-05](./principios-solid.md#sp-05-principio-de-inversión-de-dependencias-dip).

**Comprobación:** un `new` de una clase de infraestructura (`MongoCrud`, `DatabaseProvider`,
un proveedor) aparece solo bajo `runtime/`, `infra/*/factory.ts` o `core/src/infra/`.

### GP-04 Esquemas como única fuente de verdad

Una forma en tiempo de ejecución (registro, petición, respuesta, configuración) se define
una vez como un esquema Zod; su tipo TypeScript se infiere, y el mismo esquema impulsa la
validación, la serialización y el aprovisionamiento de datasets. Los tipos paralelos
escritos a mano se desvían de lo que se valida y almacena.

```typescript
// services/auth/src/storage/users/types.ts
export type UserRecord = z.infer<typeof userRecordSchema>;

// services/auth/src/storage/users/datasetdef.ts
export const userDatasetDef: DatasetDef = {
  name: 'users',
  schema: userRecordSchema,
  indexes: [recordIndexDef /* , ... */]
};
```

```typescript
// Evitar: una segunda definición de la misma forma que nadie valida.
export type UserRecord = { username: string; email: string; roles: string[] };
```

Relacionado: [CS-02](../normas/estilo-codigo.md#cs-02-esquemas-zod-y-tipos-derivados).

**Comprobación:** un tipo que refleja los campos de un esquema se declara con `z.infer`.

### GP-05 Dependencias declaradas y bloqueadas

Cada dependencia se declara en `package.json`, el lockfile se versiona en git, y la CI y los
builds de imagen instalan con un lockfile congelado. Sin un árbol bloqueado, dos builds del
mismo commit pueden resolver dependencias distintas.

```dockerfile
# services/auth/Dockerfile
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./
RUN pnpm install --frozen-lockfile
```

```yaml
# Evitar: resuelve un árbol nuevo en cada release.
- run: pnpm install --no-frozen-lockfile
```

Relacionado: GP-06, GP-12.

**Comprobación:** `pnpm-lock.yaml` está versionado en git y cada paso de instalación en CI y
en los Dockerfiles usa `--frozen-lockfile`.

### GP-06 Toolchain fijado

El gestor de paquetes y la versión mayor de Node.js están fijados y coinciden entre el
desarrollo local, la CI y las imágenes de contenedor. Un build que pasa en un runtime y
falla en otro no es reproducible.

```dockerfile
# services/auth/Dockerfile
FROM node:22-alpine AS base
RUN corepack enable && corepack prepare pnpm@12.4.1 --activate
```

```yaml
# Evitar: una herramienta no fijada y un runtime distinto del de la imagen.
- run: npm install -g npm@latest
- uses: actions/setup-node@v4
  with:
    node-version: '24'
```

Relacionado: GP-05, GP-21.

**Comprobación:** la versión mayor de Node.js en cada `node-version` de workflow, cada línea
`FROM node:` y `engines` coinciden; las herramientas instaladas en CI llevan una versión
explícita.

### GP-07 Salvaguardas de la cadena de suministro

Los paquetes publicados llevan procedencia de build, los scripts de instalación de las
dependencias están en lista de permitidos, y las versiones nuevas de dependencia respetan
una edad mínima desde su publicación. Una dependencia comprometida o recién secuestrada no
debe ejecutar código ni enviarse en silencio.

```yaml
# services/auth/pnpm-workspace.yaml
allowBuilds:
  argon2: true
  simple-git-hooks: true
minimumReleaseAgeExclude:
  - '@salamandradevs/*'
```

```json
// core/package.json
"publishConfig": { "access": "public", "provenance": true }
```

```yaml
# Evitar: deja que cada dependencia ejecute su script de instalación.
dangerouslyAllowAllBuilds: true
```

Relacionado: GP-05; [Publicación de paquetes](../infra/empaquetado.md).

**Comprobación:** cada `package.json` publicable establece `provenance: true`; cada paquete
con un script de instalación está listado en `allowBuilds`.

### GP-08 Configuración validada, fallo rápido

La configuración se valida contra un esquema antes de que el servicio comience cualquier
trabajo, y un valor inválido o ausente detiene el arranque con un error clasificado. Un
servicio que arranca con un valor malo falla más tarde, en una petición en vivo, lejos de la
causa.

```typescript
// services/auth/src/config/config.ts
try {
  const serializedConfig: string = readFileSync(configPath, 'utf8');
  const parsedConfig: unknown = JSON.parse(serializedConfig);

  validatedConfig = serviceConfigSchema.parse(parsedConfig);
} catch (error: unknown) {
  throw new CustomError({
    name: 'CONFIG_ERROR',
    message: `Configuration could not be loaded or validated at: ${configPath}`,
    cause: error
  });
}
```

```typescript
// Evitar: una lectura sin comprobar que convierte un valor ausente en una sorpresa en runtime.
const port: number = Number(process.env.PORT);
```

Relacionado: GP-09, GP-04.

**Comprobación:** cada valor leído de la configuración o del entorno pasa por un esquema Zod
antes de usarse.

### GP-09 Secretos y credenciales solo desde el entorno

Los secretos y las credenciales (tokens, contraseñas, secretos de cliente, cadenas de
conexión con credenciales) se resuelven desde el entorno en el código que los usa. El
archivo de configuración lleva solo un marcador de posición `READ FROM ENV ...`, y el valor
resuelto nunca se vuelve a almacenar en el objeto de configuración. Los archivos de
configuración se versionan en git, se copian en imágenes y se decoran en las peticiones; un
secreto ahí se filtra por las tres vías.

```typescript
// services/auth/src/application/config/schema.ts
hashSecret: z
  .string({ error: 'is required' })
  .min(1, 'must not be empty')
  .meta({ env: 'JWT_HASH_SECRET' })

// services/auth/src/runtime/services.ts
const { hashSecret } = loadEnvVariables(refreshTokenConfigSchema);
```

```json
// Evitar: una credencial versionada en el archivo de despliegue.
"dbServerUri": "mongodb://auth:s3cret@mongo:27017"
```

Relacionado: GP-08, GP-11.

**Comprobación:** ningún archivo de configuración versionado contiene una credencial; cada
campo de esquema sensible lleva `.meta({ env })`; cada variable declarada aparece en
`.env.example` y en cada manifiesto de despliegue.

### GP-10 La configuración varía por despliegue, no por build

La misma imagen se ejecuta en cada entorno. Los valores específicos del despliegue se
suministran en el momento del despliegue, y añadir un entorno (por ejemplo `staging`) no
necesita cambio de código. Una imagen que empotra los valores de un entorno no puede
promoverse sin cambios.

```yaml
# services/auth/docker-compose.yml
volumes:
  - ./config/prod.json:/app/config/prod.json:ro
environment:
  - JWT_HASH_SECRET=${JWT_HASH_SECRET}
```

```typescript
// Evitar: un interruptor fijo de dos vías que mapea cada nombre no productivo a desarrollo.
const configFileName: string =
  process.env.NODE_ENV === 'production' ? 'prod.json' : 'devel.json';
```

Relacionado: GP-12, GP-21.

**Comprobación:** el archivo de configuración se elige por un valor del momento del
despliegue que puede nombrar cualquier entorno, y la imagen no necesita contener archivos
específicos de un entorno.

### GP-11 Los servicios de respaldo son recursos adjuntos detrás de contratos

Las bases de datos, los almacenes de claves y los proveedores de identidad se alcanzan a
través de un contrato y se localizan por configuración, de modo que uno puede reemplazarse o
reubicarse sin un cambio de código. Los backends cableados convierten una mudanza de
infraestructura en una release de código.

```typescript
// services/auth/src/runtime/database.ts
const dbProvider: DatabaseProvider = new DatabaseProvider(
  config.databaseBackend,
  config.dbServerUri,
  config.databaseName
);
```

```typescript
// Evitar: un backend y su ubicación fijados en código.
const client: MongoClient = new MongoClient('mongodb://mongo:27017');
```

Relacionado: GP-09; [DP-01](./patrones-diseno.md#dp-01-factory),
[DP-02](./patrones-diseno.md#dp-02-adapter).

**Comprobación:** cambiar un backend cambia solo la configuración; su ubicación y
credenciales siguen GP-09.

### GP-12 Artefactos inmutables, desplegados por identidad

Un build produce un artefacto inmutable por commit, un release combina ese artefacto con la
configuración de despliegue, y los despliegues y rollbacks referencian el artefacto por un
identificador inmutable (SHA del commit o tag de versión). Con un tag móvil, "qué está
corriendo" y "cómo hacer rollback" no tienen una respuesta exacta.

```yaml
# services/auth/.github/workflows/deploy.yml
tags: |
  ${{ env.IMAGE }}:${{ github.sha }}
```

```yaml
# Evitar: un tag móvil en el manifiesto de despliegue.
image: ghcr.io/salamandradevs/auth:latest
```

Relacionado: GP-05, GP-10.

**Comprobación:** el manifiesto de despliegue fija una imagen por SHA o tag de versión.

### GP-13 Las puertas de calidad se ejecutan en CI

Formato, lint, comprobación de tipos y pruebas se ejecutan en CI en cada cambio y bloquean
un release cuando fallan. Los hooks locales son una conveniencia, no la puerta: pueden
saltarse y no se ejecutan sobre el código que realmente se construye.

```json
// services/auth/package.json
"scripts": {
  "lint": "eslint ./src --ext .ts --fix",
  "check": "tsc --noEmit"
},
"simple-git-hooks": {
  "pre-push": "pnpm run format && pnpm run lint"
}
```

```yaml
# Evitar: un job de release sin ninguna puerta antes.
jobs:
  build-and-push:
    steps:
      - run: pnpm run build
```

Relacionado: GP-24.

**Comprobación:** cada repositorio tiene un workflow que ejecuta `pnpm check`, `pnpm lint` y
la suite de pruebas antes de los jobs de build o publicación.

### GP-14 Procesos sin estado

Un proceso de servicio no guarda estado que otra instancia o un reinicio necesitaría; el
estado durable o compartido vive en un servicio de respaldo. El estado en memoria rompe el
escalado horizontal, los despliegues progresivos y los reinicios.

```typescript
// services/auth/src/application/sessions-service/sessions-service.ts
return createHmac('sha256', this.refreshHashSecret)
  .update(refreshToken, 'utf8')
  .digest('base64url');
```

El HMAC del token de refresco se almacena en el registro de sesión, de modo que cualquier
instancia puede verificar una petición de refresco.

```typescript
// Evitar: estado derivado de la petición retenido solo en este proceso.
private readonly pendingStates = new Map<string, OAuthPendingState>();
```

Relacionado: GP-11, GP-15.

**Comprobación:** ningún `Map`, arreglo o variable a nivel de módulo de larga vida en un
servicio retiene estado derivado de la petición.

### GP-15 Descartabilidad: arranque rápido, parada ordenada

Un servicio arranca sin pasos manuales, libera cada recurso que abrió ante `SIGTERM`/`SIGINT`
y libera los recursos ya abiertos cuando el arranque falla a mitad de camino. Los
orquestadores detienen y mueven procesos de forma rutinaria; un proceso que filtra conexiones
o se cuelga al parar degrada la plataforma.

```typescript
// services/auth/src/runtime/runtime.ts
shutdown: async (): Promise<void> => {
  try {
    await api.close();
  } finally {
    await dbProvider.close();
  }
}

// services/auth/src/index.ts
process.once('SIGINT', shutdown);
process.once('SIGTERM', shutdown);
```

```typescript
// Evitar: sale sin drenar las peticiones ni cerrar la conexión.
process.on('SIGTERM', () => process.exit(0));
```

Relacionado: GP-14, GP-16.

**Comprobación:** cada recurso abierto en `runtime/` se cierra tanto en la vía de fallo como
en la vía de apagado.

### GP-16 Liveness y readiness

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

Relacionado: GP-15, GP-11.

**Comprobación:** existe un endpoint de readiness y el Dockerfile o el archivo compose
declara un `HEALTHCHECK` o `healthcheck`.

### GP-17 Las tareas administrativas son idempotentes y propiedad del operador

Las tareas puntuales o administrativas se ejecutan contra el mismo código y configuración que
el servicio, son idempotentes, y son propiedad de un procedimiento del operador en lugar de
scripts ad hoc. El trabajo administrativo con código o configuración distintos produce estado
que el servicio no espera.

```typescript
// services/auth/src/runtime/admin.ts
const existingUser: UserRecord | null = await deps.userRecordsHandler.readByEmail(
  bootstrapAdmin.email.trim().toLowerCase()
);

if (!existingUser) {
  await deps.users.create({ /* administrador configurado */ });

  return;
}
```

La segunda ejecución encuentra al administrador y no cambia nada. El aprovisionamiento, la
rotación y la destrucción de la clave de firma son procedimientos del operador fuera del
servicio (ROADMAP INFRA-002).

```typescript
// Evitar: una tarea de arranque que crea un administrador nuevo en cada boot.
await deps.users.create({ /* administrador configurado */ });
```

Relacionado: GP-15.

**Comprobación:** cada tarea de arranque puede ejecutarse dos veces sin cambios en la segunda
ejecución.

### GP-18 Logs como flujo de eventos

Los servicios escriben logs estructurados a stdout y dejan el enrutamiento y el
almacenamiento al entorno, con el nivel configurable por despliegue. Los archivos dentro de
los contenedores se pierden con el contenedor y son invisibles para los recolectores de
logs.

```typescript
// services/auth/src/api/api.ts
const logs: Record<string, string> = loadEnvVariables(logsConfigSchema);

const base: FastifyInstance = Fastify({
  logger: isDevelopment
    ? { level: logs.consoleLevel, transport: { target: 'pino-pretty' /* , ... */ } }
    : { level: logs.consoleLevel },
  requestIdHeader: 'x-request-id'
});
```

```typescript
// Evitar: logs escritos a un archivo dentro del contenedor.
logger: { level: 'info', file: '/app/logs/auth.log' }
```

Relacionado: GP-19, GP-20.

**Comprobación:** ningún código escribe archivos de log; cada variable de logging la pasa
cada manifiesto de despliegue.

### GP-19 Correlación de peticiones

Cada petición lleva un id de correlación que se acepta de quien llama cuando está presente,
se adjunta a cada línea de log y está disponible para el código posterior. Sin él, una
petición que falla no puede rastrearse entre líneas de log ni servicios.

```typescript
// services/auth/src/api/plugins/execution-context.ts
const executionContext: ExecutionContext = {
  authenticated: { actorId: request.sessionGrant.actorId, /* ... */ },
  request: { requestId: request.id }
};
```

Fastify rellena `request.id` a partir del header `x-request-id` configurado en GP-18.

```typescript
// Evitar: una llamada a otro servicio de la plataforma que descarta el id de correlación.
await fetch(`${profilesUrl}/v1/persons/${recordId}`);
```

Relacionado: GP-18; [DP-09](./patrones-diseno.md#dp-09-ambient-context).

**Comprobación:** las llamadas salientes a otros servicios de la plataforma reenvían
`x-request-id`.

### GP-20 Errores clasificados

Cada error que el código lanza es un `CustomError` con un `ErrorName` deliberado
([CS-04](../normas/estilo-codigo.md#cs-04-errores)), y el transporte mapea los nombres a
respuestas de problema RFC 9457 con URIs de `type` de problema estables. Los errores no
clasificados llegan a los clientes como 500 genéricos y no pueden alertarse.

```typescript
// services/auth/src/api/plugins/errors.ts
} else if (error.name === 'RESOURCE_NOT_FOUND') {
  status = 404;
  type = 'https://api.slmdevs.com/problems/resource-not-found';
```

```typescript
// Evitar: un error no clasificado que el manejador solo puede reportar como 500.
throw new Error('User not found');
```

Relacionado: GP-18.

**Comprobación:** ningún `throw new Error` o `throw new TypeError` escapa de una función sin
envolverse en un `CustomError`; cada URI de `type` de problema usa una única URL base.

### GP-21 Paridad entre desarrollo y producción

El desarrollo usa los mismos tipos de servicios de respaldo y la misma imagen de contenedor
que producción, y cada diferencia es explícita, nombrada y limitada a adaptadores
seleccionados por configuración. Un backend usado solo en producción oculta fallos hasta la
release.

```typescript
// services/auth/src/infra/keys/factory.ts
case 'jose': {
  process.emitWarning(
    'JoseSigningKeyProvider is a development provider; private keys live in a local ' +
      'encrypted file on a single host. Use a vault provider in production.',
    { code: 'DEV_SIGNING_KEY_PROVIDER' }
  );

  return new JoseSigningKeyProvider(config.provider, config.algorithm);
}
```

`docker-compose.dev.yml` construye el mismo Dockerfile y se une a la misma red `services`
con MongoDB y OpenBao.

```yaml
# Evitar: un motor de base de datos solo de desarrollo que producción nunca ejecuta.
databaseBackend: sqlite
```

Relacionado: GP-10, GP-06.

**Comprobación:** cada adaptador solo de desarrollo se anuncia a sí mismo al arrancar y se
rechaza o se advierte en la configuración de producción.

### GP-22 Mínimo privilegio

Los procesos, tokens y políticas tienen solo los permisos que su trabajo necesita, de modo
que un componente comprometido no puede escalar ni destruir datos.

```dockerfile
# services/auth/Dockerfile
RUN addgroup -S nodegroup && adduser -S nodeuser -G nodegroup
USER nodeuser
```

```hcl
# docs/en/howto/openbao/auth-signing-policy.hcl
path "transit/sign/auth-signing-*" {
  capabilities = ["update"]
}

path "secret/data/auth/signing-lifecycle" {
  capabilities = ["read"]
}
```

```hcl
# Evitar: un token de servicio que también puede reescribir o destruir sus propias claves.
path "transit/keys/*" {
  capabilities = ["create", "update", "delete"]
}
```

Relacionado: GP-09, GP-23.

**Comprobación:** ningún contenedor corre como root, y la política de cada token lista solo
las operaciones que el servicio realiza.

### GP-23 Valores por defecto de transporte seguro

Los servicios HTTP habilitan headers de seguridad, CORS explícito, límites de tasa en cada
endpoint no autenticado que crea o cambia estado, y atributos de cookie seguros. Los valores
por defecto son lo que se envía cuando nadie recuerda configurar.

```typescript
// services/auth/src/api/plugins/security.ts
await fastify.register(helmet);
await fastify.register(cors, { origin: false, credentials: false });
await fastify.register(rateLimit, { global: false });

// services/auth/src/api/routes/v1/session/routes.ts
config: { rateLimit: { max: 5, timeWindow: '1 minute' } },
```

El limitador de tasa se registra con `global: false`, de modo que cada ruta opta por él; el
inicio de sesión y el registro público lo hacen. La cookie de refresco lleva el prefijo
`__Host-` con `secure`, `httpOnly` y `sameSite`.

```typescript
// Evitar: una ruta no autenticada que almacena estado por llamada, sin límite.
fastify.get('/:provider/authorize', { schema: { /* ... */ } }, startOAuthBrowserFlow);
```

Relacionado: GP-22, GP-14.

**Comprobación:** cada ruta no autenticada que crea o cambia estado declara
`config.rateLimit`.

### GP-24 Pruebas en las costuras de contrato

Las pruebas ejercitan el comportamiento a través de los mismos contratos que inyecta la raíz
de composición, usando fakes en memoria para los servicios de respaldo en el nivel rápido y
servicios de respaldo reales en un nivel de integración. Las pruebas acopladas a los drivers
son lentas y se rompen ante cambios de infraestructura que no cambian el comportamiento.

```typescript
// services/auth/src/api/api.ts
/**
 * Constructs the auth Fastify application without binding a network listener.
 */
export async function buildApi(options: BuildApiOptions): Promise<ZodFastifyInstance> {
```

Como `buildApi` toma sus servicios como opciones y no enlaza ningún puerto, una prueba puede
inyectar fakes para `ICrud`, `ISigningKeyProvider` e `IOAuthProvider` y llamar a
`app.inject`. La estrategia se planifica en `docs/plans/tests-suite.md`.

```typescript
// Evitar: una prueba unitaria que necesita una base de datos en vivo para comprobar una regla de validación.
const crud: ICrud<UserRecord> = new MongoCrud(realMongoDb.collection('users'));
```

Relacionado: GP-13, GP-03.

**Comprobación:** cada paquete tiene un comando de pruebas que se ejecuta en CI.

## Beneficios y contrapartidas

Aplicadas en conjunto, estas prácticas le dan a la plataforma:

- **Reproducibilidad**: el mismo commit construye el mismo artefacto (GP-05, GP-06, GP-12).
- **Configuración segura**: los ajustes inválidos detienen el arranque y los secretos quedan
  fuera de los archivos y las imágenes (GP-08, GP-09, GP-10).
- **Infraestructura reemplazable**: los servicios de respaldo se intercambian por
  configuración (GP-11, GP-21).
- **Operabilidad**: los procesos arrancan, paran y reportan salud de forma predecible, y los
  fallos se clasifican y se rastrean (GP-14 a GP-20).
- **Daño contenido**: el mínimo privilegio y los valores por defecto seguros limitan lo que
  un compromiso puede hacer (GP-22, GP-23).

Las prácticas tienen costes. Los toolchains fijados necesitan actualizaciones deliberadas,
las puertas de CI ralentizan ligeramente cada cambio, y las comprobaciones de readiness
añaden trabajo a cada servicio de respaldo. Aplíquelas en proporción: una práctica marcada
como "debería" puede saltarse con una razón documentada.

Se aceptan dos desviaciones deliberadas de la Twelve-Factor App:

- **Archivo de configuración más secretos del entorno**: la configuración vive en un archivo
  JSON validado, no enteramente en variables de entorno. Solo los secretos y las credenciales
  vienen del entorno (GP-09). El archivo da un documento revisado y validado por esquema por
  despliegue; el coste es que debe mantenerse libre de secretos.
- **Tareas de arranque en lugar de procesos puntuales**: el aprovisionamiento de datasets y
  la reconciliación del administrador bootstrap se ejecutan en cada arranque. Ambas son
  idempotentes (GP-17), así que ejecutarlas en cada arranque es seguro y elimina un paso
  manual.

## Relación con la Twelve-Factor App

Cómo se mapea cada factor a las prácticas anteriores:

- **I Codebase**: [GP-01](#gp-01-un-repositorio-por-paquete-o-servicio).
- **II Dependencies**: [GP-05](#gp-05-dependencias-declaradas-y-bloqueadas),
  [GP-06](#gp-06-toolchain-fijado), [GP-07](#gp-07-salvaguardas-de-la-cadena-de-suministro).
- **III Config**: [GP-08](#gp-08-configuración-validada-fallo-rápido),
  [GP-09](#gp-09-secretos-y-credenciales-solo-desde-el-entorno),
  [GP-10](#gp-10-la-configuración-varía-por-despliegue-no-por-build).
- **IV Backing services**:
  [GP-11](#gp-11-los-servicios-de-respaldo-son-recursos-adjuntos-detrás-de-contratos).
- **V Build, release, run**: [GP-12](#gp-12-artefactos-inmutables-desplegados-por-identidad),
  [GP-13](#gp-13-las-puertas-de-calidad-se-ejecutan-en-ci).
- **VI Processes**: [GP-14](#gp-14-procesos-sin-estado).
- **VII Port binding**: lo cumple Fastify escuchando en `http.host` y `http.port`; sin
  práctica aparte.
- **VIII Concurrency**: lo cumplen los procesos sin estado (GP-14); sin práctica aparte hasta
  que un servicio necesite tipos de proceso trabajador.
- **IX Disposability**: [GP-15](#gp-15-descartabilidad-arranque-rápido-parada-ordenada),
  [GP-16](#gp-16-liveness-y-readiness).
- **X Dev/prod parity**: [GP-21](#gp-21-paridad-entre-desarrollo-y-producción).
- **XI Logs**: [GP-18](#gp-18-logs-como-flujo-de-eventos).
- **XII Admin processes**:
  [GP-17](#gp-17-las-tareas-administrativas-son-idempotentes-y-propiedad-del-operador).

Dónde la plataforma va más allá de los factores:

- **Arquitectura dentro del proceso**: capas, contratos y una raíz de composición (GP-02,
  GP-03), que los factores dejan abierto.
- **Definiciones compartidas**: esquemas como única fuente de verdad (GP-04) y errores
  clasificados (GP-20).
- **Seguridad operativa**: salvaguardas de la cadena de suministro (GP-07), readiness
  (GP-16), ids de correlación (GP-19), valores por defecto de seguridad (GP-22, GP-23) y
  pruebas (GP-24).

## Referencias

### Fuentes fundamentales

- Wiggins, Adam. [The Twelve-Factor App](https://12factor.net/). Heroku, 2011.
- Humble, Jez, y David Farley. *Continuous Delivery: Reliable Software Releases through
  Build, Test, and Deployment Automation*. Addison-Wesley, 2010.
- Nottingham, Mark, Erik Wilde, y Sanjay Dalal.
  [RFC 9457: Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457). IETF,
  2023.

### Fuentes complementarias

- npm Docs.
  [Generating provenance statements](https://docs.npmjs.com/generating-provenance-statements).
- SLSA. [Supply-chain Levels for Software Artifacts](https://slsa.dev/).
</content>
