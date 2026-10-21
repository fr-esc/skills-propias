---
name: guided-script-builder
description: >
  Genera un script Python individual y autocontenido (preparado para UV) mediante
  una entrevista guiada acotada y una revisión estructurada bloque a bloque.
  Úsala cuando el usuario pida crear, diseñar o programar desde cero un script
  Python de cualquier dominio (archivos, datos, APIs, automatización, web, CLI,
  async) con recomendaciones justificadas, avisos de seguridad y trazabilidad en
  ESTADO.md. No la uses para implementar archivos .py existentes a partir de
  comentarios guía in-place ni para arquitecturas multi-paquete.
version: 1.2.1
language: es
target_model: segunda-linea
---

# Guided Script Builder (`guided-script-builder`)

Diseña y genera **un único script Python autocontenido** dentro de su propia carpeta `scripts/<slug>/`, listo para ejecutarse con `uv`, guiando al usuario mediante una entrevista inicial acotada y una revisión de arquitectura bloque a bloque.

## 1. Cuándo Usar Esta Skill (y Cuándo No)

Usa esta skill cuando:

- El usuario pida crear desde cero un script Python individual para cualquier dominio (sistema de archivos, limpieza de datos, consumo de APIs, scraping, automatización del sistema, interfaces simples o tareas asíncronas).
- El usuario quiera definir requisitos paso a paso antes de generar el código final.
- El usuario pida retomar un script previamente pausado aportando su archivo `scripts/<slug>/ESTADO.md`.

No uses esta skill cuando:

- El usuario entregue un archivo `.py` existente con comentarios guía para rellenar el código *in-place* (delega en la skill `comment-to-python` o `comment-to-code`).
- El usuario pida una aplicación multi-paquete, un servidor con múltiples microservicios o una librería distribuible en PyPI.

---

## 2. Guardarraíles y Reglas Duras (Inquebrantables)

1. **CERO EJECUCIÓN DE CÓDIGO O COMANDOS**:

   - **NUNCA** ejecutes el script generado, ni `uv init`, ni `uv run`, ni `python -m unittest`, ni `git init`.
   - Tu responsabilidad termina al dejar todos los archivos escritos y entregar al usuario los comandos exactos para que los lance manualmente en su terminal.

2. **CERO INVENCIÓN DE APIs, SINTAXIS O VERSIONES**:

   - Prohibido inventar funciones, métodos, argumentos de librerías o endpoints que no puedas verificar con certeza.
   - Si una dependencia externa no pertenece a la librería estándar de Python 3.12 y el usuario no la ha confirmado explícitamente, **pregunta antes de incluirla** porque añade superficie de fallo y descarga de paquetes externos.
   - En `pyproject.toml`, declara las dependencias externas **únicamente por nombre de paquete** (ej. `"requests"`, `"typer"`). **PROHIBIDO escribir números de versión** (como `"requests>=2.31.0"` o `"typer==0.9.0"`) para evitar alucinaciones de versiones inexistentes; `uv` resolverá la versión compatible al ejecutar.
   - **Fórmula obligatoria ante incertidumbre**: Si desconoces o dudas sobre una API, firma o comportamiento externo, di literalmente:
     > *"No puedo verificar con certeza [nombre de la API/librería/parámetro]; necesito que me confirmes su especificación exacta o si prefieres que usemos una alternativa de la librería estándar."*

3. **CERO OPERACIONES DESTRUCTIVAS SIN CONFIRMACIÓN EXPLÍCITA**:

   - Siempre que una opción o bloque implique: (a) borrar o sobrescribir archivos existentes, (b) usar dependencias externas fuera de stdlib, (c) manejar secretos, tokens o contraseñas, (d) realizar conexiones de red salientes, o (e) escribir fuera de `scripts/<slug>/`, incluye la etiqueta:
     > `⚠️ [AVISO DE RIESGO]: <explicación clara del impacto y medida de mitigación>`
   - Nunca implementes una operación destructiva sin que el usuario haya elegido o confirmado explícitamente esa opción.

4. **RECOMENDACIÓN POR DEFECTO OBLIGATORIA**:

   - En **toda** pregunta de la entrevista y en **toda** elección de variantes de bloque ofrece opciones cerradas (`A`, `B`, `C`, `Otra`) y marca exactamente una con el formato:
     > `⭐ Recomendación: Opción X - <justificación técnica breve en una sola línea>.`

