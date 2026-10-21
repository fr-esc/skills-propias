---
name: dense-doc-digester
description: Digiere documentación técnica o teórica densa (papers, RFCs, specs, whitepapers, docs extensas) en un único documento Markdown estructurado, fiel a la fuente y libre de paja. Úsala para comprender un documento completo de principio a fin; no la uses para auditar parámetros concretos contra citas literales (para eso usa extraccion-tecnica-con-verificacion).
version: 1.4.0
target_tier: second-line
annexes:
  - reference/ejemplos-digest.md
related_skills:
  - extraccion-tecnica-con-verificacion
allowed_flags:
  - --force
  - --interview
  - --mode
  - --lang-snippets
  - --focus
  - --slug
---

# `dense-doc-digester`

Convierte un documento concreto y denso (paper académico, RFC, especificación formal o documentación técnica pesada) en una síntesis rigurosa, progresiva y trazable en `digests/<slug>.md`, preservando todos los detalles críticos y eliminando el relleno sin inventar información externa.

---

## 1. Cuándo usar y cuándo NO usar esta skill

### Cuándo usarla
- El usuario proporciona o apunta a **un documento específico y denso** (URL, PDF, TXT, DOCX o texto pegado) y necesita comprender su contenido, mecanismos, fórmulas, supuestos o reglas normativas sin leer toda la paja.
- Se requiere un análisis fiel a la fuente con trazabilidad exacta por secciones/páginas y separación estricta entre lo que dice el texto (`[fuente]`) y cualquier aclaración (`[contexto añadido]`).

### Cuándo NO usarla
- **NO genera planes de proyecto, roadmaps ni arquitecturas de software a medida.**
- **NO genera tutoriales ni guías de "cómo usar o implementar una tecnología desde cero"** a partir de un tema abstracto o de la web general (ver *Protocolo cuando el usuario pide un tutorial desde cero*).
- **NO resume múltiples fuentes inconexas a la vez** (opera sobre un documento fuente por ejecución).
- **NO audita parámetros puntuales contra citas literales** ("¿cuál es el SLA P99?", "¿define el RTO?", "¿qué dice del cifrado en tránsito?"). Si la respuesta esperada es una tabla de atribución factual con veredictos de certidumbre y no un documento explicativo, delega en `extraccion-tecnica-con-verificacion`.
- **NO crea datasets de entrenamiento.** Si el siguiente paso es convertir el documento en pares instrucción-respuesta para SFT/RAG, delega en `generador-datasets`.

### Frontera explícita con `extraccion-tecnica-con-verificacion`
| Criterio | `dense-doc-digester` (esta skill) | `extraccion-tecnica-con-verificacion` |
| :--- | :--- | :--- |
| **Verbo** | Comprender | Auditar |
| **Entrada** | Un documento completo. | Un documento + una lista cerrada de parámetros a verificar. |
| **Salida** | `digests/<slug>.md`: 8 secciones, explicación progresiva, mapa conceptual, glosario. | Tabla de atribución factual + brechas/contradicciones + síntesis, en el chat. |
| **Cobertura** | Todo el documento, incluido lo que se omite y por qué. | Solo los parámetros preguntados. |
| **Uso típico** | "Explícame este RFC", "destripa este paper". | "Sácame el SLA y el RTO de este pliego y dime si se contradice". |
| **Si el usuario se equivoca de skill** | Si pide solo datos puntuales con cita, ejecuta `extraccion-tecnica-con-verificacion` y no escribas `digests/<slug>.md`. | Si pide entender el documento entero, delega aquí. |

> **Puente entre ambas:** cuando termines un digest y detectes que el usuario necesitará defender cifras concretas del documento (SLA, límites, requisitos contractuales), añade al final de la respuesta en consola una línea sugiriendo `extraccion-tecnica-con-verificacion` para auditar esos parámetros uno a uno.

### Protocolo cuando el usuario pide un tutorial desde cero (sin documento fuente)

Esta skill **no enseña tecnologías desde cero**. Es la tentación más frecuente y la que más daño hace a la fidelidad del resultado, porque abre la puerta al conocimiento de entrenamiento.

Si el usuario pide algo del tipo *"hazme un apunte de cómo instalar y usar Redis en producción"*, *"explícame Kubernetes desde cero con ejemplos"* o *"guíame para montar un clúster"* **sin aportar un documento concreto**, no escribas el tutorial, no escribas en `digests/` y no delegues en ninguna otra skill. Responde exactamente con este bloqueo:

> `ACCIÓN BLOQUEADA POR FRONTERA DE SKILL:` `dense-doc-digester` solo digiere **un documento concreto y denso** (paper, RFC, especificación, manual, archivo técnico) siendo 100% fiel a esa fuente. No genera guías de aprendizaje desde cero ni planes de implantación, porque eso exigiría conocimiento externo al documento.
>
> Tienes dos opciones:
> 1. **Pásame un documento concreto** (enlace, PDF, TXT, DOCX o el texto pegado) y lo digiero: qué propone, cómo funciona, qué límites tiene y qué se ha omitido.
> 2. **Pídeme el tutorial directamente al asistente**, sin invocar esta skill. Esta skill no es el vehículo adecuado para esa tarea y forzarla produciría un resultado peor.

**Excepción legítima:** si el usuario aporta un documento real (aunque sea corto: un KEP, un README de proyecto, un RFC) y pide "explícame esto como si empezara desde cero", eso **sí** es un digest: la fuente existe y la fidelidad es posible. La frontera no está en el nivel del lector, sino en la existencia de un documento fuente.

---

## 2. Entradas aceptadas y protocolo ante fallo de acceso (Fetch Fallido)

### Entradas válidas
1. **Archivo local o adjunto:** `.pdf`, `.txt`, `.md`, `.docx`.
2. **Texto pegado directamente** en el prompt.
3. **URL directa** a un documento (HTML técnico, PDF remoto, RFC, pre-print en arXiv, especificación W3C/IETF, etc.).

### Protocolo obligatorio ante Fetch Fallido
Si la entrada es una URL y ocurre cualquiera de estos casos: **error 404/403/5xx, paywall, pantalla de login, bloqueo anti-bot (Cloudflare/CAPTCHA), o página JS-heavy vacía cuyo contenido real no se extrae**:

1. **Intento de recuperación único:** Busca una fuente espejo oficial o abierta del **mismo documento exacto** (p. ej., versión en `arxiv.org`, mirror oficial del IETF/W3C, repositorio público de los autores o caché accesible).
2. **Prohibición de sustitución:** No sustituyas el documento inaccesible por artículos de blog de terceros, resúmenes de prensa ni tu propio conocimiento de entrenamiento sobre ese paper/RFC.
3. **Parada dura si falla el espejo:** Si el espejo no existe o también falla, **NO generes `digests/<slug>.md`**. Responde exactamente con la plantilla de bloqueo:
   > `ERROR DE ACCESO A LA FUENTE: No ha sido posible extraer el contenido de [URL] (motivo: [paywall / login / JS-heavy / 404]) ni se ha encontrado un espejo abierto verificable. Proporciona el documento en formato PDF, TXT, DOCX o pega el texto directamente para poder digerirlo sin alucinar su contenido.`

---

## 3. Flags CLI soportados

El usuario puede invocar la skill pasando cualquiera de estos flags opcionales:

| Flag | Valores admitidos | Por defecto | Semántica exacta |
| :--- | :--- | :--- | :--- |
| `--force` | *(booleano)* | `false` | Sobrescribe `digests/<slug>.md` si ya existe. Sin `--force`, si el archivo destino existe, aborta antes de procesar y avisa al usuario. |
| `--interview` | *(booleano)* | `false` | Pausa la digestión y lanza la entrevista previa acotada (≤5 preguntas) para adaptar foco y nivel. |
| `--mode` | `technical` \| `theoretical` \| `auto` | `auto` | Fuerza el modo de digestión o activa la autodetección óptima según el input. |
| `--lang-snippets` | `<lang>` \| `pseudo` \| `none` | *(según fuente)* | Fuerza el lenguaje de los bloques de código (p. ej. `python`, `rust`, `typescript`), los convierte a pseudocódigo (`pseudo`) o los suprime (`none`). |
| `--focus` | `"<sección o tema>"` | *(todo el doc)* | Prioriza y profundiza en la sección o aspecto indicado (p. ej. `--focus "algoritmo de consenso §4"`), sintetizando el resto a nivel mínimo y registrándolo en la Sección 7. |
| `--slug` | `<kebab-case-name>` | *(autogenerado)* | Define manualmente el nombre del archivo de salida en `digests/<slug>.md`. |

---

## 4. Modos de digestión y heurística de autodetección (`--mode auto`)

