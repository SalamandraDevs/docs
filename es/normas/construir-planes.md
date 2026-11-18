---
title: Construir planes
lastUpdate: 2026-11-18
lang: es
category: guia
status: published
---

## Propósito

Reglas de la casa para escribir planes de construcción: los documentos que describen un build
nuevo, como un servicio, paquete o capacidad nuevos, o un cambio a código existente, antes de
que se implemente, para que quien revisa pueda aprobar el enfoque y un agente ejecutor pueda
llevarlo a cabo sin adivinar. Esta guía llama a cualquiera de los dos "el trabajo". Estas
reglas aplican a cada contribuyente, humano o agente, y a cada plan bajo `docs/plans/`.

Un plan se fundamenta en los documentos de arquitectura y de la casa. No los reitera; los
cita, los aplica y registra cualquier desviación deliberada:

- Arquitectura: [Principios SOLID](../arquitectura/principios-solid.md) (`SP`),
  [Patrones de diseño](../arquitectura/patrones-diseno.md) (`DP`) y
  [Buenas prácticas de desarrollo](../arquitectura/buenas-practicas-desarrollo.md) (`GP`).
- Casa: [Estilo de código](./estilo-codigo.md) (`CS`),
  [Guía de estilo de documentos](./guia-estilo-documentos.md) (`DS`),
  [Construir servicios](./construir-servicios.md) (`BS`) y
  [Convenciones de nombres](./convenciones-nombres.md).

Un plan es un documento Markdown y sigue la
[Guía de estilo de documentos](./guia-estilo-documentos.md) por completo; este documento solo
añade lo específico de los planes.

Cada sección de regla lleva un ID estable (`BP-NN`) para su citación; los IDs siguen
[Convenciones de nombres, IDs de documento][naming-document-ids].

Los enlaces de esta guía son relativos a `docs/es/normas/`. Los ejemplos del lado del plan,
dentro de los bloques de código, usan rutas relativas a `docs/plans/`; copie esas, nunca los
destinos de los propios enlaces de esta guía.

## Escribir el documento

### BP-01 Ubicación, nombre y estado

- Guarde los planes en `docs/plans/` como `{YYMMDD}-{tema}-plan.md`, en minúsculas y
  kebab-case (`261029-messages-plan.md`, `261030-session-plan.md`). `{YYMMDD}` es la fecha en
  que se creó el plan, de modo que los listados de archivos ordenen por antigüedad; no cambia
  cuando el plan se edita después, lo cual registra `lastUpdate`.
- El `status` de un plan recorre estos valores, en este orden, y nunca salta ninguno:
  1. `proposed`: el plan está escrito y en revisión; sus decisiones todavía pueden cambiar.
  2. `approved`: la revisión aceptó el plan, antes de que empiece la implementación. Implemente
     solo un plan aprobado; un cambio a una decisión durante la implementación vuelve a pasar
     por revisión y se registra en el plan.
  3. `published`: el trabajo está implementado, y el plan describe lo que se construyó.
  4. `archived`: el plan ya no guía el trabajo, por ejemplo porque un plan posterior reemplaza
     su enfoque. Un plan terminado es `published`, no `archived`.
- Siga el frontmatter con un encabezado `#` que repita el valor de `title`.

### BP-02 Secciones obligatorias

Cada plan tiene estas secciones de nivel superior, en este orden:

1. `## Current Status` con una subsección `### What's the Problem`.
2. `## Goal`, con una subsección `### Decisions`.
3. `## Implementation Details`.
4. `## Acceptance Criteria`.
5. `## Out of Scope`.
6. `## Follow-ups`.

Mantenga los nombres de las secciones exactamente como están escritos para que los planes se
lean igual y puedan buscarse. Añada subsecciones libremente dentro de una sección; no añada
secciones de nivel superior entre ellas. Una sección que no tiene nada que decir igual
aparece, con una línea que lo declara.

El bloque de definiciones de enlace ([BP-03 Citar las reglas de arquitectura y de la
casa][bp-03]) cierra el archivo, después del contenido de Follow-ups. No es una sección y no
tiene encabezado.

### BP-03 Citar las reglas de arquitectura y de la casa

Un plan cita cada regla de arquitectura y de la casa en la que se apoya, para que quien revisa
pueda comprobar cada afirmación contra su fuente sin adivinar qué regla se refiere, y cada
citación es un enlace que abre la regla.