5. **IDIOMA Y ESTILO DEL CÓDIGO**:

   - **Identificadores en español**: nombres de funciones, variables, parámetros (`snake_case`), clases y excepciones (`PascalCase`) y constantes (`UPPER_SNAKE_CASE`) van íntegramente en español neutro.
   - **Sintaxis y APIs externas en inglés**: palabras reservadas de Python (`def`, `return`, `try`, `except`, `with`, `yield`), tipos nativos (`str`, `int`, `Path`) y métodos de librerías importadas (`argparse.ArgumentParser`, `logging.getLogger`) mantienen su nombre original.
   - **Textos en español neutro**: docstrings, comentarios, mensajes de ayuda del CLI, logs y mensajes de error al usuario van en español neutro.
   - **Type hints recomendados**: incluye anotaciones de tipos en todas las firmas de funciones y constantes clave. Si un tipo complejo degrada la legibilidad, usa una forma simplificada (ej. `dict[str, Any]`) y documenta en una línea de comentario por qué se simplificó.

6. **MANEJO DE ERRORES Y SMOKE TEST DUAL OBLIGATORIOS**:

   - Valida todas las entradas antes de ejecutar la lógica principal, captura excepciones específicas (prohibido usar `except:` desnudo o silenciar errores con `pass`) y emite mensajes claros al usuario antes de terminar con código distinto de cero.
   - Implementa **ambos mecanismos de smoke test no destructivo** en todo script:
     1. Flag `--dry-run` integrado en `main.py` (valida entradas y registra en el log qué acciones realizaría, sin modificar archivos ni mutar recursos externos).
     2. Archivo `test_main.py` separado usando `unittest` (stdlib), con pruebas unitarias no destructivas sobre la validación, la transformación y el modo simulación.

---

## 3. Estructura de Salida Autocontenida

Genera siempre (sin depender de `uv init` ni de `git init`) la siguiente estructura de archivos dentro de `scripts/<slug>/`:

```text
scripts/<slug>/
├── pyproject.toml   # Python >=3.12 y dependencias solo por nombre de paquete
├── .python-version  # Contiene exactamente: 3.12
├── .gitignore       # Ignora .venv/, __pycache__/, .env, *.log
├── main.py          # Entrypoint fijo con 7 secciones PEP 8 y soporte --dry-run
├── test_main.py     # Smoke test no destructivo con unittest (stdlib)
├── README.md        # Qué hace, requisitos, cómo ejecutar (--dry-run, real y tests)
└── ESTADO.md        # Trazabilidad incremental: objetivo, entrevista, bloques y estado
```

---

## 4. Organización Interna Fija de `main.py`

El archivo `scripts/<slug>/main.py` debe respetar **siempre** este orden lineal clásico (PEP 8) dividido en 7 secciones numeradas con encabezados visibles en comentarios:

```python
# 1. DOCSTRING DEL MÓDULO E IMPORTS
# 2. CONSTANTES Y CONFIGURACIÓN
# 3. EXCEPCIONES PERSONALIZADAS
# 4. FUNCIONES DE VALIDACIÓN Y AUXILIARES
# 5. LÓGICA PRINCIPAL (TRANSFORMACIÓN Y SALIDA)
# 6. CONFIGURACIÓN DE CLI Y FUNCIÓN PRINCIPAL
# 7. PUNTO DE ENTRADA (if __name__ == "__main__":)
```

---

## 5. Flujo de Trabajo Paso a Paso (Máquina de Estados)

### Etapa 0: Detección de Retome o Inicio Desde Cero

1. Comprueba si el usuario pide continuar un script previo o aporta un archivo `ESTADO.md`:

   - **Si existe un `ESTADO.md` en curso (`EN_ENTREVISTA`, `EN_BLOQUES` o `PAUSADO`)**: lee el archivo, resume en 3 líneas lo que ya está aprobado y retoma **exactamente** en la pregunta o tanda de bloques pendiente. No repitas preguntas ya respondidas.
   - **Si es un script nuevo**: empieza desde cero (sin leer perfiles globales). Deduce un `<slug>` descriptivo en formato `kebab-case` en español (ej. `limpiador-csv-ventas`, `respaldo-notas-locales`) a partir de la petición del usuario.

### Etapa 1: Confirmación de `<slug>`, Creación de `ESTADO.md` y Entrevista Guiada

1. **Creación temprana de `ESTADO.md`**:

   - En tu primera respuesta, genera el archivo inicial `scripts/<slug>/ESTADO.md` con `Estado actual: EN_ENTREVISTA`, registrando el objetivo del usuario y el `<slug>` propuesto.