### Los dos modos de operación
1. **Modo `technical` (RFC, especificación de protocolo, diseño de sistema, doc de arquitectura/API):**
   - **Enfoque:** Eliminar retórica comercial o introductoria; explicar con precisión cómo funciona el sistema, su máquina de estados, flujo de datos, contratos de interfaz y requisitos de implantación.
   - **Snippets:** Se mantienen en el lenguaje original del documento. Si el documento describe un protocolo o flujo técnico sin código y un snippet ayuda a entender su implantación, se usa el lenguaje del documento o pseudocódigo limpio marcándolo como `[contexto añadido]`.
2. **Modo `theoretical` (Paper científico, whitepaper matemático/criptográfico, monografía formal):**
   - **Enfoque:** Desempaquetar la densidad conceptual y matemática sin perder rigor. Traducir la intuición detrás de cada teorema/fórmula, explicitar todas las asunciones, cotas, complejidad e invariantes.
   - **Snippets:** **Solo se incluyen si aparecen en el documento original** o si son estrictamente imprescindibles para comprender cómo opera una fórmula o algoritmo complejo. En este segundo caso, es obligatorio marcarlos con `[contexto añadido]`.

### Heurística de autodetección óptima (`--mode auto`)
Cuando `--mode` no es forzado por el usuario, inspecciona la estructura y léxico del input aplicando esta matriz de decisión:

1. **Señales de peso `theoretical`:** Presencia de *Abstract*, *Related Work*, teoremas/lemas/demostraciones, alta densidad de ecuaciones $\LaTeX$, diseño experimental/datasets, citas bibliográficas académicas (`[1]`, `(Autor et al., 2024)`), identificadores DOI/arXiv.
2. **Señales de peso `technical`:** Lenguaje normativo IETF/RFC 2119 (`MUST`, `SHOULD`, `MAY`), definiciones de cabeceras de red/paquetes, esquemas JSON/YAML/Protobuf, firmas de métodos/endpoints, diagramas de arquitectura de componentes o guías operativas de subsistemas.
3. **Resolución óptima para documentos híbridos** (p. ej., un paper de sistemas distribuidos con pruebas de seguridad y código, o una spec criptográfica del NIST con álgebra formal e implementación):
   - Selecciona como `digest_mode` principal en el frontmatter aquel que represente el **propósito dominante (>50%)** del documento.
   - Activa automáticamente la **protección cruzada**: conserva intacta la formulación matemática/cotas en la Sección 6 (propio de `theoretical`) y detalla las reglas de implementación/máquina de estados en la Sección 4 (propio de `technical`), aplicando siempre la regla más estricta para el código: cualquier snippet no literal del documento lleva `[contexto añadido]`.

---

## 5. Flujo de trabajo paso a paso

Sigue este orden estricto en cada ejecución:

### Paso 1: Validación de frontera y cálculo de ruta (`digests/<slug>.md`)
1. Verifica que existe un documento fuente concreto. Si es una petición genérica de aprendizaje desde cero sin documento, aborta y aplica el **Protocolo cuando el usuario pide un tutorial desde cero** de la Sección 1.
2. Determina el `<slug>`: usa el valor de `--slug` si se proporcionó; de lo contrario, genera un identificador en `kebab-case` (solo letras minúsculas `a-z`, números `0-9` y guiones `-`, máx. 50 caracteres) basado en el identificador estándar del documento (p. ej. `rfc-9110-http-semantics`, `attention-is-all-you-need`).
3. **Chequeo de idempotencia:** Comprueba si `digests/<slug>.md` ya existe.
   - Si existe y **NO** se pasó `--force`: detén la ejecución inmediatamente y avisa: `"El archivo digests/<slug>.md ya existe. Ejecuta de nuevo con --force para sobrescribirlo o usa --slug <nuevo-nombre>."`
   - Si no existe o se pasó `--force`: continúa.

### Paso 2: Extracción de la fuente
1. Lee el documento completo desde el archivo local, texto pegado o URL.
2. Si falla la lectura de la URL, ejecuta el **Protocolo ante Fetch Fallido** (Sección 2) y detente si no hay espejo accesible.

