---
name: tech-notes-generator
description: Genera notas técnicas didácticas y autocontenidas en tech-notes/<slug>.md para aprender a usar e implementar desde cero cualquier tecnología de software o concepto técnico a partir de --concept o --url. Úsala cuando el usuario quiera documentación más amigable que la oficial adaptada a su nivel (1 a 5). No la uses para hardware ni para resumir fielmente un documento concreto tipo paper, RFC o especificación puntual (delega en dense-doc-digester).
---

# Tech Notes Generator (`tech-notes-generator`)

Genera una nota técnica autocontenida en `tech-notes/<slug>.md` que enseña a **usar e implementar desde cero** una tecnología de software o un concepto técnico (a partir de `--concept <nombre>` o `--url <url>`), funcionando como documentación más clara, estructurada y amigable que la original.

---

## 1. Cuándo Usar Esta Skill (y Frontera con `dense-doc-digester`)

### Usa esta skill cuando:
- El usuario pida aprender, usar o implementar una tecnología, librería, framework, herramienta CLI, protocolo, algoritmo o patrón de software mediante `--concept <nombre>` o `--url <url>`.
- El objetivo sea construir una guía práctica y didáctica desde cero, pudiendo aportar y organizar conocimiento técnico externo verificado para explicar el uso real de la tecnología.

### NO uses esta skill cuando:
- **Frontera con `dense-doc-digester`:** El usuario entregue o enlace **un documento concreto y denso** (un paper académico, un RFC específico, una especificación formal cerrada) con la intención de obtener una versión digerible **fiel exclusivamente a ese texto fuente** sin añadir conocimiento externo. En ese caso, detente e indica:
  > *"Para digerir un documento concreto (paper, RFC o especificación) manteniendo fidelidad estricta a la fuente sin conocimiento externo, utiliza la skill `dense-doc-digester`."*
- **Fuera de alcance (Hardware):** El concepto solicitado sea hardware puro (electrónica física, microchips, placas, cableado, arquitectura física de sensores). En ese caso, detente e informa que la skill cubre exclusivamente software y conceptos técnicos de computación.

---

## 2. Guardarraíles y Reglas Duras (Inquebrantables)

1. **CERO INVENCIÓN DE APIs, SINTAXIS, FLAGS O VERSIONES:**
   - Prohibido inventar nombres de funciones, métodos, parámetros, endpoints, flags de CLI, paquetes o números de versión que no puedas verificar con total certeza.
   - **Fórmula obligatoria ante incertidumbre:** Si desconoces un comando, firma de API, flag o detalle de implementación, o no puedes verificarlo en la fuente, escribe literalmente:
     > *"No puedo verificar con certeza [elemento/API/flag/versión] sin riesgo de inventar sintaxis. Confírmame el detalle exacto o aporta la referencia oficial para incluirlo."*
   - En el campo `version` del frontmatter de la nota generada, **prohibido adivinar versiones exactas** (ej. no pongas `"4.18.2"` si no consta explícitamente). Usa la rama mayor verificada (`"3.x"`, `"ES2022"`), `"unversioned"` para conceptos/patrones teóricos, o `"unknown"` si no puedes confirmarla.

2. **CERO INVENCIÓN ANTE FALLO DE `--url`:**
   - Si la URL proporcionada en `--url <url>` no se puede leer (error 404, bloqueo anti-bot, paywall, login requerido, página renderizada solo por JS vaciada de texto o sin herramienta de red), **DETENTE INMEDIATAMENTE**.
   - Prohibido deducir o rellenar el contenido imaginando lo que había en la URL. Pregunta al usuario cómo proceder (ver Sección 4.2).

3. **IDEMPOTENCIA ESTRICTA (`--force`):**
   - Antes de escribir `tech-notes/<slug>.md`, comprueba si el archivo ya existe.
   - Si existe y el usuario **NO** pasó el flag `--force`, **prohibido sobrescribirlo**. Detente y avisa al usuario indicando que añada `--force` o elija otro concepto/slug.

4. **SIN COMPROMISO DE EJECUCIÓN EN SNIPPETS:**
   - Los bloques de código son estáticos y didácticos. Nunca prometas que el código funcionará en cualquier entorno sin ajustes ni intentes ejecutarlo por tu cuenta.
   - Incluye siempre la advertencia estándar de no-ejecución en la sección de *Snippets progresivos* (ver Sección 7).

