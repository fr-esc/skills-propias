# skills-propias

Skills personalizadas para [OpenCode](https://opencode.ai), versionadas como copia de seguridad y para poder restaurarlas en cualquier máquina.

Cada carpeta es una skill: OpenCode carga su `SKILL.md` y usa su frontmatter (`name`, `description`) para decidir cuándo activarla.

## Ubicación en el sistema

```
~/.config/opencode/skills/
```

Este repositorio **es** esa carpeta. No hay copia intermedia: lo que se edita aquí es lo que OpenCode ejecuta.

## Skills incluidas

### Implementación dirigida por comentarios

| Skill | Qué hace |
|---|---|
| `comment-to-code` | Enrutador: detecta la extensión y delega en la sub-skill del lenguaje |
| `comment-to-python` | Comentarios guía `.py` → código Python in-place |
| `comment-to-typescript` | Comentarios guía `.ts/.tsx/.js/.jsx` → TypeScript/React in-place |
| `comment-to-dart` | Comentarios guía `.dart` → Dart/Flutter in-place |
| `comment-to-sql` | Comentarios guía `.sql` → SQL limpio e idempotente in-place |

### Diseño y arquitectura de proyectos

| Skill | Qué hace |
|---|---|
| `socratic-interviewer` | Entrevista por turnos → `DESIGN_LOG.md` + `DESIGN_BRIEF.md` |
| `socratic-blueprint-generator` | Brief → arquitectura modular, fichas `MOD-XXX` y backlog atómico |

### Generación de código y datos

| Skill | Qué hace |
|---|---|
| `guided-script-builder` | Script Python autocontenido para UV, con entrevista previa |
| `etl-python` | Pipeline ETL/scraping de producción desde una fuente real |
| `generador-datasets` | Documentos → datasets sintéticos JSONL para SFT/RAG |
| `finetune-script-generator` | Notebook de fine-tuning (Unsloth, QAT, TRL, ORPO) listo para Colab/Kaggle/Molab |

### Análisis de documentos

| Skill | Qué hace |
|---|---|
| `dense-doc-digester` | Documento denso → digest estructurado de 8 secciones |
| `extraccion-tecnica-con-verificacion` | Auditoría de parámetros concretos con cita literal y cero alucinación |

## Convenciones del repositorio

- **`id = name = nombre de carpeta`**, siempre en `kebab-case`. OpenCode usa la carpeta como identificador.
- **Un `SKILL.md` por skill**, con frontmatter YAML mínimo: `name` y `description`. El resto de claves (`trigger`, `related_skills`, `annexes`, etc.) son documentación para humanos.
- **`description` en tercera persona** con el patrón *«Úsala cuando… / no la uses para…»*: es lo que OpenCode usa para activarla automáticamente y lo que evita que dos skills se pisen.
- **Sin variables de plantilla**: OpenCode nunca sustituye un `{{MARCADOR}}`. Las entradas se describen en prosa y se identifican por su contenido en el mensaje del usuario, no por un hueco que nadie rellena.
- **Fronteras explícitas**: las skills que se solapan declaran a quién ceder el caso.
- **`## Eval de aceptación`** al final de cada una: entrada mínima, salida esperada, prohibiciones y un test negativo de frontera. Sirve como guion de prueba manual.
- **Anexos en `reference/`**: contenido largo (plantillas, ejemplos few-shot) que no se precarga. El `SKILL.md` indica cuándo leerlo y con qué ruta absoluta.

## Restaurar en otra máquina

```bash
git clone git@github.com:fr-esc/skills-propias.git \
  ~/.config/opencode/skills
```

Si el directorio ya existe vacío:

```bash
git clone git@github.com:fr-esc/skills-propias.git /tmp/skills-propias
cp -a /tmp/skills-propias/. ~/.config/opencode/skills/
```
