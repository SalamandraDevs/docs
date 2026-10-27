---
title: Principios SOLID
lastUpdate: 2026-08-17
lang: es
category: arquitectura
status: published
owner: Odair Trujillo
---

## Introducción

**SOLID** es un acrónimo de cinco principios de diseño de software asociados a Robert C.
Martin:

- **SRP**: Principio de Responsabilidad Única (*Single Responsibility Principle*)
- **OCP**: Principio Abierto/Cerrado (*Open/Closed Principle*)
- **LSP**: Principio de Sustitución de Liskov (*Liskov Substitution Principle*)
- **ISP**: Principio de Segregación de Interfaces (*Interface Segregation Principle*)
- **DIP**: Principio de Inversión de Dependencias (*Dependency Inversion Principle*)

Estos principios orientan los límites de responsabilidad, la extensión, la subtipificación
por comportamiento, el diseño de interfaces y la dirección de las dependencias. No
garantizan por sí solos un buen diseño; ofrecen vocabulario y restricciones para razonar
sobre el cambio.

Cada principio lleva un ID estable (`SP-NN`) para su citación; los IDs siguen
[IDs de documento](../normas/convenciones-nombres.md#ids-de-documento).

## Terminología usada en esta guía

- **Módulo de alto nivel / política**: código de dominio o de aplicación que contiene
  decisiones de negocio y casos de uso.
- **Módulo de bajo nivel / detalle**: código de infraestructura para bases de datos, redes,
  sistemas de archivos, frameworks o APIs externas.
- **Abstracción / puerto**: contrato que describe lo que necesita quien llama sin elegir un
  proveedor concreto.
- **Implementación concreta / adaptador**: clase que cumple una abstracción usando una
  tecnología específica.
- **Inyección de dependencias (DI)**: técnica que suministra dependencias desde fuera de un
  objeto. La DI suele apoyar el DIP, pero la DI no es uno de los principios SOLID.
- **Inversión de control (IoC)**: patrón más amplio en el que código reutilizable o un
  framework controla parte del flujo del programa y llama al código de aplicación en puntos
  de extensión definidos. Delegar la creación y el cableado de objetos a una factoría o
  contenedor es una forma de IoC. La IoC está relacionada con el DIP y la DI, pero no es lo
  mismo que el DIP.

En TypeScript, una abstracción puede expresarse con una `interface`, una clase abstracta o
un tipo función. Una `interface` es un contrato en tiempo de compilación; no crea
comportamiento en tiempo de ejecución y no garantiza por sí sola LSP ni DIP.

## Ejemplo de pipeline ETL

Esta guía muestra cómo se implementan los principios SOLID usando un ejemplo de pipeline
ETL. Los fragmentos de TypeScript se construyen uno sobre otro y asumen un entorno de
módulos moderno con `await` de nivel superior; las secciones posteriores reutilizan tipos,
funciones y clases introducidos antes.

Los ejemplos de código de esta guía prefijan las interfaces con `I`, como `IDataExtractor`
e `IFileChecker`. Un nombre basado en el rol como `UserRepository` también es válido y a
menudo se lee con más naturalidad. Al usar el prefijo `I`, prefiera un nombre de contrato o
capacidad, no un marcador de posición como `ISomeClass`.

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

### SP-01 Principio de Responsabilidad Única (SRP)

Un módulo de software debe tener una sola razón para cambiar: debe ser responsable ante un
actor o ante una preocupación de negocio estrechamente relacionada. Otra formulación útil
es: agrupar lo que cambia por la misma razón y separar lo que cambia por razones distintas.

Esto no significa que una solicitud de cambio deba afectar solo una clase. Una funcionalidad
puede cambiar legítimamente varios módulos cohesionados. El SRP pregunta si cada módulo
cambia por un solo tipo de razón.

En el ejemplo ETL, la configuración de conexión a base de datos, la extracción de datos y la
autenticación tienen razones distintas para cambiar. Una sola clase que concentra las tres
preocupaciones debe cambiar cada vez que cambie cualquiera de ellas:

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

Ese diseño viola el SRP porque la política de conexión, la política de autenticación y la
estrategia de extracción se mezclan en un solo módulo.

Separar esas preocupaciones mantiene cada clase responsable de un solo tipo de cambio. La
autenticación puede separarse mediante middleware, un colaborador o un decorador; el
mecanismo importa menos que mantenerla fuera de la lógica de extracción.

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

### SP-02 Principio Abierto/Cerrado (OCP)

Las entidades de software deben estar abiertas a la extensión pero cerradas a la
modificación. Las variantes nuevas deben añadirse mediante puntos de extensión estables sin
cambiar repetidamente el comportamiento establecido.

El OCP no prohíbe toda modificación, y la herencia no es su único mecanismo. La composición,
las estrategias, los plugins y las interfaces también pueden proporcionar puntos de
extensión.

Una violación común del OCP aparece cuando cada nuevo tipo de fuente obliga a editar la
misma función:

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

// Añadir una fuente de archivo obliga a modificar collectBySource.
```

Aquí, `IDataExtractor` es el punto de extensión estable. Las variantes nuevas implementan la
interfaz sin cambiar a quienes llaman ni a las implementaciones existentes:

```typescript
async function collect(extractor: IDataExtractor): Promise<void> {
  const records = await extractor.extract();
  console.log(`Collected ${records.length} records`);
}

await collect(new ApiExtractor());
await collect(new DatabaseExtractor());
```

Añadir `FileExtractor` ahora significa introducir una clase nueva, no reabrir `collect`.

### SP-03 Principio de Sustitución de Liskov (LSP)

Un subtipo debe poder usarse allí donde se espera su supertipo sin cambiar la corrección del
programa. El LSP trata sobre la **subtipificación por comportamiento**, no solo sobre
coincidir nombres o firmas de métodos.

Un subtipo debe preservar el contrato expuesto por el supertipo:

- no debe fortalecer las precondiciones;
- no debe debilitar las postcondiciones;
- debe preservar los invariantes y las transiciones de estado esperadas;
- no debe introducir comportamiento que sorprenda a quien llama confiando en el contrato del
  supertipo.

Las interfaces y las clases abstractas pueden expresar parte de este contrato, pero no
garantizan automáticamente el LSP. La documentación, las pruebas y el comportamiento de la
implementación deben preservarlo.

En TypeScript, esto también aplica a la subtipificación estructural. Una implementación
puede satisfacer una interfaz sintácticamente y, aun así, violar su contrato de
comportamiento.

El contrato existente de `IDataExtractor` promete una colección de registros extraídos y
permite llamar a `extract()` inmediatamente después de la construcción. Por lo tanto,
cualquier implementación que satisfaga el contrato debería poder usarse con la función
`collect` definida antes sin requerir inicialización adicional ni otras condiciones no
documentadas.

`DatabaseExtractor` puede sustituirse por `IDataExtractor` sin cambiar la corrección de
`collect`. Quien llama no necesita saber qué implementación concreta recibió, y no se han
introducido precondiciones adicionales.

Ahora considere una implementación que requiere un paso de inicialización no documentado:

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

Esta implementación satisface la interfaz de TypeScript, pero viola el LSP. El contrato de
`IDataExtractor` permite a quien llama invocar `extract()` inmediatamente, mientras que
`ManuallyInitializedExtractor` añade una precondición nueva: `initialize()` debe haberse
llamado primero.

Por lo tanto, reemplazar una implementación válida de `IDataExtractor` por
`ManuallyInitializedExtractor` puede cambiar el comportamiento de quien llama confiando solo
en el contrato de `IDataExtractor`.

### SP-04 Principio de Segregación de Interfaces (ISP)

Los clientes no deberían verse obligados a depender de métodos que no usan. Las interfaces
deberían enfocarse en las necesidades de un cliente o rol específico en lugar de reunir
capacidades no relacionadas.

Por ejemplo, los clientes de extracción solo necesitan `extract`. Los clientes conscientes
de archivos pueden necesitar además `fileExists`. Mantener esos contratos separados impide
que los extractores de API y quienes los llaman dependan de operaciones de archivos.

Obligar a una clase a implementar métodos irrelevantes es una violación común del ISP, igual
que obligar a los clientes a depender de métodos que nunca usan.

La siguiente interfaz amplia obliga a un extractor de API a exponer una operación de archivo
sin sentido:

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

Segregar la capacidad específica de archivos mantiene a cada cliente dependiente solo de lo
que usa:

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

`runExtraction` depende solo de `IDataExtractor`, por lo que no está obligada a conocer
`fileExists`. Los clientes específicos de archivos pueden depender de `IFileChecker` cuando
necesitan esa capacidad.

### SP-05 Principio de Inversión de Dependencias (DIP)

Robert C. Martin define el DIP en dos partes:

1. Los módulos de alto nivel no deberían depender de los módulos de bajo nivel; ambos
   deberían depender de abstracciones.
2. Las abstracciones no deberían depender de los detalles; los detalles deberían depender de
   las abstracciones.

En esta guía, **módulo de alto nivel** significa política de dominio o de aplicación,
mientras que **módulo de bajo nivel** significa un detalle de infraestructura como el acceso
a API, archivos o base de datos. La abstracción debería describir lo que necesita la
política de alto nivel. Un adaptador de bajo nivel implementa esa abstracción.

`Pipeline` satisface el DIP cuando depende de `IDataExtractor` y recibe un extractor
concreto desde fuera:

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

La política de alto nivel rompe el DIP cuando posee el detalle concreto directamente:

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

`CoupledPipeline` también rompe el DIP si su constructor requiere `ApiExtractor` en lugar de
`IDataExtractor`, porque la política de alto nivel dependería entonces de una implementación
de bajo nivel. Pasar una instancia de `ApiExtractor` a un constructor tipado como
`IDataExtractor` no rompe el DIP; acoplar la política a la clase concreta sí lo rompe.

El código que crea `ApiExtractor` y lo pasa a `Pipeline` es la **raíz de composición**. Ese
cableado es inyección de dependencias: una técnica usada para satisfacer la dirección de
dependencias que exige el DIP.

El DIP, la DI y la IoC se discuten a menudo juntos, pero no son intercambiables:

- El **DIP** define la dirección de las dependencias: la política y los detalles dependen
  ambos de abstracciones.
- La **DI** suministra esas dependencias desde fuera del módulo consumidor.
- La **IoC** transfiere el control de parte del flujo del programa a código reutilizable o a
  un framework; la creación y el cableado gestionados externamente son una aplicación de
  ella.

Aunque el DIP y el ISP usan abstracciones, abordan preocupaciones distintas. El ISP mantiene
las interfaces enfocadas para que los clientes no dependan de métodos que no usan. El DIP
mantiene la política de alto nivel independiente de los detalles de bajo nivel haciendo que
ambos dependan de una abstracción.

## Beneficios y contrapartidas

Aplicado donde se espera el cambio, SOLID puede mejorar:

- **Mantenibilidad**: los módulos cohesionados hacen que los cambios sean más fáciles de
  localizar y entender.
- **Extensibilidad**: los puntos de extensión estables permiten variantes nuevas con menos
  cambios al código existente.
- **Flexibilidad**: las abstracciones reducen el acoplamiento a infraestructura o
  implementaciones específicas.
- **Testeabilidad**: las responsabilidades enfocadas y las dependencias reemplazables hacen
  que el comportamiento sea más fácil de aislar.
- **Reutilización**: los componentes pequeños y basados en roles son más fáciles de combinar
  en contextos distintos.

Estos beneficios no son automáticos. Aplicar cada principio en todas partes puede producir
interfaces, clases, indirección y configuración innecesarias. Eso aumenta el coste de
navegación y puede hacer que un comportamiento simple sea más difícil de entender.

Use SOLID en respuesta a presiones de cambio concretas: responsabilidades que evolucionan de
forma independiente, variantes recurrentes, implementaciones sustituibles, clientes con
necesidades distintas o política acoplada a la infraestructura. Prefiera el código directo
cuando no exista tal presión, e introduzca una abstracción cuando cree un límite claro en
lugar de solo anticipar un futuro hipotético.

## Referencias

### Fuentes fundamentales

- Martin, Robert C. [Principles of Object-Oriented
  Design](http://www.butunclebob.com/ArticleS.UncleBob.PrinciplesOfOod).
- Martin, Robert C. [The Single Responsibility
  Principle](https://blog.cleancoder.com/uncle-bob/2014/05/08/SingleReponsibilityPrinciple.html).
- Martin, Robert C. [The Dependency Inversion
  Principle](https://objectmentor.com/resources/articles/dip.pdf).
- Liskov, Barbara, y Jeannette Wing. [A Behavioral Notion of
  Subtyping](https://doi.org/10.1145/197320.197383). *ACM Transactions on Programming
  Languages and Systems*, 1994.

### Fuentes complementarias

- GeeksforGeeks. [SOLID Principles with Real Life
  Examples](https://www.geeksforgeeks.org/system-design/solid-principle-in-programming-understand-with-real-life-examples/).
- Álvarez Caules, Cecilio Manuel. [*Arquitectura Java Sólida y
  Patrones*](https://www.buscalibre.es/libro-arquitectura-java-solida-y-patrones/9798322883159/p/60072104).
</content>