- **Primera citación**: el ID y el título de la regla, exactamente como se lee su encabezado
  de sección, escrito como un enlace de estilo referencia cuya etiqueta es el ID en
  minúsculas:
  `[GP-03 Contratos, cableado en la raíz de composición y backends configurados][gp-03]`.
- **Citas posteriores**: solo el ID, como un enlace de atajo: `[GP-03]`. Las etiquetas
  coinciden sin importar el caso, así que resuelve a la misma definición. Nunca escriba un ID
  sin su enlace.
- **Definiciones**: cada etiqueta se define una vez, en un bloque al final del plan, ordenado
  por etiqueta. Las definiciones no se renderizan, así que su posición no le importa a quien
  lee; mantenerlas juntas significa que mover o editar texto nunca rompe un enlace.
- **Rutas**: relativas al archivo del plan, así que comienzan con `../es/`:
  `[gp-03]: ../es/arquitectura/buenas-practicas-desarrollo.md#gp-03-...`.
- **Reglas sin un ID**: cítelas por documento y nombre de sección, con una etiqueta de estilo
  referencia hecha del nombre del documento y el ancla de la sección:
  `[Convenciones de nombres, Estructura de servicios][naming-service-structure]`. Las citas
  posteriores pueden descartar el nombre del documento y conservar la etiqueta:
  `[Estructura de servicios][naming-service-structure]`.
- **Entradas de roadmap**: el ID de la entrada como texto de enlace, con una etiqueta en
  minúsculas: `[AUTH-011][auth-011]`, definido a la sección del roadmap que lista la entrada.
- **Longitud de línea**: las líneas de definición están exentas del límite de 92 caracteres de
  [DS-01 Estilo Markdown][ds-01], porque sus URLs no pueden ajustarse. La exención cubre los
  planes y esta guía. Las líneas de prosa que citan permanecen dentro del límite; los enlaces
  de estilo referencia las mantienen cortas.
- Separe varias citaciones con punto y coma, ya que algunos títulos contienen comas:
  `[GP-03]; [DP-01 Factory][dp-01]`.

Un plan que cita una regla dos veces y otra una vez:

```markdown
### What's the Problem

- Los proveedores leen una config global a nivel de módulo; esto rompe
  [GP-03 Contratos, cableado en la raíz de composición y backends configurados][gp-03].

## Goal

### Decisions

1. **Los senders se construyen en la raíz de composición**: `bootstrapMessaging` los crea.
   - Reglas: [GP-03]; [DP-08 Composition Root][dp-08].

## Follow-ups

- ...

[dp-08]: ../es/arquitectura/patrones-diseno.md#dp-08-composition-root
[gp-03]: ../es/arquitectura/buenas-practicas-desarrollo.md#gp-03-contratos-cableado-en-la-raíz-de-composición-y-backends-configurados
```

Dónde se requieren citaciones:

- En `### What's the Problem`, junto a cada comportamiento del código actual que rompe una
  regla ([BP-04 Current Status][bp-04]).
- En cada decisión, en su línea `Rules:` ([BP-05 Objetivo y decisiones][bp-05]).
- En cada decisión que rompe una regla, en una línea `Deviation:`, y de nuevo bajo Follow-ups
  ([BP-09 Seguimientos][bp-09]).
- En línea en Implementation Details y Acceptance Criteria, dondequiera que un paso o una
  comprobación apliquen una regla.

## Secciones del plan

### BP-04 Estado actual

Describe lo que está en su lugar hoy, antes de que empiece el trabajo: para un build nuevo, el
contexto en el que aterriza; para un cambio a código existente, cómo funciona el tema
relacionado. Un plan puede cubrir ambos, como un cambio a un servicio seguido de su extracción
a un paquete nuevo; entonces describa cada parte como abajo.

- Enuncie hechos verificados leyendo el código, la configuración y los archivos de despliegue,
  no suposiciones.
- Referencie el código por ruta y símbolo: `services/auth/src/infra/keys/factory.ts`,
  `createKeysProvider`. Las rutas comienzan en la raíz del espacio de trabajo, el directorio
  que contiene `docs/`, `core/`, `tools/`, `services/` y `legacy/`, porque cada proyecto de
  código bajo él es su propio repositorio. Añada `:line` solo a la evidencia de un defecto o
  una violación en What's the Problem, donde fija el comportamiento exacto; los números de
  línea se desvían en cuanto se edita el código.
