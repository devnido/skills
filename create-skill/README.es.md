> **Nota:** este archivo es la traducción en español de `SKILL.md` para lectura humana.
> No es una skill registrada — Claude Code solo carga `SKILL.md`. No lo uses como
> contexto operativo.

# Crear y Actualizar Skills

Esta skill **crea y actualiza** skills de Claude Code de forma consistente. Se activa
tanto cuando el usuario pide crear una skill nueva como cuando pide editar, modificar
o actualizar una existente — en ambos casos mantiene sincronizados `SKILL.md` y
`README.es.md`.

## Artefactos requeridos

Para cada nueva skill llamada `<nombre-skill>`:

1. **Archivo de skill (inglés, canónico)**: `/Users/luis/dev/claude-code/skills/<nombre-skill>/SKILL.md`
   - Debe tener frontmatter YAML con los campos `name` y `description`.
   - `name` debe coincidir exactamente con el nombre de la carpeta.
   - `description` debe ser específica y enumerar frases de activación (incluyendo
     equivalentes en español cuando aplique) para que Claude rutee correctamente a la
     skill.
   - El cuerpo explica qué hace la skill, cuándo usarla, inputs requeridos y cualquier
     convención o instrucción paso a paso que Claude deba seguir.

2. **Espejo en español**: `/Users/luis/dev/claude-code/skills/<nombre-skill>/README.es.md`
   - Traducción fiel en español del cuerpo de SKILL.md. **No** incluir frontmatter YAML —
     este archivo es documentación humana, no una skill registrada.
   - Empezar con una nota corta aclarando que es traducción en español para lectura
     humana, no contexto operativo para agentes.
   - Propósito: referencia legible para el usuario. Claude Code descubre skills por el
     nombre exacto `SKILL.md`, así que `README.es.md` nunca se carga como skill.
   - El nombre `README.es.md` (en vez de `SKILL.es.md`) evita que cualquier agente lo
     confunda con un archivo de skill al listar archivos.

3. **Symlink de registro**: `~/.claude/skills/<nombre-skill>` →
   `/Users/luis/dev/claude-code/skills/<nombre-skill>`
   - Usar `ln -s` con rutas absolutas.
   - Verificar con `ls -la ~/.claude/skills/` después.

## Flujo de trabajo

1. Preguntar al usuario (o inferir de la petición):
   - `<nombre-skill>` (kebab-case, coincide con la carpeta).
   - Propósito en una frase.
   - Frases de activación que el usuario espera (inglés + español si aplica).
2. Redactar SKILL.md con un `description` rico (triggers específicos, no vagos) y un
   cuerpo claro. Evitar descripciones genéricas tipo "ayuda con X" — ser explícito
   sobre *cuándo* activarla.
3. Traducir el cuerpo a README.es.md (sin frontmatter). Mantener los títulos paralelos
   para facilitar la comparación. Agregar una línea arriba marcándolo como traducción
   en español para lectura humana.
4. Crear el symlink en `~/.claude/skills/`.
5. Confirmar el éxito al usuario con las tres rutas de artefactos y recordarle que la
   skill está disponible de inmediato (Claude Code detecta skills en `~/.claude/skills/`
   en el siguiente prompt; no requiere reinicio).

## Actualizar una skill existente

Cuando el usuario pida cambiar, editar o actualizar una skill (su descripción, cuerpo,
triggers, reglas, o cualquier contenido), DEBES aplicar el mismo cambio a ambos
archivos para mantenerlos sincronizados:

1. Actualizar `SKILL.md` (versión canónica en inglés).
2. Aplicar el cambio equivalente a `README.es.md`, traducido al español, manteniendo
   los títulos y la estructura paralelos. `README.es.md` no tiene frontmatter.
3. Si solo existe un archivo para una skill antigua, crear el espejo faltante antes
   de finalizar la edición.
4. Confirmar al usuario que ambos archivos fueron actualizados.

Esta regla aplica incluso si el usuario solo menciona un idioma — el espejo en
español nunca debe desincronizarse de la versión canónica en inglés.

## Convenciones

- Trabajar siempre dentro de `/Users/luis/dev/claude-code` (nunca rutas padre/hermanas).
- Nombre de la carpeta de la skill = campo `name` del YAML. Sin espacios, kebab-case.
- Mantener SKILL.md enfocado: qué hace la skill, cuándo activarla, inputs requeridos,
  reglas.
- NO editar settings.json para skills — el descubrimiento por filesystem lo maneja el
  symlink.
- Si la skill necesita archivos de soporte (plantillas, referencias), colocarlos en una
  subcarpeta `references/` dentro del directorio de la skill.
