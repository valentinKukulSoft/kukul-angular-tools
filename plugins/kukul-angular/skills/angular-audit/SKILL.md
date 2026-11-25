---
name: angular-audit
description: Ejecuta una auditoría técnica completa de un proyecto Angular (seguridad, performance, refactor, arquitectura, dependencias, tests, CI) con evidencia medida y genera docs/AUDITORIA_<fecha>.md priorizado P0–P3 con plan de trabajo. Usar cuando el usuario pida auditar, revisar la salud técnica o evaluar el nivel de producción de una app Angular.
---

# Auditoría técnica de proyecto Angular

Produce un documento `docs/AUDITORIA_<YYYY-MM>.md` con hallazgos **verificados con evidencia**, priorizados y con plan de ataque. Estructura: auditoría base + anexo profundo + plan de seguimiento.

## Reglas de oro

1. **Evidencia o no existe.** Cada hallazgo cita `ruta:línea` y/o una métrica obtenida con un comando. Nada de "parece que...". Si no pudiste verificarlo, va en "No verificable desde el repo".
2. **Nunca incluyas valores de secretos**, solo rutas/nombres de variables. El documento debe ser seguro de versionar.
3. **Verifica antes de afirmar.** Las heurísticas dan falsos positivos (ej. un `.subscribe(` sin `unsubscribe` que sí se libera con `takeUntilDestroyed`, `async` pipe, `first()`/`take(1)`, o un `HttpClient` que completa solo). Lee el código antes de reportar; documenta los falsos positivos descartados para cerrar la duda.
4. **Distingue basura de bug latente.** Un `@Input`/signal "sin usar" puede ser un estado que nunca se pinta; léelo antes de proponer borrarlo.
5. **Respeta decisiones de producto.** Si algo parece bug pero podría ser intencional (ej. un validador que siempre devuelve `null`, un guard que deja pasar), repórtalo como pregunta, no lo "arregles" sin confirmación.
6. **Primero lee, luego cambia.** La auditoría es read-only salvo los *quick-wins* de la Fase 3, y esos solo si el usuario lo autoriza.
7. Detecta el gestor de paquetes por el lockfile (`package-lock.json` → npm, `pnpm-lock.yaml` → pnpm, `yarn.lock` → yarn, `bun.lock*` → bun) y la versión de Node esperada (`.nvmrc`, `.node-version`, `engines`, `volta`). Usa siempre ese gestor; los ejemplos usan `npm`/`npx`.
8. Respeta el idioma del proyecto (docs, CLAUDE.md). Si ya existe un `docs/AUDITORIA_*.md`, **verifica su estado actual** en vez de rehacerlo (ver Fase 6).
9. **Documentación actual vía Context7** (ver sección siguiente) para afirmar qué es buena práctica, deprecado o recomendado en la versión de Angular *del proyecto*.

## Documentación con Context7

Lo que es "buena práctica" o "deprecado" en Angular depende de la versión (NgModules, `*ngIf`, guards de clase, Karma…). No lo afirmes de memoria.

1. **Comprueba si está disponible** al empezar: busca herramientas `resolve-library-id` / `query-docs` de Context7 (directas o diferidas vía ToolSearch).
2. **Si está disponible, úsalo** antes de clasificar un hallazgo como deprecado, inseguro o anti-patrón:
   - `resolve-library-id` con la librería (ej. "angular", "angular material", "ngrx", "rxjs") y elige el **ID de la versión instalada** cuando exista.
   - `query-docs` con una pregunta completa de **un solo concepto** (ej. "sanitización y bypassSecurityTrust en Angular 18", "takeUntilDestroyed y DestroyRef"). Una consulta por concepto.
   - En el hallazgo, enlaza o menciona la fuente oficial que lo respalda.
3. **Si está instalado pero no autenticado**, dile al usuario que lo autorice (`/mcp` en una sesión interactiva o la configuración de conectores de claude.ai) y sigue con `angular.dev` y la web mientras tanto.
4. **Si no está instalado**, recomiéndalo una vez al inicio, sin bloquear la auditoría:
   > Recomiendo instalar Context7 para contrastar los hallazgos con documentación actual y versionada de Angular y sus librerías: `/plugin marketplace add upstash/context7` y `/plugin install context7@context7-marketplace`; reinicia Claude Code y ejecuta `/mcp` para iniciar sesión en Context7 (no requiere API key). En entornos sin navegador (SSH, CI): `npx ctx7 setup --claude`.

   Si no lo instala, usa `angular.dev`, changelogs y la web, y anota en la cabecera del documento que se auditó sin Context7.

