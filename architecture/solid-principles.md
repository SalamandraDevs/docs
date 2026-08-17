---
title: SOLID Principles
---
## Introduction

**SOLID** is an acronym for five software-design principles associated with Robert C. Martin:

- **SRP** — Single Responsibility Principle
- **OCP** — Open/Closed Principle
- **LSP** — Liskov Substitution Principle
- **ISP** — Interface Segregation Principle
- **DIP** — Dependency Inversion Principle

These principles guide responsibility boundaries, extension, behavioral subtyping, interface design, and dependency direction. They do not guarantee good design by themselves; they provide vocabulary and constraints for reasoning about change.

## Terminology used in this guide

- **High-level module / policy** — domain or application code containing business decisions and use cases.
- **Low-level module / detail** — infrastructure code for databases, networks, filesystems, frameworks, or external APIs.
- **Abstraction / port** — a contract describing what a caller needs without choosing a concrete provider.
- **Concrete implementation / adapter** — a class that fulfills an abstraction using a specific technology.
- **Dependency injection (DI)** — a technique that supplies dependencies from outside an object. DI often supports DIP, but DI is not one of the SOLID principles.
- **Inversion of control (IoC)** — a broader pattern in which reusable code or a framework controls part of the program flow and calls application code at defined extension points. Delegating object creation and wiring to a factory or container is one form of IoC. IoC is related to DIP and DI, but it is not the same as DIP.

In TypeScript, an abstraction may be expressed with an `interface`, an abstract class, or a function type. An `interface` is a compile-time contract; it does not create runtime behavior and does not by itself guarantee LSP or DIP.

## ETL Pipeline Example

This guide shows how the SOLID principles are implemented using an ETL pipeline example. The TypeScript snippets build on one another and assume a modern module environment with top-level `await`; later sections reuse types, functions, and classes introduced earlier.

The code examples in this guide prefix interfaces with `I`, such as `IDataExtractor` and `IFileChecker`. A role-based name such as `UserRepository` is also valid and often reads more naturally. When using the `I` prefix, prefer a contract or capability name, not a placeholder such as `ISomeClass`.

```typescript
interface ExtractedRecord {
  readonly source: string;
  readonly [key: string]: unknown;
}

interface IDataExtractor {
  /** Implementations must support calls immediately after construction. */
  extract(): Promise<readonly ExtractedRecord[]>;
}

class ApiExtractor implements IDataExtractor {
  async extract(): Promise<readonly ExtractedRecord[]> {
    return [{ source: 'api' }];
  }
}
```

### **Single Responsibility Principle (SRP)**

A software module should have one reason to change: it should be responsible to one actor or one closely related business concern. Another useful formulation is: gather things that change for the same reason and separate things that change for different reasons.

This does not mean that one change request must affect only one class. A feature may legitimately change several cohesive modules. SRP asks whether each module changes for one kind of reason.

In the ETL example, database connection setup, data extraction, and authentication have different reasons to change. A single class that owns all three concerns must change whenever any one of them changes:

```typescript
class MonolithicExtractor implements IDataExtractor {
  async extract(): Promise<readonly ExtractedRecord[]> {
    console.log('Connecting to the database...');
    // Connection code ...
    console.log('Authenticating...');
    // Auth code code ...
    console.log('Extracting from API...');
    // Extraction code ...
    return [{ source: 'api' }];
  }
}
```

That design violates SRP because connection policy, authentication policy, and extraction strategy are mixed into one module.

Separating those concerns keeps each class responsible for one kind of change. Authentication may be separated through middleware, a collaborator, or a decorator; the mechanism is less important than keeping it out of extraction logic.

```typescript
// Handles only DB connection setup
class DBSession {
  connect(): void {
    console.log('Connecting to the database...');
  }
}

// Handles only authentication
class Authenticator {
  authenticate(): void {
    console.log('Authenticating...');
  }
}

// Coordinates collaborators without absorbing their responsibilities
class AuthenticatedExtractor implements IDataExtractor {
  constructor(
    private readonly authenticator: Authenticator,
    private readonly extractor: IDataExtractor,
  ) {}

  async extract(): Promise<readonly ExtractedRecord[]> {
    this.authenticator.authenticate();
    return this.extractor.extract();
  }
}
```

### **Open/Closed Principle (OCP)**

Software entities should be open for extension but closed for modification. New variants should be added through stable extension points without repeatedly changing established behavior.

OCP does not prohibit every modification, and inheritance is not its only mechanism. Composition, strategies, plugins, and interfaces can also provide extension points.