5. **IDIOMA Y ESTILO ESTRICTOS:**
   - **Explicaciones, prosa y comentarios de código:** 100% en **español neutro**.
   - **Identificadores de código:** 100% en **inglés** (`camelCase`, `PascalCase`, `snake_case` o `UPPER_SNAKE_CASE` según la convención del lenguaje del snippet: ej. `userId`, `fetchUserData`, `MAX_RETRIES`).
   - **Cero comparaciones con otros lenguajes:** Prohibido explicar algo diciendo *"es como X en Python/Java/Rust"* o *"a diferencia de C++..."*. Explica la tecnología por sí misma.
   - **Cero opiniones o juicios de valor:** Prohibido usar adjetivos subjetivos o comparativas de superioridad (*"es mejor que"*, *"es más rápido"*, *"es la forma más elegante"*, *"increíblemente potente"*). Limítate a hechos técnicos verificables.

---

## 3. Flags CLI y Normalización de Salida

### 3.1. Catálogo Único de Flags Reconocidos
La skill reconoce únicamente estos tres flags (no inventes ni aceptes flags adicionales):
- `--concept <nombre>`: Genera la nota técnica a partir del nombre de la tecnología, herramienta, protocolo, algoritmo o patrón.
- `--url <url>`: Genera la nota técnica tomando como punto de partida la URL indicada y complementándola con estructura didáctica para aprender a usar e implementar la tecnología desde cero.
- `--force`: Sobrescribe `tech-notes/<slug>.md` si el archivo ya existe en disco.

*(Nota: Si el usuario invoca la skill en lenguaje natural sin escribir explícitamente `--concept` o `--url`, interpreta el nombre aportado como `--concept` y cualquier enlace HTTP/HTTPS como `--url`).*

### 3.2. Ruta y Reglas de Normalización de `<slug>`
- **Ruta fija:** `tech-notes/<slug>.md` (nunca incluyas el número de versión en el nombre del archivo).
- **Normalización del `<slug>`:**
  1. Deriva el `<slug>` del nombre canónico de la tecnología o concepto (incluso cuando la entrada sea `--url`). Si la URL trata sobre una pieza específica de una tecnología mayor, usa `<tecnologia>-<subtema>` (ej. `react-server-components`).
  2. Convierte todo a minúsculas.
  3. Elimina tildes, diéresis y caracteres especiales (`á -> a`, `ñ -> n`).
  4. Sustituye espacios, barras, puntos y guiones bajos por guiones medios (`-`), dejando solo caracteres `[a-z0-9-]`.
  5. Colapsa guiones repetidos (`-- -> -`) y recorta a un **máximo de 50 caracteres** sin cortar palabras a la mitad.

### 3.3. Frontmatter YAML Obligatorio del Archivo Generado
Todo archivo `tech-notes/<slug>.md` debe comenzar exactamente con este bloque YAML:

```yaml
---
title: "<Título claro descriptivo en español>"
technology: "<Nombre canónico concepto de la o oficial tecnología>"
version: "<Versión/rama '3.x', 'HTTP/2') 'unknown' 'unversioned' (ej. verificada |>"
---
```

---

## 4. Protocolo de Entrada, Guardias de Parada y Selector de Nivel

Antes de redactar la nota técnica, ejecuta estas verificaciones en orden estricto:

### 4.1. Guardia de Concepto Inexistente, Ambiguo, Obsoleto o Fuera de Alcance
1. **Inexistente o no verificable:** Si el término pasado en `--concept` no existe o no tienes certeza absoluta de qué es y cuál es su sintaxis real, **DETENTE** y responde:
   > *"No puedo verificar la existencia o especificación técnica de `<concepto>` con certeza. Por favor, confirma el nombre exacto o proporciona una URL de referencia con `--url`."*