## Escala de prioridad

- **P0** bloquea release / riesgo real (secretos abusables en el bundle, `bypassSecurityTrust*` con datos de usuario, build prod apuntando a dev, auth solo en el cliente).
- **P1** alto impacto (fuga de PII/tokens a consola o a Sentry, CI ausente, suite de tests rota, `strict` desactivado, vulnerabilidades altas en `npm audit` explotables en runtime).
- **P2** calidad (change detection excesiva, cobertura, `any` masivo, componentes-Dios, bundle fuera de budget, números mágicos).
- **P3** pulido (imports muertos, `catch` vacíos, TODOs, APIs deprecadas, versiones a medias).

## Fase 0 — Contexto (no saltar)

```bash
cat package.json angular.json tsconfig.json tsconfig.app.json 2>/dev/null
ls .nvmrc .node-version .github .gitlab-ci.yml .eslintrc* eslint.config.* .prettierrc* 2>&1
cat CLAUDE.md README.md 2>/dev/null | head -100; ls docs
git log --oneline | head -20
npx ng version
find src -name '*.ts' -not -name '*.spec.ts' | wc -l; find src -name '*.ts' -not -name '*.spec.ts' | xargs cat | wc -l
find src -name '*.html' | wc -l
ls src/app src/app/features src/app/core src/app/shared 2>/dev/null
```

Anota: tamaño, nº de features/rutas lazy, versión de Angular, ¿workspace multi-proyecto o Nx?, ¿standalone o NgModules (`grep -rl "@NgModule" src | wc -l`)?, ¿signals o RxJS-céntrico?, state management (NgRx Store/SignalStore, NGXS, Akita, servicios con `BehaviorSubject`), librería UI (Material, PrimeNG, Tailwind…), cliente HTTP e interceptores, auth (OIDC, Firebase, JWT propio), SSR/hydration, PWA/service worker, i18n (`@angular/localize`, Transloco, ngx-translate), builder (`@angular/build:application` / `browser-esbuild` / `browser` webpack), test runner (Karma, Jest, Vitest) y E2E (Cypress, Playwright), entornos (`environments/*.ts`, `fileReplacements`), y si la documentación del repo describe la realidad.

## Fase 1 — Baseline objetivo

```bash
npx ng build --configuration production 2>&1 | tail -40   # ¿compila? warnings, budgets, tamaño inicial
npx ng lint 2>&1 | tail -40                              # ¿hay lint configurado? ¿cuántos errores?
npx tsc -p tsconfig.app.json --noEmit 2>&1 | tail -20
npx prettier --check "src/**/*.{ts,html,scss}" 2>&1 | tail -5   # si hay prettier
npx ng test --watch=false 2>&1 | tail -30                # ¿compila? ¿pasa? (Karma necesita ChromeHeadless: --browsers=ChromeHeadless)
npm outdated 2>&1 | head -40
npm audit --omit=dev 2>&1 | tail -30
```

- Si `ng lint` falla con "Cannot find builder", **no hay lint configurado**: hallazgo P1 (sugerir `ng add angular-eslint`).
- Revisa `tsconfig.json`: `strict`, `noImplicitAny`, `strictTemplates`, `noPropertyAccessFromIndexSignature`. `strict: false` o `strictTemplates` ausente = P1 (los errores de plantilla solo aparecen en runtime).
- Un solo `.spec.ts` que no compila tumba toda la suite y **oculta** otros fallos: repórtalo y anota que al arreglarlo pueden aparecer más.
- `npm audit`: separa vulnerabilidades de **dependencias de runtime** (llegan al navegador) de las de tooling de build/dev (riesgo mucho menor). No copies el listado entero.

## Fase 2 — Barridos por categoría

Ejecuta los greps, registra los conteos en la tabla de métricas y **abre los archivos** de los resultados relevantes.