- Para un build nuevo, como un servicio, paquete o capacidad nuevos sin código todavía,
  describa el contexto en el que aterriza: las partes de la plataforma de las que dependerá o
  con las que se integrará, los consumidores que lo esperan, los proyectos existentes cuyo
  layout y patrones debería reflejar, cualquier predecesor legacy, y las restricciones externas
  como APIs de proveedor, cuotas o límites de sandbox. Enuncie las restricciones como hechos
  aquí; Acceptance Criteria enuncia solo los umbrales que el trabajo terminado debe cumplir
  ([BP-07 Criterios de aceptación][bp-07]).
- Para un cambio a código existente, cubra las partes que el trabajo tocará: módulos y sus
  contratos, datos y almacenamiento, interfaces públicas (endpoints, comandos, eventos,
  exports de librería), configuración y variables de entorno, manifiestos de despliegue, y
  cualquier código legacy que se reemplace.
- Para el código legacy que se reemplaza, describa lo que hace aquí; ponga sus defectos y
  violaciones de reglas en What's the Problem, y cómo el agente ejecutor debe tratarlo (por
  ejemplo, solo como referencia, nunca copiado) bajo Rules for the executing agent
  ([BP-06 Detalles de implementación][bp-06]).
- Nombre los patrones existentes que el trabajo debe seguir, cada uno con el archivo que lo
  muestra, citados como exige [BP-03].
- No tome decisiones aquí.

Un patrón nombrado en Current Status:

```markdown
- **Selección de proveedor**: `createKeysProvider`
  (`services/auth/src/infra/keys/factory.ts`) selecciona el proveedor nombrado por
  `SIGNING_KEY_PROVIDER`, siguiendo [DP-01 Factory][dp-01].
```

`### What's the Problem` enuncia por qué se necesita el trabajo. Incluya los ítems que
apliquen; un build o funcionalidad nuevos pueden no corregir ningún defecto ni romper ninguna
regla:

- **Necesidad nueva o capacidad faltante**: qué falta y la evidencia, como una entrada de
  roadmap (`[AUTH-011][auth-011]`).
- **Defecto o brecha**: la vía que falla o el comportamiento erróneo, con la evidencia.
- **Violaciones de reglas**: cuando el código actual rompe una regla de arquitectura o de la
  casa, cítela junto al comportamiento que la rompe, como exige [BP-03].
- **Impacto**: a quién o qué afecta, y el coste de dejarlo como está.

Enuncie al menos uno de los tres primeros ítems, y enuncie siempre el impacto.

Una violación de regla con su evidencia:

```markdown
- Los proveedores leen un `localConfig` global a nivel de módulo
  (`legacy/emailservice/src/providers/smtp.ts:11`); esto rompe
  [GP-03 Contratos, cableado en la raíz de composición y backends configurados][gp-03].
```

### BP-05 Objetivo y decisiones

`## Goal` abre con un párrafo breve: el resultado que entrega el plan y su límite.

`### Decisions` es una lista numerada. Cada ítem es una decisión, ya sea que el humano la tomó
o que el autor la propone; el plan es el único registro de ellas, y la revisión del plan las
aprueba o las cambia:

- Para un build nuevo, las primeras decisiones son sus fundacionales: dónde vive
  ([GP-01 Un repositorio por paquete o servicio][gp-01]), su nombre e identificadores bajo
  [Convenciones de nombres](./convenciones-nombres.md), sus contratos y límites con el resto
  de la plataforma, y el lenguaje, framework y proveedores que usa.
- Comience cada decisión con un término principal en negrita y dos puntos, luego la decisión y
  su razón: `1. **Los contratos viven en el paquete de transporte**: ...`.
- Termine con una línea `Rules:` que liste cada regla de arquitectura y de la casa que la
  decisión aplica, citada como exige [BP-03]:

  ```markdown
  1. **Los contratos viven en el paquete de transporte**: decisión y razón.
     - Rules: [SP-05 Principio de inversión de dependencias (DIP)][sp-05]; [GP-02];
       [DP-02 Adapter][dp-02].
  ```

- Mencione una alternativa rechazada en una oración cuando explique la elección; las opciones
  se sopesan antes de escribir el plan, y el plan registra el resultado.
- Cuando una decisión deba romper una regla, añada una línea `Deviation:` que cite la regla y
  dé la razón, y lístela de nuevo bajo Follow-ups para su aprobación. Nunca rompa una regla en
  silencio.