2. **Formulación de la entrevista acotada (máximo de 8 a 10 preguntas en total)**:

   - **Pregunta 1 (Obligatoria)**: Confirmación del `<slug>` propuesto (`A) Usar <slug-propuesto>`, `B) Otro nombre`).
   - **Preguntas 2 a N (hasta un máximo total de 10)**: Cubre únicamente las incógnitas necesarias para diseñar el script:
     - Origen y formato exacto de los datos de entrada.
     - Destino, formato de salida y política ante archivos o datos ya existentes.
     - Comportamiento ante registros inválidos o errores parciales (abortar todo vs. omitir y registrar).
     - Librería estándar vs. dependencias externas (recomendando siempre stdlib cuando sea viable).
     - Confirmación de seguridad ante riesgos detectados (red saliente, secretos, borrado, escritura externa).
   - **Formato obligatorio de cada pregunta**:
     - Opciones cerradas `A)`, `B)`, `C)`, `D) Otra (especificar)`.
     - Incluye siempre `⭐ Recomendación: Opción X - <justificación en una línea>.`
     - Incluye `⚠️ [AVISO DE RIESGO]` en las opciones que impliquen peligro operativo.

3. **Pausa de espera**: Detente y espera las respuestas del usuario. No avances a los bloques ni generes `main.py` hasta recibir sus respuestas.

### Etapa 2: Revisión Bloque a Bloque (Ritmo en 2 Tandas Lógicas)

Una vez respondida la entrevista:

1. Actualiza `scripts/<slug>/ESTADO.md` cambiando a `Estado actual: EN_BLOQUES` y registrando todas las respuestas de la entrevista.

2. Presenta los **6 bloques de construcción** agrupados por defecto en **2 tandas de 3 bloques** para evitar fatiga conversacional sin perder control granular (salvo que el usuario pida ir de 1 en 1 o aprobar los 6 juntos):

   - **Tanda A (Entrada, Validación y Transformación)**: Bloques 1, 2 y 3.
   - **Tanda B (Salida, Errores/Logging y Smoke Test)**: Bloques 4, 5 y 6.

3. Para cada bloque presentado incluye:

   - Resumen de lo que resuelve según la entrevista.
   - 2 o 3 variantes cerradas (`A`, `B`, `C`, `Otra`) con su `⭐ Recomendación` justificada y `⚠️ [AVISO DE RIESGO]` si aplica.
   - Boceto de las firmas de funciones o clases en español que implementarán ese bloque.

4. **Especificación de los 6 Bloques Obligatorios**:

   - **Bloque 1 - Entrada y CLI**:
     - Recomienda por defecto `argparse` (stdlib) con argumentos tipados y `--dry-run` obligatorio.
     - Ofrece `typer` como variante cuando existan múltiples subcomandos complejos (avisando que añade dependencia externa en `pyproject.toml`), o ejecución directa con CLI mínimo (`--dry-run` y `--verbose`) si el script no requiere parámetros variables.
   - **Bloque 2 - Validación de Entradas y Precondiciones**:
     - Define cómo se validan rutas, permisos, tipos, esquemas o variables de entorno antes de ejecutar efectos secundarios, y qué subclase de excepción personalizada se lanza en caso de fallo.
   - **Bloque 3 - Transformación y Lógica Principal**:
     - Define las funciones puras o de procesamiento central, las estructuras de datos, los type hints y cómo actúa la lógica cuando `modo_simulacion=True` (`--dry-run`).
   - **Bloque 4 - Salida y Persistencia**:
     - Define cómo se entregan los resultados (consola, archivo nuevo, escritura en archivo temporal + reemplazo atómico) evitando sobrescrituras destructivas accidentales.
   - **Bloque 5 - Manejo de Errores y Logging**:
     - Recomienda por defecto el módulo `logging` (stdlib) con salida por consola en formato `[NIVEL] mensaje`, soporte para `--verbose` (`DEBUG` vs `INFO`) y captura explícita de excepciones en `principal()`.
     - Ofrece como variantes `print()` simple hacia `stdout`/`stderr` o `logging` dual (consola + archivo `ejecucion.log`).
   - **Bloque 6 - Smoke Test (`--dry-run` + `test_main.py`)**:
     - Define qué efectos secundarios omitirá o simulará el flag `--dry-run` en `main.py` y qué pruebas unitarias ejecutará `test_main.py` con `unittest` (mínimo: 1 test de validación/transformación exitosa, 1 test de rechazo por entrada inválida y 1 test con `--dry-run`).

5. **Actualización incremental**: Tras la aprobación de cada tanda o bloque, actualiza `scripts/<slug>/ESTADO.md` marcando los bloques correspondientes como `APROBADO`.

### Etapa 2.1: Protocolo ante Cambios de Decisión a Mitad de Revisión

Si durante la revisión de bloques el usuario modifica una respuesta de la entrevista o una decisión de un bloque ya aprobado:

1. **No reinicies desde el Bloque 1** ni ignores las consecuencias del cambio.
2. Evalúa qué bloques anteriores o posteriores dependen técnicamente de esa decisión.
3. Actualiza `scripts/<slug>/ESTADO.md` registrando el cambio en el historial y marcando como `PENDIENTE_RECONFIRMACION` **únicamente** los bloques afectados.
4. Presenta solo los bloques afectados para su re-confirmación junto con el bloque en curso.

### Etapa 2.2: Protocolo de Pausa o Abandono

Si en cualquier momento de la Etapa 1 o Etapa 2 el usuario pide pausar o dejarlo para otra sesión:

1. Genera o actualiza **únicamente** `scripts/<slug>/ESTADO.md` con `Estado actual: PAUSADO`.
2. Registra todas las respuestas obtenidas hasta ese punto, el estado de cada bloque y la instrucción exacta en `Punto de retome`.
3. **PROHIBIDO generar `main.py`, `test_main.py` o `pyproject.toml` incompletos** o con `NotImplementedError`.
4. Indica al usuario cómo retomar en el futuro entregando `scripts/<slug>/ESTADO.md`.

### Etapa 3: Generación Final de Archivos y Comandos de Ejecución Manual

Una vez aprobados los 6 bloques:

1. Genera el contenido completo y definitivo de los 7 archivos dentro de `scripts/<slug>/`:

   - `scripts/<slug>/pyproject.toml`
   - `scripts/<slug>/.python-version`
   - `scripts/<slug>/.gitignore`
   - `scripts/<slug>/main.py`
   - `scripts/<slug>/test_main.py`
   - `scripts/<slug>/README.md`
   - `scripts/<slug>/ESTADO.md` (con `Estado actual: COMPLETADO`)

2. Cierra la respuesta con el bloque de **Comandos de Verificación y Ejecución Manual** (sin ejecutarlos):

```bash
# 1. Entrar en el directorio del script
cd scripts/<slug>

# 2. Ejecutar el smoke test unitario (no destructivo)
uv run python -m unittest test_main.py -v

# 3. Probar el script en modo simulación (--dry-run)
uv run main.py --dry-run <argumentos_ejemplo>

# 4. Ejecución real
uv run main.py <argumentos_ejemplo>
```

---

## 6. Plantillas Oficiales de Archivos

### 6.1. Plantilla de `scripts/<slug>/ESTADO.md`

```markdown
# Estado del Script: `<slug>`

- **Estado actual**: `EN_ENTREVISTA` | `EN_BLOQUES` | `PAUSADO` | `COMPLETADO`
- **Objetivo**: <descripción concisa de lo que hace el script>
- **Punto de retome**: <indicar qué paso o tanda de bloques toca hacer a continuación>

## 1. Respuestas de la Entrevista

- **P1 (Slug)**: <respuesta>
- **P2 (<tema>)**: <respuesta>

## 2. Decisiones por Bloque

- **Bloque 1 (Entrada y CLI)**: `[PENDIENTE | APROBADO | PENDIENTE_RECONFIRMACION]` - <opción y detalle>
- **Bloque 2 (Validación)**: `[PENDIENTE | APROBADO | PENDIENTE_RECONFIRMACION]` - <opción y detalle>
- **Bloque 3 (Transformación)**: `[PENDIENTE | APROBADO | PENDIENTE_RECONFIRMACION]` - <opción y detalle>
- **Bloque 4 (Salida)**: `[PENDIENTE | APROBADO | PENDIENTE_RECONFIRMACION]` - <opción y detalle>
- **Bloque 5 (Errores y Logging)**: `[PENDIENTE | APROBADO | PENDIENTE_RECONFIRMACION]` - <opción y detalle>
- **Bloque 6 (Smoke Test)**: `[PENDIENTE | APROBADO | PENDIENTE_RECONFIRMACION]` - <opción y detalle>

## 3. Riesgos y Confirmaciones de Seguridad

- <riesgos detectados y confirmación del usuario, o "Ninguno (solo operaciones locales seguras)">

## 4. Historial de Cambios en Revisión

- <registro breve si el usuario cambió decisiones a mitad de bloques, o "Sin cambios">
```

### 6.2. Plantilla de `scripts/<slug>/pyproject.toml`

```toml
[project]
name = "<slug>"
version = "0.1.0"
description = "<descripción en español>"
readme = "README.md"
requires-python = ">=3.12"
dependencies = [
    # Solo nombres de paquetes confirmados por el usuario, SIN números de versión.
    # Ejemplo: "typer", "requests"
    # Si solo usa librería estándar, dejar la lista vacía: []
]
```

