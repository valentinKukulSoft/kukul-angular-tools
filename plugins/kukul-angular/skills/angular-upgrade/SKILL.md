---
name: angular-upgrade
description: Actualiza un proyecto Angular a la última versión estable (major a major con ng update) y luego sus paquetes, por fases, investigando breaking changes que afecten al proyecto, y genera docs/angular-upgrade/README.md como tablero de seguimiento. Usar cuando el usuario pida actualizar Angular, subir dependencias, migrar a la última versión, o planear/continuar una actualización de Angular y paquetes.
argument-hint: [versión-angular-objetivo]
---

# Actualización de Angular y dependencias por fases

Actualiza Angular y luego los paquetes, con **evidencia** de qué rompe en *este* proyecto y un tablero de seguimiento versionado. Argumento opcional: versión objetivo (`$ARGUMENTS`); si no se pasa, la última estable.

## Reglas de oro

1. **Investiga antes de tocar.** Entra en modo plan: lee el proyecto, averigua versiones reales y breaking changes, escribe el plan y pide aprobación. Nada de ediciones antes.
2. **Datos reales, no memoria.** Las versiones cambian: consulta `npm view`, `ng update`, la guía oficial (`angular.dev/update-guide`) y changelogs. No afirmes una versión "más reciente" ni una compatibilidad Node/TypeScript de memoria.
3. **Un major de Angular a la vez.** `ng update` no salta majors (16 → 18 no está soportado): 16 → 17 → 18. Cada salto es una fase verificable.
4. **Una fase = un commit (o uno por paquete en majors de riesgo).** Si el usuario pide "un solo commit", un solo commit. **Commitea solo cuando lo pida.** Ojo: `ng update` exige árbol limpio; si el usuario no quiere commits intermedios, usa `--allow-dirty` y explícalo.
5. **Stage explícito, nunca `git add -A`.** Hay cambios del usuario sin commitear que no deben colarse. Revisa `git status` antes de cada commit.
6. **Cada paso se verifica:** build de producción + lint + test. No declares una fase completa sin haberlos corrido.
7. **Las pruebas visuales/manuales las hace el usuario** salvo que pida lo contrario: dile exactamente qué probar a mano al cerrar cada fase.
8. Usa el gestor de paquetes del proyecto (lockfile: npm/pnpm/yarn/bun) y la versión de Node que fija (`.nvmrc`, `.node-version`, `volta`, `engines`); con nvm, `nvm use`. Los ejemplos usan `npm`/`npx`. **Nunca** cambies de gestor ni regeneres el lockfile desde cero sin preguntar.
9. Respeta el idioma y las reglas del proyecto (CLAUDE.md: comentarios mínimos, no formatear carpetas completas, convenciones de commits).
10. Si algo huele a decisión de producto o de riesgo (cambiar de builder, de test runner, activar zoneless, migrar a standalone/signals, cambiar de librería UI), **pregunta** con una recomendación; no decidas solo. Las migraciones **opcionales** no son parte del upgrade salvo que el usuario las pida.
11. **Documentación actual vía Context7** (ver sección siguiente): úsalo para toda consulta de APIs, migraciones y configuración de Angular y librerías.

## Documentación con Context7

Los breaking changes, schematics y flags de Angular cambian en cada major: **no confíes en la memoria del modelo**. Context7 (MCP) da documentación versionada y actual.

1. **Comprueba si está disponible** al empezar: busca herramientas `resolve-library-id` / `query-docs` de Context7 (directas o diferidas vía ToolSearch).
2. **Si está disponible, úsalo siempre** antes de afirmar algo sobre una API, migración o configuración:
   - `resolve-library-id` con el nombre de la librería (ej. "angular", "angular material", "ngrx", "angular-eslint", "rxjs", "primeng") y elige el ID por coincidencia de nombre, reputación y nº de snippets. Usa el **ID específico de versión** cuando exista (versión origen y destino).
   - `query-docs` con una pregunta completa y de **un solo concepto** (ej. "breaking changes al migrar de Angular 19 a 20 en el router", "cómo migrar de Karma a Vitest con @angular/build:unit-test"). Si hay varios conceptos, una consulta por concepto.
   - Cita en el tablero qué se verificó con Context7 y contrástalo con el changelog/update-guide oficial cuando haya discrepancia.
