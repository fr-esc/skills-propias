---
name: comment-to-code
alias: file-from-comments
description: Enrutador para peticiones de implementación que abarcan varios archivos de lenguajes distintos (.py, .ts/.tsx/.js/.jsx, .sql, .dart) o cuyo lenguaje no está claro; delega cada archivo en la skill especializada sin mezclar reglas. Úsala solo cuando haya más de un lenguaje implicado o la extensión sea desconocida; si la petición afecta a un único archivo de extensión conocida, no la uses y carga directamente comment-to-python, comment-to-typescript, comment-to-sql o comment-to-dart.
version: 1.2.0
language: es
trigger: "implementa|implanta|genera el código de *"
supported_extensions:
  - .py
  - .ts
  - .tsx
  - .js
  - .jsx
  - .sql
  - .dart
outputs:
  - skill-routing-and-execution
---

# Skill: `comment-to-code` (Enrutador Principal)

## 1. Propósito y Cuándo Activarse
Esta skill actúa como punto de entrada único cuando el usuario pide implementar, implantar o generar el código de cualquier archivo que contenga comentarios guía.

Para evitar contaminar la ventana de contexto en modelos pequeños (<= 7B), **esta skill NO contiene reglas de múltiples lenguajes mezcladas**: su única responsabilidad es inspeccionar la extensión del archivo solicitado y activar exclusivamente la sub-skill del lenguaje correspondiente.

---

## 2. Tabla de Enrutamiento Estricto por Extensión

| Extensión del archivo | Sub-Skill que se debe cargar y ejecutar | Comentario ancla esperado | Formato del bloque `Realizado:` final |
| :--- | :--- | :--- | :--- |
| **`.py`** | `comment-to-python` | `# ...` | `"""\nRealizado:\n...\n"""` |
| **`.ts`, `.tsx`, `.js`, `.jsx`** | `comment-to-typescript` | `// ...` | `/*\nRealizado:\n...\n*/` |
| **`.sql`** | `comment-to-sql` | `-- ...` | `/*\nRealizado:\n...\n*/` |
| **`.dart`** | `comment-to-dart` | `// ...` | `/*\nRealizado:\n...\n*/` |

---

## 3. Protocolo de Ejecución Paso a Paso

1. **Identificar el archivo objetivo:** Lee el nombre y la extensión del archivo que el usuario indicó en el prompt o que tiene abierto en el editor (ej. `prueba.py`, `Tarjeta.tsx`, `migracion.sql`, `boton.dart`).
2. **Seleccionar una única sub-skill:**
   - Si termina en `.py` -> Carga y aplica estrictamente las reglas de `comment-to-python`.
   - Si termina en `.ts`, `.tsx`, `.js` o `.jsx` -> Carga y aplica estrictamente las reglas de `comment-to-typescript`.
   - Si termina en `.sql` -> Carga y aplica estrictamente las reglas de `comment-to-sql`.
   - Si termina en `.dart` -> Carga y aplica estrictamente las reglas de `comment-to-dart`.
   - Si la extensión no está en la tabla -> Informa en una sola línea qué extensiones están soportadas (`.py`, `.ts`, `.tsx`, `.js`, `.jsx`, `.sql`, `.dart`) y detente.
3. **Invariantes Globales (Comunes a todas las sub-skills):**
   - Mantén **todos** los comentarios originales del usuario intactos en su posición.
   - Escribe el código justo debajo de cada comentario sin añadir documentación adicional no solicitada (sin docstrings, sin JSDoc, sin `///`).
   - Si un comentario pide usar una función, clase, componente o tabla externa, **impórtala o referénciala sin recrearla**.
   - Añade siempre al final del archivo el bloque multilínea temporal `Realizado:` para que el usuario revise los cambios antes de borrarlo.

---

## 4. System Prompt Directo para el Enrutador (Modelos <= 7B)

```text
You are a strict file-extension router and code implementation dispatcher inside an IDE.
Task: Inspect the target file extension and apply ONLY the matching language rule-set to implement code directly below each user comment.

ROUTING TABLE:
- .py -> Apply 'comment-to-python' (PEP 8, type hints, keep # comments, append """\nRealizado:\n...""" at the end).
- .ts, .tsx, .js, .jsx -> Apply 'comment-to-typescript' (Strict TS, no 'any', ES Modules, React/RN detection, keep // comments, append /*\nRealizado:\n...*/ at the end).
- .sql -> Apply 'comment-to-sql' (Uppercase SQL keywords, snake_case, IF NOT EXISTS, keep -- comments, append /*\nRealizado:\n...*/ at the end).
- .dart -> Apply 'comment-to-dart' (Dart 3+ null safety, const Flutter widgets, keep // comments, append /*\nRealizado:\n...*/ at the end).

GLOBAL INVARIANTS:
1. NEVER delete or modify user comments.
2. NEVER add docstrings, JSDoc, or inline explanatory comments.
3. NEVER recreate external functions/classes/tables mentioned as existing in other modules; import or reference them.
4. Output ONLY the complete valid file content in-place.
```

---

## 5. Estructura Recomendada de Carpetas en tu Proyecto

Para que el asistente de tu IDE (Cursor, Windsurf, Continue, VS Code + Ollama) las encuentre de forma limpia y modular:

```text
skills/
├── comment-to-code/
│   └── SKILL.md          # Este enrutador principal
├── comment-to-python/
│   └── SKILL.md          # Reglas exclusivas de Python (.py)
├── comment-to-typescript/
│   └── SKILL.md          # Reglas exclusivas de TS/React/RN (.ts, .tsx, .js, .jsx)
├── comment-to-sql/
│   └── SKILL.md          # Reglas exclusivas de SQL (.sql)
└── comment-to-dart/
    └── SKILL.md          # Reglas exclusivas de Dart/Flutter (.dart)
```

---

## Eval de aceptación

Prueba manual mínima del enrutador. Se considera **APROBADA** solo si se cumplen todos los puntos.

**Entrada**
> «implementa el código de `boton.dart`»
> (el archivo contiene dos comentarios guía `//`: uno pidiendo un `StatelessWidget` y otro pidiendo un `Text`)

**Salida esperada**
- Se carga **una única** sub-skill: `comment-to-dart`.
- Los dos comentarios originales permanecen intactos y en su posición.
- Debajo de cada comentario aparece el código Dart 3 correspondiente, con null safety.
- Al final del archivo, bloque `/* Realizado: ... */`.

**Prohibido en esta prueba**
- Cargar dos o más sub-skills o mezclar reglas de dos lenguajes.
- Aplicar convenciones de Python o SQL a un archivo `.dart`.
- Añadir doc comments `///`.

**Test negativo**
> «implementa el código de `consulta.rs`»

**Resultado correcto del test negativo:** una sola línea informando las extensiones soportadas (`.py`, `.ts`, `.tsx`, `.js`, `.jsx`, `.sql`, `.dart`) y parada inmediata, sin generar código.
