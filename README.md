# kukul-angular-tools

Marketplace de plugins de Claude Code del equipo Kukul para proyectos Angular.

## Plugins

| Plugin | Qué incluye |
|---|---|
| `kukul-angular` | Skills `angular-audit` y `angular-upgrade` |

## Instalación

Desde la terminal:

```bash
claude plugin marketplace add valentinKukulSoft/kukul-angular-tools
claude plugin install kukul-angular@kukul-angular-tools
```

O desde una sesión de Claude Code:

```
/plugin marketplace add valentinKukulSoft/kukul-angular-tools
/plugin install kukul-angular@kukul-angular-tools
```

Para recibir cambios nuevos (reinicia Claude Code para aplicarlos):

```bash
claude plugin marketplace update kukul-angular-tools
claude plugin update kukul-angular@kukul-angular-tools
```

En sesión: `/plugin marketplace update kukul-angular-tools`.

## Skills

- **`angular-audit`**: auditoría técnica completa de un proyecto Angular (seguridad, performance, refactor, arquitectura, dependencias, tests, CI). Cada hallazgo lleva evidencia y se genera `docs/AUDITORIA_<fecha>.md` priorizado de P0 a P3, con plan de trabajo. Se activa al pedir auditar la salud técnica de una app Angular.
- **`angular-upgrade [versión]`**: actualiza Angular a la última estable, un major a la vez con `ng update`, y luego sus paquetes, por fases, investigando los breaking changes que afectan al proyecto (Node, TypeScript, RxJS, librerías de terceros, builder y test runner). Mantiene `docs/angular-upgrade/README.md` como tablero de seguimiento.

### Recomendado: Context7

Ambas skills usan [Context7](https://github.com/upstash/context7) (MCP) para consultar documentación actual y versionada de Angular y sus librerías en vez de depender de la memoria del modelo. Si no está instalado, la skill lo recomienda y continúa con las fuentes oficiales.

```bash
claude plugin marketplace add upstash/context7
claude plugin install context7@context7-marketplace
```

Reinicia Claude Code, ejecuta `/mcp`, selecciona Context7 e inicia sesión en el navegador (no requiere API key). En entornos sin navegador (SSH, CI) usa `npx ctx7 setup --claude`.

Claude las invoca solo cuando la petición coincide con su descripción, o puedes llamarlas por nombre (por ejemplo `/kukul-angular:angular-upgrade 21`).

## Estructura

```
.claude-plugin/marketplace.json
plugins/kukul-angular/
├── .claude-plugin/plugin.json
└── skills/
    ├── angular-audit/SKILL.md
    └── angular-upgrade/SKILL.md
```

Este repositorio es público: no subas secretos, rutas locales ni datos internos de proyectos.
