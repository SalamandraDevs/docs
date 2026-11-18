---
title: Estilo de código
lastUpdate: 2026-11-18
lang: es
category: guia
status: published
---

## Propósito

Estilo de código de la casa para TypeScript en toda la plataforma de Salamandra Devs. Estas
reglas aplican a cada contribuyente, humano o agente. Las reglas de comentarios de código
fuente viven en la [Guía de estilo de documentos](./guia-estilo-documentos.md), y cómo los
servicios Fastify componen los hooks de ruta vive en
[Construir servicios](./construir-servicios.md#hooks-de-ruta).

Cada sección de regla lleva un ID estable (`CS-NN`) para su citación; los IDs siguen
[IDs de documento](./convenciones-nombres.md#ids-de-documento). CS-09 a CS-12 están
retiradas: se trasladaron a BS-05 a BS-08.

## TypeScript

### CS-01 Importaciones

- Mantenga las declaraciones de import en una sola línea. Cuando un import excedería la
  longitud de línea configurada en Prettier, divídalo en varias declaraciones de import de
  una sola línea desde el mismo módulo en lugar de usar un bloque de import multilínea.
- Al editar imports, conserve la preferencia de agrupación actual del repositorio: primero
  los imports externos, luego los imports de tipo externos, luego una línea en blanco, luego
  los imports de valor internos, luego los imports de tipo, priorizando los imports por
  alias. Esto es estilo de la casa para la legibilidad, no una recomendación de TypeScript.

```ts
import Fastify from 'fastify';
import { serializerCompiler, validatorCompiler } from 'fastify-type-provider-zod';
import { CustomError } from '@salamandradevs/tools/shared';
import type { FastifyBaseLogger, FastifyInstance } from 'fastify';
import type { RawReplyDefaultExpression } from 'fastify';
import type { BaseRecord } from '@salamandradevs/core';

import { OAuthService } from '#application/oauth-service/oauth-service.js';
import { UsersService } from '#application/users-service/users-service.js';
import { UserRecordsHandler } from '#storage/users/index.js';
import type { IPasswordsService } from '#application/contracts.js';
import type { ServiceConfig } from '../config/types.js';
```

- No importe clases con `import type`, aunque actualmente se usen solo en posiciones de tipo.
  Importe las clases con imports de valor ordinarios para que su naturaleza en tiempo de
  ejecución quede clara.

```ts
// Evitar
import type { UsersService } from '../application/users-service/users-service.js';

// Preferir
import { UsersService } from '../application/users-service/users-service.js';
```

### CS-02 Esquemas Zod y tipos derivados

- No añada anotaciones de tipo TypeScript explícitas a las constantes de esquema Zod en los
  archivos de esquema. Los archivos de esquema deben apoyarse en el override global de
  ESLint que deshabilita `@typescript-eslint/typedef` para las definiciones de esquema. Si la
  excepción global no está presente, use la regla de alcance de archivo
  `/* eslint-disable @typescript-eslint/typedef */`.

```ts
// Evitar
export const recordParamsSchema: z.ZodType<RecordParams> = z
  .object({ recordId: z.uuid() })
  .strict();

// Preferir
export const recordParamsSchema = z
  .object({ recordId: z.uuid() })
  .strict();
```

- Prefiera derivar los tipos de los esquemas Zod con `z.infer` en lugar de codificar a mano
  formas TypeScript equivalentes cuando el esquema es la fuente de validación en tiempo de
  ejecución. Evite los imports circulares: los archivos de esquema no deben importar los
  tipos que se infieren de esos mismos esquemas.

```ts
// schema.ts
export const recordParamsSchema = z
  .object({ recordId: z.uuid() })
  .strict();

// types.ts
import type { z } from 'zod';

import type { recordParamsSchema } from './schema.js';

/** Parámetros de ruta de registro aceptados por las rutas de ítem. */
export type RecordParams = z.infer<typeof recordParamsSchema>;
```

- Use Zod para validación, proyección, normalización y serialización a nivel de forma cuando
  un esquema sea la fuente de verdad en tiempo de ejecución. Use funciones normales para
  comportamiento, orquestación y política. Evite los wrappers de conveniencia que solo
  renombran, reenvían o parsean a través de un esquema existente sin añadir transformación o
  validación real.

### CS-03 Wrappers

- No cree una función wrapper cuyo único propósito sea darle a una función existente un
  nombre distinto. Si un wrapper llama a otra función con los mismos argumentos, el mismo
  tipo de retorno y ninguna lógica añadida, es peso muerto; llame directamente a la función
  original en su lugar. Esta es una aplicación específica de TypeScript de la
  [regla general de wrappers de conveniencia](#cs-07-wrappers-de-conveniencia).

```ts
// Evitar
export async function hashPassword(password: string): Promise<string> {
  return argon2.hash(password);
}

// Preferir
const passwordHash: string = await argon2.hash(password);
```

- No cree tipos wrapper que solo renombren otro tipo sin añadir significado de dominio,
  validación o restricciones estructurales. Use el tipo original directamente.

```ts
// Evitar
export type UserId = string;

// Preferir
const recordId: string = request.params.recordId;
```

### CS-04 Errores

- Como la aplicación específica de errores de la regla general de wrappers de conveniencia,
  no cree funciones wrapper para lanzar errores; lance un `new CustomError()` en su lugar.

```ts
// Evitar
function throwAuthenticationFailed(): never {
  throw new CustomError({
    name: 'AUTHENTICATION_FAILED',
    message: 'Authenticated session grant is required'
  });
}

// Preferir
throw new CustomError({
  name: 'AUTHENTICATION_FAILED',
  message: 'Authenticated session grant is required'
});
```

- Cada error que el código de aplicación lanza o pasa a un callback debe ser un
  `CustomError` con un `ErrorName` deliberado. No se apoye en el mapeo de respaldo del
  manejador de errores de una API para clasificar un error. Un `Error` plano se permite solo
  dentro de un bloque `try` cuando su `catch` contenedor lo convierte inmediatamente en un
  `CustomError` y lo retiene como la causa.

```ts
try {
  const response: Response = await fetch(url);

  if (!response.ok) {
    throw new Error('Provider request failed');
  }
} catch (error: unknown) {
  throw new CustomError({
    name: 'EXTERNAL_API_ERROR',
    message: 'OAuth provider request failed',
    cause: error
  });
}
```

### CS-05 Archivos y módulos

- No escriba declaraciones de tipo en el mismo archivo que define una clase. Ponga esos
  tipos en un archivo hermano `types.ts` e impórtelos desde el archivo de implementación de
  la clase.

```ts
// users-service.ts
import type { CreateUserInput, UserView } from './types.js';

export class UsersService {
  async create(input: CreateUserInput): Promise<UserView> {
    // ...
  }
}

// types.ts
export type CreateUserInput = z.infer<typeof createUserInputSchema>;
```

- Cada `index.ts` de módulo debe ser un archivo barril usando `export * from './module.js'`.
  No use re-exports individuales de valor o de tipo en los archivos index de módulo.

```ts
// Evitar
export { UsersService } from './users-service.js';
export type { UserView } from './types.js';

// Preferir
export * from './users-service.js';
export * from './types.js';
```

## Reglas generales de estilo de código

### CS-06 Líneas en blanco

- Los bloques condicionales (if, switch, etc.), los bloques de bucle (for, for...of, while,
  etc.), try-catch y los bloques de declaración de función deben estar precedidos y seguidos
  por una línea en blanco para separarlos visualmente del código circundante. Trate una
  cadena if-else o una sentencia try-catch-finally como un solo constructo: añada líneas en
  blanco antes y después del constructo completo, no entre sus cláusulas conectadas. No se
  requiere una línea en blanco entre un constructo y su llave contenedora inmediata o un
  límite de archivo. Las sentencias return siempre deben estar precedidas por una línea en
  blanco, salvo cuando son la única sentencia dentro de su bloque contenedor.

```ts
const sessionId: string | undefined = request.sessionGrant?.sessionId;

if (!sessionId) {
  throw new CustomError({
    name: 'AUTHENTICATION_FAILED',
    message: 'Authenticated session grant is required'
  });
}

const session: SessionView = await services.sessions.read(sessionId);

return session;
```

### CS-07 Wrappers de conveniencia

- No cree wrappers de conveniencia que solo oculten un acceso a propiedad, devuelvan una
  entrada sin cambios o reenvíen a otra función sin añadir validación, transformación o
  política. Inline la operación directa en el sitio de llamada en su lugar. Las secciones de
  [Wrappers](#cs-03-wrappers) y [Errores](#cs-04-errores) proveen aplicaciones específicas
  para los wrappers de solo renombrar y de lanzar errores.
- Como excepción, un método que reenvía a un manejador u otro servicio únicamente para
  satisfacer un contrato publicado (una interfaz) a través de un límite arquitectónico no es
  un wrapper de conveniencia, aun con los mismos argumentos, el mismo tipo de retorno y
  ninguna lógica añadida; la delegación misma es el límite, e inlinearla en el sitio de
  llamada rompería el capas. Un ejemplo es un método de servicio de aplicación que cumple su
  interfaz delegando a un manejador de registros, de modo que un consumidor depende del
  contrato del servicio en lugar del almacenamiento.

```ts
// Evitar
function readSessionId(request: FastifyRequest): string {
  return request.sessionGrant.sessionId;
}

// Preferir
const sessionId: string = request.sessionGrant.sessionId;
```

### CS-08 Nombres de operaciones delegadas

- No renombre una operación delegada. Cuando un método o función solo reenvía a otra
  operación sin añadir validación, transformación, política u orquestación, déle el mismo
  nombre que la operación a la que delega; no introduzca un sinónimo que renombre el concepto
  en el límite. Una operación conserva un nombre al cruzar capas, igual que un `recordId`
  conserva su nombre del transporte al almacenamiento, de modo que quien lee puede rastrearlo
  y los conceptos no se remapean en silencio; un `deactivateItem` que simplemente llama a
  `deleteRow` oculta que elimina. Esto complementa la excepción de delegación de límite de
  arriba: la delegación se permite, y debe permanecer transparente. Cuando el método
  genuinamente añade comportamiento, nómbrelo por ese comportamiento en su lugar.

```ts
// Evitar
async revokeAllForSubject(subject: string): Promise<number> {
  return this.sessionRecordsHandler.deleteBySubject(subject);
}

// Preferir
async deleteBySubject(subject: string): Promise<number> {
  return this.sessionRecordsHandler.deleteBySubject(subject);
}
```
