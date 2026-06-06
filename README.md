# Claude Code Skills

Colección de [Agent Skills](https://docs.anthropic.com/en/docs/claude-code/skills) para Claude Code. Cada skill extiende al agente con instrucciones especializadas que se activan automáticamente según el contexto de la conversación.

## Skills disponibles

| Skill | Descripción |
|---|---|
| [create-skill](./create-skill/) | Crea y actualiza skills de Claude Code de forma consistente (scaffold, symlink, espejo en español). |
| [hexagonal-architecture](./hexagonal-architecture/) | Impone arquitectura hexagonal (Ports & Adapters) con vertical slicing en proyectos TypeScript: NestJS, Vue, React, React Native y Next.js. |
| [criteria-pattern](./criteria-pattern/) | Aplica el patrón Criteria (CodelyTV) para búsquedas dinámicas con filtros, orden y paginación en microservicios DDD/hexagonal. |

## Estructura de cada skill

```
<skill-name>/
├── SKILL.md          # Canónico (inglés) — lo que Claude Code carga como skill
├── README.es.md      # Traducción al español para lectura humana
└── references/       # Material de soporte opcional (plantillas, guías, etc.)
```

- **`SKILL.md`** es el archivo operativo. Debe tener frontmatter YAML con `name` y `description` (con frases de activación explícitas).
- **`README.es.md`** es documentación humana. Claude Code **no** lo carga como skill.
- El nombre de la carpeta debe coincidir exactamente con el campo `name` del frontmatter (kebab-case).

## Registro

Las skills se registran con un symlink en `~/.claude/skills/`:

```bash
ln -s /Users/luis/dev/claude-code/skills/<skill-name> ~/.claude/skills/<skill-name>
```

Claude Code detecta las skills en ese directorio en el siguiente prompt; no requiere reinicio.

Skills registradas actualmente desde este repo:

```bash
ls -la ~/.claude/skills/ | grep claude-code/skills
```

## Crear una skill nueva

Pide al agente *"crear una skill"* o *"crear una skill para …"* — activará la skill [create-skill](./create-skill/), que generará `SKILL.md`, `README.es.md` y el symlink de registro.

## Skills relacionadas (fuera de este repo)

Otras skills del workspace viven bajo `patina/workflow/patina-backend-nestjs/skills/` (workflow Spec-Driven Development: `create-use-case`, `generate-dev-plan`, `implement-dev-plan`, `spec-nido`, `use-case-status`, etc.) y también se registran vía symlink en `~/.claude/skills/`.