### 1. Seguridad
- **Secretos en el bundle**: todo lo que esté en `environment*.ts` o en código fuente **acaba en el JS público**. `grep -rnEi "(api[_-]?key|secret|token|password|AIza|sk_live|pk_|client_secret)" src` y revisa `environments/`. Matiza: Firebase apiKey, Stripe *publishable*, Maps key restringida por referrer son públicos por diseño; el riesgo real son claves de servidor (`sk_live`, `client_secret`, credenciales de BD/servicios) y Maps keys sin restringir. ¿`.env` versionado? `git ls-files | grep -E "\.env|\.pem|\.key|credentials"`.
- **Fail-open a dev**: ¿`environment.prod.ts` existe y `fileReplacements` está en la configuración `production` de `angular.json`? ¿Alguna URL de API se resuelve con fallback a dev (`|| 'https://dev...'`)? ¿El build de despliegue usa realmente `--configuration production`?
- **XSS / sanitización**: `grep -rn "bypassSecurityTrust" src` (cada uso: ¿de dónde viene el valor? Con datos de usuario/backend = P0), `grep -rn "\[innerHTML\]\|innerHTML\s*=" src`, `nativeElement`, `document.write`, `eval(`, `new Function(`, `Renderer2.setProperty(.*innerHTML`. Revisa librerías de markdown/rich text y si sanitizan.
- **Fugas a consola/monitorización**: `grep -rnE "console\.(log|debug|info|warn|error)" src | wc -l` y abre los que imprimen usuarios, tokens, `HttpErrorResponse` completos o payloads. A diferencia de Flutter, **no hay eliminación automática** en prod salvo que se configure. Revisa el `ErrorHandler`/Sentry/Datadog (`grep -rn "captureException\|setUser\|setContext\|ErrorHandler"`): no deben llevar PII ni cuerpos de request/response.
- **Auth y almacenamiento de tokens**: `grep -rn "localStorage\|sessionStorage" src` → ¿guarda access/refresh tokens? (expuestos a cualquier XSS; valorar cookies `HttpOnly` según backend). Interceptor: ¿qué hace ante 401/403? ¿refresh con cola o bucle infinito? ¿adjunta el token a **cualquier** dominio (fuga a terceros) o solo a la API propia?
- **Guards**: ¿las rutas protegidas tienen `canActivate`/`canMatch`? Recuerda que los guards son UX, no seguridad: verifica que la autorización también exista en backend (si no es verificable desde el repo, va a "No verificable").
- **Cabeceras / CSP**: busca CSP (`<meta http-equiv="Content-Security-Policy"`, config de nginx/Firebase Hosting/Vercel/`server.ts`), `autoCsp` en `angular.json`, `X-Frame-Options`. Reporta explícitamente si "depende del hosting".
- **SSR** (si aplica): ¿`server.ts` filtra hosts permitidos? ¿se filtran secretos de servidor al `TransferState`? ¿llamadas HTTP absolutas a dev desde el servidor?

### 2. Performance
- **Change detection**: `grep -rc "ChangeDetectionStrategy.OnPush" src` vs nº de componentes (`grep -rl "@Component" src | wc -l`). ¿La app es zoneless (`provideZonelessChangeDetection`) o usa `zone.js`? Componentes grandes con `Default` + timers/streams = ciclos de más.
- **Llamadas a métodos en plantillas**: `grep -rnE "\{\{[^}]*\w\([^)]*\)[^}]*\}\}" src --include=*.html` y bindings `[x]="metodo()"` (se ejecutan en cada ciclo; las lecturas de signals `sig()` son la excepción y están bien).
- **Listas**: `*ngFor` sin `trackBy` (`grep -rn "\*ngFor" src | grep -vc trackBy`), `@for` con `track $index` en listas que se reordenan, listas largas sin virtual scroll (`cdk-virtual-scroll-viewport`).
- **Suscripciones**: `grep -rc "\.subscribe(" src` vs `takeUntilDestroyed`/`takeUntil(`/`async` pipe/`toSignal`/`unsubscribe(`. **Verifica cada candidato**: `HttpClient` completa solo, `ActivatedRoute` se limpia con el componente; `interval`, `fromEvent`, `valueChanges`, stores y `Subject`s propios no. `setInterval`/`setTimeout`/`addEventListener` sin limpieza en `ngOnDestroy`/`DestroyRef`.
- **Subscribe anidados** (`subscribe` dentro de `subscribe`): condiciones de carrera; proponer `switchMap`/`concatMap`.
- **Bundle**: salida de `ng build` (initial total, lazy chunks), `budgets` en `angular.json` (¿existen? ¿se han subido para "pasar"?), warnings de CommonJS (`allowedCommonJsDependencies`), imports pesados (`moment`, `lodash` completo, iconos completos). Para detalle: `ng build --stats-json` y analizar con `esbuild` metafile / `source-map-explorer`.
- **Lazy loading**: rutas con `loadComponent`/`loadChildren` vs rutas eager; `@defer` para bloques pesados; módulos de terceros cargados en el arranque (`app.config.ts`/`AppModule`).
- **Imágenes**: `<img` sin `NgOptimizedImage` (`ngSrc`) en imágenes LCP, sin `width/height` (CLS), assets pesados (`find src/assets public -size +500k`).
- **SSR/hydration**: si hay SSR, ¿`provideClientHydration()` activo? ¿hay errores de hydration mismatch documentados (`ngSkipHydration`)?