3. **Si está instalado pero no autenticado**, dile al usuario que lo autorice (`/mcp` en una sesión interactiva o la configuración de conectores de claude.ai) y continúa con las fuentes oficiales mientras tanto.
4. **Si no está instalado**, recomiéndalo una vez al inicio, antes de la investigación de la Fase 1, sin bloquear el trabajo:
   > Recomiendo instalar Context7 para consultar documentación actual y versionada de Angular y sus librerías durante el upgrade: `/plugin marketplace add upstash/context7` y `/plugin install context7@context7-marketplace`; reinicia Claude Code y ejecuta `/mcp` para iniciar sesión en Context7 (no requiere API key). En entornos sin navegador (SSH, CI): `npx ctx7 setup --claude`.

   Si el usuario no lo instala, sigue con `npm view`, `angular.dev/update-guide`, changelogs de GitHub y la web, y anota en el tablero que la investigación se hizo sin Context7.

## Fase 0 — Contexto y línea base

1. Rama: trabaja en la actual salvo que el usuario pida una nueva. Anota la rama base.
2. Versiones actuales: `npx ng version`, `node -v`, gestor y versión, `package.json` (`@angular/*`, `@angular/cli`, `@angular-devkit/*`/`@angular/build`, `typescript`, `rxjs`, `zone.js`, `tslib`, librerías UI, NgRx, test runner), **`overrides`/`resolutions` y comentarios** sobre por qué existen, `.npmrc` (¿`legacy-peer-deps=true`?), `angular.json` (builders por proyecto), `tsconfig*.json` (`target`, `module`, `moduleResolution`, `strict*`), `browserslist`, y versión de Node en CI.
3. Última estable real y compatibilidades:
   ```bash
   npm view @angular/core dist-tags --json        # latest, next, v<N>-lts
   npm view @angular/core@<objetivo> peerDependencies --json   # rxjs, zone.js, typescript (aprox.)
   npm view @angular/cli@<objetivo> engines --json             # rango de Node exigido
   npm view @angular/compiler-cli@<objetivo> peerDependencies --json   # rango exacto de TypeScript
   ```
   Contrasta con `angular.dev/reference/versions` (tabla Angular ↔ Node ↔ TypeScript ↔ RxJS) y `angular.dev/reference/releases` (soporte activo/LTS).
4. Paquetes: `npx ng update` (lista los paquetes con schematics de migración) y `npm outdated` (Current / Wanted / Latest). Separa: `@angular/*` (framework), ecosistema Angular con versión acoplada al major (`@angular/material`/`cdk`, `@ngrx/*`, `angular-eslint`, `@angular/fire`, PrimeNG, `ngx-*`), dependencias de runtime y tooling.
5. **Línea base:** `npx ng build --configuration production`, `npx ng lint`, `npx ng test --watch=false` (Karma: añade `--browsers=ChromeHeadless`) y, si hay, E2E. Anota tamaño inicial del bundle y warnings. Si ya hay tests rojos, errores de lint o warnings de deprecación, repórtalos y **resuélvelos antes** (para no confundirlos con regresiones). Causas típicas de tests rojos heredados: specs plantilla `should create` sin providers (`HttpClient`, `ActivatedRoute`), `fit`/`fdescribe` olvidados, servicios que hacen HTTP real sin `provideHttpClientTesting`.
6. Limpia deprecaciones del major actual antes de saltar: `ng lint` con reglas de deprecación, warnings de compilación, APIs marcadas `@deprecated` que el siguiente major elimina (consulta el update-guide con origen/destino).

