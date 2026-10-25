---
title: Design Patterns
lastUpdate: 2026-10-25
lang: en
category: architecture
status: published
---

## Introduction

A **design pattern** is a named, reusable solution to a recurring design problem. Its name
gives a team shared vocabulary: saying "this is an Adapter" tells a reader what the class
does, what it hides, and what it must not do. This guide covers the patterns that the
Salamandra Devs code uses today, grouped by the classic categories, additional patterns can 
be found at [Refactoring Guru](https://refactoring.guru/design-patterns/):

- **Creational**: provide various object creation mechanisms, which increase flexibility
  and reuse of existing code.
  - DP-01 Factory
- **Structural**: explain how to assemble objects and classes into larger structures, while
  keeping these structures flexible and efficient.
  - DP-02 Adapter
  - DP-03 Bridge
  - DP-04 Facade
- **Behavioral**: are concerned with algorithms and the assignment of responsibilities
  between objects.
  - DP-05 Strategy
  - DP-06 Chain of Responsibility
- **Application architecture**: organize an application beyond individual objects, deciding
  where persistence is hidden, where objects are built and wired, how request-scoped data
  reaches the code that needs it, and how concurrent work is shared.
  - DP-07 Repository
  - DP-08 Composition Root
  - DP-09 Ambient Context
  - DP-10 Single Flight

Patterns are vocabulary and tools, not goals. A pattern earns its place only when it solves
a problem the code actually has; see [Benefits and trade-offs](#benefits-and-trade-offs).
The principles the patterns serve are defined in [SOLID Principles](./solid-principles.md).

Each pattern carries a stable ID (`DP-NN`) for citation; IDs follow
[Document IDs](../house/naming-conventions.md#document-ids).

## Terminology used in this guide

- **Intent**: the problem a pattern solves, stated independently of any language.
- **Participant**: a role in the pattern, such as the context, the strategy, or the
  adaptee.
- **Contract**: an `interface` or function type that describes what a caller needs
  (`ICrud`, `ISigningKeyProvider`, `SchemaConverter`). Also called a port.
- **Implementation**: a class that fulfills a contract with a specific technology
  (`MongoCrud`, `OpenBaoSigningKeyProvider`). Also called an adapter when it wraps a
  vendor API.
- **Composition root**: the startup code that builds concrete objects and injects them
  (`services/auth/src/runtime/`).
- **Record**: persisted data shaped by `baseRecordSchema`. **Records handler**; the class
  that stores one record type (`UserRecordsHandler`).
- **Fastify decoration**: a value attached to the Fastify instance with `decorate`. Despite
  the name, it is not the GoF Decorator pattern.

## Platform Examples

The examples are trimmed excerpts from `core/`, `tools/`, and `services/auth/`. Each one
starts with a comment naming its source file; doc comments and unrelated members are
removed for brevity. Counter-examples are marked `// Avoid` and are hypothetical: they show
what would break the pattern, not code that exists.

Most examples build on one contract from `core`, the backend-neutral CRUD surface:

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

A Factory creates objects without the caller naming the concrete class. The GoF catalog
defines Factory Method (subclasses decide the product) and Abstract Factory (families of
related products). The platform uses a simpler, common form: a **parameterized factory**,
one function or constructor that picks the concrete class from a value.

`createSigningKeyProvider` builds the signing-key provider named in configuration:

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

`DatabaseProvider` (connection class by `backend`) and `DatabaseAdapter.createCrud` (CRUD
adapter by connected backend) are factories of the same kind. Both `switch` on a
discriminated union and end in a `never` check, so a new backend without a `case` fails to
compile.

The pattern breaks when callers choose the class themselves:

```typescript
// Avoid: every caller repeats the selection and imports every provider.
const provider: ISigningKeyProvider =
  config.provider.name === 'openbao'
    ? new OpenBaoSigningKeyProvider(config.provider, config.algorithm, 300)
    : new JoseSigningKeyProvider(config.provider, config.algorithm);
```

Related principle: **OCP**, with a trade-off: a new product means a new `case`, so the
factory itself is closed only to callers, not to its own edits.

### DP-02 Adapter

An Adapter converts the interface of an existing class into the interface its clients
expect. It is the usual way to put a vendor library behind a platform contract.

`MongoCrud` adapts the MongoDB driver to `ICrud<T>`. It also removes what is specific to
MongoDB, such as the storage-owned `_id`, so callers receive backend-neutral records:

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

`MySqlCrud` adapts `mysql2` to the same contract, and `MongoDataset` / `MySqlDataset` adapt
each driver's administration API to `IDataset`. In `services/auth`,
`GoogleOAuthProvider` and `GithubOAuthProvider` adapt two provider APIs to
`IOAuthProvider`.

The pattern breaks when vendor types leak through the contract:

```typescript
// Avoid: callers now depend on MongoDB types and on `_id`.
interface ICrud<T> {
  read(filter: Filter<T>): Promise<WithId<T> | null>;
}
```

Related principles: **DIP** (the contract belongs to the caller, the driver detail stays
behind it) and **LSP** (`MongoCrud` and `MySqlCrud` must be substitutable for each other).

### DP-03 Bridge

A Bridge separates an abstraction from its implementation so the two can vary
independently, instead of multiplying subclasses for every combination.

In the platform, the two dimensions are the record type and the database backend. The
abstraction (`RecordsHandler` and its subclasses) holds a reference to the implementation
(`ICrud<T>`), received through its constructor:

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

Two handlers and two backends give four combinations with four classes, not four
subclasses. Adding a third backend adds one `ICrud` implementation and changes no handler.

The pattern breaks when the backend is fixed by inheritance:

```typescript
// Avoid: one subclass per record type and backend.
class MongoUserRecordsHandler extends MongoRecordsHandler<UserRecord> {}
class MySqlUserRecordsHandler extends MySqlRecordsHandler<UserRecord> {}
```

Related principles: **OCP** (new backends extend without editing handlers) and **DIP**.

### DP-04 Facade

A Facade gives a simple interface to a subsystem of collaborating classes. Clients use the
facade and do not need to know the subsystem's parts.

`DatabaseAdapter` is the facade over connections, provisioners, schema converters, and
CRUD adapters. The composition root needs only two calls:

```typescript
// services/auth/src/runtime/database.ts
const dbHandle: DatabaseHandle = await dbProvider.connect();
const dbAdapter: DatabaseAdapter = new DatabaseAdapter(dbHandle);

await dbAdapter.provision(collectDatasetDefs());

// ... then one dbAdapter.createCrud<T>(name) per records handler
```

Behind those calls, `DatabaseAdapter` picks `MongoDataset` or `MySqlDataset`, the matching
schema converter, and `MongoCrud` or `MySqlCrud`.

The pattern breaks when clients reach past the facade into its parts:

```typescript
// Avoid: the composition root now knows every driver class and converter.
const provisioner = new DatasetProvisioner(new MongoDataset(db), zodToMongoSchema);
const userCrud = new MongoCrud<UserRecord>(db.collection('users'));
```

Related principle: **SRP** (backend selection lives in one class).

### DP-05 Strategy

A Strategy defines a family of interchangeable algorithms behind one contract, and the
context receives the chosen one from outside instead of branching on a flag.

`DatasetProvisioner` runs one provisioning workflow for every backend. The backend-specific
steps are two strategies it receives: an `IDataset` and a `SchemaConverter` function.

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

In `services/auth`, `TokensService` receives any `ISigningKeyProvider`, and `OAuthService`
receives a list of `IOAuthProvider` strategies and selects one per request by name.

The pattern breaks when the context branches on the variant:

```typescript
// Avoid: every new backend reopens the workflow.
async provision(def: DatasetDef, backend: DatabaseBackend): Promise<void> {
  if (backend === 'mongodb') {
    // MongoDB-specific steps ...
  } else {
    // MySQL-specific steps ...
  }
}
```

Related principles: **OCP** and **DIP**. Strategy differs from Bridge in intent: Strategy
swaps an algorithm inside one workflow, Bridge splits a whole abstraction from its
implementation.

### DP-06 Chain of Responsibility

A Chain of Responsibility passes a request along ordered handlers. Each handler either
handles its concern and passes the request on, or stops the chain.

Fastify `preHandler` arrays are chains. Shared composers build them from decorated hooks,
and every protected route declares its chain:

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

`authenticateRequest` stops the chain when the bearer token is missing
(`AUTHENTICATION_FAILED`) or fails verification, so `requireAdminRole` and the controller
never run for that request. Controllers can therefore assume the chain's preconditions
([Code Style](../house/code-style.md#cs-11-controllers)).

The pattern breaks when a controller repeats or reorders the checks:

```typescript
// Avoid: the controller re-authenticates and duplicates hook policy.
async function deleteUser(request: FastifyRequest, reply: FastifyReply): Promise<void> {
  const grant: SessionGrant = await request.server.services.tokens.verify(/* ... */);

  if (!grant.roles.includes('admin')) {
    // ...
  }
}
```

Related principle: **SRP** (each hook owns one precondition).

### DP-07 Repository

A Repository mediates between use cases and persistence through a collection-like API, so
use cases ask for records by meaning ("the user with this username") instead of building
queries. Martin Fowler catalogs it in *Patterns of Enterprise Application Architecture*.

In the platform, `RecordsHandler<T>` is the base repository. It adds what every record
needs (tenant scope, identity, audit stamps) on top of `ICrud<T>`, and a subclass adds
lookups for one record type:

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

`UsersService` asks `userRecordsHandler.readByUsername(...)` and never sees a collection,
a table, or a tenant filter.

The pattern breaks when a use case builds persistence queries itself:

```typescript
// Avoid: the use case owns the query and must remember the tenant filter.
const user: UserRecord | null = await crud.read({ username });
```

That call forgets `tenantId`, which is exactly the kind of mistake the repository exists
to make impossible. Related principles: **SRP** (persistence rules live in one class) and
**DIP** (use cases depend on the handler, not on a driver).

### DP-08 Composition Root

A Composition Root is the single place, as close to the entry point as possible, where the
application builds its concrete objects and injects them. Mark Seemann named it as the
anchor of dependency injection.

`createRuntime` is the auth service's composition root. Each `bootstrap*` step builds one
topic, and the order is explicit:

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
    // ... buildApi(...) and return { start, shutdown }
  } catch (error: unknown) {
    await dbProvider.close();

    throw error;
  }
}
```

Every other module receives its collaborators through constructor parameters, which keeps
dependencies visible and replaceable.

The pattern breaks when policy code builds its own collaborators:

```typescript
// Avoid: the service hides a concrete dependency and its configuration.
export class SessionsService {
  private readonly tokens: ITokenService = new TokensService(loadConfig().auth, /* ... */);
}
```

Related principles: **DIP** and dependency injection, as distinguished in
[SOLID Principles](./solid-principles.md#sp-05-dependency-inversion-principle-dip).

### DP-09 Ambient Context

An Ambient Context makes request-scoped data available to any code running within that
request, without passing it through every call.

`core` binds an `ExecutionContext` (tenant, actor, request id) to the async scope with
Node's `AsyncLocalStorage`. Transport binds it once per request; `RecordsHandler` reads it
on every operation:

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

The benefit is structural tenant isolation: no caller can pass the wrong tenant, because
no caller passes a tenant at all.

The cost is a hidden dependency. Seemann and van Deursen list Ambient Context among the
dependency injection anti-patterns for that reason. The platform limits the cost in three
ways: the context holds data, not services; it is set only by transport hooks and the
bootstrap step; and reading it outside a bound scope fails loudly instead of returning a
default.

The pattern breaks when the context is used to carry services or is set from deep inside
policy code:

```typescript
// Avoid: a service locator in disguise, and a context set far from transport.
runWithExecutionCtx({ ...context, users: new UsersService(/* ... */) }, next);
```

### DP-10 Single Flight

Single Flight (also called request coalescing) makes concurrent callers of the same async
work share one execution. Go's `golang.org/x/sync/singleflight` popularized the name.

`tools` provides `singleFlight`, and the OpenBao signing-key provider uses it so that a
burst of token requests triggers one key-state refresh, not one per request:

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

The stored promise is cleared when it settles, so results are never cached past the
in-flight window, and a rejection reaches every joined caller and is retried on the next
call.

The pattern breaks when each caller starts its own refresh:

```typescript
// Avoid: N concurrent requests send N reads to OpenBao.
async getCurrentSigner(): Promise<SigningKeyHandle> {
  await this.refreshSnapshot();
  // ...
}
```

## Benefits and trade-offs

Applied where they fit, these patterns give the platform:

- **Replaceable infrastructure**: Adapter, Bridge, Strategy, and Factory let MongoDB and
  MySQL, OpenBao and jose, and Google and GitHub swap by configuration.
- **One place per decision**: Facade and Factory keep backend selection in one class or
  function; Composition Root keeps wiring in `runtime/`.
- **Safety by construction**: Repository plus Ambient Context make cross-tenant access
  impossible to express; Chain of Responsibility guarantees preconditions before
  controllers run.
- **Shared vocabulary**: a reviewer who reads "Strategy" knows what to look for and what
  would be wrong.

Patterns have costs. Each adds indirection, and a pattern applied without a matching
problem makes simple code harder to follow. Two trade-offs in the current code are
accepted on purpose:

- The parameterized factories (`createSigningKeyProvider`, `DatabaseProvider`,
  `DatabaseAdapter`) must be edited for each new variant. The exhaustiveness `never` check
  makes that edit impossible to forget.
- Ambient Context hides a dependency, as described in its section.

Patterns the platform does not use, and why:

- **Singleton**: one shared instance per process is achieved by the composition root
  creating it once, without a global access point.
- **Abstract Factory**: no family of related products must change together yet.
- **Template Method**: `RecordsHandler` subclasses add lookups but override no steps, so
  there is no algorithm with hook steps.
- **Decorator (GoF)**: cross-cutting behavior is attached through Fastify hooks and
  plugins instead of wrapper objects.
- **Observer**: there are no domain events yet. Events and an outbox are proposals
  (ROADMAP AUTH-017).
- **Active Record**: records are plain data validated by Zod; persistence stays in
  records handlers, as Repository requires.
- **Command, State, Mediator, Visitor**: no current problem needs queued or undoable
  requests, state-dependent behavior objects, a central coordinator, or double dispatch.

Introduce one of these only when the code shows the problem it solves, and document the
decision here when you do.

## References

### Foundational sources

- Gamma, Erich, Richard Helm, Ralph Johnson, and John Vlissides. *Design Patterns: Elements
  of Reusable Object-Oriented Software*. Addison-Wesley, 1994.
- Fowler, Martin. *Patterns of Enterprise Application Architecture*. Addison-Wesley, 2002.
  [Repository](https://martinfowler.com/eaaCatalog/repository.html).
- Seemann, Mark. [Composition Root](https://blog.ploeh.dk/2011/07/28/CompositionRoot/).
- Seemann, Mark, and Steven van Deursen. *Dependency Injection Principles, Practices, and
  Patterns*. Manning, 2019.

### Supplementary sources

- Refactoring Guru.
  [Design Patterns catalog](https://refactoring.guru/design-patterns/catalog).
- Go project. [singleflight](https://pkg.go.dev/golang.org/x/sync/singleflight).