- Numere las decisiones una vez y nunca las renumere; las secciones posteriores las citan como
  "decisión 3".

### BP-06 Detalles de implementación

La sección que un agente ejecutor sigue paso a paso. Contiene dos subsecciones fijas, luego
las etapas:

1. `### Rules for the executing agent`: qué documentos leer, qué archivos existentes reflejar,
   los límites de dependencia entre capas, qué comandos del proyecto ejecutar, cómo tratar el
   código legacy, y cuándo detenerse y preguntar.
2. `### Interface layout`: cada interfaz pública que el trabajo añade o altera, en todas las
   etapas (endpoints HTTP, comandos de CLI, eventos, exports de librería, claves de
   configuración, variables de entorno), con su nombre o ruta, entradas, salidas y errores.
   Las rutas HTTP siguen [Convenciones de nombres, APIs][naming-apis]. Cuando el trabajo no
   cambia ninguna interfaz pública, decláralo en una línea.
3. Una o más etapas (`### Stage 1: senders inside auth`), cada una con pasos ordenados
   numerados desde 1 (`#### Step 1: dependencies`). Un plan con una etapa igual la nombra.

Los pasos siguen el orden de dependencia dentro de cada etapa y cubren, donde apliquen:

- **Scaffolding**: el primer paso de una etapa que crea un repositorio o servicio nuevo.
  Configura el layout de código fuente, el manifiesto de dependencias y el lockfile, el
  toolchain fijado, las puertas de calidad en CI, la imagen de contenedor y los ajustes de
  publicación, antes de cualquier trabajo de funcionalidad en ese repositorio. Cuando el
  repositorio nuevo viene antes que todo el resto del trabajo, su etapa es
  `### Stage 0: scaffolding`. Aplica [GP-01];
  [GP-05 Dependencias bloqueadas y toolchain fijado][gp-05];
  [GP-07 Salvaguardas de la cadena de suministro][gp-07]; y
  [GP-13 Las puertas de calidad se ejecutan en CI][gp-13].
- **File layout**: en cada etapa que añade o mueve archivos, el primer paso después de
  cualquier scaffolding; un árbol de directorios de archivos nuevos y cambiados, con el
  elemento principal que contiene cada uno (una clase, módulo o función). Los nombres siguen
  [Convenciones de nombres](./convenciones-nombres.md).
- **Code examples**: suficientes para eliminar las conjeturas, comenzando cada bloque con un
  comentario que nombra su archivo destino por ruta completa. Para un formato sin comentarios,
  como JSON, nombre el archivo destino en la oración antes del bloque. Los ejemplos son código
  que el agente copiará, así que ya deben seguir las reglas de estilo de código y de
  comentarios de fuente para el lenguaje destino; para TypeScript,
  [Estilo de código](./estilo-codigo.md) y
  [DS-09 Comentarios de documentación y comentarios de línea][ds-09] a
  [DS-13 Referencias cruzadas con `{@link}`][ds-13]. Muestre un ejemplo completo por patrón y
  extractos para sus repeticiones.
- **Configuration and environment**: cambios de esquema o formato de configuración, marcadores
  de posición de secretos, variables de entorno nuevas, y cada manifiesto de despliegue que
  debe pasarlas ([GP-08 Configuración validada, fallo rápido][gp-08];
  [GP-09 Secretos y credenciales solo desde el entorno][gp-09];
  [GP-10 La configuración varía por despliegue, no por build][gp-10]).
- **Composition root**: dónde se construyen y cablean los componentes nuevos
  ([DP-08 Composition Root][dp-08];
  [GP-03 Contratos, cableado en la raíz de composición y backends configurados][gp-03]).
- **Documentation changes**: actualizaciones de casa, arquitectura, roadmap y traducción que
  el trabajo requiere.

Los comandos de validación y las comprobaciones en runtime pertenecen a Acceptance Criteria,
no a un paso final. Cite las reglas en línea, como exige [BP-03], donde un paso las aplica.
Cuando un ejemplo depende de una API de terceros, dígale al agente que compruebe esa API en la
versión instalada (sus tipos, headers o documentación de referencia) antes de copiar el
ejemplo.

### BP-07 Criterios de aceptación

Condiciones que quien revisa o un agente puede comprobar para dar el trabajo por terminado.
Cada una es concreta: un comando, un comportamiento observable o un número medible.