### Paso 3: Entrevista previa condicional (solo si `--interview` está activo)
Si el usuario incluyó `--interview`, **no generes el digest todavía**. Formula exactamente este bloque de 5 preguntas breves y espera su respuesta:
1. **Objetivo de lectura:** ¿Para qué necesitas digerir este documento (entender la idea general, auditar límites/seguridad, implementar el mecanismo, replicar el paper)?
2. **Nivel previo:** ¿Cuál es tu familiaridad con el dominio del documento (básico, intermedio, avanzado/especialista)?
3. **Profundidad deseada:** ¿Prefieres énfasis en la intuición conceptual (Sección 4 ligera) o máximo desglose técnico/matemático (Sección 4 y 6 exhaustivas)?
4. **Idioma/formato de snippets:** ¿En qué lenguaje prefieres los ejemplos de código o pseudocódigo (original del documento, `pseudo`, `none`, u otro lenguaje)?
5. **Prioridad (`focus`):** ¿Hay alguna sección, teorema o componente concreto que debamos priorizar por encima del resto?

*(Si `--interview` NO está activo, salta directamente al Paso 4 sin hacer preguntas).*

### Paso 4: Estrategia según longitud del documento
Evalúa la extensión del documento fuente antes de redactar:

- **Caso A — Documento muy corto (<300 palabras o <30 líneas):**
  - Procesa automáticamente en **modo compacto**. No rellenes con texto artificial. Omite justificadamente las secciones que no aporten valor en un texto tan breve (típicamente Sección 3 *Mapa conceptual* y Sección 5 *Términos clave*, dejando una línea de justificación bajo su encabezado) y concentra el rigor en TL;DR, Tesis, Detalles críticos y Referencias.
- **Caso B — Documento estándar (entre 300 y 8.000 palabras / ≤15 páginas):**
  - Procesa en una sola pasada directa cubriendo las 8 secciones.
- **Caso C — Documento largo (>8.000 palabras / ~12k tokens / >15 páginas):**
  - Activa obligatoriamente la **construcción en 2 pasadas internas** para evitar truncamiento o pérdida de precisión en la mitad final del documento:
    - *Pasada 1 (Extracción estructural interna):* Recorre el documento de principio a fin extrayendo un índice de bloques con sus localizadores (`§sección`, `p. página`), todas las fórmulas/parámetros críticos, las ambigüedades detectadas y qué secciones son paja (motivación repetitiva, *related work* accesorio).
    - *Pasada 2 (Consolidación):* A partir del mapa extraído, redacta y ensambla el archivo único `digests/<slug>.md` garantizando que las 8 secciones sean coherentes entre sí y no se pierdan detalles de las últimas páginas.
- **Caso D — Documento enorme (>50 páginas / libro completo / tesis doctoral):**
  - Si se indicó `--focus`: aplica extracción profunda sobre los capítulos/secciones del foco y resume el resto en la Sección 7.
  - Si **NO** se indicó `--focus`: ejecuta las 2 pasadas a escala **macro-estructural**. Digiere la tesis global, la arquitectura completa y los teoremas/mecanismos troncales del documento, asigna `status: partial` en el frontmatter, y detalla en la Sección 7 qué capítulos secundarios se han condensado a alto nivel recomendando usar `--focus "<capítulo>"` para bajar al detalle línea a línea de esas partes.

### Paso 5: Escritura, validación y entrega
1. Escribe el archivo `digests/<slug>.md` siguiendo la plantilla de la Sección 6 y las reglas de marcado de la Sección 7.
2. Ejecuta mentalmente el **Checklist final de verificación** (Sección 12) antes de dar por cerrada la tarea.

---

## 6. Estructura interna obligatoria de `digests/<slug>.md`

El archivo generado debe ser **un único archivo Markdown autocontenido** que comience con el Frontmatter YAML y contenga exactamente las siguientes **8 secciones numeradas**.

> **Regla de omisión justificada:** Ninguno de los 8 encabezados puede borrarse silenciosamente. Si una sección (p. ej. el Glosario en un texto corto sin jerga, o ciertos bloques en un digest acotado con `--focus`) no aplica, **mantén el título de la sección** y escribe debajo en una sola línea en cursiva: `*[Sección omitida justificadamente: <motivo concreto>]*`.
> **Prohibición expresa:** No añadas secciones extra (está terminantemente prohibido añadir secciones de "Preguntas de autoevaluación", "Plan de implementación" o "Próximos pasos").