### 3. Refactor / calidad de código
- **Componentes/servicios-Dios**: `find src -name '*.ts' -not -name '*.spec.ts' | xargs wc -l | sort -rn | head -15` (umbral >600 líneas en componentes, >900 en general) y lo mismo para `.html` (>400 líneas). Componentes con muchas dependencias inyectadas (>7 en el constructor/`inject`).
- **Tipado**: `grep -rcE ":\s*any\b|<any>|as any" src` (conteo total y top archivos), `// @ts-ignore`, `@ts-expect-error`, `!` non-null masivo. Respuestas HTTP sin tipar (`http.get(` sin genérico) o interfaces que no reflejan el backend. Detecta **duplicación del unwrap** de la respuesta (`res.data`, `res[0]`) repetida por servicio en vez de un interceptor/mapper.
- **Errores silenciados**: `catchError(() => of(null))`, `catchError(() => EMPTY)`, `catch (e) {}`, `subscribe({ error: () => {} })`. Clasifica: aceptables (telemetría que no debe romper) vs fallos invisibles en flujos de negocio/dinero.
- **Números/strings mágicos**: IDs de dominio como enteros crudos (`=== 1`, `role === 'admin'`, `statusId == 3`), estados como strings. Proponer `enum`/union types/`as const`. Cuidado con `==` vs `===`.
- **Duplicación de constantes** (URLs base, endpoints, mails, teléfonos en 2+ sitios; URLs fuera de `environment`).
- **i18n**: si hay i18n, ratio de textos traducidos vs literales en plantillas (`grep -rnE ">\s*[A-Za-zÁÉÍÓÚáéíóúñ][^<{]{3,}<" src --include=*.html`); verifica que las claves existan en **todos** los idiomas. Si no hay i18n y el producto lo necesita, anótalo.
- **APIs deprecadas / legado**: NgModules en app nueva, `*ngIf`/`*ngFor` frente a control flow (`@if`/`@for`), constructor injection vs `inject()`, `@Input()`/`@Output()` vs `input()`/`output()`, guards/resolvers de clase, `HttpClientModule`, `RouterTestingModule`, `TestBed` con `HttpClientTestingModule`. No son bugs: **inventaria** y remite a `angular-upgrade` (hay schematics oficiales para migrarlos).
- **TODO/FIXME/HACK**: `grep -rnE "TODO|FIXME|HACK|XXX" src | wc -l`; separa notas de deuda funcional real y detecta TODOs obsoletos.
- Código muerto, imports sin usar (`ng lint`, `ng g @angular/core:cleanup-unused-imports` si existe en la versión), dependencias de `package.json` que no se importan (`npx depcheck` o grep), `devDependencies` en `dependencies` y viceversa.