A common OCP violation appears when each new source type requires editing the same function:

```typescript
class DatabaseExtractor implements IDataExtractor {
  async extract(): Promise<readonly ExtractedRecord[]> {
    return [{ source: 'database' }];
  }
}

async function collectBySource(
  source: 'api' | 'database',
): Promise<void> {
  let extractor: IDataExtractor;

  if (source === 'api') {
    extractor = new ApiExtractor();
  } else {
    extractor = new DatabaseExtractor();
  }

  const records = await extractor.extract();
  console.log(`Collected ${records.length} records`);
}

// Adding a file source requires modifying collectBySource.
```

Here, `IDataExtractor` is the stable extension point. New variants implement the interface without changing callers or existing implementations:

```typescript
async function collect(extractor: IDataExtractor): Promise<void> {
  const records = await extractor.extract();
  console.log(`Collected ${records.length} records`);
}

await collect(new ApiExtractor());
await collect(new DatabaseExtractor());
```

Adding `FileExtractor` now means introducing a new class, not reopening `collect`.

### **Liskov Substitution Principle (LSP)**

A subtype must be usable wherever its supertype is expected without changing the correctness of the program. LSP is about **behavioral subtyping**, not only matching method names or signatures.

A subtype must preserve the contract exposed by the supertype:

- it must not strengthen preconditions;
- it must not weaken postconditions;
- it must preserve invariants and expected state transitions;
- it must not introduce behavior that surprises callers relying on the supertype contract.

Interfaces and abstract classes can express part of this contract, but they do not automatically guarantee LSP. Documentation, tests, and implementation behavior must preserve it.

In TypeScript, this applies to structural subtyping as well. An implementation can satisfy an interface syntactically while still violating its behavioral contract.

The existing `IDataExtractor` contract promises a collection of extracted records and permits `extract()` to be called immediately after construction. Therefore, any implementation that satisfies the contract should be usable by the previously defined `collect` function without requiring additional initialization or other undocumented conditions.

`DatabaseExtractor` can be substituted for `IDataExtractor` without changing the correctness of `collect`. The caller does not need to know which concrete implementation it received, and no additional preconditions have been introduced.

Now consider an implementation that requires an undocumented initialization step:

```typescript
class ManuallyInitializedExtractor implements IDataExtractor {
  private initialized = false;

  initialize(): void {
    this.initialized = true;
  }

  async extract(): Promise<readonly ExtractedRecord[]> {
    if (!this.initialized) {
      throw new Error('Extractor must be initialized before extract()');
    }

    return [
      { source: 'manual' },
    ];
  }
}

await collect(new ManuallyInitializedExtractor());
```

This implementation satisfies the TypeScript interface, but it violates LSP. The `IDataExtractor` contract allows callers to invoke `extract()` immediately, while `ManuallyInitializedExtractor` adds a new precondition: `initialize()` must have been called first.

Therefore, replacing a valid `IDataExtractor` implementation with `ManuallyInitializedExtractor` can change the behavior of a caller that relies only on the `IDataExtractor` contract.

### **Interface Segregation Principle (ISP)**

Clients should not be forced to depend on methods they do not use. Interfaces should be focused on the needs of a specific client or role instead of collecting unrelated capabilities.

For example, extraction clients need only `extract`. File-aware clients may additionally need `fileExists`. Keeping those contracts separate prevents API extractors and their callers from depending on file operations.

Forcing a class to implement irrelevant methods is a common ISP violation, as is forcing clients to depend on methods they never use.

The following broad interface forces an API extractor to expose a meaningless file operation:

```typescript
interface IBloatedExtractor extends IDataExtractor {
  fileExists(path: string): Promise<boolean>;
}

class BloatedApiExtractor implements IBloatedExtractor {
  async extract(): Promise<readonly ExtractedRecord[]> {
    return [{ source: 'api' }];
  }

  async fileExists(_path: string): Promise<boolean> {
    throw new Error('Not supported');
  }
}
```

Segregating the file-specific capability keeps each client dependent only on what it uses:

```typescript
interface IFileChecker {
  fileExists(path: string): Promise<boolean>;
}

class FileExtractor implements IDataExtractor, IFileChecker {
  async extract(): Promise<readonly ExtractedRecord[]> {
    return [{ source: 'file' }];
  }

  async fileExists(path: string): Promise<boolean> {
    return path.length > 0;
  }
}

async function runExtraction(extractor: IDataExtractor): Promise<void> {
  const records = await extractor.extract();
  console.log(`Extracted ${records.length} records`);
}
```