### Frontmatter YAML de salida (Esquema compatible con Obsidian/Logseq)
```yaml
---
title: "Título descriptivo del documento digerido"
source: "URL o ruta exacta del archivo, identificador estándar (RFC/arXiv/DOI) o «texto pegado»"
source_type: <paper | rfc | spec | tech-doc | whitepaper | other>
date: YYYY-MM-DD
status: <complete | partial | unverified-claims>
digest_mode: <technical | theoretical>
lang_snippets: "<lenguaje principal | pseudo | none>"
source_version: "<versión, fecha de publicación o número de RFC/arXiv, o 'unknown' si no consta>"
tags:
  - digest
  - <tag-dominio-1>
  - <tag-dominio-2>
aliases:
  - "<Nombre acrónimo corto del o paper/RFC>"
---
```
*Criterio para `status`:*
- `complete`: Documento digerido íntegramente sin afirmaciones dudosas.
- `partial`: Se usó `--focus` o es un documento enorme (>50 págs.) donde capítulos secundarios se sintetizaron a nivel macro.
- `unverified-claims`: El documento original contiene contradicciones internas, referencias rotas o huecos marcados con `[verificar]`.

### Las 8 Secciones del Cuerpo

1. `## 1. TL;DR`
   - **Máximo 10 líneas.** Resumen ejecutivo directo: qué es el documento, qué propone o especifica, cuál es su aportación o mecanismo clave y su principal límite o requisito.

2. `## 2. Qué problema resuelve / Tesis central`
   - Qué limitación previa, cuello de botella o vacío formal ataca el documento y cuál es su tesis o propuesta arquitectónica central.

3. `## 3. Mapa conceptual`
   - Representación visual o jerárquica de cómo se conectan las piezas del documento.
   - Usa una lista jerárquica de bullets o un diagrama `mermaid` (`flowchart`, `sequenceDiagram` o `stateDiagram-v2`) **solo si aporta claridad real** sobre la relación entre componentes/conceptos.

4. `## 4. Explicación progresiva`
   - Desglose estructurado en 3 niveles de profundidad creciente (o agrupado por los bloques lógicos del documento pasando de lo simple a lo complejo):
     - **Nivel 1 — Intuición base:** Cómo funciona el mecanismo o idea central sin jerga pesada.
     - **Nivel 2 — Arquitectura / Flujo operativo:** Cómo interactúan las partes, pasos del algoritmo o flujo del protocolo.
     - **Nivel 3 — Mecánica profunda e implantación:** Detalles internos, casos borde, estructura de datos o demostración clave, incluyendo snippets si corresponden según el modo.

5. `## 5. Términos clave`
   - Glosario mínimo indispensable para leer el documento sin atascarse.
   - **Regla multi-idioma:** Redacta la definición en español y conserva siempre el término original entre paréntesis: `- **Traducción / Concepto** (*Original Term*): definición precisa según el documento [fuente].`

6. `## 6. Detalles críticos que no se pueden perder`
   - Todo aquello que, si se simplifica de más, induce a error:
     - **Fórmulas y ecuaciones esenciales** (en $\LaTeX$ exacto respecto al original, explicando cada variable).
     - **Supuestos e invariantes** (bajo qué condiciones funciona y cuándo se rompe).
     - **Límites, cotas de complejidad, timeouts o parámetros duros.**
     - **Cláusulas normativas críticas:** Si una cláusula de un RFC/spec o una definición formal pierde precisión al traducirse al español, **incluye la cita textual breve en su idioma original entre comillas** seguida de su explicación exacta en español.
   - *En esta sección toda afirmación debe llevar localizador inline obligatorio (ver Sección 7).*

7. `## 7. Qué se ha omitido y por qué`
   - Inventario transparente de dos elementos:
     1. **Paja y material descartado:** Qué secciones o páginas del original se han omitido (motivación genérica, *related work* histórico, agradecimientos, demostraciones rutinarias en anexos, tablas redundantes) y por qué no afectan al núcleo.
     2. **Ambigüedades o contradicciones de la fuente:** Si el documento original se contradice entre dos secciones, deja una variable sin definir o promete un dato que no aparece, documéntalo aquí sin intentar inventar la solución.

8. `## 8. Referencias exactas a la fuente por bloque`
   - Mapa o tabla de trazabilidad que conecta cada subsección del digest (Secciones 2, 4, 5 y 6) con los localizadores exactos del documento original (`§Sección`, `p. Página`, `Líneas X–Y`, `Tabla/Figura N` o `#anchor`).

---

## 7. Reglas de marcado epistemológico y formato de referencias

Para garantizar trazabilidad total sin ensuciar la lectura, aplica el **esquema híbrido por criticidad**:

### Las 4 etiquetas obligatorias
1. `[fuente]` o `[fuente: <localizador>]`: Afirmación, dato, fórmula o snippet extraído directamente del documento original.
2. `[inferencia: <localizador base>]`: Deducción lógica directa e inevitable a partir de lo que dice el documento (p. ej., deducir que un paso es $O(n^2)$ porque el texto muestra dos bucles anidados sobre $n$ en `§3.1`), pero que el autor no escribió literalmente con esas palabras.
3. `[verificar: <qué comprobar y dónde>]`: Afirmación cuya validez no puede confirmarse al 100% con el documento proporcionado (p. ej., el texto cita una constante de un Anexo no adjunto, una tabla está truncada en el PDF, o la Sección 3 contradice a la Sección 5).
4. `[contexto añadido]`: Aclaración breve, definición de un concepto previo que el documento da por sabido, o **cualquier snippet de código/pseudocódigo generado o traducido a otro lenguaje** para ilustrar una fórmula o protocolo que no venía así en la fuente.

### Dónde usar etiqueta simple vs. localizador inline
- **En prosa general (Secciones 2, 4 y 5):**
  - Si es extracción directa, usa `[fuente]` al final del párrafo o bloque lógico (el detalle exacto de páginas/secciones va en la tabla de la Sección 8).
  - Si es `[inferencia: ...]`, `[verificar: ...]` o `[contexto añadido]`, **incluye siempre el detalle inline**.
- **En la Sección 6 (*Detalles críticos*):**
  - **Localizador inline obligatorio en cada punto:** Toda fórmula, parámetro, supuesto o límite debe llevar su ubicación exacta al lado: `[fuente: §4.2, p. 11, Ec. 3]` o `[fuente: §5.1, L210-225]`.

### Formato estándar de localizadores
Usa el identificador más preciso que permita el formato de entrada:
- **PDF / Paper paginado:** `§3.2, p. 8` (más `Ec. N`, `Tabla N` o `Fig. N` si aplica).
- **RFC / Spec numerada:** `§4.1.2` (y `p. N` si está paginado).
- **TXT / Markdown / Código:** `§Encabezado, L45-72` (rango de líneas).
- **URL HTML sin páginas:** `§Nombre-de-sección` o `#anchor-id`.

---

## 8. Reglas duras anti-alucinación y política de idiomas

1. **Cero invención de APIs, sintaxis, versiones o parámetros:** Si el documento menciona una función o configuración sin detallar todos sus argumentos, muestra solo lo que aparece en la fuente. Nunca rellenes parámetros, flags ni versiones de librerías sacados de tu memoria de entrenamiento.
2. **Prohibido rellenar huecos en silencio:** Si el documento da un salto lógico o asume un conocimiento previo indispensable para entender un paso, no lo camufles como parte del documento; explícalo en 1-2 líneas y márcalo explícitamente con `[contexto añadido]`.
3. **Respeto a las contradicciones de la fuente:** Si en la página 4 el documento dice que el timeout es `500ms` y en la página 19 dice `5000ms`, **no elijas uno**. Expón ambos valores con `[verificar: contradicción entre p. 4 (500ms) y p. 19 (5000ms)]`, regístralo en la Sección 7 y pon `status: unverified-claims` en el frontmatter.
4. **Política de snippets multi-lenguaje:**
   - Si no se usa `--lang-snippets`, conserva cada snippet en el lenguaje exacto en que aparece en el documento original (incluso si el documento usa C en una sección y Python en otra).
   - Si el usuario fuerza `--lang-snippets <lang>` o `pseudo`, traduce el snippet manteniendo la lógica exacta del original y añade en la primera línea del bloque un comentario o marca externa `[contexto añadido: traducido desde <lenguaje-original> en §X]`.
   - Si el usuario pasa `--lang-snippets none`, no incluyas bloques de código; explica la lógica en prosa o lista de pasos numerados.

---

## 9. Ejemplos Few-Shot

Los dos digests completos de referencia (uno en modo `technical` y otro en `theoretical`, con frontmatter, las 8 secciones y el marcado epistemológico aplicado) viven en `reference/ejemplos-digest.md`, en el directorio de esta skill (`~/.config/opencode/skills/dense-doc-digester/`).

**Cuándo leerlo:**
- Antes del primer digest de una sesión, para fijar el formato exacto de salida.
- Si dudas de cómo aplicar `[fuente]`, `[inferencia]`, `[verificar]` o `[contexto añadido]` en un caso concreto.
- Si el documento es híbrido (spec con matemáticas, o paper con código) y hay que decidir qué modo domina.