### 4. Arquitectura y buenas prácticas
- ¿Estructura feature-first (`core/`, `shared/`, `features/`) coherente? ¿Lógica de negocio dentro de componentes? ¿componentes que llaman directamente a `HttpClient` en vez de a un servicio/repositorio?
- **Smart vs presentational**: componentes de UI que inyectan servicios de datos; `shared/` que importa de `features/` (dependencias circulares: `npx madge --circular --extensions ts src`).
- **Inyección y testabilidad**: `providedIn: 'root'` vs providers en componentes, `InjectionToken` para config, servicios con estado global mutable.
- **Router**: guards, resolvers, `**` wildcard / página 404, rutas placeholder, `withComponentInputBinding`, preloading strategy.
- **Estado**: `BehaviorSubject` expuestos públicamente (`.next()` desde fuera), mezclas incoherentes de signals/RxJS/NgRx, `effect()` que escribe en signals (bucles), stores "gordos", lecturas de estado en `ngOnInit` que no reaccionan a cambios (UI desactualizada: es caza de bugs, no cosmético).
- **Formularios**: `FormGroup` sin tipar (`UntypedFormGroup`), validación solo en plantilla, lógica duplicada entre formularios.
- **Observabilidad**: `ErrorHandler` global personalizado, interceptor de errores HTTP centralizado, logger con niveles y redacción, analytics real vs solo `console`.
- **Config por entorno**: `environments` + `fileReplacements` vs config en runtime (`assets/config.json` cargado con `provideAppInitializer`/`APP_INITIALIZER`) — importa si la misma imagen se despliega en varios entornos.
- **CI/CD**: existencia de `.github/workflows`, `.gitlab-ci.yml`, Azure Pipelines, Vercel/Netlify/Firebase config. ¿Corre lint + test + build prod? Si no hay CI: P1 (mayor multiplicador de calidad por esfuerzo).
- **Documentación**: ¿README/CLAUDE.md/ARQUITECTURA describen la realidad actual (versión de Angular, comandos, entornos)?

### 5. Tests
- `find src -name '*.spec.ts' | wc -l` vs nº de componentes/servicios. Mapa de qué features tienen cobertura y cuáles no. Si hay cobertura configurada: `ng test --watch=false --code-coverage` y reporta el resumen.
- **Specs plantilla**: `it('should create'` como único test del archivo (`grep -rl "should create" src | xargs grep -c "it(" | grep ":1$"`) — cobertura aparente, no real.
- **Cobertura en flujos de dinero/críticos** (pagos, checkout, permisos, auth, interceptores, guards): suele ser cero → P2 con prioridad concreta.
- Specs con `xit`/`xdescribe`/`fit`/`fdescribe` (un `fit` olvidado hace que **solo** corra ese test), mocks desactualizados que no compilan, tests que prueban tautologías.
- E2E: ¿existen (Cypress/Playwright)? ¿corren en CI o están abandonados?
- Cuando destrabes la compilación, **corre la suite completa**: pueden aparecer fallos ocultos; investígalos (¿bug de código o test desactualizado? ¿decisión de producto?).

### 6. Dependencias e integraciones
- `package.json`: versión de Angular vs soportadas (cada major tiene ~18 meses de soporte: revisa `angular.dev/reference/releases`); paquetes `@angular/*` con versiones desalineadas entre sí; librerías de terceros atadas a un major viejo de Angular (bloquean el upgrade); `overrides`/`resolutions` y su motivo; rangos `^0.x`; deps abandonadas (último release > 2 años) o deprecadas (`npm view <pkg> deprecated`); `--legacy-peer-deps` en scripts/CI/`.npmrc` (oculta conflictos reales).
- Versión de Node: `.nvmrc`/`engines` vs Node que exige la versión de Angular instalada; ¿CI usa la misma?
- Scripts y SDKs de terceros en `index.html` (analytics, chat, tag managers): impacto en rendimiento y privacidad, ¿cargados con consentimiento?
- Integraciones **comentadas o con feature flag** (verifica si siguen comentadas: el estado cambia).
- Polyfills y `browserslist`: ¿se apunta a navegadores que ya no hace falta soportar?

## Fase 3 — Quick-wins (solo con autorización del usuario)

