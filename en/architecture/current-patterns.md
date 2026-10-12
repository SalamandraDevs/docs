---
title: Current Design Patterns in the Mongo Entity Stack
owner: OdairTrujillo
lastUpdate: 2026-08-17
category: architecture
lang: en
---

# Current Design Patterns in the Mongo Entity Stack

This document maps the shared database layer (`tools/src/node/database/mongo/`) and its consumers (e.g. `authservice`, `identity`) to established design patterns and software design principles. **Matching Patterns** documents structures present in current code, including well-implemented and broken uses plus a **Principles** block. **Not Matching Patterns** explains absent structures and gives brief refactor sketches without claiming they already exist. See [solid-principles.md](./solid-principles.md) for principle definitions.

SOLID (**SRP**, **OCP**, **LSP**, **ISP**, **DIP**) definitions, terminology, and examples: [solid-principles.md](./solid-principles.md). Principle names below use those acronyms.


## Layer overview

**Call flow:** `Users` / `Persons` → `MongoEntity` → `MongoCrud` → `MongoDataset` → MongoDB driver

**Interfaces:** `MongoCrud` implements `ICrud` · `MongoDataset` implements `IDataset`

- **Infrastructure** — `MongoDataset` → connection, collections, indexes
- **Data access** — `MongoCrud` → generic CRUD over any collection
- **Entity service** — `MongoEntity` → schema, init, default entity lifecycle
- **Domain** — `Users`, `Persons`, … → business rules, DTO mapping, validation

## Matching Patterns

### 1. Repository

