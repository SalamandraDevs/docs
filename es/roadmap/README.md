---
title: Reglas del roadmap
lastUpdate: 2026-10-25
lang: es
category: planificacion
status: active
---

# Reglas del roadmap

Estas reglas aplican a todos los archivos `{REPO}-ROADMAP.md` de este directorio (`AUTH`,
`CORE`, `TOOLS`).

## Formato de línea

Cada funcionalidad es una línea con un ID estable, para que las personas y los agentes
puedan buscarla con grep y citarla:

```text
- [estado] ID Nombre: qué hace. (refs: archivos fuente o documentos de plan)
```

Ejemplo:

```markdown
- [implementado] AUTH-004 Sesiones: emite, lee, termina y refresca con tokens de refresco
  opacos y rotativos almacenados como hashes HMAC. (refs: `docs/session-plan.md`,
  `src/application/sessions-service/`)
```

## Etiquetas de estado

Estas etiquetas marcan funcionalidades individuales en línea dentro de los roadmaps y
los documentos de planificación; no son valores de `status` del frontmatter.

- `implementado`: la funcionalidad está completa y presente en el código del árbol de
  trabajo principal. El comportamiento está verificado y disponible para su uso. El
  documento de plan que la implementa, cuando existe, aparece entre los `(refs: ...)` del
  elemento junto a las rutas de código.
- `planificado`: la funcionalidad tiene un documento de plan aprobado y está lista para
  implementarse. La revisión de diseño está completa; el orden de implementación está
  decidido.
- `propuesto`: la funcionalidad está inferida, en discusión o diseñada pero aún no
  aprobada. Puede existir un documento de diseño abierto a comentarios; el orden de
  implementación no está decidido.
- `cancelado`: la funcionalidad fue rechazada o aplazada indefinidamente. Se registra por
  contexto pero no requiere trabajo de implementación.

## Alcance y rutas

- Alcance: `core/`, `tools/` y `services/auth/`, cada uno con su propio archivo de roadmap
  en este directorio.
- Las rutas son relativas a la raíz de cada repositorio. Las rutas de ruta HTTP son
  relativas a la raíz del servicio auth, antes de cualquier prefijo de gateway.

## Reglas de edición

- Cambie un estado solo después de revisar el código; cite el archivo o el plan en `refs`.
- Nunca reutilice ni renumere un ID. Añada los IDs nuevos al final del archivo de roadmap
  del repositorio.
- Al sincronizar el estado de las funcionalidades implementadas, actualice `lastUpdate` en
  cada archivo de roadmap que modifique para reflejar la instantánea actual.
</content>