**Cuándo NO leerlo:** si ya has producido un digest en esta misma sesión y el formato está fijado. Es referencia, no requisito.

A continuación quedan los dos guardarraíles cortos, que **sí** son de obligado cumplimiento:

### Ejemplo 3: Guardarraíl de frontera (tutorial desde cero sin documento fuente)

**Entrada del usuario:**
> Usa `dense-doc-digester` para hacerme una guía completa desde cero de cómo instalar, configurar y usar Kubernetes en producción con ejemplos de despliegue.

**Respuesta obligatoria del modelo (sin generar archivo en `digests/`):**
> `ACCIÓN BLOQUEADA POR FRONTERA DE SKILL:` `dense-doc-digester` está diseñada exclusivamente para digerir **un documento concreto y denso** (un paper, un RFC, una especificación o archivo técnico específico) siendo 100% fiel a esa fuente, y no genera guías de aprendizaje desde cero ni planes de implantación.
>
> Para generar una nota práctica de cómo instalar, configurar y usar Kubernetes desde cero, pídeselo directamente al asistente **sin invocar esta skill**: no es el vehículo adecuado y forzarla produciría un tutorial inventado, no un digest. Si lo que deseas es digerir un documento de diseño o KEP (*Kubernetes Enhancement Proposal*) concreto, pásame el enlace, PDF o texto de ese documento y lo analizo con total fidelidad a la fuente.

---

### Ejemplo 4: Guardarraíl de Fetch Fallido (Paywall / JS-heavy sin mirror)

**Entrada del usuario:**
> Digiere este paper: `https://dl.proprietary-journal.org/doi/10.9999/closed-paper-2026`

**Respuesta obligatoria del modelo (tras comprobar que exige suscripción y no está en arXiv ni en otro repositorio abierto):**
> `ERROR DE ACCESO A LA FUENTE: No ha sido posible extraer el contenido de https://dl.proprietary-journal.org/doi/10.9999/closed-paper-2026 (motivo: paywall de suscripción sin acceso al texto completo) ni se ha encontrado un espejo abierto verificable (arXiv / repositorio institucional). Proporciona el documento en formato PDF, TXT, DOCX o pega el texto directamente para poder digerirlo sin alucinar su contenido.`

---

## 10. Si el modelo destino se desvía (Protocolo de recuperación)

Si durante la generación detectas que estás incurriendo en alguno de los siguientes desvíos, **detén la escritura del bloque actual y aplica la corrección inmediata**:

1. **Desvío 1 — Invención de código o completado de APIs por inercia:**
   - *Síntoma:* Estás escribiendo un bloque de código con imports, métodos o flags de configuración que conoces por tu entrenamiento pero que **no están en el documento fuente**.
   - *Recuperación:* Borra los métodos/parámetros no presentes en la fuente. Si el bloque es un ejemplo conceptual para explicar una ecuación o flujo, redúcelo a la lógica estrictamente contenida en el documento y añade la etiqueta `[contexto añadido]` explicando su alcance.
2. **Desvío 2 — Deriva hacia un tutorial "Desde Cero" o Plan de Proyecto:**
   - *Síntoma:* Estás añadiendo pasos como *"Paso 1: Instala la librería con pip install..."*, *"Arquitectura recomendada para tu empresa"* o *"Roadmap de fases"*.
   - *Recuperación:* Elimina todo contenido prescriptivo o de proyecto. Limítate a describir lo que el documento fuente expone. Las guías prácticas de implantación no se redactan desde aquí: si el usuario las quiere, aplica el bloqueo de frontera de la Sección 1.
3. **Desvío 3 — Omisión de marcas epistemológicas (`[fuente]`, `[inferencia]`, `[verificar]`, `[contexto añadido]`):**
   - *Síntoma:* Has redactado varios párrafos en las Secciones 2, 4 o 6 sin etiquetas de trazabilidad.
   - *Recuperación:* Revisa párrafo por párrafo y asigna su etiqueta correspondiente. Verifica que en la Sección 6 **cada ítem** tenga localizador inline (`[fuente: §X, p. Y]`).
4. **Desvío 4 — Alteración de las 8 secciones o adición de "Preguntas de autoevaluación":**
   - *Síntoma:* Has cambiado el orden de las secciones, has borrado una sección sin dejar la nota de omisión justificada, o has añadido una sección 9 de autoevaluación/conclusión.
   - *Recuperación:* Restaura exactamente los 8 encabezados numerados del apartado 6 y elimina cualquier sección adicional.