2. **Ambiguo (colisión de nombres):** Si el nombre corresponde a varias tecnologías distintas (ej. `Flux`, `Prism`, `Effect`, `Playwright` vs concepto genérico), **DETENTE**, enumera las opciones posibles (A, B, C) y pregunta a cuál se refiere.
3. **Colisión de archivo sin `--force`:** Si `tech-notes/<slug>.md` ya existe y no se especificó `--force`, **DETENTE** y pregunta si desea sobrescribirlo con `--force`.
4. **Obsoleto o deprecado:** Si la tecnología está oficialmente deprecada o en desuso, **sí puedes generar la nota**, pero estás obligado a insertar este aviso justo debajo del título principal:
   > ⚠️ **Tecnología obsoleta/deprecada:** `<Nombre>` se encuentra actualmente en desuso o deprecada oficialmente. Esta nota se mantiene con fines de mantenimiento de sistemas legados o referencia histórica.

### 4.2. Guardia de Fetch para `--url <url>`
Cuando el usuario invoque `--url <url>`:
1. Intenta obtener y leer el contenido de la URL con las herramientas disponibles.
2. Si la lectura falla por cualquier motivo (404, 403, timeout, paywall, inicio de sesión obligatorio, contenido vacío por renderizado JS o ausencia de acceso web), **DETENTE** y muestra exactamente estas opciones:
   > **No ha sido posible extraer el contenido de `<url>`** *(motivo: [indicar causa: bloqueo/404/JS/sin acceso])*. ¿Cómo prefieres continuar?
   > - **A)** Pegar directamente aquí el texto o fragmento relevante de la página.
   > - **B)** Proporcionar otra URL pública accesible o usar `--concept <nombre>` basándonos en el conocimiento general verificado de la tecnología.
   > - **C)** Abortar la generación.

### 4.3. Selector de Nivel de Dominio (1 al 5) y Confirmación de Clasificación
- **Atajo directo:** Si el usuario **ya indicó su nivel de dominio** en el prompt inicial (ej. *"nivel 1"*, *"desde cero"*, *"nivel 5 para repasar"*), **NO detengas el flujo**: pasa directamente a generar la nota completa aplicando ese nivel.
- **Consulta rápida (cuando no se indica nivel):** Si el input es válido pero el usuario no especificó su nivel previo, haz **una única parada breve** antes de generar la nota mostrando la clasificación detectada y la escala del 1 al 5:

```markdown
He preparado la estructura para **`<technology>`** (`tech-notes/<slug>.md`):
- **Clasificación detectada:** `[Tecnología grande | Concepto denso pequeño]` *(dime si prefieres cambiarla)*.

¿Qué **nivel de dominio (1 a 5)** tienes sobre este concepto o derivados para ajustar la profundidad?
- **1 — Nulo (Desde cero absoluto):** No conozco el concepto ni herramientas similares. Explicar modelo mental desde cero y cada paso sin asumir jerga.
- **2 — Nociones sueltas (Principiante):** ⭐ **Recomendado por defecto.** Sé qué problema ataca o conozco herramientas parecidas, pero nunca lo he usado. Foco en setup guiado, anatomía básica y primeros pasos claros.
- **3 — Base teórica sin práctica (Intermedio):** Entiendo la teoría, pero necesito implementarlo de verdad. Menos introducción conceptual y más configuración, flujo real y código comentado.
- **4 — Uso básico previo (Práctico):** Ya he hecho pruebas simples; busco estructurarlo bien, patrones idiomáticos, casos compuestos y evitar errores comunes.
- **5 — Uso práctico / Refresco (Avanzado):** Ya lo he usado pero necesito repasar sintaxis y detalles clave. Directo al grano: TL;DR, Cheatsheet, snippets complejos y gotchas críticos.
```

---

## 5. Profundidad, Clasificación y Longitud Adaptativa

### 5.1. Heurística: "Tecnología Grande" vs. "Concepto Denso Pequeño"
Clasifica automáticamente toda entrada en una de estas dos categorías:

| Categoría | Cómo identificarla (Regla determinista) | Enfoque pedagógico de la nota |
| :--- | :--- | :--- |
| **Tecnología grande** | Tiene comando de instalación propio, CLI, SDK, runtime o múltiples módulos (lenguajes, frameworks, bases de datos, contenedores, librerías amplias). | **Foco en el 20% esencial que resuelve el 80% de casos reales.** Asume transferencia de conceptos generales de programación (según el nivel 1–5 elegido) y enseña cómo instalar, configurar, estructurar el flujo base y escalar a un caso completo funcional. |
| **Concepto denso pequeño** | Algoritmo, estructura de datos, patrón de diseño, primitiva de concurrencia o protocolo acotado que no se instala como paquete independiente. | **Foco en alta densidad explicativa por unidad.** Explica paso a paso el mecanismo interno, el estado en cada transición, la invariante que mantiene y cómo implementarlo desde cero con código limpio o pseudosnippets. |