**Source:** [Repository — Martin Fowler (EAA Catalog)](https://martinfowler.com/eaaCatalog/repository.html)  
*Not in the [Guru GoF catalog](https://refactoring.guru/design-patterns/catalog); Fowler defines it as a collection-like API that mediates between domain and data mapping layers, hiding persistence details.*

**Intent (Fowler):** A Repository encapsulates the logic required to access data sources. It centralizes common data access functionality, provides better testability, and keeps domain objects ignorant of database APIs.

**How it maps here:**

- `MongoCrud` — low-level persistence primitives (create/read/update/delete by key).
- `MongoEntity` — entity-scoped repository: owns collection name, schema, indexes, uniqueness checks, and default `entityId` lookups.
- `Users` / `Persons` — domain-facing repository with business-specific methods.

#### Well implemented — `MongoEntity` as entity repository

`MongoEntity` exposes a stable, domain-shaped API (`create`, `read`, `update`, `delete`) and hides collection setup and CRUD delegation:

```typescript
// tools/src/node/database/mongo/mongoentity.ts
export class MongoEntity<T extends MongoRecord<any>> implements IEntity<T> {
  #crud: MongoCrud<T>;
  #collectionName: string;

  public async create(input: Omit<T, keyof ExcludeMongoEntity>): Promise<T | null> {
    // uniqueness checks, stamp _id/entityId/timestamps ...
    const record = { _id: new ObjectId(), entityId: uuidv4(), /* ... */ ...input } as T;
    return await this.#crud.create(this.#collectionName, 'entityId', record);
  }

  public async read(id: string): Promise<T | null> {
    return await this.#crud.read(this.#collectionName, 'entityId', id);
  }
}
```

#### Well implemented — `Persons` delegates persistence to the base repository

`Persons` builds a domain record, then uses `super.create()` / `super.read()` / `super.update()` instead of talking to Mongo directly for the standard lifecycle:

```typescript
// identity/src/entities/persons/persons.ts
const created: Person | null = await super.create(person);
return created ? personZodSchema.parse(created) : null;

// ...

const person: Person | null = await super.read(userId);

// ...

const updated: Person | null = await super.update(userId, updates);
```

#### Broken — `Users` bypasses the repository abstraction

`Users` duplicates `#crud` and `#collectionName` (already owned by `MongoEntity`), skips `super.create()`, and calls `MongoCrud` directly. It also hardcodes the collection name `'users'` instead of using `this.#collectionName` from config:

```typescript
// authservice/src/users/users.ts
export class Users extends MongoEntity<UserAuth> {
  #crud: MongoCrud<UserAuth>;       // duplicate of parent's #crud
  #collectionName: string;           // duplicate of parent's #collectionName

  public override async create(data: UserCreateDto): Promise<UserAuth | null> {
    const UserAuth: UserAuth = { /* builds full record manually */ };

    // skips super.create() — misses base uniqueness checks and standard stamping
    return await this.#crud.create(this.#collectionName, 'username', UserAuth);
  }

  public override async read(id?: string, username?: string): Promise<UserAuth | null> {
    if (username) {
      return await this.#crud.read('users', 'username', username); // hardcoded collection
    }
    if (id) {
      return await this.#crud.read('users', 'entityId', id);       // hardcoded collection
    }
    return null;
  }
}
```

**Impact:** Two persistence paths for entities (`MongoEntity` vs raw `MongoCrud`), inconsistent uniqueness handling, and config drift if `collectionName` changes.

**Principles:**

- **Meets:** **SRP** — `MongoEntity` keeps collection lifecycle behind one public API when used as intended (one kind of reason to change per module). `Persons` delegates persistence to that API instead of mixing domain rules with driver calls.
- **Breaks:** **SRP** — `Users` duplicates `#crud` and `#collectionName`, so persistence wiring and domain policy change for different reasons in one class. **DIP** — `Users` and `MongoEntity` depend on concrete `MongoCrud` instead of an injected port; `Persons` still extends infrastructure (`MongoEntity`) rather than depending on an abstraction such as `PersonRepository`. **DI** — repeated `new MongoCrud(options.db)` in base and subclasses performs hidden concrete construction outside the composition root. **LSP** — `Users.create` takes `UserCreateDto` and `Users.read` widens `IEntity.read(id)` to `read(id?, username?)`, strengthening preconditions and changing the behavioral contract callers expect from `IEntity<UserAuth>`.

---

### 2. Template Method

**Source:** [Template Method — Refactoring Guru](https://refactoring.guru/design-patterns/template-method)

**Intent (Guru):** Defines the skeleton of an algorithm in a superclass but lets subclasses override specific steps without changing the algorithm's structure. Common steps live in the base class; variable steps are overridden in subclasses.

**Applicability (Guru):** Use when several classes share almost identical algorithms with minor differences, or when clients should extend particular steps but not the whole algorithm.

**How it maps here:**

The pattern lives in **`MongoEntity`** that is the superclass that defines the algorithm skeleton; entity subclasses (`Persons`, `Users`, …) are supposed to customize steps — not replace the whole flow.

- **Template method (base):** `MongoEntity.create()` — uniqueness check → stamp metadata → persist.
- **Intended customization:** subclasses override specific steps or prepare input, then delegate to `super.*`.
- **Invariant structure:** persist goes through the same `entityId` key and collection from options when the template is respected.

#### Well implemented — `MongoEntity` defines the base algorithm

`MongoEntity` owns the fixed skeleton. Subclasses are meant to plug into it, not reimplement persistence:

```typescript
// tools/src/node/database/mongo/mongoentity.ts
public async create(input: Omit<T, keyof ExcludeMongoEntity>): Promise<T | null> {
  // uniqueness checks on defined indexes ...
  const record = { _id: new ObjectId(), entityId: uuidv4(), /* timestamps */ ...input } as T;
  return await this.#crud.create(this.#collectionName, 'entityId', record);
}
```

This is the template method. Domain-specific classes extend `MongoEntity`, not `Person`.

#### Partial implemented — `Persons` delegates to the template but does not override it

`Persons` does not override `create` / `read` / `update`. It adds domain methods (`createPerson`, `readById`) that build or validate data **outside** the template, then call `super.create()` / `super.read()`. That respects the base algorithm, but it is not textbook Template Method — there are no protected hook methods inside `MongoEntity` for subclasses to override. `Persons` is a well-behaved **consumer** of the template, not a full structural example of the pattern.

```typescript
// identity/src/entities/persons/persons.ts
public async createPerson(input: CreatePersonInput): Promise<Person | null> {
  const person: Person = personZodSchema.parse({ /* domain fields */ });
  const created: Person | null = await super.create(person);
  return created ? personZodSchema.parse(created) : null;
}
```

`readById` follows the same idea: `super.read()` then domain filtering and validation. For alternate keys, `readByEmail` reaches into `#crud` directly — a smaller leak, same duplicate-wiring smell as `Users`:

```typescript
// identity/src/entities/persons/persons.ts
public async readByEmail(email: string): Promise<Person | null> {
  const person: Person | null = await this.#crud.read(
    this.#collectionName,
    'email',
    email.toLowerCase()
  );
  // ...
}
```

#### Broken — `Users.create` replaces the template instead of extending it

`Users` overrides `create` but reimplements persistence from scratch, changing the uniqueness key from `entityId` to `username` and skipping the base class steps:

```typescript
// authservice/src/users/users.ts
public override async create(data: UserCreateDto): Promise<UserAuth | null> {
  const UserAuth: UserAuth = {
    _id: new ObjectId(),
    entityId: uuidv4(),
    // manually stamps all metadata — duplicates MongoEntity.create logic
    password: await hashData(data.password),
    // ...
  };

  // does NOT call super.create(person)
  return await this.#crud.create(this.#collectionName, 'username', UserAuth);
}
```

**What Guru warns about:** subclasses should override steps, not the template algorithm itself. Here the entire create flow is forked.

**Principles:**

- **Meets:** **OCP** — `MongoEntity` exposes a stable extension point: new entity classes can plug in through delegation without editing the base create/read/update algorithm. **SRP** — invariant steps stay in one module when subclasses call `super.*`.
- **Breaks:** **OCP** — `Users.create` modifies the established template by reimplementing persistence instead of extending through hooks. **SRP** — `Users.create` duplicates metadata stamping owned by the base algorithm. **DIP** / **DI** — duplicate `#crud` in `Persons` / `Users` repeats low-level construction inside high-level subclasses instead of receiving an injected port from the composition root.

---

### 3. Bridge

**Source:** [Bridge — Refactoring Guru](https://refactoring.guru/design-patterns/bridge)

**Intent (Guru):** Split a large class into two hierarchies — **abstraction** and **implementation** — that can vary independently. The abstraction delegates real work to the implementation via composition instead of multiplying subclasses (e.g. `BlueCircle`, `RedSquare`).

**Applicability (Guru):** Use when you need to extend a class in several orthogonal dimensions (e.g. entity type × database backend), or when you may switch implementations at runtime.

**How it maps here:**

- `IEntity` / `MongoEntity` → `MongoCrud`
- `ICrud` → `MongoDataset` + Mongo driver
- `IDataset` → `MongoDataset`

#### Well implemented — interface split from Mongo implementation

```typescript
// tools/src/node/base/database.ts
export interface ICrud<T> {
  create<K extends keyof T>(datasetName: string, key: K, data: T): Promise<T | null>;
  read<K extends keyof T>(datasetName: string, key: K, value: T[K]): Promise<T | null>;
  update<K extends keyof T>(datasetName: string, key: K, value: T[K], updateData: Partial<T>): Promise<T | null>;
  // ...
}

// tools/src/node/database/mongo/mongocrud.ts
export class MongoCrud<T extends WithId<Document>>
  extends MongoDataset
  implements ICrud<T>
{
  public async create<K extends keyof T>(datasetName: string, key: K, data: T): Promise<T | null> {
    const collection: Collection = super.getDataset(datasetName);
    // Mongo-specific insertOne / findOne
  }
}
```

`MongoEntity` (abstraction) holds a `MongoCrud` reference (implementation) and never imports Mongo driver types in its public API.

#### Broken — domain layer couples to concrete `MongoCrud`

Subclasses import and instantiate `MongoCrud` directly, weakening the Bridge at the domain boundary:

```typescript
// authservice/src/users/users.ts
import { MongoCrud, MongoEntity } from '@mss/libs/node';

export class Users extends MongoEntity<UserAuth> {
  #crud: MongoCrud<UserAuth>;

  constructor(options: EntityOptions, userIndexes: EntityIndex<UserAuth>[], ...) {
    super(options, userIndexes);
    this.#crud = new MongoCrud(options.db);  // second implementation instance
  }
}
```

A cleaner Bridge would expose protected lookup helpers on `MongoEntity` (e.g. `readByField(key, value)`) so subclasses never touch `MongoCrud`.

**Principles:**

- **Meets:** **DIP** — `ICrud` and `IDataset` are abstractions; Mongo implementations sit on the detail side of the boundary.
- **Breaks:** **DIP** — `MongoEntity` constructs `MongoCrud`, and domain subclasses construct it again, so high-level policy depends on low-level concrete types instead of injected abstractions. **DI** — each `new MongoCrud(options.db)` performs wiring that belongs at the composition root and multiplies client stacks. **ISP** — `MongoCrud extends MongoDataset`, so CRUD consumers inherit dataset/admin methods they do not use, and implementers carry a broader surface than the port requires.

---

### 4. Adapter

**Source:** [Adapter — Refactoring Guru](https://refactoring.guru/design-patterns/adapter)

**Intent (Guru):** Convert the interface of a class into another interface clients expect. Lets classes work together that could not otherwise because of incompatible interfaces.

**Relation to Bridge (Guru):** Adapter is often applied after the fact to integrate existing code; Bridge is usually designed up front. Both use composition and delegation.

**How it maps here:** `MongoCrud` adapts the MongoDB driver API (`collection.findOne`, `findOneAndUpdate`, `insertOne`) to the project's `ICrud<T>` contract.

#### Well implemented — driver calls wrapped in `ICrud` shape

```typescript
// tools/src/node/database/mongo/mongocrud.ts
public async read<K extends keyof T>(
  datasetName: string,
  key: K,
  value: T[K]
): Promise<T | null> {
  const collection: Collection = super.getDataset(datasetName);
  const result = await collection.findOne({ [key]: value });
  return result !== null ? (result as T) : null;
}
```

Compare-and-set updates are also adapted cleanly:

```typescript
// tools/src/node/database/mongo/mongocrud.ts
public async updateByFilter(
  datasetName: string,
  filter: Filter<T>,
  data: Partial<T>
): Promise<T | null> {
  const result = await collection.findOneAndUpdate(
    filter,
    { $set: data },
    { returnDocument: 'after' }
  );
  return result !== null ? (result as T) : null;
}
```

#### Minor smell — delete projection is domain-specific inside generic adapter

```typescript
// tools/src/node/database/mongo/mongocrud.ts
const result = await collection.findOneAndDelete(
  { [key]: value },
  { projection: { username: 1, email: 1 } }  // auth-specific fields in generic CRUD
);
```

A generic Adapter should not know about `username` / `email`. This couples the data layer to one domain entity.

**Principles:**

- **Meets:** **SRP** — `read`, `update`, and `updateByFilter` each perform one adaptation job from driver API to `ICrud`.
- **Breaks:** **SRP** — `delete` embeds auth-domain projection `{ username: 1, email: 1 }` in generic CRUD. **OCP** — new domains cannot extend delete behavior without modifying the shared adapter. **LSP** — `delete` is typed to return full `T`, but the projection can return a partial document, weakening the postcondition the `ICrud` contract promises.

---

### 5. Facade

**Source:** [Facade — Refactoring Guru](https://refactoring.guru/design-patterns/facade)

**Intent (Guru):** Provide a simplified interface to a library, framework, or complex subsystem. The Facade defines what clients need; subsystems stay internal.

**How it maps here:** `MongoEntity` is the Facade over collection creation, JSON schema validation, index setup, uniqueness checks, and `MongoCrud` delegation.

#### Well implemented — `init()` hides collection bootstrap complexity

```typescript
// tools/src/node/database/mongo/mongoentity.ts
async init(onFinish?: () => void | Promise<void>): Promise<void> {
  const collectionStatus = await this.#createCollection();
  // logs, then optional callback
}

async #createCollection(): Promise<boolean> {
  if (!await this.#crud.datasetExists(this.#collectionName)) {
    await this.#crud.createDataset(this.#collectionName, this.#schema);
    await this.#crud.createIndex(this.#collectionName, { entityId: 1 }, { unique: true });
    // custom indexes from EntityIndex<T>[]
  }
}
```

Callers call `entity.init()` once; they do not manage collections or indexes themselves.

#### Broken — `Users` pierces the Facade

By keeping a private `#crud` and calling it directly, `Users` clients effectively have two facades: `MongoEntity` methods and raw CRUD paths inside overrides. The simplified interface is no longer single-entry.

#### Partial leak — `Persons.listByIds` uses `getDataset` directly

```typescript
// identity/src/entities/persons/persons.ts
const collection: Collection = this.#crud.getDataset(this.#collectionName);
const docs = await collection.find({ entityId: { $in: userIds } }).toArray();
```

Valid for complex queries, but bypasses `MongoEntity.findByFilter`. Consider lifting `findByFilter` / `$in` support into the Facade so domain code stays above the driver.

**Principles:**

- **Meets:** **SRP** — `MongoEntity.init()` gives callers one bootstrap entry point; collection and index setup stay out of domain modules.
- **Breaks:** **SRP** — `Users` exposes a second persistence entry through raw `#crud`; `MongoEntity` also bootstraps connections in its constructor. **DIP** — the facade constructs `MongoCrud` instead of receiving an injected abstraction from the composition root. **DI** — connection wiring lives inside the facade rather than external bootstrap code. **OCP** — `Persons.listByIds` bypasses the facade's stable API (`findByFilter`) and reaches for `getDataset` directly.

---

### 6. Decorator

**Source:** [Decorator — Refactoring Guru](https://refactoring.guru/design-patterns/decorator)

**Intent (Guru):** Attach new behaviors to objects by placing them inside wrapper objects that contain the behaviors. Lets you combine behavior at runtime without subclass explosion.

**How it maps here:** The `@errorHandler` method decorator wraps `MongoCrud` operations with retry logic and normalized `CustomError` throwing — cross-cutting concern without polluting each method body.

#### Well implemented — error handling wrapped around CRUD methods

```typescript
// tools/src/node/database/mongo/mongocrud.ts
@errorHandler({ errorName: 'MONGO_CRUD_ERROR', message: 'Failed to create document' })
public async create<K extends keyof T>(datasetName: string, key: K, data: T): Promise<T | null> {
  // pure persistence logic only
}
```

```typescript
// tools/src/shared/utils/decorators.ts
export function errorHandler(options = {}) {
  return function (_target, propertyKey, descriptor: PropertyDescriptor) {
    const originalMethod = descriptor.value;
    descriptor.value = async function (...args) {
      // retry loop, then throw CustomError with errorName / message
      return await originalMethod.apply(this, args);
    };
    return descriptor;
  };
}
```

This matches Guru's structure: the wrapper (`descriptor.value`) delegates to the original method and adds behavior before/after (retries, error normalization).

#### No break identified

Keep `@errorHandler` on the data layer (`MongoCrud`), not on domain entities (`Users`, `Persons`), so business logic errors stay separate from infrastructure errors.

**Principles:**

- **Meets:** **SRP** — retry and error normalization live in one wrapper; CRUD method bodies stay focused on persistence. **OCP** — cross-cutting behavior extends through the decorator without modifying each method body.
- **Breaks:** none in current code while `@errorHandler` stays on `MongoCrud` — moving it onto domain entities would violate **SRP** by mixing infrastructure concerns into business modules. (TypeScript method decorator: analogous to Decorator, not the textbook object-composition form.)

---

### 7. DAO (Data Access Object)

**Source:** [DAO — Oracle Java EE documentation](https://www.oracle.com/java/technologies/data-access-object.html)  
*Enterprise pattern; not in the [Guru catalog](https://refactoring.guru/design-patterns/catalog).*

**Intent:** Abstract and encapsulate all access to the data source. The DAO manages the connection and exposes CRUD operations without exposing database semantics to callers.

**How it maps here:** `MongoCrud` is the DAO — collection-agnostic, no domain logic. `MongoEntity` adds entity lifecycle on top (closer to Repository).

#### Well implemented — `MongoCrud` stays domain-agnostic

All methods take `datasetName` + key/value; no imports from `authservice` or `identity`. Domain knowledge stays in entity subclasses.

**Principles:**

- **Meets:** **SRP** — `MongoCrud` methods stay domain-agnostic; no imports from `authservice` or `identity`.
- **Breaks:** **SRP** — `MongoCrud extends MongoDataset`, so data access and connection lifecycle change for different reasons in one inheritance chain. **ISP** — CRUD consumers inherit dataset/admin capabilities they never call; implementers must carry the full combined surface. **DIP** / **DI** — each `new MongoCrud(options.db)` from `MongoEntity`, subclasses, or `MongoBucket` constructs low-level details outside the composition root and duplicates client ownership.

---

## Not Matching Patterns

Patterns below are not implemented by the current Mongo entity stack. Each entry explains why the structure is absent and what would need to change before the pattern name applies.

Any future refactor should keep policy-owned ports such as `IUserRepository` and `IPersonRepository`, not expose `ICrud`, collection names, or Mongo types to domain/application code. Concrete Mongo adapters belong in infrastructure wiring at the composition root. That preserves DIP, DI, and IoC: bootstrap code controls construction, while policy classes depend only on abstractions.

### Strategy

**Source:** [Strategy — Refactoring Guru](https://refactoring.guru/design-patterns/strategy)

**Intent (Guru):** Defines a family of algorithms, puts each into a separate class, and makes their objects interchangeable. Strategy lets the algorithm vary independently from clients that use it.

**Why it does not match:** `ICrud` is an interface, but `MongoEntity` constructs `MongoCrud` directly; no family of interchangeable algorithms is supplied through composition. `Users.update` option flags are conditional branches, not strategy objects.

**Refactor Direction:** encapsulate each variation of an algorithm in a separate strategy behind a common contract. The context receives the selected strategy through dependency injection and delegates the variable behavior to it instead of branching on flags. Construction and strategy selection stay at the composition root. Strategy applies only when multiple algorithms are genuinely interchangeable and the context can switch among them without modification.

### Factory Method

**Source:** [Factory Method — Refactoring Guru](https://refactoring.guru/design-patterns/factory-method)

**Intent (Guru):** Provides an interface for creating objects in a superclass, but allows subclasses to alter the type of objects that will be created.

**Why it does not match:** `Persons.createPerson` constructs data and delegates persistence, but no superclass factory method defers the choice of concrete product to subclasses. Record preparation alone is not Factory Method.

**Refactor Direction:** define a creator superclass with a factory method that returns a product contract, then let subclasses decide which concrete product to instantiate. Client code calls the stable creation workflow without naming concrete products. Keep persistence outside the factory method unless persistence is itself part of the creation workflow. The composition root selects the concrete creator and supplies any external collaborators.

### Active Record

**Source:** [Active Record — Martin Fowler (EAA Catalog)](https://martinfowler.com/eaaCatalog/activeRecord.html)  
*Not in the [Guru GoF catalog](https://refactoring.guru/design-patterns/catalog); Fowler defines it as an object that wraps a database row and adds domain logic while encapsulating database access.*

**Intent (Fowler):** An object that wraps a row in a database table or view, encapsulates the database access, and adds domain logic on that data.

**Why it does not match:** `UserAuth` and `Person` are plain data types; records do not persist themselves.

**Refactor Direction:** place persistence operations such as save, update, and delete on each domain record. Back those operations with a narrow persistence abstraction supplied externally so records do not construct database infrastructure. This creates the defining Active Record behavior but intentionally couples records to persistence capability. Prefer Repository when domain objects should remain persistence-ignorant.

### Singleton

**Source:** [Singleton — Refactoring Guru](https://refactoring.guru/design-patterns/singleton)

**Intent (Guru):** Ensures that a class has only one instance, while providing a global point of access to it.

**Why it does not match:** no class enforces a single instance. Repeated `new MongoCrud(options.db)` calls currently create multiple client stacks.

**Refactor Direction:** enforce one process-wide instance only when shared identity—not merely shared lifetime—is required. Create or retrieve that instance in infrastructure bootstrap and inject dependent abstractions into consumers. Application and domain code should not access a global instance directly. If ordinary composition-root ownership can provide one shared instance, prefer it; a globally accessible Singleton can become a service locator that hides dependencies.

### Command

**Source:** [Command — Refactoring Guru](https://refactoring.guru/design-patterns/command)

**Intent (Guru):** Turns a request into a stand-alone object that contains all information about the request. This transformation lets you pass requests as method arguments, delay or queue a request's execution, and support undoable operations.

**Why it does not match:** callers invoke entity methods directly; operations are not encapsulated as command objects.

**Refactor Direction:** represent each request as an object implementing a common execution contract. Store all data and abstractions required to perform the request inside that object, while concrete dependencies are injected externally. An invoker can execute commands without knowing their internal work, enabling queues, retries, logging, scheduling, or undo when those capabilities are required.

### State

**Source:** [State — Refactoring Guru](https://refactoring.guru/design-patterns/state)

**Intent (Guru):** Lets an object alter its behavior when its internal state changes. The object will appear to change its class.

**Why it does not match:** `status` values are data, not polymorphic State objects; `updateByFilter` is compare-and-set behavior.

**Refactor Direction:** encapsulate behavior for each lifecycle state in a separate object behind a common state contract. The context holds its current state, delegates state-dependent operations to it, and changes state through explicit transitions. Persistence remains behind an independent port. This replaces growing conditionals only when behavior—not merely stored data—changes meaningfully by state.

### Abstract Factory

**Source:** [Abstract Factory — Refactoring Guru](https://refactoring.guru/design-patterns/abstract-factory)

**Intent (Guru):** Lets you produce families of related objects without specifying their concrete classes.

**Why it does not match:** the stack has one Mongo product family and no factory interface that creates related persistence objects.

**Refactor Direction:** define a factory contract that creates a family of related product abstractions. Each concrete factory produces a compatible family for one technology or deployment mode. Bootstrap selects one factory, creates the family, and injects those products into policy code. Abstract Factory is worthwhile only when several related products must remain compatible and the entire family must be replaceable as a unit.

### Observer

**Source:** [Observer — Refactoring Guru](https://refactoring.guru/design-patterns/observer)

**Intent (Guru):** Lets you define a subscription mechanism to notify multiple objects about any events that happen to the object they're observing.

**Why it does not match:** entity operations do not publish events to subscribed listeners.

**Refactor Direction:** let a subject maintain subscribers behind an observer contract and notify them after meaningful state changes. Subscribers handle independent reactions such as audit logging, cache invalidation, or event publication. Registration and concrete subscriber wiring belong at the composition root. Define failure and ordering semantics explicitly so one observer cannot silently corrupt the primary operation.

### Mediator

**Source:** [Mediator — Refactoring Guru](https://refactoring.guru/design-patterns/mediator)

**Intent (Guru):** Lets you reduce chaotic dependencies between objects. The pattern restricts direct communications between the objects and forces them to interact only via a mediator object.

**Why it does not match:** entities communicate through direct calls; no central object coordinates interactions among them.

**Refactor Direction:** introduce a coordinator that owns interactions among several components for a specific workflow. Components communicate through the mediator instead of referencing one another, while the mediator depends on narrow abstractions supplied at the composition root. This centralizes collaboration rules and reduces peer-to-peer coupling, but should not become a catch-all service containing unrelated business logic.

### Visitor

**Source:** [Visitor — Refactoring Guru](https://refactoring.guru/design-patterns/visitor)

**Intent (Guru):** Lets you separate algorithms from the objects on which they operate.

**Why it does not match:** there is no entity object hierarchy exposing an `accept` operation, and no external operation dispatches by concrete entity type.

**Refactor Direction:** define a visitor contract with one operation per concrete element type, and let each element dispatch itself to the matching visitor operation. New cross-cutting operations then live in new visitors instead of conditionals spread across clients. Visitor fits a stable element hierarchy with frequently added operations; it is a poor fit when element types change often because every visitor must then change.