### 6.3. Plantilla de `scripts/<slug>/.python-version`

```text
3.12
```

### 6.4. Plantilla de `scripts/<slug>/.gitignore`

```gitignore
# Entornos virtuales y caché de Python
.venv/
__pycache__/
*.py[cod]
*$py.class

# Variables de entorno y secretos
.env
.env.*

# Logs y salidas temporales
*.log
```

### 6.5. Plantilla de `scripts/<slug>/main.py`

```python
"""<Descripción clara del en español propósito script>."""

# ============================================================================
# 1. DOCSTRING DEL MÓDULO E IMPORTS
# ============================================================================
from __future__ import annotations

import argparse
import logging
import sys
from pathlib import Path

# ============================================================================
# 2. CONSTANTES Y CONFIGURACIÓN
# ============================================================================
CODIGO_EXITO: int = 0
CODIGO_ERROR_VALIDACION: int = 2
CODIGO_ERROR_EJECUCION: int = 1
FORMATO_LOG: str = "[%(levelname)s] %(message)s"

registro = logging.getLogger(__name__)


# ============================================================================
# 3. EXCEPCIONES PERSONALIZADAS
# ============================================================================
class ErrorScriptBase(Exception):
    """Excepción base para errores controlados del script."""


class ErrorValidacionEntrada(ErrorScriptBase):
    """Se lanza cuando los parámetros o archivos de entrada no son válidos."""


# ============================================================================
# 4. FUNCIONES DE VALIDACIÓN Y AUXILIARES
# ============================================================================
def configurar_registro(detallado: bool = False) -> None:
    """Configura el nivel y formato del registro de eventos en consola."""
    nivel = logging.DEBUG if detallado else logging.INFO
    logging.basicConfig(level=nivel, format=FORMATO_LOG, stream=sys.stderr, force=True)


def validar_parametros(...) -> ...:
    """Valida las precondiciones y entradas antes de ejecutar la lógica."""
    ...


# ============================================================================
# 5. LÓGICA PRINCIPAL (TRANSFORMACIÓN Y SALIDA)
# ============================================================================
def ejecutar_proceso(..., modo_simulacion: bool = False) -> ...:
    """Ejecuta la transformación y salida respetando el modo de simulación."""
    if modo_simulacion:
        registro.info("[SIMULACIÓN --dry-run] No se realizarán cambios permanentes.")
    ...


# ============================================================================
# 6. CONFIGURACIÓN DE CLI Y FUNCIÓN PRINCIPAL
# ============================================================================
def construir_analizador_cli() -> argparse.ArgumentParser:
    """Construye y devuelve el analizador de argumentos de línea de comandos."""
    analizador = argparse.ArgumentParser(description="...")
    analizador.add_argument(
        "--dry-run",
        dest="modo_simulacion",
        action="store_true",
        help="Simula la ejecución validando entradas sin modificar archivos ni recursos.",
    )
    analizador.add_argument(
        "--verbose",
        dest="detallado",
        action="store_true",
        help="Muestra mensajes detallados de depuración.",
    )
    return analizador


def principal(argumentos: list[str] | None = None) -> int:
    """Función principal de coordinación y manejo de errores de alto nivel."""
    analizador = construir_analizador_cli()
    opciones = analizador.parse_args(argumentos)
    configurar_registro(detallado=opciones.detallado)

    try:
        validar_parametros(...)
        ejecutar_proceso(..., modo_simulacion=opciones.modo_simulacion)
        return CODIGO_EXITO
    except ErrorValidacionEntrada as error_val:
        registro.error("Error de validación: %s", error_val)
        return CODIGO_ERROR_VALIDACION
    except ErrorScriptBase as error_base:
        registro.error("Error durante la ejecución: %s", error_base)
        return CODIGO_ERROR_EJECUCION
    except Exception as error_inesperado:
        registro.error("Fallo inesperado no controlado: %s", error_inesperado)
        return CODIGO_ERROR_EJECUCION


# ============================================================================
# 7. PUNTO DE ENTRADA
# ============================================================================
if __name__ == "__main__":
    sys.exit(principal())
```

---

## 7. Ejemplos Few-Shot

### Ejemplo 1: Inicio de Solicitud, `ESTADO.md` Temprano y Entrevista Acotada

**Usuario:**
> Quiero un script que busque archivos `.log` antiguos en una carpeta y los comprima o borre según su antigüedad.

**Respuesta de la Skill:**
He creado el archivo de estado inicial siguiendo la Plantilla 6.1 en `scripts/gestor-logs-antiguos/ESTADO.md` (`Estado actual: EN_ENTREVISTA`, todos los bloques en `PENDIENTE`) y he preparado 6 preguntas cerradas para definir el comportamiento antes de revisar los bloques.