- **Validation**: los comandos de formato, lint, comprobación de tipos o compilación, y build
  del proyecto que deben pasar, más las comprobaciones del repositorio tomadas de la línea
  **Check:** de cada práctica GP que el plan aplica, acotadas a los archivos, manifiestos y
  paquetes que el trabajo añade o cambia; por ejemplo, ningún archivo de la capa de transporte
  importa del almacenamiento, para [GP-02 Estructura de servicio en capas][gp-02]. Una
  comprobación que ya falla por razones anteriores al trabajo no es un criterio de aceptación;
  lístela bajo Follow-ups como una brecha ([BP-09]).
- **Verification**: comportamiento en runtime a confirmar, como que el servicio se niega a
  arrancar con un error de configuración clasificado cuando una variable requerida no está
  definida ([GP-08]), o que un endpoint devuelve una respuesta de error dada.
- **Timings and capacities**: presupuestos de latencia, timeouts, límites de tasa, cuotas o
  volúmenes que el trabajo terminado debe cumplir o respetar, solo cuando el trabajo depende de
  ellos (por ejemplo, el límite de tasa de un endpoint,
  [BS-01 Valores seguros de transporte por defecto][bs-01]). Refiérase a los límites de
  proveedor enunciados en Current Status en lugar de repetirlos. Cuando ninguno aplica,
  decláralo en una línea.
- **Tests**: solo cuando el humano los pidió explícitamente. Entonces nombre las pruebas, la
  costura de contrato que ejercitan, y el comando que las ejecuta
  ([GP-24 Pruebas en las costuras de contrato][gp-24]). Si no, escriba una línea que declare
  que las pruebas están fuera de alcance.

### BP-08 Fuera de alcance

Una lista de lo que el plan deliberadamente no hace, cada ítem con una razón de una línea o un
puntero a dónde se maneja (otro plan, una entrada de roadmap). Incluya el trabajo que quien lee
podría esperar razonablemente y que el plan deja fuera, como flujos relacionados, pruebas no
solicitadas o código legacy dejado en su lugar.

### BP-09 Seguimientos

Lo que el agente considera pendiente para que el humano revise, decida o programe:

- Actualizaciones de registro y documentación que un build nuevo requiere, donde existe un
  registro: un id de servicio bajo [Convenciones de nombres, Servicios][naming-services],
  entradas de roadmap (y, para un repositorio fuera del alcance listado en
  [Reglas del roadmap](../roadmap/README.md), la extensión de ese alcance), enlaces de índice
  en la documentación, e instrucciones de agente. Cuando ningún registro cubre el build nuevo,
  como un nombre de paquete, decláralo como una pregunta abierta.
- Preguntas abiertas que el plan no pudo resolver, y valores que el agente no conoce (hosts,
  regiones, direcciones de sender).
- Desviaciones de reglas de arquitectura o de la casa que necesitan aprobación ([BP-05]), cada
  una con la regla citada, o una línea que declare que no hay ninguna.
- Brechas encontradas en código existente al escribir el plan que están fuera de su alcance,
  cada una con la regla que rompe citada, incluyendo comprobaciones del repositorio que ya
  fallan ([BP-07]).
- Trabajo que se vuelve posible o necesario después de que el plan aterrice: traducciones y
  pasos operativos como solicitudes de acceso a proveedores.

## Revisar el plan

### BP-10 Revisión de cumplimiento antes de la entrega

Antes de presentar un plan, el autor lo contrasta con los documentos de arquitectura y de la
casa y corrige lo que no cumple:

- Cada decisión lleva su línea `Rules:`, cada regla que el código actual rompe se cita en
  `### What's the Problem`, y cada citación sigue [BP-03], apunta a una regla que existe, y
  dice lo que el plan afirma.
- Cada etiqueta tiene exactamente una definición en el bloque de cierre, cada definición se
  usa, y cada ruta comienza con `../es/`.
- El documento sigue la [Guía de estilo de documentos](./guia-estilo-documentos.md), en
  particular [DS-01]; [DS-02 Redacción][ds-02]; y
  [DS-04 Frontmatter para archivos docs/][ds-04].
- Los ejemplos de código siguen las reglas para el lenguaje destino listadas en [BP-06].
- Los nombres siguen [Convenciones de nombres](./convenciones-nombres.md) y las convenciones
  del lenguaje y el layout destino.
- La dirección de dependencia, la raíz de composición, la configuración, los secretos y los
  errores siguen [GP-02]; [GP-03]; [GP-08]; [GP-09]; y [GP-20 Errores clasificados][gp-20].
