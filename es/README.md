---
title: Documentación de Salamandra Devs
lastUpdate: 2026-10-25
lang: es
category: guia
status: active
---

# Documentación de Salamandra Devs

Base de conocimiento mantenida por la comunidad **Salamandra Devs**. Aquí viven guías,
notas de arquitectura y runbooks operativos para compartir lo que aprendemos, mejorarlo
en conjunto y mantener las prácticas alineadas entre proyectos.

Este repositorio está abierto a las contribuciones de miembros y colaboradores. Si usa
estos documentos, detecta una carencia o quiere documentar algo nuevo, está invitado a
participar.

**Otros idiomas:** [English index](../en/README.md)

## Tabla de contenidos

- [Temas](#temas)
  - [Arquitectura](#arquitectura)
  - [Normas](#normas)
  - [Roadmap](#roadmap)
- [Cómo usar este repositorio](#cómo-usar-este-repositorio)
- [Contribuir](#contribuir)
  - [Formas de participar](#formas-de-participar)
  - [Flujo de contribución](#flujo-de-contribución)
  - [Pautas para los pull requests](#pautas-para-los-pull-requests)
  - [Reportar incidencias](#reportar-incidencias)
  - [Pautas de redacción](#pautas-de-redacción)

## Temas

### Arquitectura

Principios, patrones y prácticas de diseño de software usados en los proyectos de
Salamandra Devs.

- [Principios SOLID](./arquitectura/principios-solid.md): definiciones, terminología,
  ejemplos en TypeScript, beneficios, contrapartidas y referencias para SRP, OCP, LSP, ISP
  y DIP.
- [Patrones de diseño](./arquitectura/patrones-diseno.md): los patrones creacionales,
  estructurales, de comportamiento y de arquitectura de aplicación que el código de
  Salamandra Devs usa hoy (DP-01 a DP-10), agrupados por categoría con el problema que
  resuelve cada uno.
- [Buenas prácticas de desarrollo](./arquitectura/buenas-practicas-desarrollo.md): guía
  general sobre modularidad, encapsulamiento y diseño de sistemas mantenibles, con ejemplos
  prácticos.

### Normas

Convenciones de toda la plataforma para código, documentación y comentarios de API a nivel
de código fuente.

- [Estilo de código](./normas/estilo-codigo.md): estilo de código TypeScript y general;
  imports, esquemas Zod, wrappers, errores, módulos, líneas en blanco y hooks de rutas
  de las APIs RESTful.
- [Guía de estilo de documentos](./normas/guia-estilo-documentos.md): estilo Markdown,
  redacción, frontmatter, valores de estado y comentarios de código fuente para TypeDoc,
  con ejemplos trabajados.
- [Convenciones de nombres](./normas/convenciones-nombres.md): el único estándar de
  nombres para servicios, rutas, registros y símbolos, con las reglas de abreviatura y el
  esquema de IDs de documento.

### Roadmap

Estado de las funcionalidades por paquete, registrado en línea con etiquetas de estado.

- [Roadmap de la librería Core](./roadmap/CORE-ROADMAP.md): estado de las funcionalidades de
  `core/`.
- [Roadmap de la librería Tools](./roadmap/TOOLS-ROADMAP.md): estado de las funcionalidades
  de `tools/`.
- [Roadmap del servicio Auth](./roadmap/AUTH-ROADMAP.md): estado de las funcionalidades de
  `services/auth/`.

## Cómo usar este repositorio

- Explore los temas por área usando las listas de arriba.
- Prefiera enlazar una guía existente en lugar de duplicar contenido en los README de los
  proyectos.
- Trate los documentos como material vivo: cuando una práctica cambie en el código o en
  operaciones, actualice aquí la guía correspondiente.
- Abra una incidencia cuando algo no esté claro, esté desactualizado o falte.

## Contribuir

Queremos que este repositorio refleje la experiencia real de la comunidad. Las
contribuciones pueden ser correcciones pequeñas, guías nuevas, ejemplos más claros o
ajustes cuando algo ya no coincide con la práctica actual.

### Formas de participar

- **Mejorar documentos existentes**: corregir erratas, aclarar la redacción, actualizar
  pasos desactualizados o añadir ejemplos.
- **Añadir guías nuevas**: documentar decisiones de arquitectura, flujos de incorporación,
  runbooks o la configuración de herramientas.
- **Revisar pull requests**: ayudar a validar la exactitud, la legibilidad y la coherencia
  con el material existente.
- **Abrir incidencias**: reportar carencias, hacer preguntas o proponer temas nuevos antes
  de escribir un cambio grande.

### Flujo de contribución

1. **Haga un fork** de este repositorio a su cuenta de GitHub.
2. **Clone** su fork localmente:

   ```bash
   git clone https://github.com/<su-usuario>/<nombre-repo>.git
   cd <nombre-repo>
   ```

3. **Cree una rama** para su cambio:

   ```bash
   git checkout -b docs/su-tema
   ```

4. **Haga sus cambios** en Markdown. Mantenga las ediciones enfocadas y fáciles de revisar.
5. **Haga commit** con un mensaje claro que explique qué cambió y por qué.
6. **Haga push** de su rama a su fork:

   ```bash
   git push -u origin docs/su-tema
   ```

7. **Abra un pull request** contra la rama principal del repositorio original.
8. **Responda a los comentarios de la revisión** y actualice el PR hasta que esté listo
   para fusionar.

Si no tiene acceso de escritura, este flujo de fork y PR es el camino estándar para todas
las contribuciones.

### Pautas para los pull requests

- Mantenga un cambio lógico por pull request cuando sea posible.
- Enlace las incidencias relacionadas en la descripción del PR cuando aplique.
- Resuma qué cambió y por qué debería importarle a quien revisa.
- Prefiera una redacción precisa y práctica antes que consejos genéricos.
- Incluya comandos, fragmentos de configuración o ejemplos cuando ayuden a aplicar la guía.
- Evite cambios de solo formato no relacionados, salvo que mejoren la legibilidad en los
  archivos que ya está editando.

### Reportar incidencias

Use las incidencias de GitHub para:

- Reportar instrucciones incorrectas o desactualizadas
- Solicitar documentación de un tema que falta
- Pedir aclaraciones sobre una guía existente
- Proponer mejoras estructurales al conjunto de documentos

Al abrir una incidencia, incluya:

- El documento o tema involucrado
- Qué esperaba encontrar
- Qué no está claro, falta o está mal
- Cualquier contexto relevante como el entorno, las versiones de las herramientas o el área
  del proyecto

Para propuestas más grandes, abrir una incidencia antes de empezar el trabajo ayuda a
alinear el alcance y evitar esfuerzo duplicado.

### Pautas de redacción

- Escriba para quienes aplican el conocimiento: pasos concretos, terminología clara y
  contexto del mundo real.
- Use Markdown de forma coherente con los documentos existentes de este repositorio.
- Prefiera conceptos estables y patrones con nombre antes que jerga específica del
  proyecto cuando la guía está pensada para compartirse ampliamente.
- Enlace guías relacionadas en lugar de repetir secciones grandes.
- Añada o actualice las traducciones bajo `docs/es/` cuando contribuya contenido en
  español.
- Mantenga los nombres de archivo alineados entre idiomas cuando sea posible (por ejemplo,
  `solid-principles.md` / `principios-solid.md`).
- Marque con claridad las propuestas, borradores o supuestos específicos de un entorno
  cuando aún no sean práctica estándar.
- No haga commit de secretos, credenciales, nombres de host privados ni datos específicos
  de clientes.

Gracias por ayudar a Salamandra Devs a construir documentación en la que la comunidad pueda
confiar y mejorar en conjunto.
</content>
</invoke>
