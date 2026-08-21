---
title: Principios SOLID
owner: OdairTrujillo
lastUpdate: 2026-08-17
category: arquitectura
lang: es
status: publicado
---

## Introducción

**SOLID** es un acrónimo de cinco principios de diseño de software asociados a Robert C. Martin:

- **SRP** — Principio de Responsabilidad Única (*Single Responsibility Principle*)
- **OCP** — Principio Abierto/Cerrado (*Open/Closed Principle*)
- **LSP** — Principio de Sustitución de Liskov (*Liskov Substitution Principle*)
- **ISP** — Principio de Segregación de Interfaces (*Interface Segregation Principle*)
- **DIP** — Principio de Inversión de Dependencias (*Dependency Inversion Principle*)

Estos principios orientan los límites de responsabilidad, la extensión, la subtipificación por comportamiento, el diseño de interfaces y la dirección de las dependencias. No garantizan por sí solos un buen diseño; ofrecen vocabulario y restricciones para razonar sobre el cambio.

## Terminología usada en esta guía

- **Módulo de alto nivel / política** — código de dominio o de aplicación que contiene decisiones de negocio y casos de uso.
- **Módulo de bajo nivel / detalle** — código de infraestructura para bases de datos, redes, sistemas de archivos, frameworks o APIs externas.
- **Abstracción / puerto** — contrato que describe lo que necesita quien llama sin elegir un proveedor concreto.
- **Implementación concreta / adaptador** — clase que cumple una abstracción usando una tecnología específica.
- **Inyección de dependencias (DI)** — técnica que suministra dependencias desde fuera de un objeto. La DI suele apoyar el DIP, pero la DI no es uno de los principios SOLID.
- **Inversión de control (IoC)** — patrón más amplio en el que código reutilizable o un framework controla parte del flujo del programa y llama al código de aplicación en puntos de extensión definidos. Delegar la creación y el cableado de objetos a una factoría o contenedor es una forma de IoC. La IoC está relacionada con el DIP y la DI, pero no es lo mismo que el DIP.

En TypeScript, una abstracción puede expresarse con una `interface`, una clase abstracta o un tipo función. Una `interface` es un contrato en tiempo de compilación; no crea comportamiento en tiempo de ejecución y por sí sola no garantiza el LSP ni el DIP.

## Ejemplo de pipeline ETL

Esta guía muestra cómo se aplican los principios SOLID mediante un ejemplo de pipeline ETL. Los fragmentos de TypeScript se construyen uno sobre otro y asumen un entorno de módulos moderno con `await` de nivel superior; las secciones posteriores reutilizan tipos, funciones y clases introducidos antes.

Los ejemplos de código en esta guía prefijan las interfaces con `I`, como `IDataExtractor` e `IFileChecker`. Un nombre basado en el rol, como `UserRepository`, también es válido y a menudo resulta más natural. Al usar el prefijo `I`, prefiera un nombre de contrato o capacidad, no un marcador de posición como `ISomeClass`.

```typescript
interface ExtractedRecord {
  readonly source: string;
  readonly [key: string]: unknown;
}

interface IDataExtractor {
  /** Las implementaciones deben admitir llamadas inmediatamente después de la construcción. */
  extract(): Promise<readonly ExtractedRecord[]>;
}

class ApiExtractor implements IDataExtractor {
  async extract(): Promise<readonly ExtractedRecord[]> {
    return [{ source: 'api' }];
  }
}
```

### **Principio de Responsabilidad Única (SRP)**

Un módulo de software debe tener una sola razón para cambiar: debe ser responsable ante un actor o ante una preocupación de negocio estrechamente relacionada. Otra formulación útil es: agrupar lo que cambia por la misma razón y separar lo que cambia por razones distintas.

Esto no significa que una solicitud de cambio deba afectar solo una clase. Una funcionalidad puede cambiar legítimamente varios módulos cohesionados. El SRP pregunta si cada módulo cambia por un solo tipo de razón.

En el ejemplo ETL, la configuración de conexión a base de datos, la extracción de datos y la autenticación tienen razones distintas para cambiar. Una sola clase que concentra las tres preocupaciones debe cambiar cada vez que cambie cualquiera de ellas:

```typescript
class MonolithicExtractor implements IDataExtractor {
  async extract(): Promise<readonly ExtractedRecord[]> {
    console.log('Connecting to the database...');
    // Código de conexión ...
    console.log('Authenticating...');
    // Código de autenticación ...
    console.log('Extracting from API...');
    // Código de extracción ...
    return [{ source: 'api' }];
  }
}
```

Ese diseño viola el SRP porque la política de conexión, la política de autenticación y la estrategia de extracción se mezclan en un solo módulo.