## Fase 1 — Investigación de breaking changes (modo plan)

Para **cada salto de major** de Angular y para cada paquete con salto major:
- Guía oficial: `https://angular.dev/update-guide?v=<origen>.0-<destino>.0&l=3` (nivel avanzado) — lista acciones antes, durante y después del update. Changelog: `github.com/angular/angular/blob/main/CHANGELOG.md` (secciones *Breaking Changes* y *Deprecations*), y los de CLI, Material, NgRx, etc. Con Context7 (ver sección arriba), consulta los cambios de cada librería con su ID de versión destino.
- Anota: mínimos nuevos (Node, TypeScript, RxJS, zone.js, navegadores), APIs eliminadas/renombradas, cambios de comportamiento por defecto (ej. `standalone: true` por defecto, builder por defecto, zoneless por defecto en proyectos nuevos, cambios en `HttpClient`, router, forms, SSR), y qué migraciones aplica `ng update` automáticamente.
- **Cruza con el código:** `grep` de cada API afectada en `src/` (y `projects/` en workspaces) para saber si te toca. Lista archivos y líneas.
- **Terceros que bloquean:** para cada librería con peer `@angular/core`, `npm view <pkg>@latest peerDependencies` y busca la primera versión compatible con el major destino. Si una librería crítica no soporta aún el destino, **eso fija el techo del upgrade**: anótalo y pregunta. Agrupa paquetes que deben subir juntos (`@angular/material` + `@angular/cdk`; todos los `@ngrx/*`; `angular-eslint` + `typescript-eslint`).
- **Toolchain:** Node exigido (¿hay que subir `.nvmrc`, CI, Dockerfile, hosting?), TypeScript, y builder. Alinéate con lo que genera `ng new` en la versión destino (créalo en el scratchpad con `npx @angular/cli@<v> new tmp --defaults --skip-install` para comparar `angular.json`/`tsconfig`) en vez de inventar configuración.
- `overrides`/`resolutions` existentes con comentario: verifica si su causa sigue vigente antes de conservarlos.

Escribe el plan con fases (un salto de major por fase), riesgos y verificación; presenta con ExitPlanMode.

## Fase 2 — Angular (major a major)

Por cada salto `N → N+1`:
1. Node: si el destino exige otro Node, cámbialo primero (`nvm install`/`nvm use`, actualiza `.nvmrc`/`engines`) y anota que **CI, Docker y hosting** también deben subir.
2. Framework y CLI juntos:
   ```bash
   npx ng update @angular/core@<N+1> @angular/cli@<N+1>
   ```
   Lee la salida completa: lista las migraciones aplicadas y los archivos tocados. Si falla por peers, **lee el mensaje** (indica qué paquete bloquea); no recurras a `--force` sin entender el conflicto y avisar al usuario.
   **Ojo con las migraciones opcionales:** en una terminal no interactiva (como la de Claude) `ng update` **no las ejecuta ni pregunta**; solo las lista con su comando `ng update <pkg> --name <migración>`. Cópialas al tablero y ofrécelas al usuario: se aplican después, una a una, con `--name` (ver Fase 5).
3. Ecosistema acoplado en el mismo salto: `npx ng update @angular/material@<N+1>` (incluye cdk), `@ngrx/store@<N+1>`, `angular-eslint@<N+1>`, `@angular/fire`… cada uno ejecuta sus propios schematics.
4. Revisa el diff de las migraciones automáticas (no lo aceptes a ciegas): comentarios `TODO` que dejan los schematics, cambios en `angular.json`/`tsconfig`, imports reescritos.
5. Sube `typescript`, `zone.js` y `rxjs` a los rangos que exige el destino (normalmente `ng update` lo hace; verifícalo).
6. `npm install` limpio (respeta el lockfile), build prod, lint, test. Actualiza docs (CLAUDE.md/README) con las versiones.
7. Registra en el tablero y repite con el siguiente major.

## Fase 3 — Minor/patch de dependencias directas