`runExtraction` depends only on `IDataExtractor`, so it is not forced to know about `fileExists`. File-specific callers can depend on `IFileChecker` when they need that capability.

### **Dependency Inversion Principle (DIP)**

Robert C. Martin defines DIP in two parts:

1. High-level modules should not depend on low-level modules; both should depend on abstractions.
2. Abstractions should not depend on details; details should depend on abstractions.

In this guide, **high-level module** means domain or application policy, while **low-level module** means an infrastructure detail such as API, file, or database access. The abstraction should describe what the high-level policy needs. A low-level adapter implements that abstraction.

`Pipeline` satisfies DIP when it depends on `IDataExtractor` and receives a concrete extractor from outside:

```typescript
// High-level policy
class Pipeline {
  constructor(private readonly extractor: IDataExtractor) {}

  async run(): Promise<readonly ExtractedRecord[]> {
    const records = await this.extractor.extract();
    console.log(`Validating ${records.length} records`);
    return records;
  }
}

// Composition root: concrete details are selected and injected here.
const pipeline = new Pipeline(new ApiExtractor());
await pipeline.run();
```

The high-level policy breaks DIP when it owns the concrete detail directly:

```typescript
class CoupledPipeline {
  private readonly extractor = new ApiExtractor();

  async run(): Promise<readonly ExtractedRecord[]> {
    const records = await this.extractor.extract();
    console.log(`Validating ${records.length} records`);
    return records;
  }
}
```

`CoupledPipeline` also breaks DIP if its constructor requires `ApiExtractor` instead of `IDataExtractor`, because high-level policy would then depend on a low-level implementation. Passing an `ApiExtractor` instance into a constructor typed as `IDataExtractor` does not break DIP; coupling policy to the concrete class does.

The code that creates `ApiExtractor` and passes it to `Pipeline` is the **composition root**. That wiring is dependency injection: a technique used to satisfy the dependency direction required by DIP.

DIP, DI, and IoC are often discussed together, but they are not interchangeable:

- **DIP** defines dependency direction: policy and details both depend on abstractions.
- **DI** supplies those dependencies from outside the consuming module.
- **IoC** transfers control of part of the program flow to reusable code or a framework; externally managed creation and wiring is one application of it.

Although DIP and ISP both use abstractions, they address different concerns. ISP keeps interfaces focused so clients do not depend on methods they do not use. DIP keeps high-level policy independent from low-level details by making both depend on an abstraction.

## Benefits and trade-offs

Applied where change is expected, SOLID can improve:

- **Maintainability** — cohesive modules make changes easier to locate and understand.
- **Extensibility** — stable extension points allow new variants with fewer changes to existing code.
- **Flexibility** — abstractions reduce coupling to specific infrastructure or implementations.
- **Testability** — focused responsibilities and replaceable dependencies make behavior easier to isolate.
- **Reusability** — small, role-based components are easier to combine in different contexts.

These benefits are not automatic. Applying every principle everywhere can produce unnecessary interfaces, classes, indirection, and configuration. That increases navigation cost and can make simple behavior harder to understand.

Use SOLID in response to concrete change pressures: responsibilities that evolve independently, recurring variants, substitutable implementations, clients with different needs, or policy coupled to infrastructure. Prefer direct code when no such pressure exists, and introduce an abstraction when it creates a clear boundary rather than merely anticipating a hypothetical future.

## References

### Foundational sources

- Martin, Robert C. [Principles of Object-Oriented Design](http://www.butunclebob.com/ArticleS.UncleBob.PrinciplesOfOod).
- Martin, Robert C. [The Single Responsibility Principle](https://blog.cleancoder.com/uncle-bob/2014/05/08/SingleReponsibilityPrinciple.html).
- Martin, Robert C. [The Dependency Inversion Principle](https://objectmentor.com/resources/articles/dip.pdf).
- Liskov, Barbara, and Jeannette Wing. [A Behavioral Notion of Subtyping](https://doi.org/10.1145/197320.197383). *ACM Transactions on Programming Languages and Systems*, 1994.

### Supplementary sources

- GeeksforGeeks. [SOLID Principles with Real Life Examples](https://www.geeksforgeeks.org/system-design/solid-principle-in-programming-understand-with-real-life-examples/).
- Álvarez Caules, Cecilio Manuel. [*Arquitectura Java Sólida y Patrones*](https://www.buscalibre.es/libro-arquitectura-java-solida-y-patrones/9798322883159/p/60072104).