Cambios mecánicos y sin riesgo, aplicables en la misma pasada. Reporta antes/después con números:
- Eliminar `console.*` con PII/tokens; los demás tras un logger o condicionados a `isDevMode()`; recortar cuerpos de response de los reportes de error.
- Quitar `fit`/`fdescribe` olvidados; borrar specs plantilla vacías solo si el usuario lo prefiere a completarlas.
- Mover dependencias mal clasificadas (`dependencies` ↔ `devDependencies`); borrar dependencias no importadas (verificado).
- Añadir `trackBy`/`track` correctos donde sea trivial.
- `ng lint --fix` (inspecciona el diff antes).
Deja **fuera** cualquier cambio de lógica, formato masivo (`prettier --write` va en commit aislado), activar `strict`/`strictTemplates` (sube el conteo de errores de golpe: MR aparte) y migraciones con schematics (van en `angular-upgrade` o en su propio MR).

## Fase 4 — Redactar el documento

Estructura de `docs/AUDITORIA_<YYYY-MM>.md`:

1. **Cabecera**: fecha, versión de Angular/Node, nº archivos/líneas/componentes/features, nota "cita rutas, nunca valores".
2. **Resumen ejecutivo**: 1 párrafo: qué está sólido y que las brechas son de endurecimiento o estructurales. Leyenda P0–P3.
3. **Secciones numeradas** por categoría (Seguridad, Refactor, Arquitectura, Performance, Limpieza, Dependencias). Cada hallazgo:
   `### P<n> — <título>` → archivo:línea, snippet corto si ayuda, **por qué importa** (riesgo concreto, no genérico), matiz (qué es falso positivo/público por diseño) y **Acción** concreta (sin estimar viabilidad ni tiempo).
4. **Verificado y limpio**: falsos positivos descartados.
5. **Métricas base** (tabla con conteo + comando fuente): archivos·líneas, errores de build/lint/tsc, tamaño inicial del bundle vs budget, componentes OnPush/total, `.subscribe(` vs mecanismos de limpieza, `any`, `bypassSecurityTrust`, `console.*`, `catchError` silenciosos, archivos >600/900 líneas, specs·componentes, specs plantilla, vulnerabilidades runtime (alta/crítica), archivos sin formatear.
6. **Plan de ataque** ordenado: P0 ahora → P1 sprint 1 → P2 sprint 2 → P3 continuo.
7. **Checklist de seguimiento** en tandas por esfuerzo/impacto, con IDs estables (`B1…Bn`) y casillas `- [ ]`:
   - Tanda 1: trivial, mecánico, sin riesgo.
   - Tanda 2: mecánico con criterio (≈0.5–1 día).
   - Tanda 3: transversal, requiere revisión (≈1–3 días).
   - Tanda 4: especializado, alto impacto (por sprint).
   Anota **dependencias entre ítems** (ej. "CI después de arreglar tests y configurar lint", "activar `strictTemplates` después de tipar formularios", "dividir componentes grandes después de tener tests") y los que no son cerrables desde el repo (ej. restringir API key en consola cloud, cabeceras en el hosting).
8. **Tabla de seguimiento** con columnas de fecha (antes/después/objetivo).
9. **Pendiente fuera del código**: lo que requiere confirmar en consolas externas, hosting o backend.

## Fase 5 — Entrega

Resume al usuario: top 3 riesgos, métricas clave, qué quick-wins aplicaste (con números) y qué decisiones necesitas de él (producto/backend). No pegues el documento entero en el chat.

## Fase 6 — Re-auditoría (si ya existe un doc previo)

Añade sección "Plan de trabajo y seguimiento (fecha)": verifica **cada** hallazgo anterior contra el repo hoy (lee los archivos, reejecuta los greps), clasifica en **A. Resuelto ✅ (con evidencia)** y **B. Pendiente**, anota cambios no documentados (ej. una integración que estaba comentada y ya está activa, un upgrade de Angular), actualiza la tabla de métricas y registra hallazgos nuevos destapados al arreglar bloqueantes. No reescribas el historial: añade.

## Anti-patrones

- Listar problemas genéricos de Angular sin evidencia del repo.
- Reportar conteos sin abrir los archivos (falsos positivos, sobre todo en suscripciones).
- Tratar el uso de APIs "legado" (NgModules, `*ngIf`) como bugs en vez de deuda migrable.
- Mezclar decisiones de producto con bugs.
- Proponer refactors grandes (dividir componentes de 2k líneas, migrar a signals) sin red de tests previa.
- Incluir valores de secretos en el documento.
