---
title: Patrones de diseño
lastUpdate: 2026-10-25
lang: es
category: arquitectura
status: published
---

## Introducción

Un **patrón de diseño** es una solución reutilizable con nombre para un problema de diseño
recurrente. Su nombre da al equipo un vocabulario compartido: decir "esto es un Adapter" le
dice a quien lee qué hace la clase, qué oculta y qué no debe hacer. Esta guía cubre los
patrones que el código de Salamandra Devs usa hoy, agrupados por las categorías clásicas.

Los patrones son vocabulario y herramientas, no fines. Un patrón gana su lugar solo cuando
resuelve un problema que el código realmente tiene; véase
[Beneficios y contrapartidas](#beneficios-y-contrapartidas). Los principios a los que
sirven los patrones se definen en [Principios SOLID](./principios-solid.md).

Cada patrón lleva un ID estable (`DP-NN`) para su citación; los IDs siguen
[IDs de documento](../normas/convenciones-nombres.md#ids-de-documento).

## Terminología usada en esta guía

- **Intención**: el problema que resuelve un patrón, enunciado con independencia de
  cualquier lenguaje.
- **Participante**: un rol en el patrón, como el contexto, la estrategia o el adaptado.
- **Contrato**: una `interface` o tipo función que describe lo que necesita quien llama
  (`ICrud`, `ISigningKeyProvider`, `SchemaConverter`). También llamado puerto.
- **Implementación**: una clase que cumple un contrato con una tecnología específica
  (`MongoCrud`, `OpenBaoSigningKeyProvider`). También llamada adaptador cuando envuelve una
  API de proveedor.
- **Raíz de composición**: el código de arranque que construye los objetos concretos y los
  inyecta (`services/auth/src/runtime/`).
- **Registro**: datos persistidos con la forma de `baseRecordSchema`. **Manejador de
  registros**; la clase que almacena un tipo de registro (`UserRecordsHandler`).
- **Decoración de Fastify**: un valor adjuntado a la instancia de Fastify con `decorate`. A
  pesar del nombre, no es el patrón Decorator del GoF.

## Ejemplos de la plataforma

Los ejemplos son extractos recortados de `core/`, `tools/` y `services/auth/`. Cada uno
comienza con un comentario que nombra su archivo fuente; los comentarios de documentación y
los miembros no relacionados se eliminan por brevedad. Los contraejemplos se marcan con
`// Evitar` y son hipotéticos: muestran lo que rompería el patrón, no código que exista.

La mayoría de los ejemplos se construyen sobre un contrato de `core`, la superficie CRUD
neutral al backend:

```typescript
// core/src/infra/database/contracts.ts
export interface ICrud<T> {
  create(record: T): Promise<T>;
  read(criteria: Partial<T>): Promise<T | null>;
  update(criteria: Partial<T>, changes: Partial<T>): Promise<T | null>;
  delete(criteria: Partial<T>): Promise<T | null>;
  find(criteria: Partial<T>, options: FindOptions<T>): Promise<FindResult<T>>;
}
```

### DP-01 Factory

Una Factory crea objetos sin que quien llama nombre la clase concreta. El catálogo del GoF
define Factory Method (las subclases deciden el producto) y Abstract Factory (familias de
productos relacionados). La plataforma usa una forma más simple y común: una **factoría
parametrizada**, una función o constructor que elige la clase concreta a partir de un valor.

`createSigningKeyProvider` construye el proveedor de clave de firma nombrado en la
configuración:

```typescript
// services/auth/src/infra/keys/factory.ts
export function createSigningKeyProvider(config: SigningKeysConfig): ISigningKeyProvider {
  switch (config.provider.name) {
    case 'openbao': {
      return new OpenBaoSigningKeyProvider(
        config.provider,
        config.algorithm,
        config.jwks.cacheMaxAgeSecs
      );
    }

    case 'jose': {
      return new JoseSigningKeyProvider(config.provider, config.algorithm);
    }

    default: {
      throw new CustomError({
        name: 'CONFIG_ERROR',
        message: `Unknown signing-key provider '${JSON.stringify(config.provider)}'`
      });
    }
  }
}
```

`DatabaseProvider` (clase de conexión por `backend`) y `DatabaseAdapter.createCrud`
(adaptador CRUD por el backend conectado) son factorías del mismo tipo. Ambos hacen `switch`
sobre una unión discriminada y terminan en una comprobación `never`, de modo que un backend
nuevo sin un `case` no compila.

El patrón se rompe cuando quien llama elige la clase por su cuenta:

```typescript
// Evitar: cada quien llama repite la selección e importa cada proveedor.
const provider: ISigningKeyProvider =
  config.provider.name === 'openbao'
    ? new OpenBaoSigningKeyProvider(config.provider, config.algorithm, 300)
    : new JoseSigningKeyProvider(config.provider, config.algorithm);
```

Principio relacionado: **OCP**, con una contrapartida: un producto nuevo significa un `case`
nuevo, de modo que la factoría misma está cerrada solo a quienes llaman, no a sus propias
ediciones.

### DP-02 Adapter

Un Adapter convierte la interfaz de una clase existente en la interfaz que esperan sus
clientes. Es la forma habitual de poner una librería de proveedor detrás de un contrato de
la plataforma.

`MongoCrud` adapta el driver de MongoDB a `ICrud<T>`. También elimina lo específico de
MongoDB, como el `_id` propiedad del almacenamiento, de modo que quienes llaman reciben
registros neutrales al backend:

```typescript
// core/src/infra/database/mongo/mongo-crud.ts
export class MongoCrud<T extends GeneralRecord> implements ICrud<T> {
  constructor(private readonly collection: Collection<T>) {}

  async read(criteria: Partial<T>): Promise<T | null> {
    assertCriteria(criteria);
    const document: WithId<T> | null = await this.collection.findOne(
      criteria as Filter<T>
    );

    if (!document) {
      return null;
    }

    return this.returnRecord(document);
  }

  private returnRecord(document: WithId<T>): T {
    const { _id: mongoId, ...record } = document;
    void mongoId;

    return record as unknown as T;
  }
}
```

`MySqlCrud` adapta `mysql2` al mismo contrato, y `MongoDataset` / `MySqlDataset` adaptan la
API de administración de cada driver a `IDataset`. En `services/auth`,
`GoogleOAuthProvider` y `GithubOAuthProvider` adaptan dos APIs de proveedor a
`IOAuthProvider`.

El patrón se rompe cuando los tipos del proveedor se filtran a través del contrato:

```typescript
// Evitar: quienes llaman ahora dependen de los tipos de MongoDB y de `_id`.
interface ICrud<T> {
  read(filter: Filter<T>): Promise<WithId<T> | null>;
}
```

Principios relacionados: **DIP** (el contrato pertenece a quien llama, el detalle del driver
queda detrás de él) y **LSP** (`MongoCrud` y `MySqlCrud` deben ser sustituibles entre sí).

### DP-03 Bridge

Un Bridge separa una abstracción de su implementación para que ambas puedan variar de forma
independiente, en lugar de multiplicar subclases para cada combinación.

En la plataforma, las dos dimensiones son el tipo de registro y el backend de base de datos.
La abstracción (`RecordsHandler` y sus subclases) mantiene una referencia a la
implementación (`ICrud<T>`), recibida a través de su constructor:

```typescript
// services/auth/src/runtime/database.ts
return {
  dbProvider,
  userRecordsHandler: new UserRecordsHandler(
    dbAdapter.createCrud<UserRecord>('users')
  ),
  sessionRecordsHandler: new SessionRecordsHandler(
    dbAdapter.createCrud<SessionRecord>('sessions')
  )
};
```

Dos manejadores y dos backends dan cuatro combinaciones con cuatro clases, no cuatro
subclases. Añadir un tercer backend añade una implementación de `ICrud` y no cambia ningún
manejador.

El patrón se rompe cuando el backend queda fijado por herencia:

```typescript
// Evitar: una subclase por tipo de registro y backend.
class MongoUserRecordsHandler extends MongoRecordsHandler<UserRecord> {}
class MySqlUserRecordsHandler extends MySqlRecordsHandler<UserRecord> {}
```

Principios relacionados: **OCP** (los backends nuevos extienden sin editar los manejadores) y
**DIP**.

### DP-04 Facade

Una Facade da una interfaz simple a un subsistema de clases que colaboran. Los clientes usan
la fachada y no necesitan conocer las partes del subsistema.

`DatabaseAdapter` es la fachada sobre conexiones, aprovisionadores, convertidores de esquema
y adaptadores CRUD. La raíz de composición necesita solo dos llamadas:

```typescript
// services/auth/src/runtime/database.ts
const dbHandle: DatabaseHandle = await dbProvider.connect();
const dbAdapter: DatabaseAdapter = new DatabaseAdapter(dbHandle);

await dbAdapter.provision(collectDatasetDefs());

// ... luego un dbAdapter.createCrud<T>(name) por cada manejador de registros
```

Detrás de esas llamadas, `DatabaseAdapter` elige `MongoDataset` o `MySqlDataset`, el
convertidor de esquema correspondiente, y `MongoCrud` o `MySqlCrud`.

El patrón se rompe cuando los clientes alcanzan más allá de la fachada hacia sus partes:

```typescript
// Evitar: la raíz de composición ahora conoce cada clase de driver y convertidor.
const provisioner = new DatasetProvisioner(new MongoDataset(db), zodToMongoSchema);
const userCrud = new MongoCrud<UserRecord>(db.collection('users'));
```

Principio relacionado: **SRP** (la selección de backend vive en una sola clase).

### DP-05 Strategy

Una Strategy define una familia de algoritmos intercambiables detrás de un contrato, y el
contexto recibe el elegido desde fuera en lugar de ramificar sobre una bandera.

`DatasetProvisioner` ejecuta un único flujo de aprovisionamiento para todos los backends.
Los pasos específicos del backend son dos estrategias que recibe: un `IDataset` y una
función `SchemaConverter`.

```typescript
// core/src/infra/database/dataset-provisioner.ts
export class DatasetProvisioner<
  TDef extends DatasetDef<TIndex>,
  TSchema,
  TIndex extends { name: string }
> implements IDatasetProvisioner<TDef> {
  constructor(
    private readonly dataset: IDataset<TSchema, TIndex>,
    private readonly convertSchema: SchemaConverter<TSchema>
  ) {}

  async provision(def: TDef): Promise<void> {
    const convertedSchema: TSchema = this.convertSchema(def.schema);

    await this.ensureDataset(def.name, convertedSchema);
    await this.ensureIndexes(def.name, def.indexes);
  }
}

// core/src/infra/database/database-adapter.ts
new DatasetProvisioner<DatasetDef, MongoJSONSchema, IndexDef>(
  new MongoDataset(db),
  zodToMongoSchema
);
```

En `services/auth`, `TokensService` recibe cualquier `ISigningKeyProvider`, y `OAuthService`
recibe una lista de estrategias `IOAuthProvider` y selecciona una por petición según el
nombre.

El patrón se rompe cuando el contexto ramifica sobre la variante:

```typescript
// Evitar: cada backend nuevo reabre el flujo.
async provision(def: DatasetDef, backend: DatabaseBackend): Promise<void> {
  if (backend === 'mongodb') {
    // Pasos específicos de MongoDB ...
  } else {
    // Pasos específicos de MySQL ...
  }
}
```

Principios relacionados: **OCP** y **DIP**. La Strategy difiere del Bridge en la intención:
la Strategy intercambia un algoritmo dentro de un flujo, el Bridge separa toda una
abstracción de su implementación.

### DP-06 Chain of Responsibility

Una Chain of Responsibility pasa una petición por una secuencia ordenada de manejadores.
Cada manejador o bien atiende su preocupación y pasa la petición adelante, o bien detiene la
cadena.

Los arreglos `preHandler` de Fastify son cadenas. Los compositores compartidos los
construyen a partir de hooks decorados, y cada ruta protegida declara su cadena:

```typescript
// services/auth/src/api/routes/v1/hooks/authenticate.ts
export function protectionHooks(fastify: FastifyInstance): PreHandlerHooks {
  return [fastify.authenticateRequest, fastify.bindExecutionContext];
}

// services/auth/src/api/routes/v1/hooks/authorize.ts
export function adminProtectionHooks(fastify: FastifyInstance): PreHandlerHooks {
  return [...protectionHooks(fastify), fastify.requireAdminRole];
}

// services/auth/src/api/routes/v1/users/routes.ts
fastify.delete<{ Params: RecordParams }>(
  '/:recordId',
  { preHandler: adminProtectionHooks(fastify), schema: { /* ... */ } },
  deleteUser
);
```

`authenticateRequest` detiene la cadena cuando falta el token bearer
(`AUTHENTICATION_FAILED`) o no se verifica, de modo que `requireAdminRole` y el controlador
nunca se ejecutan para esa petición. Los controladores pueden por tanto asumir las
precondiciones de la cadena
([Estilo de código](../normas/estilo-codigo.md#cs-11-controladores)).

El patrón se rompe cuando un controlador repite o reordena las comprobaciones:

```typescript
// Evitar: el controlador vuelve a autenticar y duplica la política de los hooks.
async function deleteUser(request: FastifyRequest, reply: FastifyReply): Promise<void> {
  const grant: SessionGrant = await request.server.services.tokens.verify(/* ... */);

  if (!grant.roles.includes('admin')) {
    // ...
  }
}
```

Principio relacionado: **SRP** (cada hook posee una precondición).

### DP-07 Repository

Un Repository media entre los casos de uso y la persistencia a través de una API similar a
una colección, de modo que los casos de uso piden registros por significado ("el usuario con
este username") en lugar de construir consultas. Martin Fowler lo cataloga en *Patterns of
Enterprise Application Architecture*.

En la plataforma, `RecordsHandler<T>` es el repositorio base. Añade lo que todo registro
necesita (alcance de tenant, identidad, sellos de auditoría) sobre `ICrud<T>`, y una
subclase añade búsquedas para un tipo de registro:

```typescript
// core/src/storage/records-handler.ts
export abstract class RecordsHandler<T extends BaseRecord> {
  protected constructor(
    protected readonly crud: ICrud<T>,
    private readonly handlerName: T['handlerName']
  ) {}

  read(recordId: string): Promise<T | null> {
    const { tenantId } = getExecutionCtx().authenticated;

    return this.crud.read({ recordId, tenantId } as Partial<T>);
  }
}

// services/auth/src/storage/users/user-records-handler.ts
export class UserRecordsHandler extends RecordsHandler<UserRecord> {
  constructor(crud: ICrud<UserRecord>) {
    super(crud, 'users');
  }

  readByUsername(username: string): Promise<UserRecord | null> {
    const { tenantId } = getExecutionCtx().authenticated;

    return this.crud.read({ username, tenantId });
  }
}
```

`UsersService` le pide a `userRecordsHandler.readByUsername(...)` y nunca ve una colección,
una tabla ni un filtro de tenant.

El patrón se rompe cuando un caso de uso construye las consultas de persistencia por su
cuenta:

```typescript
// Evitar: el caso de uso posee la consulta y debe recordar el filtro de tenant.
const user: UserRecord | null = await crud.read({ username });
```

Esa llamada olvida `tenantId`, que es exactamente el tipo de error que el repositorio existe
para hacer imposible. Principios relacionados: **SRP** (las reglas de persistencia viven en
una sola clase) y **DIP** (los casos de uso dependen del manejador, no de un driver).

### DP-08 Composition Root

Una Composition Root es el único lugar, tan cerca del punto de entrada como sea posible,
donde la aplicación construye sus objetos concretos y los inyecta. Mark Seemann lo nombró
como el ancla de la inyección de dependencias.

`createRuntime` es la raíz de composición del servicio auth. Cada paso `bootstrap*`
construye un tema, y el orden es explícito:

```typescript
// services/auth/src/runtime/runtime.ts
export async function createRuntime(config: ServiceConfig): Promise<ServiceRuntime> {
  const { dbProvider, userRecordsHandler, sessionRecordsHandler }: StorageComposition =
    await bootstrapDatabase(config);

  try {
    const signingKeys: ISigningKeyProvider = await bootstrapSigningKeys(config);
    const oauth: OAuthComposition = bootstrapOAuth(config, userRecordsHandler);
    const services: ServicesComposition = bootstrapServices(config, {
      userRecordsHandler,
      sessionRecordsHandler,
      signingKeys
    });

    await ensureBootstrapAdmin(config, { userRecordsHandler, users: services.users });
    // ... buildApi(...) y return { start, shutdown }
  } catch (error: unknown) {
    await dbProvider.close();

    throw error;
  }
}
```

Todos los demás módulos reciben sus colaboradores a través de parámetros del constructor, lo
que mantiene las dependencias visibles y reemplazables.

El patrón se rompe cuando el código de política construye sus propios colaboradores:

```typescript
// Evitar: el servicio oculta una dependencia concreta y su configuración.
export class SessionsService {
  private readonly tokens: ITokenService = new TokensService(loadConfig().auth, /* ... */);
}
```

Principios relacionados: **DIP** e inyección de dependencias, según se distinguen en
[Principios SOLID](./principios-solid.md#sp-05-principio-de-inversión-de-dependencias-dip).

### DP-09 Ambient Context

Un Ambient Context hace que los datos con alcance de petición estén disponibles para
cualquier código que se ejecute dentro de esa petición, sin pasarlos por cada llamada.

`core` enlaza un `ExecutionContext` (tenant, actor, id de petición) al alcance asíncrono con
el `AsyncLocalStorage` de Node. El transporte lo enlaza una vez por petición;
`RecordsHandler` lo lee en cada operación:

```typescript
// core/src/utils/context.ts
const executionContextStorage: AsyncLocalStorage<ExecutionContext> =
  new AsyncLocalStorage<ExecutionContext>();

export function runWithExecutionCtx<T>(context: ExecutionContext, callback: () => T): T {
  return executionContextStorage.run(context, callback);
}

export function getExecutionCtx(): ExecutionContext {
  const context: ExecutionContext | undefined = executionContextStorage.getStore();

  if (!context) {
    throw new CustomError({
      name: 'RESOURCE_NOT_FOUND',
      message: 'Execution context is not available'
    });
  }

  return context;
}

// services/auth/src/api/plugins/execution-context.ts
const executionContext: ExecutionContext = {
  authenticated: {
    actorId: request.sessionGrant.actorId,
    tenantId: request.sessionGrant.tenantId
  },
  request: { requestId: request.id }
};

runWithExecutionCtx(executionContext, done);
```

El beneficio es el aislamiento estructural de tenant: ningún quien llama puede pasar el
tenant equivocado, porque ninguno pasa un tenant en absoluto.

El coste es una dependencia oculta. Seemann y van Deursen listan el Ambient Context entre
los antipatrones de inyección de dependencias por esa razón. La plataforma limita el coste de
tres maneras: el contexto contiene datos, no servicios; solo lo establecen los hooks de
transporte y el paso de arranque; y leerlo fuera de un alcance enlazado falla ruidosamente
en lugar de devolver un valor por defecto.

El patrón se rompe cuando el contexto se usa para transportar servicios o se establece desde
lo profundo del código de política:

```typescript
// Evitar: un localizador de servicios disfrazado, y un contexto establecido lejos del transporte.
runWithExecutionCtx({ ...context, users: new UsersService(/* ... */) }, next);
```

### DP-10 Single Flight

Single Flight (también llamado coalescencia de peticiones) hace que quienes llaman de forma
concurrente al mismo trabajo asíncrono compartan una sola ejecución. El
`golang.org/x/sync/singleflight` de Go popularizó el nombre.

`tools` provee `singleFlight`, y el proveedor de clave de firma de OpenBao lo usa para que
una ráfaga de peticiones de token dispare un solo refresco del estado de clave, no uno por
petición:

```typescript
// tools/src/shared/concurrency.ts
export function singleFlight<T>(operation: () => Promise<T>): () => Promise<T> {
  let inFlight: Promise<T> | undefined;

  return async (): Promise<T> => {
    const current: Promise<T> = (inFlight ??= operation());

    try {
      return await current;
    } finally {
      if (inFlight === current) {
        inFlight = undefined;
      }
    }
  };
}

// services/auth/src/infra/keys/openbao/provider.ts
private readonly synchronize = singleFlight(() => this.refreshSnapshot());
```

La promesa almacenada se limpia cuando se resuelve, de modo que los resultados nunca se
cachean más allá de la ventana en vuelo, y un rechazo alcanza a cada quien llama unido y se
reintenta en la siguiente llamada.

El patrón se rompe cuando cada quien llama inicia su propio refresco:

```typescript
// Evitar: N peticiones concurrentes envían N lecturas a OpenBao.
async getCurrentSigner(): Promise<SigningKeyHandle> {
  await this.refreshSnapshot();
  // ...
}
```

## Beneficios y contrapartidas

Aplicados donde encajan, estos patrones le dan a la plataforma:

- **Infraestructura reemplazable**: Adapter, Bridge, Strategy y Factory permiten que
  MongoDB y MySQL, OpenBao y jose, y Google y GitHub se intercambien por configuración.
- **Un lugar por decisión**: Facade y Factory mantienen la selección de backend en una sola
  clase o función; Composition Root mantiene el cableado en `runtime/`.
- **Seguridad por construcción**: Repository más Ambient Context hacen imposible expresar el
  acceso entre tenants; Chain of Responsibility garantiza las precondiciones antes de que
  los controladores se ejecuten.
- **Vocabulario compartido**: quien revisa y lee "Strategy" sabe qué buscar y qué estaría
  mal.

Los patrones tienen costes. Cada uno añade indirección, y un patrón aplicado sin un problema
que lo justifique hace que el código simple sea más difícil de seguir. Dos contrapartidas en
el código actual se aceptan a propósito:

- Las factorías parametrizadas (`createSigningKeyProvider`, `DatabaseProvider`,
  `DatabaseAdapter`) deben editarse por cada variante nueva. La comprobación de
  exhaustividad `never` hace que esa edición sea imposible de olvidar.
- El Ambient Context oculta una dependencia, como se describe en su sección.

Patrones que la plataforma no usa, y por qué:

- **Singleton**: una sola instancia compartida por proceso se logra con la raíz de
  composición creándola una vez, sin un punto de acceso global.
- **Abstract Factory**: todavía ninguna familia de productos relacionados debe cambiar en
  conjunto.
- **Template Method**: las subclases de `RecordsHandler` añaden búsquedas pero no
  sobrescriben ningún paso, así que no hay un algoritmo con pasos gancho.
- **Decorator (GoF)**: el comportamiento transversal se adjunta mediante hooks y plugins de
  Fastify en lugar de objetos wrapper.
- **Observer**: todavía no hay eventos de dominio. Los eventos y un outbox son propuestas
  (ROADMAP AUTH-017).
- **Active Record**: los registros son datos planos validados por Zod; la persistencia queda
  en los manejadores de registros, como exige Repository.
- **Command, State, Mediator, Visitor**: ningún problema actual necesita peticiones
  encoladas o reversibles, objetos de comportamiento dependientes del estado, un coordinador
  central ni doble despacho.

Introduzca uno de estos solo cuando el código muestre el problema que resuelve, y documente
la decisión aquí cuando lo haga.

## Referencias

### Fuentes fundamentales

- Gamma, Erich, Richard Helm, Ralph Johnson, y John Vlissides. *Design Patterns: Elements
  of Reusable Object-Oriented Software*. Addison-Wesley, 1994.
- Fowler, Martin. *Patterns of Enterprise Application Architecture*. Addison-Wesley, 2002.
  [Repository](https://martinfowler.com/eaaCatalog/repository.html).
- Seemann, Mark. [Composition Root](https://blog.ploeh.dk/2011/07/28/CompositionRoot/).
- Seemann, Mark, y Steven van Deursen. *Dependency Injection Principles, Practices, and
  Patterns*. Manning, 2019.

### Fuentes complementarias

- Refactoring Guru.
  [Design Patterns catalog](https://refactoring.guru/design-patterns/catalog).
- Proyecto Go. [singleflight](https://pkg.go.dev/golang.org/x/sync/singleflight).
</content>