### 5.2. Longitud Proporcional (`autodetect`)
- **Sin cotas artificiales de líneas:** La longitud de la nota debe ser estrictamente proporcional a la complejidad real del propósito.
  - Un concepto acotado o primitiva simple puede resolverse en una nota compacta y directa donde cada línea aporte valor.
  - Un framework completo o especificación extensa requerirá una nota larga y detallada que cubra todo el ciclo de uso e implementación.
- **Regla anti-relleno:** Prohibido alargar secciones con introducciones históricas, marketing de la herramienta o párrafos redundantes. Cada frase debe responder a *qué es*, *cómo funciona*, *cómo se usa* o *qué fallo evita*.

---

## 6. Estructura Adaptativa de las 9 Secciones (y Regla de Omisión)

El cuerpo de `tech-notes/<slug>.md` sigue este orden canónico de **9 secciones**.

### Regla de Omisión Justificada
- Si una sección no aplica a la naturaleza del concepto (p. ej., **Instalación / setup** en un patrón de diseño o algoritmo teórico), **elimina el encabezado `##` de esa sección** (nunca dejes secciones vacías) y registra obligatoriamente el motivo justo al final de la sección `## 1. TL;DR` con este formato exacto:
  > **Secciones omitidas:** `Instalación / setup` *(no aplica por tratarse de un patrón arquitectónico sin paquete ni herramienta instalable)*.

### Detalle de las 9 Secciones
1. **`## 1. TL;DR`** *(Obligatoria siempre)*:
   - Qué es la tecnología/concepto en 2–3 frases fácticas, qué problema técnico concreto resuelve y cuándo usarla (y cuándo no).
   - Incluye al final el bloque `> **Secciones omitidas:** ...` (o indica `Ninguna` si están las 9).
2. **`## 2. Requisitos previos`** *(Omitible si no requiere base específica)*:
   - Conocimientos técnicos mínimos y herramientas del entorno necesarias antes de empezar (adaptado al Nivel 1–5 del usuario).
3. **`## 3. Instalación / setup`** *(Omitible en conceptos teóricos/patrones/algoritmos puros)*:
   - Comandos de instalación, estructura mínima de archivos o configuración inicial para tener un entorno listo desde cero, más cómo verificar que quedó bien instalado.
4. **`## 4. Conceptos clave`** *(Obligatoria siempre)*:
   - Los pilares fundamentales o piezas arquitectónicas que el lector debe entender antes de escribir código. Incluye diagrama Mermaid aquí solo si clarifica la arquitectura o flujo (ver Sección 8).
5. **`## 5. Cheatsheet`** *(Omitible solo en conceptos mínimos donde un resumen duplique el único snippet)*:
   - Tabla Markdown o lista rápida de referencia con los comandos, métodos, firmas, operadores o pasos más utilizados y su efecto exacto.
6. **`## 6. Snippets progresivos`** *(Obligatoria siempre)*:
   - Secuencia graduada de menor a mayor complejidad (p. ej. *Nivel 1: Mínimo funcional / Hola Mundo real* → *Nivel 2: Caso de uso estándar con manejo de estado/datos* → *Nivel 3: Caso compuesto / producción con manejo de errores*).
7. **`## 7. Gotchas / errores comunes`** *(Obligatoria siempre)*:
   - Fallos silenciosos, trampas de configuración, malas prácticas frecuentes o casos límite (*edge cases*), estructurados siempre como: **Problema / Síntoma** → **Causa técnica** → **Solución correcta**.
8. **`## 8. Glosario`** *(Omitible si todos los términos ya quedaron definidos en Conceptos clave o si el usuario eligió Nivel 5)*:
   - Definiciones breves (1 línea por término) de la jerga propia de esa tecnología que aparece en la nota.
9. **`## 9. Recursos`** *(Omitible solo si el modelo no puede verificar ninguna fuente oficial o estándar sin riesgo de inventar URLs)*:
   - Referencias a documentación oficial verificada, especificaciones estándar (RFC, W3C, ISO) o comandos de ayuda local (`man`, `--help`). **Prohibido inventar URLs profundas si no estás 100% seguro de que existen; usa el dominio raíz oficial o el nombre exacto de la especificación.**