Sube el resto de dependencias a las últimas versiones compatibles dentro del major (`npm update` o ajustando rangos) y reinstala. Si npm falla con `ERESOLVE`, **lee el árbol**: casi siempre indica el peer que obliga a mover otro paquete. Mueve el paquete conflictivo a la fase de majors. **No** añadas `--legacy-peer-deps` para "pasar": si ya estaba en el proyecto, anota el motivo y valora retirarlo al final.

## Fase 4 — Majors de terceros

Uno por uno (o por bloque si comparten peers), de menor a mayor riesgo. Tras cada uno: install, build, lint, test. Registra en el tablero.
- Usa `ng update <pkg>` si el paquete trae schematics; si no, `npm install <pkg>@<v>` y adapta el código a las APIs eliminadas (grep primero).
- Un tercero puede bloquear un major (librería abandonada con peer `@angular/core` antiguo): **no fuerces con `overrides`** salvo que el usuario lo acepte sabiendo el riesgo; anota alternativas (fork mantenido, sustituto, reimplementar) y sigue.
- Una transitiva con vulnerabilidad o rota: prefiere subir la dependencia directa que la arrastra; `overrides` solo como último recurso, con comentario y entrada en "Pendientes".
- Librerías UI (Material, PrimeNG…) con cambios visuales: anota componentes afectados para la prueba manual; los cambios de tema/tokens (ej. Material M2 → M3) son **decisión de producto**.

## Fase 5 — Toolchain y migraciones de configuración

Solo lo que el upgrade exige o el usuario aprueba; cada punto en su propio commit:
- **Builder**: si el proyecto sigue en `@angular-devkit/build-angular:browser` (webpack), la migración a `@angular/build:application` (esbuild/Vite) se ofrece con `npx ng update @angular/cli --name use-application-builder`. Cambia la estructura de `dist/` (subcarpeta `browser/`), `main.server.ts`/SSR, y la sintaxis de algunos imports de estilos: revisa scripts de despliegue, Dockerfile y config del hosting **antes** de proponerlo. Requiere aprobación.
- **Test runner**: si Karma está deprecado en el destino, presenta opciones (mantener Karma mientras funcione, Vitest vía `@angular/build:unit-test`, Jest) con recomendación. Requiere aprobación. La migración oficial va en dos pasos:
  1. Configuración: migración opcional de `ng update` (desde CLI v22) `npx ng update @angular/cli --name migrate-karma-to-vitest` → convierte el target `test` a `@angular/build:unit-test` con `runner: vitest` y mapea el `main` de test a `setupFiles` (quita de ese archivo cualquier `TestBed.initTestEnvironment`). **No** reescribe los specs.
  2. Specs: schematic **experimental** `npx ng g @schematics/angular:refactor-jasmine-vitest <proyecto>` (opciones útiles: `--include` para ir por carpetas, `--fakeAsync`, `--browserMode`; genera un informe `jasmine-vitest-<fecha>.md`). Revisa el diff y el informe: lo que no sabe convertir queda marcado.
- **ESLint**: `angular-eslint` alineado al major; config plana (`eslint.config.js`) si el destino lo exige.
- **TypeScript config**: `target`/`module`/`moduleResolution` alineados con lo que genera `ng new` en el destino; `useDefineForClassFields` según el update-guide.
- **Node en CI/Docker/hosting**: actualizar `.github/workflows`, `Dockerfile`, `netlify.toml`/`vercel.json`/Firebase y similares.
- Tras cambios de builder o de dependencias nativas, si el build falla de forma inexplicable: borra `.angular/cache` (y `node_modules` + reinstalar con lockfile) antes de investigar.

### Migraciones opcionales (solo si el usuario las pide)

