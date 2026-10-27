---
title: Guía de estilo de documentos
lastUpdate: 2026-10-24
lang: es
category: guia
status: published
---

## Propósito

Convenciones de la casa para la documentación y los comentarios de API a nivel de código
fuente en toda la plataforma de Salamandra Devs. Estas reglas aplican a cada contribuyente,
humano o agente. El estilo de código vive en la guía de
[Estilo de código](./estilo-codigo.md).

Cada sección de regla lleva un ID estable (`DS-NN`) para su citación; los IDs siguen
[IDs de documento](./convenciones-nombres.md#ids-de-documento).

## Documentos Markdown

Pautas para escribir y mantener documentación, documentos de planificación y
especificaciones de diseño en toda la plataforma.

### DS-01 Estilo Markdown

- Ajuste las líneas de prosa ordinaria en los archivos Markdown a un máximo de 92
  caracteres.
- Preserve los bloques de código, tablas, encabezados y árboles de directorios al reflujar
  el texto. Esta regla de preservación tiene precedencia sobre la regla de longitud de línea
  de la prosa.
- Prefiera las listas (viñetas y listas numeradas) antes que las tablas en la documentación
  y los documentos de planificación. Use tablas solo cuando una comparación estructurada de
  múltiples atributos entre muchos ítems mejore genuinamente la claridad; por ejemplo,
  tablas de referencia de configuración o matrices de funcionalidades de proveedores. Cuando
  los ítems tienen de uno a tres atributos clave, descríbalos en forma de lista en su lugar:
  `- Nombre del ítem: atributo 1, atributo 2, atributo 3.`
- Use punto y coma en lugar de rayas (—), sin espacio antes del punto y coma. Prefiera
  `texto; más texto` antes que `texto — más texto` para la claridad y la consistencia en el
  procesamiento de texto plano.
- Cuando un ítem de lista comienza con un término principal (una etiqueta en negrita, un
  nombre o un identificador de código) seguido de su definición o descripción, sepárelos con
  dos puntos, no con punto y coma:
  `- **Factory**: selecciona y construye una implementación concreta.` Reserve el punto y
  coma para unir cláusulas dentro de una oración.

### DS-02 Redacción

- Evite la palabra "canonical" (y la española "canónico"); se usa rara vez en español y se
  lee como jerga. Prefiera un calificador llano como "id de servicio", "referencia" u
  "oficial", o descarte el calificador cuando la regla circundante ya lleve el significado.
  Consérvela solo cuando sea un identificador literal (una bandera de CLI, un nombre de
  campo) o el nombre del término de un estándar externo, como la forma de texto UUID
  canónica.
- Evite la palabra "entity" (y la española "entidad"); la plataforma ya no tiene una capa de
  entidad. Nombre el sustantivo sobre el que trata un módulo de ruta o de registro como un
  "recurso" (`users`, `sessions`), y use `{Resource}` / `{resource}` en los patrones de
  nombres (`{Resource}Record`, `{resource}ListQuerySchema`). Llame "registro" a los datos
  persistidos y "manejador de registros" a su clase de almacenamiento. No use "subject" para
  esto: ya significa el principal autenticado (`sub` del JWT, `SessionRecord.subject`).
  Conserve "entity" solo en nombres literales de código anterior (`entityId`, `Entities`,
  `#entities`), en citas de fuentes externas y en términos de proveedor como una entidad de
  identidad de Vault.

### DS-03 Planes y requisitos de diseño

- No cree diagramas de flujo en los planes ni en los requisitos de diseño.
- Use encabezados descriptivos y listas para estructurar los documentos de diseño.
- Referencie los principios arquitectónicos y los patrones existentes de
  `docs/en/architecture/`.
- Enlace los documentos de planificación y los artefactos de implementación relacionados
  cuando sea apropiado.
- Mantenga los metadatos del frontmatter para la capacidad de búsqueda y el seguimiento del
  ciclo de vida del documento.

### DS-04 Frontmatter para archivos docs/

Cada archivo Markdown creado en el directorio `docs/` debe incluir frontmatter YAML en la
parte superior con los siguientes campos requeridos:

```yaml
---
title: Título breve y descriptivo
lastUpdate: YYYY-MM-DD
lang: en o es
category: architecture, infrastructure, planning u other
status: <uno de los valores de estado de abajo>
owner: Nombre (opcional, recomendado para documentos activos)
---
```

Descripciones de los campos:

- `title`: descripción de una línea del propósito del documento. Ejemplos: "Auth Service
  Architecture Build Plan", "SOLID Principles", "Durable Signing Key Provider Implementation
  Plan".
- `lastUpdate`: la fecha en que el documento se actualizó por última vez, en formato
  `YYYY-MM-DD`. Use `today` como marcador de posición al crear un documento nuevo;
  reemplácelo por la fecha real antes de hacer commit.
- `lang`: el idioma del documento. Use `en` para inglés o `es` para español. Mantenga
  archivos separados para cada idioma en `docs/en/` y `docs/es/` respectivamente.
- `category`: una clasificación para el documento. Ejemplos: `architecture`,
  `infrastructure`, `planning`, `guide`, `operations`. Use snake_case si es de varias
  palabras.
- `status`: el estado actual del documento. Véase
  [Valores de estado](#ds-05-valores-de-estado).
- `owner`: (opcional) la persona o el equipo responsable de mantener este documento.
  Recomendado para documentos activos y en evolución. Formato: `Nombre Apellido` o `Nombre
  del Equipo`.

### DS-05 Valores de estado

El campo `status` del frontmatter usa uno de los siguientes valores. Úselos de forma
consistente para señalar la completitud y la fiabilidad del documento.

Estados del ciclo de vida del documento:

- `published`: el documento está completo, revisado y listo para su referencia. El contenido
  es estable y preciso. Ejemplo: `auth-plan.md` con decisiones de arquitectura que han sido
  implementadas.
- `active`: el documento es actual y se mantiene activamente. El contenido puede evolucionar
  a medida que la plataforma cambia. Usado para guías vivas, runbooks operativos y hojas de
  ruta que permanecen sincronizados con la implementación. Ejemplo: `ROADMAP.md` que rastrea
  el estado de las funcionalidades entre servicios.
- `draft`: el documento está incompleto y todavía se está escribiendo. No recomendado para
  referencia externa; destinado solo a la discusión del equipo.
- `archived`: el documento describe un enfoque obsoleto o una iniciativa completada. El
  contenido se preserva por referencia histórica pero no debería guiar trabajo nuevo.
- `proposed`: el documento describe un cambio planificado o una funcionalidad nueva en
  discusión. El contenido puede ser especulativo y estar sujeto a revisión antes de que
  comience la implementación.

Las etiquetas de estado de funcionalidad (`implementado`, `planificado`, `propuesto`,
`cancelado`) no son valores del frontmatter; marcan funcionalidades individuales en línea.
Sus definiciones, el formato de línea del roadmap y las reglas de edición del roadmap viven
en [el README del roadmap](../roadmap/README.md).

### DS-06 Ejemplos de frontmatter

Documento de planificación de arquitectura:

```yaml
---
title: Auth Session Issuance Implementation Plan
lastUpdate: 2026-09-05
lang: en
category: architecture
status: published
owner: Odair Trujillo
---
```

Runbook operativo:

```yaml
---
title: Mail Servers Backup and Sync
lastUpdate: 2026-09-20
lang: en
category: infrastructure
status: active
owner: DevOps Team
---
```

Traducción al español:

```yaml
---
title: Principios SOLID
lastUpdate: 2026-08-21
lang: es
category: arquitectura
status: published
owner: Odair Trujillo
---
```

Documento de roadmap:

```yaml
---
title: Salamandra Devs Platform Roadmap
lastUpdate: 2026-09-22
lang: en
category: planning
status: active
---
```

### DS-07 Cuándo actualizar lastUpdate

- Actualice `lastUpdate` siempre que el contenido del documento cambie de forma material.
- Las correcciones menores de formato o erratas no requieren actualizar la fecha.

### DS-08 Traducir entre idiomas

- Mantenga los nombres de los campos del frontmatter en inglés (`title`, `lastUpdate`,
  `lang`, `category`, `status`, `owner`).
- Traduzca el valor de `title` al idioma del documento.
- Use el campo `lang` para indicar el idioma (`en` o `es`).
- Al traducir un documento existente, use la misma fecha de `lastUpdate` que la versión en
  inglés para mostrar que son traducciones sincronizadas.
- Si una versión traducida se actualiza después de la inglesa, suba su `lastUpdate` a la
  fecha nueva.

## Comentarios de código fuente para TypeDoc

TypeDoc genera el sitio de documentación de API de las librerías de la plataforma a partir de
los comentarios TSDoc `/** */` en las declaraciones exportadas.

### DS-09 Comentarios de documentación y comentarios de línea

- Escriba comentarios de documentación TSDoc `/** */` en cada declaración exportada (clase,
  función, método, propiedad, tipo, interfaz, enum) para que el sitio generado documente la
  superficie pública.
- Solo un comentario `/** */` adjunto a una declaración es documentación. Las notas de
  implementación que viven dentro del cuerpo de una función o método permanecen como
  comentarios de línea `//`; nunca convierta una nota de cuerpo en un `/** */` flotante, que
  no se adjunta a ninguna declaración y que `eslint-plugin-jsdoc` marca.

```ts
export class DatabaseAdapter {
  /** Ejecuta el DDL de dataset e índice a través del `DatasetProvisioner` del backend. */
  private readonly provisioner: IDatasetProvisioner<DatasetDef>;

  constructor(handle: DatabaseHandle) {
    // La etiqueta discriminada `backend` selecciona el driver una vez; esta nota de cuerpo
    // permanece como comentario de línea.
    switch (handle.backend) {
      // ...
    }
  }
}
```

- Los miembros privados (`private` o `#`) se excluyen del sitio generado
  (`excludePrivate: true`). Coméntelos solo como ayudas para leer el código y no cuente con
  que se rendericen.
- Las líneas de comentario están limitadas a 92 caracteres, incluyendo la indentación
  inicial y los marcadores de comentario.

### DS-10 Etiquetas

Use las siguientes etiquetas como estilo de la casa, consistentes con los comentarios que el
repositorio ya escribe:

- `@param name - description`, usando la forma con guion.
- `@returns` (no `@return`) para un retorno no void, sin guion.
- `@throws {CustomError} <condición>` para cada vía de fallo, nombrando el `ErrorName`
  relevante en el texto de la condición. El tipo `{CustomError}` lo requiere la regla
  `jsdoc/require-throws-type` y coincide con la regla de que cada error lanzado es un
  `CustomError` con un `ErrorName` deliberado.
- `@remarks` para el detalle más allá del resumen de una línea; `@example` con un bloque de
  código delimitado para el uso.
- `{@link Symbol}` para las referencias cruzadas en lugar de un identificador desnudo.
- `@category` o `@group` para organizar la barra lateral generada; `@internal` para mantener
  fuera del sitio un símbolo exportado pero no público.
- No reitere un tipo que TypeScript ya codifica; nunca añada una anotación de tipo
  `@param {string}`.

### DS-11 Documentación de tipos y esquemas

- Para los tipos no derivados de un esquema Zod, no adjunte comentarios JSDoc por campo a las
  propiedades individuales dentro de un tipo. En su lugar, escriba un solo bloque JSDoc sobre
  el tipo mismo, con una línea de viñeta por campo que resuma su propósito en una línea.
  Mantenga toda la documentación consolidada a nivel del tipo en lugar de dispersa en línea.

```ts
// Evitar
export type SessionGrant = Readonly<{
  /** Identificador de sesión del lado del servidor. */
  sessionId: string;
  /** Actor local representado por la sesión. */
  actorId: string;
}>;

// Preferir
/**
 * Identidad confiable reconstruida a partir de un token de acceso verificado.
 * - sessionId: Identificador de sesión del lado del servidor.
 * - actorId: Actor local representado por la sesión.
 */
export type SessionGrant = Readonly<{
  sessionId: string;
  actorId: string;
}>;
```

- Como la excepción específica de Zod a la regla anterior, ponga el JSDoc detallado a nivel
  de campo en la constante de esquema y mantenga breve el JSDoc del tipo inferido. El JSDoc
  del esquema lleva las viñetas que de otro modo habrían vivido en el tipo codificado a mano.
  Ambos alimentan a TypeDoc tal como están escritos.

```ts
/**
 * Esquema del identificador de registro de ruta de ítem.
 * - recordId: Identificador de registro direccionado por la ruta.
 */
export const recordParamsSchema = z
  .object({ recordId: z.uuid() })
  .strict();

/** Parámetros de ruta de registro aceptados por las rutas de ítem. */
export type RecordParams = z.infer<typeof recordParamsSchema>;
```

### DS-12 Descripción (resumen) y `@remarks`

- La descripción es el párrafo principal sin etiqueta del comentario de documentación. TSDoc
  no tiene una etiqueta `@description` y el estilo de la casa no añade una; el párrafo de
  resumen es la descripción. Manténgalo en una línea que enuncie qué es o qué hace el
  símbolo.
- `@remarks` lleva el detalle más allá de ese resumen de una línea: mecanismo, invariantes,
  ciclo de vida, idempotencia y los colaboradores involucrados. Ponga la explicación extensa
  aquí, nunca en el resumen.
- Para una clase que `implements` un contrato, escriba `@remarks` priorizando el rol: empiece
  por lo que la clase hace o cómo funciona. No abra con una reformulación apositiva como "The
  shipped {@link IContract}." o "The backend-neutral {@link IContract}:". La cláusula
  `implements`, más el propio puntero de la interfaz de vuelta a su implementación, ya
  documentan el enlace clase-a-contrato, así que repetirlo en los remarks de la clase es
  redundante.
- Mantenga el puntero de contrato-a-implementación en el lado de la interfaz (en los propios
  `@remarks` de la interfaz, por ejemplo "Implemented by {@link DatabaseProvider}" o
  "{@link DatabaseProvider} is the shipped implementation"), no duplicado en la clase.

```ts
// Evitar; reitera el enlace que la cláusula `implements` ya codifica.
/**
 * Provisioning workflow that converts schemas and coordinates dataset setup.
 *
 * @remarks
 * The backend-neutral {@link IDatasetProvisioner}: it converts a dataset's Zod schema
 * through the injected {@link SchemaConverter}, then drives an {@link IDataset}.
 */
export class DatasetProvisioner ... implements IDatasetProvisioner<TDef> {}

// Preferir; el resumen nombra el rol, `@remarks` empieza por el mecanismo.
/**
 * Provisioning workflow that converts schemas and coordinates dataset setup.
 *
 * @remarks
 * Converts a dataset's Zod schema through the injected {@link SchemaConverter}, then
 * drives an {@link IDataset} to create the dataset and its indexes.
 */
export class DatasetProvisioner ... implements IDatasetProvisioner<TDef> {}
```

### DS-13 Referencias cruzadas con `{@link}`

- Use `{@link Symbol}` para cualquier referencia a otro símbolo exportado, en lugar de un
  identificador desnudo o un nombre entre comillas invertidas, para que TypeDoc renderice un
  enlace navegable en el sitio generado.
- `{@link Symbol}` renderiza el nombre del símbolo; use `{@link Symbol | texto
  personalizado}` solo cuando la prosa circundante necesite un texto de enlace distinto.
- Enlace los colaboradores a los que quien lee saltaría realmente: el contrato que una clase
  implementa, los tipos que una función consume o devuelve, y el hermano que consume su
  salida.
- `{@link}` resuelve solo símbolos exportados. No enlace miembros privados ni locales no
  exportados; se excluyen del sitio generado y el enlace no resolverá.

```ts
/**
 * Returns the connected, backend-tagged database handle.
 *
 * @returns Connected database handle.
 * @throws {CustomError} DB_BACKEND_ERROR when called before
 *   {@link DatabaseProvider.connect}.
 */
getHandle(): DatabaseHandle;
```
</content>
