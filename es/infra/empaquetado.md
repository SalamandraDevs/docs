---
title: Publicación y consumo de paquetes
lastUpdate: 2026-10-12
lang: es
category: operaciones
status: active
owner: haldrik
---

## Propósito

Cómo se publican en npm las librerías compartidas de Salamandra Devs y cómo las consumen
otros paquetes, servicios y apps. Registra las convenciones que debe seguir cada paquete
publicable, el pipeline de publicación, las reglas de instalación para los consumidores, la
política de versionado y las trampas aprendidas al migrar el scope `@salamandradevs` a npm.

Los paquetes de referencia son `@salamandradevs/tools` y `@salamandradevs/core`. Los
primeros consumidores reales son `services/auth` (instala ambos) y el sitio `yesenia`.

## Tabla de contenidos

- [Alcance y propiedad](#alcance-y-propiedad)
- [Reglas para un paquete publicable](#reglas-para-un-paquete-publicable)
- [Pipeline de publicación](#pipeline-de-publicación)
- [Consumo de paquetes publicados](#consumo-de-paquetes-publicados)
- [Política de versionado (semver)](#política-de-versionado-semver)
- [Checklist de publicación](#checklist-de-publicación)
- [Trampas y lecciones aprendidas](#trampas-y-lecciones-aprendidas)

## Alcance y propiedad

- Todos los paquetes compartidos se publican bajo el scope de npm **`@salamandradevs`**, que
  refleja la organización de GitHub `SalamandraDevs`.
- El scope debe ser una organización de npm **que poseamos**. Originalmente usamos
  `@slmdevs`, pero ese scope ya pertenece a una cuenta no relacionada
  (`@slmdevs/celeraone`), por lo que publicar allí falla. Verifique la propiedad del scope
  antes de nombrar un paquete.
- Los paquetes son **públicos**. Un scope público más un repositorio de GitHub público es
  también lo que habilita la procedencia de build.

## Reglas para un paquete publicable

Cada paquete es su propio repositorio de GitHub bajo `SalamandraDevs/` (no son un único
workspace; `~/dev/slmdevs` es solo una carpeta contenedora). Un `package.json` publicable
lleva:

- `"name"`: `@salamandradevs/<pkg>`.
- `"publishConfig": { "access": "public", "provenance": true }`: `access: public` es
  obligatorio para los paquetes con scope; `provenance: true` registra una atestación de
  build firmada.
- `"repository"`, `"homepage"`, `"bugs"` apuntando al repositorio de GitHub exacto.
  **`repository.url` debe coincidir con el repositorio exactamente** o la procedencia se
  rechaza.
- `"files": ["dist", "types"]`: solo se envía la salida del build.
- `"exports"`: solo subrutas explícitas; no declare una subruta que no tenga salida
  construida. Por ejemplo, `@salamandradevs/tools` expone `./browser`, `./node`, `./shared`
  y **no tiene export de raíz (`.`)**; los consumidores importan la entrada específica que
  necesitan.
- `"prepublishOnly": "pnpm run build"`: garantiza un build fresco antes de empaquetar.
- El build es `tsc --build`, que emite `dist/` (JS) y `types/` (`.d.ts`).

**Los artefactos construidos se versionan en git.** `dist/` y `types/` intencionadamente
**no** están en gitignore. Esto permite a una persona desarrolladora copiar la salida
construida de una dependencia directamente en el `node_modules` de otro paquete durante el
trabajo local, evitando el ciclo de publicar y reinstalar.

## Pipeline de publicación

La publicación usa token pero es por etapas, de modo que ninguna versión se activa sin una
persona. Cada repositorio tiene `.github/workflows/release.yml`:

1. **Disparador:** hacer push de un tag semver, por ejemplo
   `git tag v0.1.0 && git push origin v0.1.0`.
2. **Build:** checkout, `pnpm install`, `pnpm run build`. El job instala `npm@latest`
   porque la publicación por etapas necesita **npm >= 11.15.0** (y Node >= 22.14).
3. **Guarda:** rechaza continuar si la versión ya está en npm.
4. **Etapa (stage):** `npm stage publish`, autenticado por el secreto `NPM_TOKEN`. Esto
   **no** activa la versión; aparca la publicación en la cola de staging de npm.
5. **Procedencia:** se adjunta automáticamente porque el workflow tiene
   `permissions: id-token: write` y el repositorio y el paquete son públicos. No se necesita
   la bandera `--provenance`.
6. **Aprobar (manual, humano):** una persona mantenedora promueve la publicación en staging
   con 2FA. La UI web más passkey funciona; la vía por CLI `npm stage approve` no funciona
   cuando el segundo factor es una passkey. Apruebe desde la página del paquete en
   npmjs.com.
7. **Verificar:** `npm view @salamandradevs/<pkg> version` devuelve la versión nueva.

### El secreto `NPM_TOKEN`

- Es un **token de acceso granular de npm** con scope `@salamandradevs` y permiso de
  **escritura (staged)**, creado en npmjs.com y luego almacenado por repositorio como el
  secreto de GitHub Actions `NPM_TOKEN` (`gh secret set NPM_TOKEN -R SalamandraDevs/<repo>`).
- Es una credencial de npm, no de GitHub. `secrets.GITHUB_TOKEN` no puede publicar en npm.
- npm limita los tokens de escritura granulares a ~90 días, así que cuente con rotarlo.

### Orden de dependencias

Publique de abajo hacia arriba: un paquete debe estar activo en npm antes de que cualquier
cosa que dependa de él pueda construirse en CI. Orden actual: **`tools` → `core` →
consumidores** (`core` depende de `tools`; `auth` depende de ambos).

## Consumo de paquetes publicados

- Dependa del **rango de versión de npm** (`^0.1.0`), nunca de una ruta `file:../...`. Las
  rutas `file:` solo se resuelven dentro del layout del monorepo y rompen cualquier CI fuera
  del monorepo (esto es exactamente lo que impidió que Netlify construyera `yesenia`). `core`
  consume `@salamandradevs/tools` de esta manera, y `services/auth` consume `core` y `tools`
  de esta manera.
- **Desarrollo local** contra cambios no publicados: construya la dependencia y copie sus
  `dist/`/`types/` en el `node_modules/@salamandradevs/<pkg>/` del consumidor. Por eso los
  artefactos se versionan; no se requiere subir versión ni publicar para probar un cambio.
- **Cuando un consumidor necesita solo una porción**, prefiera localizarla antes que tomar
  la dependencia. `yesenia` usaba solo el logger de Winston de `tools`, así que el logger se
  copió dentro de la app (`src/lib/globals/logger.ts`) y la dependencia se eliminó por
  completo; el sitio ahora instala desde su propio repositorio sin dependencias de scope
  privado.

## Política de versionado (semver)

Seguimos [semver](https://semver.org). La versión en `package.json` es la fuente de verdad;
la publicación se corta etiquetando `v<version>`.

- **Pre-1.0 (`0.y.z`), donde estamos ahora:** la API aún se está asentando. Suba **minor**
  (`0.1.0 → 0.2.0`) ante cualquier cambio incompatible o funcionalidad notable, y **patch**
  (`0.1.0 → 0.1.1`) ante correcciones compatibles hacia atrás. Los consumidores deberían
  fijar con `^0.y.z`, entendiendo que bajo 0.x un caret solo permite actualizaciones de
  patch.
- **1.0 en adelante:** **major** para cambios incompatibles, **minor** para añadidos
  compatibles hacia atrás, **patch** para correcciones compatibles hacia atrás.
- **Publicación = tag.** Suba la versión en `package.json`, haga commit, luego
  `git tag vX.Y.Z` y haga push del tag. El paso de guarda bloquea un tag cuya versión ya
  esté publicada, así que suba la versión antes de etiquetar.
- Nunca republique una versión. Si una publicación en staging está mal, recházela y corte
  el siguiente patch.

## Checklist de publicación

1. Aterrice el cambio en `main`; asegúrese de que `pnpm run build` y el lint pasen.
2. Suba `version` en `package.json` según la política anterior; haga commit.
3. `git tag vX.Y.Z && git push origin main --tags`.
4. Observe el workflow `Release`; debería terminar en "staged with id ...".
5. Apruebe la publicación en staging en npmjs.com con 2FA.
6. `npm view @salamandradevs/<pkg> version` para confirmar que está activa.
7. Publique los dependientes solo después de que sus dependencias estén activas.

## Trampas y lecciones aprendidas

- **La propiedad del scope se comprueba en la publicación, de forma opaca.** Hacer staging
  a un scope que no posee falla con
  `400 Bad Request ... Version uniqueness check failed unexpectedly`; no es un "forbidden"
  evidente. Si lo ve, confirme que la organización posee el scope.
- **Precondiciones de la procedencia:** repositorio público, paquete público,
  `id-token: write` y `repository.url` coincidiendo con el repositorio exactamente. Si falta
  alguna, la procedencia no se adjuntará en silencio (o la publicación da error).
- **Aviso de restricción de tokens:** npm está restringiendo los tokens que evitan el 2FA
  para publicación directa; la publicación por etapas (CI hace staging, un humano aprueba)
  es la vía soportada.
- **Adaptadores/plugins acoplados al framework:** fije las versiones al major del framework
  del consumidor. `yesenia` sobre SvelteKit 2 necesita `@sveltejs/adapter-netlify@^6` (la
  7.x apunta a kit 3 y falla con `builder.config.files` undefined). `latest` no siempre es la
  elección correcta.
- **Los hooks de pre-push reescriben archivos.** Estos repositorios ejecutan
  `pnpm format && pnpm lint` en `pre-push`, que edita el árbol *después* de su commit,
  dejándolo sucio y el commit enviado sin formatear. Ejecute format y lint y haga commit
  antes de hacer push para que el hook sea una operación sin efecto.
</content>