- Los patrones que ya están en la plataforma se reflejan, no se reinventan.
- Las desviaciones restantes se enuncian en su decisión y se listan bajo Follow-ups.

## Plantilla

```markdown
---
title: {Tema} Implementation Plan
lastUpdate: YYYY-MM-DD
lang: en
category: architecture
status: proposed
---

# {Tema} Implementation Plan

## Current Status

El contexto en el que aterriza un build nuevo, o lo que existe hoy para un cambio, con
referencias de ruta y símbolo y los patrones que el trabajo debe seguir.

### What's the Problem

Por qué se necesita el trabajo, con evidencia. Cite cada regla que el código actual rompe
junto al comportamiento que la rompe: "... esto rompe [XX-NN Título][xx-nn]".

## Goal

El resultado que entrega este plan y su límite.

### Decisions

1. **Término principal**: decisión y razón.
   - Rules: [XX-NN]; [YY-NN Título][yy-nn].
   - Deviation: [XX-NN]; razón (solo cuando la decisión rompe una regla).

## Implementation Details

### Rules for the executing agent

### Interface layout

### Stage 1: {nombre}

#### Step 1: {nombre}

## Acceptance Criteria

- Validation:
- Verification:
- Timings and capacities:
- Tests: fuera de alcance salvo que se soliciten.

## Out of Scope

## Follow-ups

[xx-nn]: ../es/{arquitectura|normas}/{documento}.md#xx-nn-título
[yy-nn]: ../es/{arquitectura|normas}/{documento}.md#yy-nn-título
```

[bp-03]: #bp-03-citar-las-reglas-de-arquitectura-y-de-la-casa
[bp-04]: #bp-04-estado-actual
[bp-05]: #bp-05-objetivo-y-decisiones
[bp-06]: #bp-06-detalles-de-implementación
[bp-07]: #bp-07-criterios-de-aceptación
[bp-09]: #bp-09-seguimientos
[bs-01]: ./construir-servicios.md#bs-01-valores-seguros-de-transporte-por-defecto
[dp-08]: ../arquitectura/patrones-diseno.md#dp-08-composition-root
[ds-01]: ./guia-estilo-documentos.md#ds-01-estilo-markdown
[ds-02]: ./guia-estilo-documentos.md#ds-02-redacción
[ds-04]: ./guia-estilo-documentos.md#ds-04-frontmatter-para-archivos-docs
[ds-09]: ./guia-estilo-documentos.md#ds-09-comentarios-de-documentación-y-comentarios-de-línea
[ds-13]: ./guia-estilo-documentos.md#ds-13-referencias-cruzadas-con-link
[gp-01]: ../arquitectura/buenas-practicas-desarrollo.md#gp-01-un-repositorio-por-paquete-o-servicio
[gp-02]: ../arquitectura/buenas-practicas-desarrollo.md#gp-02-estructura-de-servicio-en-capas
[gp-03]: ../arquitectura/buenas-practicas-desarrollo.md#gp-03-contratos-cableado-en-la-raíz-de-composición-y-backends-configurados
[gp-05]: ../arquitectura/buenas-practicas-desarrollo.md#gp-05-dependencias-bloqueadas-y-toolchain-fijado
[gp-07]: ../arquitectura/buenas-practicas-desarrollo.md#gp-07-salvaguardas-de-la-cadena-de-suministro
[gp-08]: ../arquitectura/buenas-practicas-desarrollo.md#gp-08-configuración-validada-fallo-rápido
[gp-09]: ../arquitectura/buenas-practicas-desarrollo.md#gp-09-secretos-y-credenciales-solo-desde-el-entorno
[gp-10]: ../arquitectura/buenas-practicas-desarrollo.md#gp-10-la-configuración-varía-por-despliegue-no-por-build
[gp-13]: ../arquitectura/buenas-practicas-desarrollo.md#gp-13-las-puertas-de-calidad-se-ejecutan-en-ci
[gp-20]: ../arquitectura/buenas-practicas-desarrollo.md#gp-20-errores-clasificados
[gp-24]: ../arquitectura/buenas-practicas-desarrollo.md#gp-24-pruebas-en-las-costuras-de-contrato
[naming-apis]: ./convenciones-nombres.md#apis
[naming-document-ids]: ./convenciones-nombres.md#ids-de-documento
[naming-services]: ./convenciones-nombres.md#servicios