Inventaria su alcance (grep) y ofrécelas como fases aparte, una por commit, con la verificación completa tras cada una. Comprueba cuáles existen en la versión instalada con `npx ng g @angular/core: --help`:
- Standalone: `ng g @angular/core:standalone` (en 3 pasos: convertir, quitar NgModules, bootstrap standalone).
- Control flow: `ng g @angular/core:control-flow` (`*ngIf`/`*ngFor`/`ngSwitch` → `@if`/`@for`/`@switch`).
- `inject()`: `ng g @angular/core:inject`.
- Rutas lazy: `ng g @angular/core:route-lazy-loading`.
- Signals: `ng g @angular/core:signal-input-migration`, `output-migration`, `signal-queries-migration` (o `ng g @angular/core:signals` para todas).
- Limpieza: `ng g @angular/core:cleanup-unused-imports`, `self-closing-tag`.
- Zoneless / OnPush: cambio de comportamiento en runtime, **no** mecánico: requiere pruebas manuales amplias.

## Fase 6 — Transitivas y limpieza

- Actualiza transitivas dentro de rangos (`npm update`), sin regenerar el lockfile desde cero; luego verificación completa.
- `npm outdated` y `npx ng update` finales: lo que quede debe estar explicado (bloqueado por un tercero, por Node del hosting, por decisión de producto).
- `npm audit --omit=dev`: compara con la línea base.
- Retira `overrides`/`--legacy-peer-deps` que ya no hagan falta.
- Documenta pendientes y de quién dependen (tabla "Pendientes y dependencias de terceros").

## Verificación (cada fase)

```bash
npx ng build --configuration production     # sin errores; compara bundle inicial y warnings con la línea base
npx ng lint                                 # 0 errores
npx ng test --watch=false                   # todo en verde (Karma: --browsers=ChromeHeadless)
npx ng build --configuration development    # si hay configuraciones distintas (staging, etc.), constrúyelas todas
npm run e2e                                 # si existe
npx ng serve                                # arranque de humo: sin errores en consola del navegador
```

Si hay SSR, también `npm run serve:ssr:<proyecto>` (o el script equivalente) y una carga de página con hydration. En workspaces multi-proyecto o Nx, verifica **todos** los proyectos afectados (`nx affected -t build,test,lint`). Al cerrar cada fase entrega al usuario una **lista concreta de pruebas manuales** según lo tocado (login/sesión/refresh ↔ interceptores y auth; formularios y validaciones ↔ `@angular/forms`; navegación, guards y deep links ↔ router; diálogos, tablas, datepickers y temas ↔ Material/CDK/librería UI; carga inicial y SEO ↔ SSR/hydration; subida/descarga de archivos ↔ `HttpClient`; i18n ↔ cambio de idioma; PWA ↔ service worker y actualización).

## Tablero de seguimiento

Crea y mantén `docs/angular-upgrade/README.md` (en el idioma del proyecto) con:
- Cabecera: versión inicial → objetivo, rama, gestor de paquetes, entorno (Node, TypeScript, RxJS, builder, test runner, navegadores objetivo) y Node en CI/hosting.
- Tabla de **estado por fase** (✅ / 🟡 / ⬜), con una fila por salto de major.
- Por fase: checklist de pasos, tabla de paquetes (de → a, migraciones aplicadas, impacto en el código, hecho) y **Notas** con hallazgos, decisiones e incidencias (la causa y la solución, para que el siguiente que actualice no repita el problema).
- Tamaño del bundle inicial antes/después de cada fase.
- Sección **Pendientes y dependencias de terceros**: qué, de quién depende, impacto, acción.
- Checklist de verificación al cerrar cada fase y tabla de registro de incidencias.
Actualízalo en cada paso y mantenlo dentro del commit de la fase que documenta.

## Cierre

Resume al usuario: versiones antes/después (Angular, Node, TypeScript, RxJS), qué rompió y cómo se resolvió, qué migraciones se aplicaron y cuáles quedan opcionales, qué quedó pendiente y de quién depende (incluido subir Node en CI/hosting), y qué debe probar a mano. No hagas commits ni push sin que lo pida.