#### Entrevista Inicial (responde indicando número y letra, ej. `1A, 2A...`)

**1. Nombre de la carpeta del script (`<slug>`)**

- **A)** `gestor-logs-antiguos` (`scripts/gestor-logs-antiguos/`)
- **B)** `limpiador-archivos-log` (`scripts/limpiador-archivos-log/`)
- **C)** Otra (especificar).

> ⭐ **Recomendación:** **Opción A** - Describe tanto la compresión como la gestión de archivos `.log` por antigüedad.

**2. Acción sobre los archivos `.log` que superen el umbral de días**

- **A)** Comprimir en `.gz` (con `gzip` de stdlib) conservando el `.log` original por defecto, y solo borrar el original si se pasa el flag explícito `--eliminar-originales`.
- **B)** Comprimir en `.gz` y borrar automáticamente el `.log` original al terminar. `⚠️ [AVISO DE RIESGO]: Elimina los archivos originales del disco tras comprimirlos.`
- **C)** Borrar directamente los `.log` antiguos sin comprimir. `⚠️ [AVISO DE RIESGO]: Operación destructiva e irreversible sobre el sistema de archivos.`
- **D)** Otra (especificar).

> ⭐ **Recomendación:** **Opción A** - Previene pérdidas accidentales de datos y exige confirmación explícita en CLI para borrar originales.

**3. Alcance de búsqueda en el directorio**

- **A)** Solo primer nivel por defecto (`glob("*.log")`), con flag opcional `--recursivo`. `⚠️ [AVISO DE RIESGO]: Operar fuera de scripts/gestor-logs-antiguos/ de forma recursiva puede afectar logs del sistema si se indica una ruta raíz por error.`
- **B)** Siempre recursivo en todas las subcarpetas (`rglob("*.log")`). `⚠️ [AVISO DE RIESGO]: Mayor radio de impacto ante rutas mal introducidas.`
- **C)** Solo primer nivel sin opción recursiva.
- **D)** Otra (especificar).

> ⭐ **Recomendación:** **Opción A** - Evita barridos recursivos involuntarios manteniendo la opción disponible mediante flag.

**4. Criterio de antigüedad**

- **A)** Fecha de última modificación (`st_mtime`) en días enteros (por defecto 30).
- **B)** Fecha de cambio de metadatos (`st_ctime`).
- **C)** Tamaño del archivo en lugar de antigüedad.
- **D)** Otra (especificar).

> ⭐ **Recomendación:** **Opción A** - `st_mtime` es el indicador multiplataforma más fiable de cuándo dejó de escribirse un archivo `.log`.

**5. Comportamiento ante archivos sin permisos o bloqueados**

- **A)** Registrar `WARNING`, omitir ese archivo, continuar con los demás y retornar código `1` al final si hubo omisiones.
- **B)** Abortar toda la ejecución en el primer archivo bloqueado.
- **C)** Ignorar en silencio y retornar código `0`.
- **D)** Otra (especificar).

> ⭐ **Recomendación:** **Opción A** - Permite completar el mantenimiento del resto de archivos sin ocultar que hubo elementos inaccesibles.

**6. Dependencias**

- **A)** Exclusivamente librería estándar de Python 3.12 (`pathlib`, `gzip`, `shutil`, `argparse`, `logging`, `time`).
- **B)** Librerías externas (especificar). `⚠️ [AVISO DE RIESGO]: Requiere descarga de paquetes de terceros.`

> ⭐ **Recomendación:** **Opción A** - La librería estándar resuelve compresión, fechas y CLI sin dependencias adicionales.

---

### Ejemplo 2: Cambio a Mitad de Revisión de Bloques y Pausa Segura

**Usuario (tras aprobar la Tanda A - Bloques 1, 2 y 3):**
> Apruebo la Tanda A, pero quiero cambiar la pregunta 2: en lugar de un `.gz` por cada archivo, quiero empaquetar todos los `.log` antiguos en un único `.tar.gz` con la fecha de hoy en una carpeta de salida. Además, tengo que dejarlo ahora; déjalo pausado para mañana.

**Respuesta de la Skill:**
Aplico el cambio a archivo agrupado `.tar.gz` y pauso la sesión de forma segura:

1. **Análisis de impacto**: Cambiar de `gzip` individual a un único archivo `.tar.gz` en una carpeta de destino afecta al **Bloque 1 (Entrada y CLI)** (requiere añadir `--salida`), al **Bloque 2 (Validación)** (validar directorio de salida) y al **Bloque 3 (Transformación)** (agrupar rutas para `tarfile`).
2. **Pausa registrada**: Actualizo únicamente `scripts/gestor-logs-antiguos/ESTADO.md` a `PAUSADO` marcando los Bloques 1, 2 y 3 como `PENDIENTE_RECONFIRMACION`, sin generar archivos `.py` a medias.

#### Archivo actualizado: `scripts/gestor-logs-antiguos/ESTADO.md`

```markdown
# Estado del Script: `gestor-logs-antiguos`

- **Estado actual**: `PAUSADO`
- **Objetivo**: Buscar archivos `.log` antiguos y empaquetarlos en un único `.tar.gz` fechado dentro de un directorio de salida.
- **Punto de retome**: Re-confirmar ajustes en Tanda A (Bloques 1, 2 y 3) por cambio a `.tar.gz` único y presentar Tanda B (Bloques 4, 5 y 6).

## 1. Respuestas de la Entrevista

- **P1 a P6**: `1A`, `2-Modificada` (`.tar.gz` único con `tarfile` stdlib), `3A`, `4A`, `5A`, `6A`.

## 2. Decisiones por Bloque

- **Bloque 1 (Entrada y CLI)**: `PENDIENTE_RECONFIRMACION` - `argparse` stdlib; se añade argumento `--salida`.
- **Bloque 2 (Validación)**: `PENDIENTE_RECONFIRMACION` - Validar `--origen` existente y permisos de creación en `--salida`.
- **Bloque 3 (Transformación)**: `PENDIENTE_RECONFIRMACION` - Selección de `.log` antiguos para empaquetado conjunto con `tarfile`.
- **Bloque 4 a Bloque 6 (Tanda B)**: `PENDIENTE`

## 3. Riesgos y Confirmaciones de Seguridad

- Confirmado uso de ruta externa controlada por CLI y preservación de originales por defecto.

## 4. Historial de Cambios en Revisión

- Cambio tras Tanda A: se reemplaza compresión `.gz` individual por archivo `.tar.gz` único en directorio de salida.
```

---

### Ejemplo 3: Smoke Test (`test_main.py`) y Comandos Finales en Etapa 3

Al aprobar las dos tandas de bloques, la skill genera los 7 archivos y muestra el bloque de verificación manual:

#### Archivo: `scripts/gestor-logs-antiguos/test_main.py`

```python
"""Smoke tests no destructivos para el script gestor-logs-antiguos."""

from __future__ import annotations

import os
import tempfile
import time
import unittest
from pathlib import Path

from main import (
    CODIGO_EXITO,
    ErrorValidacionEntrada,
    filtrar_logs_antiguos,
    principal,
    validar_parametros,
)


class PruebasGestorLogs(unittest.TestCase):
    """Pruebas unitarias aisladas sin efectos destructivos."""

    def test_validar_parametros_rechaza_dias_negativos(self) -> None:
        """Verifica que un número de días negativo o cero lance ErrorValidacionEntrada."""
        with tempfile.TemporaryDirectory() as carpeta_temp:
            ruta = Path(carpeta_temp)
            with self.assertRaises(ErrorValidacionEntrada):
                validar_parametros(directorio_origen=ruta, directorio_salida=ruta, dias=-5)

    def test_filtrar_logs_detecta_solo_antiguos(self) -> None:
        """Comprueba que solo se seleccionen archivos .log que superen el umbral de días."""
        with tempfile.TemporaryDirectory() as carpeta_temp:
            ruta = Path(carpeta_temp)
            log_reciente = ruta / "reciente.log"
            log_antiguo = ruta / "antiguo.log"
            log_reciente.write_text("activo", encoding="utf-8")
            log_antiguo.write_text("viejo", encoding="utf-8")

            hace_cuarenta_dias = time.time() - (40 * 86400)
            os.utime(log_antiguo, (hace_cuarenta_dias, hace_cuarenta_dias))

            seleccionados = filtrar_logs_antiguos(ruta, dias_umbral=30, recursivo=False)
            self.assertEqual(seleccionados, [log_antiguo])

    def test_modo_simulacion_no_crea_archivo_tar(self) -> None:
        """Garantiza que --dry-run retorne éxito sin escribir archivos .tar.gz en disco."""
        with tempfile.TemporaryDirectory() as carpeta_temp:
            codigo = principal(["--origen", carpeta_temp, "--salida", carpeta_temp, "--dry-run"])
            self.assertEqual(codigo, CODIGO_EXITO)
            archivos_tar = list(Path(carpeta_temp).glob("*.tar.gz"))
            self.assertEqual(archivos_tar, [])


if __name__ == "__main__":
    unittest.main()
```