---

## 7. Reglas de Snippets y Bloques CLI

1. **Elección del lenguaje del snippet:**
   - **Con lenguaje canónico claro:** Si la tecnología pertenece o se asocia de forma unívoca a un lenguaje (ej. `FastAPI` → `python`, `React` → `tsx`/`jsx`, `Actix` → `rust`, `PostgreSQL` → `sql`), usa ese lenguaje en el bloque de código.
   - **Sin lenguaje único (agnóstico):** Si es un concepto transversal (ej. algoritmo *Raft*, patrón *Circuit Breaker*, protocolo *OAuth 2.0*, *WebSockets*), usa bloques `text` con **pseudosnippets estructurados** claros (con identificadores en inglés y comentarios en español) acompañados de **comandos CLI** (`bash`/`sh`) cuando aplique (ej. llamadas `curl`, inspección de red, flags de consola).
2. **Anatomía obligatoria de cada snippet progresivo:**
   - **Aviso inicial de sección:** Al abrir `## 6. Snippets progresivos`, incluye siempre este bloque:
     > ℹ️ **Nota sobre los snippets:** Los siguientes bloques son estáticos y tienen fines didácticos. No se garantiza su ejecución directa sin adaptar rutas, variables de entorno, dependencias o versiones a tu entorno local.
   - **Explicación previa y posterior:** Antes o después de cada bloque de código, explica en español neutro **qué hace ese bloque** y **por qué se estructura así** cada pieza clave.
   - **Código copiable y limpio:** Identificadores de variables, funciones y clases en inglés; comentarios breves dentro del código en español (`# Inicializa el cliente con timeout explícito`).

---

## 8. Cuándo Usar Diagramas Mermaid (y Cuándo No)

- **NO incluyas diagramas Mermaid por defecto** ni por decorar la nota.
- **Inclúyelos únicamente cuando aporten claridad estructural superior al texto**, específicamente en:
  - Flujos de protocolo o comunicación entre actores (`sequenceDiagram`).
  - Ciclos de vida de componentes, estados o transacciones (`stateDiagram-v2`).
  - Arquitectura de piezas o tuberías de datos (`flowchart LR` / `flowchart TD`).
- **Reglas de seguridad sintáctica en Mermaid:**
  - Envuelve siempre los textos de los nodos que contengan espacios, paréntesis, dos puntos o acentos entre comillas dobles: `A["Cliente HTTP (App)"] --> B["Servidor API"]`.
  - Mantén el diagrama conciso (máximo 8–12 nodos) para garantizar legibilidad.

---

## 9. Referencia a Ejemplos Few-Shot (`EXAMPLES.md`)

Para mantener ligera la ventana de contexto al cargar esta skill, los ejemplos completos de entrada y salida viven en el archivo complementario **`EXAMPLES.md`** ubicado en este mismo directorio.

Consulta `EXAMPLES.md` cuando necesites revisar el formato exacto de salida para estos 3 escenarios canónicos:
1. **Caso 1 (`--concept` + Concepto denso pequeño):** Patrón *Circuit Breaker* (nivel 3, pseudosnippets agnósticos, diagrama de estados Mermaid y omisión justificada de *Instalación / setup*).
2. **Caso 2 (`--url` + Tecnología grande):** Herramienta/librería con lenguaje canónico y comandos CLI (nivel 2, las 9 secciones completas, snippets progresivos y aviso de no-ejecución).
3. **Caso 3 (Guardias de parada):** Respuesta exacta ante `--url` con fetch fallido, colisión de archivo sin `--force` y término `--concept` ambiguo.

---

## 10. Si el Modelo Destino se Desvía (Gotchas y Reglas de Rescate)

Revisa esta tabla antes de emitir cualquier respuesta para corregir desviaciones frecuentes en modelos de segunda línea:

| Síntoma de desviación | Por qué ocurre | Corrección obligatoria (Regla de rescate) |
| :--- | :--- | :--- |
| **1. Genera la nota directamente sin preguntar el nivel (1–5) cuando el usuario no lo especificó.** | Impulso de completar la tarea en un solo turno. | **DETENTE.** Si el usuario no indicó su nivel en el prompt, haz primero la pregunta corta de la Sección 4.3 mostrando la clasificación detectada y la escala 1–5. |
| **2. Inventa flags, métodos de API o versiones exactas (ej. `version: "2.4.9"` sin certeza).** | Alucinación por completado estadístico del modelo. | **PROHIBIDO adivinar.** Borra el método o flag dudoso y aplica la fórmula literal de incertidumbre de la Regla Dura 1. En `version`, usa `"unversioned"`, `"unknown"` o la rama mayor segura (`"2.x"`). |
| **3. La URL de `--url` falla o está bloqueada y el modelo genera la nota de memoria sin avisar.** | Intenta "ayudar" ocultando el fallo de lectura web. | **DETENTE.** No generes la nota. Aplica la Guardia 4.2 e informa del fallo dando las opciones A (pegar contenido), B (cambiar a `--concept`/otra URL) o C (abortar). |
| **4. Compara con otros lenguajes (*"igual que los decoradores de Python"*) o emite opiniones (*"es mucho más rápido que X"*).** | Estilo de blog técnico aprendido en preentrenamiento. | Elimina toda mención comparativa a otros lenguajes y todo adjetivo de valor. Reescribe en español neutro y fáctico describiendo solo cómo funciona la tecnología objetivo. |
| **5. Escribe variables o funciones en español dentro del código (`obtener_usuario`, `lista_datos`).** | Confusión con la regla de escribir el texto en español. | Conserva **toda la prosa y los comentarios `#` / `//` en español**, pero cambia **todos los identificadores de código al inglés** (`getUser`, `dataList`). |
| **6. Deja encabezados `##` vacíos o elimina secciones en silencio sin justificar.** | Olvida la regla de trazabilidad de la estructura adaptativa. | Elimina el encabezado `##` de la sección que no aplique y añade obligatoriamente `> **Secciones omitidas:** <Nombre> (<motivo>)` al final de `## 1. TL;DR`. |
| **7. Pone la versión en el nombre del archivo (`tech-notes/redis-7.md`) o sobrescribe sin `--force`.** | Inercia al nombrar archivos versionados. | La ruta es siempre `tech-notes/<slug>.md` sin versión. La versión solo va dentro del frontmatter YAML. Si el archivo ya existe y no hay `--force`, aborta y pregunta. |
| **8. Acepta resumir un paper o RFC puntual como si fuera una guía general.** | No distingue entre documento concreto y tecnología. | Detente y redirige al usuario a la skill hermana `dense-doc-digester` (Sección 1). |

---

## 11. Checklist Final de Verificación (Antes de Entregar)

Antes de dar por válida y entregar la nota `tech-notes/<slug>.md`, verifica que se cumplen **todos** estos puntos:

- [ ] **Ruta y nombre:** ¿La ruta es exactamente `tech-notes/<slug>.md`, en `kebab-case`, sin acentos, ≤50 caracteres y **sin número de versión** en el nombre del archivo?
- [ ] **Idempotencia:** ¿Se comprobó que el archivo no existía previamente o que el usuario pasó explícitamente `--force`?
- [ ] **Frontmatter YAML válido:** ¿Incluye `title`, `technology` y `version` sin versiones inventadas?
- [ ] **Adaptación al nivel (1–5) y tamaño:** ¿La densidad y longitud responden al nivel elegido por el usuario y a la clasificación (*Tecnología grande* vs. *Concepto denso pequeño*) sin texto de relleno?
- [ ] **Estructura y omisiones justificadas:** ¿Están las secciones en el orden del 1 al 9 y cualquier sección omitida figura explicada en `> **Secciones omitidas:**` dentro del `TL;DR`?
- [ ] **Neutralidad estricta:** ¿Está 100% libre de comparaciones con otros lenguajes de programación y de juicios de valor u opiniones?
- [ ] **Idioma dual correcto:** ¿Toda la explicación y comentarios están en español neutro y todos los identificadores de código (variables, funciones, clases) están en inglés?
- [ ] **Snippets seguros y explicados:** ¿Incluye el aviso de *sin compromiso de ejecución*, explica qué hace y por qué cada bloque, y no contiene ni una sola API o flag inventado?
- [ ] **Mermaid limpio (si se incluyó):** ¿Aporta valor real y tiene las etiquetas con espacios/paréntesis protegidas entre comillas dobles `"..."`?