5. **Desvío 5 — Truncamiento silencioso en documentos largos (>8.000 palabras):**
   - *Síntoma:* Notas que las últimas secciones del documento original apenas aparecen reflejadas en las Secciones 4, 6 y 8.
   - *Recuperación:* Detén la redacción, ejecuta la **Pasada 1 de extracción estructural** sobre la segunda mitad del documento fuente e integra los puntos críticos y referencias antes de cerrar el archivo.
6. **Desvío 6 — Deriva hacia auditoría puntual (confusión con `extraccion-tecnica-con-verificacion`):**
   - *Síntoma:* Estás construyendo una tabla de parámetros con veredictos de certidumbre (`Explícito` / `No especificado`) en lugar de secciones explicativas, o el usuario solo preguntó por dos o tres datos concretos del documento.
   - *Recuperación:* No fuerces el digest. Si la petición real es auditar parámetros contra citas literales, responde con la tabla de atribución propia de `extraccion-tecnica-con-verificacion` y **no generes** `digests/<slug>.md`. Solo vuelve a esta skill si el usuario confirma que quiere la comprensión completa del documento.

---

## 11. Checklist final de verificación (Obligatorio antes de entregar)

Antes de dar por finalizado el archivo `digests/<slug>.md`, comprueba que se cumplen **todos** estos puntos:

- [ ] **Ruta e idempotencia:** El archivo se guarda en `digests/<slug>.md` y no se ha sobrescrito ningún archivo existente sin el flag `--force`.
- [ ] **Frontmatter completo:** Incluye los 5 campos obligatorios (`title`, `source`, `source_type`, `date` en `YYYY-MM-DD`, `status`) más los metadatos (`digest_mode`, `lang_snippets`, `source_version`, `tags`, `aliases`).
- [ ] **8 secciones exactas:** Están presentes las 8 secciones numeradas en orden, el TL;DR tiene ≤10 líneas, no existe sección de autoevaluación, y cualquier sección vacía contiene su justificación explícita de omisión.
- [ ] **Marcado riguroso:** Todos los bloques tienen su marca (`[fuente]`, `[inferencia: ...]`, `[verificar: ...]`, `[contexto añadido]`), y todos los puntos de la Sección 6 tienen localizador inline exacto.
- [ ] **Fidelidad multi-idioma y código:** La prosa está en español, los términos del glosario conservan el original entre paréntesis, las cláusulas normativas/formales críticas mantienen la cita textual original si la traducción pierde matices, y cualquier snippet no literal está marcado como `[contexto añadido]`.
- [ ] **Cero alucinación:** Ninguna contradicción del documento ha sido resuelta inventando datos (se han registrado en la Sección 7 y marcado con `[verificar]`).
- [ ] **Frontera respetada:** La petición era comprender el documento completo, no auditar parámetros puntuales con cita literal (en cuyo caso correspondía delegar en `extraccion-tecnica-con-verificacion` sin escribir en `digests/`).

---

## Eval de aceptación

Prueba manual mínima. Se considera **APROBADA** solo si se cumplen todos los puntos.

**Entrada**
> «Digiere este extracto: *(unas 300 palabras de una especificación que fija un timeout de 500 ms en §2 y de 5.000 ms en §9)*»

**Salida esperada**
- Un único archivo `digests/<slug>.md` con frontmatter de 5 campos obligatorios: `title`, `source`, `source_type` (valor de la lista cerrada), `date` en `YYYY-MM-DD` y `status`.
- Las 8 secciones numeradas, en orden, sin secciones extra.
- En §6, localizador inline en cada ítem (`[fuente: §2]`, `[fuente: §9]`).
- La contradicción del timeout aparece **con ambos valores** en §6 y en §7, con `status: unverified-claims`.
- Prosa en español y términos originales entre paréntesis en el glosario.

**Prohibido en esta prueba**
- Elegir uno de los dos timeouts y presentarlo como el correcto.
- Añadir secciones de autoevaluación, conclusiones o próximos pasos.
- Rellenar parámetros ausentes con conocimiento de entrenamiento.

**Test negativo 1**
> «enséñame Kubernetes desde cero con ejemplos de despliegue»

**Resultado correcto:** bloqueo de frontera, sin escribir nada en `digests/`.

**Test negativo 2:** ejecutar dos veces con el mismo `<slug>` sin `--force`.

**Resultado correcto:** la segunda ejecución aborta antes de procesar y avisa de que el archivo existe.