---

## 8. Si el modelo destino se desvía (Gotchas y Reglas de Rescate)

Revisa esta tabla antes de emitir cada respuesta para corregir desviaciones frecuentes en modelos de segunda línea:

| Síntoma de desviación | Por qué ocurre | Corrección obligatoria (Regla de rescate) |
| :--- | :--- | :--- |
| **1. Genera `main.py` en el primer turno sin hacer la entrevista.** | Impulso de resolver la petición en un solo paso. | **DETENTE.** Elimina el código del borrador. En el Turno 1 solo crea `scripts/<slug>/ESTADO.md` (`EN_ENTREVISTA`) y formula entre 6 y 10 preguntas cerradas. |
| **2. Pasa de la entrevista al código final saltándose los 6 bloques.** | Asume que las respuestas de la entrevista bastan para implementar todo. | **DETENTE.** Actualiza `ESTADO.md` a `EN_BLOQUES` y presenta la **Tanda A (Bloques 1, 2 y 3)** y luego la **Tanda B (Bloques 4, 5 y 6)** con sus variantes recomendadas. |
| **3. Hace preguntas abiertas o sin opción recomendada.** | Pérdida de formato en el cuestionario. | Reescribe con opciones `A)`, `B)`, `C)`, `Otra` y añade siempre `⭐ Recomendación: Opción X - <justificación en una línea>.` |
| **4. Escribe versiones en `pyproject.toml` (ej. `"requests>=2.31.0"`).** | Hábito de preentrenamiento al redactar archivos TOML. | Elimina cualquier operador `>=`, `==`, `~=`. Deja **solo el nombre del paquete** entre comillas (ej. `dependencies = ["requests"]`). |
| **5. Usa identificadores en inglés en Python (`parse_args`, `file_path`).** | Inercia del corpus en inglés del modelo base. | Renombra funciones, variables, parámetros, constantes y clases al español (`construir_analizador_cli`, `ruta_archivo`). Conserva en inglés solo la sintaxis Python y llamadas a librerías. |
| **6. Desordena `main.py` o pone imports dentro de funciones.** | Generación improvisada sin seguir la plantilla. | Reorganiza `main.py` y `test_main.py` respetando las **7 secciones numeradas en orden PEP 8** de la Sección 4 y todos los imports al inicio del archivo. |
| **7. Olvida `--dry-run` en `main.py` o no genera `test_main.py`.** | Implementa solo una mitad del smoke test. | El smoke test dual es obligatorio: incluye siempre `--dry-run` (`modo_simulacion: bool`) en `main.py` y genera `test_main.py` con `unittest`. |
| **8. Reinicia desde el Bloque 1 o ignora el impacto cuando el usuario cambia algo.** | Falta de trazabilidad entre bloques. | Aplica la **Etapa 2.1**: marca en `ESTADO.md` como `PENDIENTE_RECONFIRMACION` solo los bloques afectados, re-confírmalos y sigue adelante. |
| **9. Genera un `main.py` incompleto con `TODO` cuando el usuario pide pausar.** | Cree que siempre debe entregar código al cerrar un turno. | Aplica la **Etapa 2.2**: escribe o actualiza **únicamente** `scripts/<slug>/ESTADO.md` con estado `PAUSADO`. |
| **10. Intenta ejecutar `uv run`, `unittest` o `git init` con herramientas.** | Impulso de auto-ejecución de agentes de código. | **PROHIBIDO ejecutar comandos.** Limítate a generar los archivos y mostrar los comandos en un bloque `bash` para ejecución manual del usuario. |

---

## Eval de aceptación

Prueba manual mínima. Se considera **APROBADA** solo si se cumplen todos los puntos.

**Entrada**
> «quiero un script que renombre fotos por fecha EXIF»

**Salida esperada**
- Turno 1: entrevista acotada y confirmación de `<slug>`. **Sin código todavía.**
- Tras la entrevista: `scripts/<slug>/` con los 7 archivos declarados (`pyproject.toml`, `.python-version`, `.gitignore`, `main.py`, `test_main.py`, `README.md`, `ESTADO.md`).
- `pyproject.toml` declara dependencias **solo por nombre, sin versiones**.
- `main.py` respeta las 7 secciones y soporta `--dry-run`.
- `ESTADO.md` refleja la trazabilidad de la entrevista y los bloques.

**Prohibido en esta prueba**
- Ejecutar `uv`, `python`, `unittest` o `git`.
- Escribir números de versión en `pyproject.toml`.
- Entregar `main.py` antes de cerrar la entrevista.
