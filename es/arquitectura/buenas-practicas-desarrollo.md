---
title: Buenas prácticas de desarrollo
lastUpdate: 2026-11-18
lang: es
category: arquitectura
status: published
---

## Introducción

Una **buena práctica** es una regla de ingeniería que previene una clase conocida de fallo:
builds que no pueden reproducirse, secretos que se filtran, procesos que pierden estado o
incidentes que nadie puede rastrear. Esta guía cubre las prácticas que mantienen los
paquetes y servicios de Salamandra Devs modulares, reproducibles, desplegables, observables
y seguros por defecto. Aplican a `core/`, `tools/` y a todos los servicios bajo `services/`.

Cada práctica lleva un ID estable (`GP-NN`) para su citación; los IDs siguen
[IDs de documento](../normas/convenciones-nombres.md#ids-de-documento). Dónde el código aún
no cumple una práctica se registra en la
[Revisión de Arquitectura](../../architecture-review.md), no en esta guía. Cómo los
servicios Fastify aplican estas prácticas (seguridad de transporte, endpoints de salud,
correlación de peticiones, respuestas de problema) vive en la guía de la casa
[Construir servicios](../normas/construir-servicios.md) (`BS`).

IDs retirados, nunca reutilizados: GP-06 fusionado en GP-05, GP-11 en GP-03 y GP-22 en
GP-25; las reglas de servicio de GP-16, GP-19 y GP-23 se trasladaron a BS-02, BS-03 y BS-01,
y las prácticas que las respaldan son parte de GP-15, GP-18 y GP-25.

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
hipotéticos: muestran lo que rompería la práctica, no código que exista.

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

### GP-03 Contratos, cableado en la raíz de composición y backends configurados

Una clase que se consume a través de un límite se consume mediante un contrato, aunque tenga
una sola implementación, y las clases concretas se crean solo en la raíz de composición o en
una factoría que esta llama. Un límite es una costura entre las secciones de
[GP-02](#gp-02-estructura-de-servicio-en-capas) (transporte, aplicación, almacenamiento,
adaptadores externos), entre dos módulos de casos de uso, o entre un servicio y un paquete
compartido. Quien consume depende solo de lo que llama, y la raíz de composición decide qué
recibe, así que la clase detrás del contrato puede reemplazarse por una segunda
implementación o por un fake de prueba sin tocar a quien consume. Construir colaboradores
dentro del código de política oculta dependencias y bloquea ambas cosas. Los servicios de
respaldo (bases de datos, almacenes de claves, proveedores de identidad) también se localizan
por configuración, de modo que reemplazar o reubicar uno es un cambio de configuración, no
una release de código.

- **Necesita un contrato**: el transporte que consume un caso de uso (`ISessionsService` en
  `api/types.ts`), un caso de uso que consume otro (`ITokensService` en `SessionsService`), y
  cualquier consumidor de un adaptador (`ISigningKeyProvider`, `IOAuthProvider`).
- **No lo necesita**: las clases usadas solo dentro de su propio módulo o solo en `runtime/`
  (`DatabaseProvider`, `DatabaseAdapter`), y los manejadores de registros, cuyo punto de
  sustitución es el `ICrud` que tienen debajo ([DP-03](./patrones-diseno.md#dp-03-bridge)).

```typescript
// services/auth/src/runtime/services.ts
const tokens: ITokensService = new TokensService(config.auth, deps.signingKeys);
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
  private readonly tokens: ITokensService = new TokensService(loadConfig().auth, /* ... */);
}

// Evitar: un backend y su ubicación fijados en código.
const client: MongoClient = new MongoClient('mongodb://mongo:27017');
```

Relacionado: GP-09; [DP-01](./patrones-diseno.md#dp-01-factory),
[DP-08](./patrones-diseno.md#dp-08-composition-root),
[SP-05](./principios-solid.md#sp-05-principio-de-inversión-de-dependencias-dip).

**Comprobación:** un `new` de una clase de infraestructura (`MongoCrud`, `DatabaseProvider`,
un proveedor) aparece solo bajo `runtime/`, `infra/*/factory.ts` o `core/src/infra/`; ningún
tipo bajo `src/api/` ni en el constructor de un caso de uso nombra una clase concreta de
aplicación o de adaptador; cambiar o reubicar un servicio de respaldo cambia solo la
configuración.

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

### GP-05 Dependencias bloqueadas y toolchain fijado

Cada dependencia se declara y se bloquea, y las instalaciones usan el lockfile congelado;
pnpm y la versión mayor de Node están fijados por igual en desarrollo, CI e imágenes, o dos
builds de un mismo commit pueden diferir.

```dockerfile
# services/auth/Dockerfile
FROM node:22-alpine
RUN corepack enable && corepack prepare pnpm@12.4.1 --activate
RUN pnpm install --frozen-lockfile
```

```yaml
# Evitar: una instalación sin bloqueo y un runtime distinto del de la imagen.
- run: pnpm install --no-frozen-lockfile
- uses: actions/setup-node@v4
  with: { node-version: '24' }
```

Relacionado: GP-12, GP-21.

**Comprobación:** `pnpm-lock.yaml` está versionado y las instalaciones usan
`--frozen-lockfile`; `node-version`, `FROM node:` y `engines` coinciden; las herramientas de
CI llevan versiones explícitas.

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
conexión con credenciales) se resuelven desde el entorno en el código que los usa. El archivo
de configuración lleva solo un marcador de posición `READ FROM ENV ...`, y el valor resuelto
nunca se vuelve a almacenar en el objeto de configuración. Los archivos de configuración se
versionan en git, se copian en imágenes y se decoran en las peticiones; un secreto ahí se
filtra por las tres vías.

El archivo se valida (GP-08) antes de resolver los valores del entorno, así que un marcador
de posición debe pasar el formato de su campo: un `baoAddr` validado por `z.url()` usa
`http://READ-FROM-ENV-BAO_ADDR`, no `READ FROM ENV BAO_ADDR`.

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

Relacionado: GP-08, GP-03.

**Comprobación:** ningún archivo de configuración versionado contiene una credencial; cada
campo de esquema sensible lleva `.meta({ env })`; cada marcador de posición en un archivo de
configuración versionado parsea bajo el esquema de su campo; cada variable declarada aparece
en `.env.example` y en cada manifiesto de despliegue.

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

### GP-12 Artefactos inmutables, desplegados por identidad

Un build produce un artefacto inmutable por commit, y los despliegues y rollbacks referencian
el artefacto por un identificador inmutable (SHA del commit o tag de versión). Con un tag
móvil, "qué está corriendo" y "cómo hacer rollback" no tienen una respuesta exacta.

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

Formato, lint, comprobación de tipos y pruebas se ejecutan en CI en cada cambio y bloquean un
release cuando fallan. Los hooks locales son una conveniencia, no la puerta: pueden saltarse
y no se ejecutan sobre el código que realmente se construye.

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

Relacionado: GP-03, GP-15.

**Comprobación:** ningún `Map`, arreglo o variable a nivel de módulo de larga vida en un
servicio retiene estado derivado de la petición.

### GP-15 Descartabilidad: arranque rápido, parada ordenada

Un servicio arranca sin pasos manuales, libera cada recurso que abrió ante `SIGTERM`/`SIGINT`
y libera los recursos ya abiertos cuando el arranque falla a mitad de camino. Mientras se
ejecuta, le dice a su orquestador si está vivo y si puede servir, de modo que el tráfico
alcance solo una instancia cuyos servicios de respaldo requeridos son alcanzables. Los
orquestadores arrancan, detienen y mueven procesos de forma rutinaria; un proceso que filtra
conexiones, se cuelga al parar o acepta tráfico que no puede servir degrada la plataforma.

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

Relacionado: GP-14, GP-03;
[BS-02](../normas/construir-servicios.md#bs-02-liveness-y-readiness).

**Comprobación:** cada recurso abierto en `runtime/` se cierra tanto en la vía de fallo como
en la vía de apagado, y cada proceso desplegado declara una comprobación de salud que usa su
orquestador.

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
almacenamiento al entorno, con el nivel configurable por despliegue. Cada línea de log lleva
el id de correlación de la unidad de trabajo a la que pertenece (una petición, un job, un
mensaje); el id se acepta de quien llama cuando está presente y se reenvía en cada llamada a
otro servicio de la plataforma. Los archivos dentro de los contenedores se pierden con el
contenedor y son invisibles para los recolectores de logs, y las líneas de log sin id de
correlación no pueden rastrearse hasta un fallo entre líneas o servicios.

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

// Evitar: una llamada a otro servicio de la plataforma que descarta el id de correlación.
await fetch(`${profilesUrl}/v1/persons/${recordId}`);
```

Relacionado: GP-20; [DP-09](./patrones-diseno.md#dp-09-ambient-context);
[BS-03](../normas/construir-servicios.md#bs-03-correlación-de-peticiones).

**Comprobación:** ningún código escribe archivos de log; cada variable de logging la pasa
cada manifiesto de despliegue; las llamadas salientes a otros servicios de la plataforma
reenvían el id de correlación.

### GP-20 Errores clasificados

Cada error que el código lanza es un `CustomError` con un `ErrorName` deliberado
([CS-04](../normas/estilo-codigo.md#cs-04-errores)), y un fallo de proveedor o driver se
convierte en el adaptador que lo captura, conservando el original como la causa. Cada límite
que expone errores a un cliente mapea esos nombres a una respuesta estable y documentada. Los
errores no clasificados llegan a los clientes como fallos genéricos y no pueden alertarse.

```typescript
// core/src/infra/database/mongo/mongo-crud.ts
} catch (error: unknown) {
  if (isMongoDupKeyError(error)) {
    throw new CustomError({
      name: 'DB_DUPLICATE_KEY_ERROR',
      message: errMessages.DB_DUPLICATE_KEY,
      cause: error
    });
  }

  throw new CustomError({
    name: 'DB_BACKEND_ERROR',
    message: 'Failed to create MongoDB record',
    cause: error
  });
}
```

```typescript
// Evitar: un error no clasificado que el límite solo puede reportar como un fallo genérico.
throw new Error('User not found');
```

Relacionado: GP-18; [BS-04](../normas/construir-servicios.md#bs-04-respuestas-de-problema).

**Comprobación:** ningún `throw new Error` ni `throw new TypeError` escapa de una función sin
envolverse en un `CustomError`.

### GP-21 Paridad entre desarrollo y producción

El desarrollo usa los mismos tipos de servicios de respaldo que producción, y cada diferencia
es explícita, nombrada y limitada a adaptadores seleccionados por configuración. Un backend
usado solo en producción oculta fallos hasta la release.

```typescript
// Evitar: un adaptador solo de desarrollo que arranca en silencio, de modo que una mala
// configuración lo envía a producción sin que nadie lo note.
case 'dev-insecure': {
  return new InsecureDevKeyProvider(config.provider);
}
```

`docker-compose.dev.yml` construye el mismo Dockerfile y se une a la misma red `services` con
MongoDB y OpenBao. `JoseKeysProvider` no es un adaptador solo de desarrollo: es un proveedor
de producción para despliegues pequeños (un host o réplicas que comparten un archivo de
clave), así que no anuncia nada.

```yaml
# Evitar: un motor de base de datos solo de desarrollo que producción nunca ejecuta.
databaseBackend: sqlite
```

Relacionado: GP-10, GP-05.

**Comprobación:** cada adaptador solo de desarrollo se anuncia a sí mismo al arrancar y se
rechaza o se advierte en la configuración de producción; un adaptador apto para producción,
como `JoseKeysProvider`, no lleva tal advertencia.

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
`app.inject`. La estrategia se planifica en `docs/plans/261029-tests-suite-plan.md`.

Los contratos son lo que abarata esos fakes. TypeScript compara de forma nominal una clase
con miembros `private`, así que un objeto plano no puede sustituir a `SessionsService`, pero
sí a `ISessionsService`. Una prueba de transporte simula los casos de uso con fakes y
comprueba el enrutado, los hooks y el mapeo de errores; una prueba de caso de uso simula los
contratos que consume y comprueba la política; ninguna necesita un servicio de respaldo.

```typescript
// Evitar: una prueba unitaria que necesita una base de datos en vivo para comprobar una regla de validación.
const crud: ICrud<UserRecord> = new MongoCrud(realMongoDb.collection('users'));
```

Relacionado: GP-13, GP-03.

**Comprobación:** ninguna prueba de nivel rápido abre una conexión a un servicio de respaldo;
los fakes entran a través de los mismos contratos que inyecta la raíz de composición.

### GP-25 Seguro por defecto y mínimo privilegio

Cada componente nuevo que expone o concede algo (una ruta HTTP, un comando de CLI, un
consumidor de mensajes, un token, una cookie, un contenedor, una política de acceso) se crea
en su estado más restrictivo, y mientras se ejecuta tiene solo los permisos que su trabajo
necesita, de modo que un componente comprometido no puede escalar ni destruir datos. Aflojar
cualquiera de los dos es una declaración explícita y nombrada en el código o la
configuración, y cuando falta una declaración requerida el componente se niega a arrancar o a
construirse en lugar de recaer en abierto. Un estado seguro que hay que recordar se pierde la
primera vez que alguien lo olvida.

```typescript
// http/src/plugins/security.ts
fastify.addHook('onRoute', (routeOptions: RouteOptions): void => {
  const onRequest: unknown[] = [routeOptions.onRequest ?? []].flat();
  const tiers: number = onRequest.filter(
    (hook: unknown): boolean => hook === publicRateLimit || hook === exemptRateLimit
  ).length;

  if (tiers !== 1 || routeOptions.config?.rateLimit !== undefined) {
    throw new CustomError({
      name: 'CONFIG_ERROR',
      message: `Route ${String(routeOptions.method)} ${url} needs one rate-limit tier`
    });
  }
});
```

```dockerfile
# services/auth/Dockerfile
USER nodeuser
```

Una ruta nueva no puede arrancar sin límite, y el nivel exento es la única salida, revisable.

```typescript
// Evitar: un componente que queda abierto a menos que alguien recuerde cerrarlo.
const requireAuth: boolean = options.requireAuth ?? false;
```

```hcl
# Evitar: un token de servicio que también puede reescribir o destruir sus propias claves.
path "transit/keys/*" {
  capabilities = ["create", "update", "delete"]
}
```

Relacionado: GP-08, GP-09, GP-20;
[BS-01](../normas/construir-servicios.md#bs-01-valores-seguros-de-transporte-por-defecto).

**Comprobación:** cada relajación de un valor por defecto de seguridad o exposición es una
declaración explícita, y ningún componente recae en abierto cuando falta; ningún contenedor
corre como root, y la política de cada token lista solo las operaciones que el servicio
realiza.

## Beneficios y contrapartidas

Aplicadas en conjunto, estas prácticas hacen los builds reproducibles (GP-05, GP-12), la
configuración segura (GP-08 a GP-10), la infraestructura reemplazable (GP-03, GP-21), los
procesos operables y rastreables (GP-14 a GP-20) y el daño contenido (GP-25).

Las prácticas tienen costes. Los toolchains fijados necesitan actualizaciones deliberadas,
las puertas de CI ralentizan ligeramente cada cambio, y las comprobaciones de readiness
añaden trabajo a cada servicio de respaldo. Aplíquelas en proporción: una práctica marcada
como "debería" puede saltarse con una razón documentada.

Se aceptan dos desviaciones deliberadas de la Twelve-Factor App:

- **Archivo de configuración más secretos del entorno**: un archivo JSON validado por
  despliegue contiene la configuración, y solo los secretos vienen del entorno (GP-09); el
  coste es que el archivo debe mantenerse libre de secretos.
- **Tareas de arranque en lugar de procesos puntuales**: el aprovisionamiento de datasets y
  la reconciliación del administrador bootstrap se ejecutan en cada arranque, lo cual es
  seguro porque ambas son idempotentes (GP-17).

## Relación con la Twelve-Factor App

- **I Codebase**: GP-01. **II Dependencies**: GP-05, GP-07. **III Config**: GP-08 a GP-10.
- **IV Backing services**: GP-03. **V Build, release, run**: GP-12, GP-13.
- **VI Processes** y **VIII Concurrency**: GP-14; todavía sin tipos de proceso trabajador.
- **VII Port binding**: Fastify escucha en `http.host` y `http.port`; sin práctica aparte.
- **IX Disposability**: GP-15. **X Dev/prod parity**: GP-21. **XI Logs**: GP-18.
- **XII Admin processes**: GP-17.
- **Más allá de los factores**: capas y contratos (GP-02, GP-03), esquemas (GP-04), errores
  clasificados (GP-20), cadena de suministro (GP-07), pruebas (GP-24) y valores por defecto
  seguros (GP-25).

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