Separar esas preocupaciones mantiene cada clase responsable de un solo tipo de cambio. La autenticación puede separarse mediante middleware, un colaborador o un decorador; el mecanismo importa menos que mantenerla fuera de la lógica de extracción.

```typescript
// Solo maneja la configuración de conexión a BD
class DBSession {
  connect(): void {
    console.log('Connecting to the database...');
  }
}

// Solo maneja la autenticación
class Authenticator {
  authenticate(): void {
    console.log('Authenticating...');
  }
}

// Coordina colaboradores sin absorber sus responsabilidades
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

### **Principio Abierto/Cerrado (OCP)**

Las entidades de software deben estar abiertas a la extensión pero cerradas a la modificación. Las nuevas variantes deben añadirse mediante puntos de extensión estables sin modificar repetidamente el comportamiento establecido.

El OCP no prohíbe toda modificación, y la herencia no es su único mecanismo. La composición, las estrategias, los plugins y las interfaces también pueden ofrecer puntos de extensión.

Una violación común del OCP aparece cuando cada nuevo tipo de origen obliga a editar la misma función:

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

// Añadir un origen de archivo requiere modificar collectBySource.
```

Aquí, `IDataExtractor` es el punto de extensión estable. Las nuevas variantes implementan la interfaz sin cambiar a quienes llaman ni las implementaciones existentes:

```typescript
async function collect(extractor: IDataExtractor): Promise<void> {
  const records = await extractor.extract();
  console.log(`Collected ${records.length} records`);
}

await collect(new ApiExtractor());
await collect(new DatabaseExtractor());
```

Añadir `FileExtractor` ahora significa introducir una nueva clase, no reabrir `collect`.

### **Principio de Sustitución de Liskov (LSP)**

Un subtipo debe poder usarse donde se espera su supertipo sin cambiar la corrección del programa. El LSP trata de **subtipificación por comportamiento**, no solo de coincidir nombres o firmas de métodos.

Un subtipo debe preservar el contrato expuesto por el supertipo:

- no debe reforzar las precondiciones;
- no debe debilitar las postcondiciones;
- debe preservar invariantes y transiciones de estado esperadas;
- no debe introducir comportamiento que sorprenda a quienes confían en el contrato del supertipo.

Las interfaces y las clases abstractas pueden expresar parte de este contrato, pero no garantizan automáticamente el LSP. La documentación, las pruebas y el comportamiento de la implementación deben preservarlo.

En TypeScript, esto también aplica a la subtipificación estructural. Una implementación puede satisfacer una interfaz sintácticamente y aun así violar su contrato de comportamiento.

El contrato existente de `IDataExtractor` promete una colección de registros extraídos y permite llamar a `extract()` inmediatamente después de la construcción. Por tanto, cualquier implementación que cumpla el contrato debe poder usarse con la función `collect` definida antes sin requerir inicialización adicional ni otras condiciones no documentadas.

`DatabaseExtractor` puede sustituir a `IDataExtractor` sin cambiar la corrección de `collect`. Quien llama no necesita saber qué implementación concreta recibió, y no se han introducido precondiciones adicionales.

Considere ahora una implementación que requiere un paso de inicialización no documentado:

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

Esta implementación satisface la interfaz de TypeScript, pero viola el LSP. El contrato de `IDataExtractor` permite invocar `extract()` de inmediato, mientras que `ManuallyInitializedExtractor` añade una nueva precondición: primero debe haberse llamado a `initialize()`.

Por tanto, reemplazar una implementación válida de `IDataExtractor` por `ManuallyInitializedExtractor` puede cambiar el comportamiento de quien confía solo en el contrato de `IDataExtractor`.

### **Principio de Segregación de Interfaces (ISP)**

Los clientes no deben verse obligados a depender de métodos que no usan. Las interfaces deben centrarse en las necesidades de un cliente o rol concreto en lugar de reunir capacidades no relacionadas.

Por ejemplo, los clientes de extracción solo necesitan `extract`. Los clientes orientados a archivos pueden necesitar además `fileExists`. Mantener esos contratos separados evita que extractores de API y sus llamadores dependan de operaciones de archivos.

Obligar a una clase a implementar métodos irrelevantes es una violación común del ISP, al igual que obligar a los clientes a depender de métodos que nunca usan.

La siguiente interfaz amplia obliga a un extractor de API a exponer una operación de archivos sin sentido:

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

Segregar la capacidad específica de archivos mantiene a cada cliente dependiendo solo de lo que usa:

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

`runExtraction` depende solo de `IDataExtractor`, por lo que no se ve obligada a conocer `fileExists`. Los llamadores orientados a archivos pueden depender de `IFileChecker` cuando necesiten esa capacidad.

### **Principio de Inversión de Dependencias (DIP)**

Robert C. Martin define el DIP en dos partes:

1. Los módulos de alto nivel no deben depender de los módulos de bajo nivel; ambos deben depender de abstracciones.
2. Las abstracciones no deben depender de los detalles; los detalles deben depender de las abstracciones.

En esta guía, **módulo de alto nivel** significa política de dominio o de aplicación, mientras que **módulo de bajo nivel** significa un detalle de infraestructura como acceso por API, archivos o base de datos. La abstracción debe describir lo que necesita la política de alto nivel. Un adaptador de bajo nivel implementa esa abstracción.

`Pipeline` cumple el DIP cuando depende de `IDataExtractor` y recibe un extractor concreto desde fuera:

```typescript
// Política de alto nivel
class Pipeline {
  constructor(private readonly extractor: IDataExtractor) {}

  async run(): Promise<readonly ExtractedRecord[]> {
    const records = await this.extractor.extract();
    console.log(`Validating ${records.length} records`);
    return records;
  }
}

// Raíz de composición: los detalles concretos se seleccionan e inyectan aquí.
const pipeline = new Pipeline(new ApiExtractor());
await pipeline.run();
```

La política de alto nivel viola el DIP cuando posee el detalle concreto directamente:

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

`CoupledPipeline` también viola el DIP si su constructor exige `ApiExtractor` en lugar de `IDataExtractor`, porque la política de alto nivel dependería entonces de una implementación de bajo nivel. Pasar una instancia de `ApiExtractor` a un constructor tipado como `IDataExtractor` no viola el DIP; acoplar la política a la clase concreta sí.

El código que crea `ApiExtractor` y lo pasa a `Pipeline` es la **raíz de composición**. Ese cableado es inyección de dependencias: una técnica usada para satisfacer la dirección de dependencias que exige el DIP.

El DIP, la DI y la IoC se discuten a menudo juntos, pero no son intercambiables:

- **DIP** define la dirección de las dependencias: política y detalles dependen de abstracciones.
- **DI** suministra esas dependencias desde fuera del módulo consumidor.
- **IoC** transfiere el control de parte del flujo del programa a código reutilizable o a un framework; la creación y el cableado gestionados externamente son una aplicación de ello.

Aunque el DIP y el ISP usan abstracciones, abordan preocupaciones distintas. El ISP mantiene las interfaces enfocadas para que los clientes no dependan de métodos que no usan. El DIP mantiene la política de alto nivel independiente de los detalles de bajo nivel haciendo que ambos dependan de una abstracción.

## Beneficios y contrapartidas

Aplicado donde se espera cambio, SOLID puede mejorar:

- **Mantenibilidad** — módulos cohesionados facilitan localizar y entender los cambios.
- **Extensibilidad** — puntos de extensión estables permiten nuevas variantes con menos cambios en el código existente.
- **Flexibilidad** — las abstracciones reducen el acoplamiento a infraestructura o implementaciones concretas.
- **Capacidad de prueba** — responsabilidades enfocadas y dependencias reemplazables facilitan aislar el comportamiento.
- **Reutilización** — componentes pequeños y basados en roles son más fáciles de combinar en distintos contextos.

Estos beneficios no son automáticos. Aplicar cada principio en todas partes puede producir interfaces, clases, indirección y configuración innecesarias. Eso aumenta el coste de navegación y puede hacer más difícil de entender un comportamiento simple.

Use SOLID en respuesta a presiones de cambio concretas: responsabilidades que evolucionan de forma independiente, variantes recurrentes, implementaciones sustituibles, clientes con necesidades distintas o política acoplada a infraestructura. Prefiera código directo cuando no exista esa presión, e introduzca una abstracción cuando cree un límite claro en lugar de anticipar solo un futuro hipotético.

## Referencias

### Fuentes fundamentales

- Martin, Robert C. [Principles of Object-Oriented Design](http://www.butunclebob.com/ArticleS.UncleBob.PrinciplesOfOod).
- Martin, Robert C. [The Single Responsibility Principle](https://blog.cleancoder.com/uncle-bob/2014/05/08/SingleReponsibilityPrinciple.html).
- Martin, Robert C. [The Dependency Inversion Principle](https://objectmentor.com/resources/articles/dip.pdf).
- Liskov, Barbara, and Jeannette Wing. [A Behavioral Notion of Subtyping](https://doi.org/10.1145/197320.197383). *ACM Transactions on Programming Languages and Systems*, 1994.

### Fuentes complementarias

- GeeksforGeeks. [SOLID Principles with Real Life Examples](https://www.geeksforgeeks.org/system-design/solid-principle-in-programming-understand-with-real-life-examples/).
- Álvarez Caules, Cecilio Manuel. [*Arquitectura Java Sólida y Patrones*](https://www.buscalibre.es/libro-arquitectura-java-solida-y-patrones/9798322883159/p/60072104).
