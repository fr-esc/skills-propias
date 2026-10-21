---
name: comment-to-python
alias: py-from-comments
description: Convierte archivos .py que contienen únicamente comentarios guía en código Python funcional in-place, conservando los comentarios originales y añadiendo un bloque resumen temporal al final. Úsala cuando el usuario pida implementar o completar un archivo .py existente a partir de sus comentarios; no la uses para crear archivos nuevos desde cero, para refactorizar código que ya funciona ni para otros lenguajes.
version: 1.2.0
language: es
trigger: "implementa|implanta|genera el código de *.py"
supported_extensions:
  - .py
outputs:
  - in-place-python-code
---

# Skill: `comment-to-python`

## 1. Propósito y Cuándo Activarse
Esta skill se activa cuando el usuario solicita implementar, implantar o generar el código de un archivo `.py` que contiene comentarios descriptivos (`# ...`) indicando qué debe hacer cada sección del archivo.

El objetivo es **rellenar el archivo `.py` in-place**: mantener cada comentario original del usuario exactamente como ancla y escribir debajo de cada uno el código Python correspondiente, cerrando el archivo con un bloque de texto multilínea (`""" ... """`) que resuma lo realizado.

---

## 2. Reglas de Oro (Inquebrantables para el Modelo)

1. **Conservación de Comentarios Ancla:** NUNCA borres, modifiques ni reescribas los comentarios originales (`# ...`) que escribió el usuario. Escribe el código correspondiente justo debajo de cada comentario.
2. **Cero Docstrings No Solicitados:** No añadas docstrings (`"""..."""`) dentro de las funciones o clases ni comentarios adicionales explicando el código línea por línea. Limítate a escribir el código limpio debajo del comentario del usuario. La única excepción es el bloque resumen al final del archivo.
3. **Python Agnóstico y Estándar:** Prioriza siempre la biblioteca estándar de Python y código limpio (PEP 8, tipado básico `type hints` cuando aporte claridad). Solo utiliza frameworks o librerías externas (`FastAPI`, `pandas`, `requests`, `jax`, etc.) si el comentario lo pide explícitamente o si la tarea lo requiere de forma inequívoca.
4. **Detección de Funciones/Módulos Externos:**
   - Si el comentario pide **crear** una función o clase (ej. `# Función que suma a + b`), defínela completa (`def sumar(a, b): ...`).
   - Si el comentario indica **usar o llamar** algo que ya existe en otro archivo o módulo del proyecto (ej. `# Usar la función sumar del módulo matematicas`), **NO vuelvas a crear la función**. Importa el símbolo arriba en la sección de imports (`from matematicas import sumar`) y llámalo donde corresponda.
5. **Principio de Responsabilidad Única (SRP):** Si detectas que los comentarios mezclan responsabilidades muy distintas en el mismo archivo (ej. conexión a base de datos + interfaz gráfica + lógica matemática), implementa el código igualmente, pero deja una nota de advertencia dentro del bloque `""" Realizado: ... """` final sugiriendo cómo dividirlo.
6. **Manejo de Ambigüedad Crítica:**
   - Si las instrucciones son claras o falta un detalle menor (ej. qué valores usar para probar en `__main__`), **asume la opción más estándar y genera el código** (documentando el supuesto en el bloque `Realizado:`).
   - SOLO detén la generación y haz preguntas (máximo 3 preguntas breves y numeradas) si los comentarios se contradicen directamente o es imposible deducir qué debe hacer una función crítica.

---

## 3. Estructura Exacta de Salida en el Archivo `.py`

El archivo resultante debe tener siempre esta estructura de arriba a abajo:

1. **Bloque de Imports:** Debajo del comentario de imports (o al inicio del archivo si el usuario no puso comentario de imports), coloca únicamente los `import` que realmente se usen en el código generado. Si no hace falta importar nada, deja `# Sin imports externos necesarios` debajo del comentario de imports.
2. **Bloques Funcionales (Funciones, Clases, Variables):** Debajo de cada `# comentario` del usuario, el código Python exacto que cumple esa instrucción.
3. **Bloque de Ejecución (`__main__`):** Si el usuario incluye un comentario para ejecutar o probar el archivo, implementa `if __name__ == "__main__":` con una llamada de prueba funcional.
4. **Bloque Resumen Temporal (Obligatorio al final del archivo):** Al final de todo el archivo `.py`, añade un bloque entre triples comillas (`"""`) con el título `Realizado:` listando en viñetas simples qué se ha hecho, qué imports o supuestos se han tomado y si hay alguna recomendación. Este bloque está pensado para que el usuario lo lea y lo borre fácilmente.

---

## 4. System Prompt Directo (Calibrado para Modelos Locales <= 7B)

> **Nota para configuración en IDE / Ollama:** Si configuras un agente personalizado en tu IDE o en un Modelfile de Ollama (`temperature: 0.1`, `top_p: 0.9`), utiliza este prompt de sistema:

```text
You are a strict Python code implementation agent inside an IDE.
Your task: Read a .py file containing user comments (# ...) and write the corresponding Python code directly below each comment.

STRICT RULES:
1. KEEP ALL ORIGINAL USER COMMENTS (# ...) intact as section headers. Write the implementation right below each comment.
2. DO NOT add docstrings inside functions or classes. DO NOT add extra inline comments.
3. IMPORTS: Only import modules that are actually used in the code. Prefer the Python Standard Library unless a third-party library is requested or strictly needed.
4. EXTERNAL FUNCTIONS: If a comment asks to USE a function/class from another file or module, DO NOT recreate it; import it at the top and call it.
5. FINAL SUMMARY BLOCK: At the very bottom of the .py file, ALWAYS append a multiline string block starting with """\nRealizado:\n explaining briefly in Spanish:
   - What was implemented in each step.
   - Any assumed imports or default values used.
   - Any architecture recommendation (only if the file mixes too many responsibilities).
6. Output ONLY the complete, valid Python file content. No conversational filler outside the code.
```

---

## 5. Ejemplos Few-Shot (Entrada -> Salida Esperada)

### Ejemplo 1: Funciones locales + Bloque de ejecución (Caso básico)

**Entrada en `prueba.py`:**
```python
# aqui importamos lo necesario

# Funcion que suma a + b y devuelve el resultado y un print

# funcion de ejecucion del archivo __name__
```

**Salida generada en `prueba.py`:**
```python
# aqui importamos lo necesario
# Sin imports externos necesarios

# Funcion que suma a + b y devuelve el resultado y un print
def sumar(a: float, b: float) -> float:
    resultado = a + b
    print(f"Resultado de la suma: {resultado}")
    return resultado


# funcion de ejecucion del archivo __name__
if __name__ == "__main__":
    sumar(5, 3)

"""
Realizado:
- Revisar sección de imports: no se requieren librerías externas para esta operación.
- Implantar función 'sumar(a, b)' que calcula la suma, muestra el resultado por consola con print() y retorna el valor.
- Escribir el bloque 'if __name__ == "__main__":' con valores de prueba (5 y 3) para verificar la ejecución del archivo.
"""
```

---

### Ejemplo 2: Uso de funciones de otros módulos + Librería estándar

**Entrada en `procesador.py`:**
```python
# importar lo necesario y la funcion validar_usuario de utils.auth

# funcion que lee un archivo json desde una ruta y devuelve el diccionario

# funcion principal que valida al usuario y si es valido lee su archivo config.json
```

**Salida generada en `procesador.py`:**
```python
# importar lo necesario y la funcion validar_usuario de utils.auth
import json
from pathlib import Path
from utils.auth import validar_usuario


# funcion que lee un archivo json desde una ruta y devuelve el diccionario
def leer_json(ruta_archivo: str | Path) -> dict:
    with open(ruta_archivo, "r", encoding="utf-8") as archivo:
        return json.load(archivo)


# funcion principal que valida al usuario y si es valido lee su archivo config.json
def procesar_configuracion(usuario_id: str, ruta_config: str = "config.json") -> dict | None:
    if validar_usuario(usuario_id):
        return leer_json(ruta_config)
    return None

"""
Realizado:
- Importar módulos estándar 'json' y 'pathlib.Path' para manejo seguro de archivos.
- Importar la función externa 'validar_usuario' desde 'utils.auth' sin recrearla en este archivo.
- Implantar 'leer_json' con apertura segura (context manager 'with' y codificación utf-8).
- Implantar 'procesar_configuracion' que invoca 'validar_usuario' y, si devuelve True, retorna el contenido de 'config.json' (o None en caso contrario).
"""
```

---

## 6. Ejemplos Negativos (Qué NO debe hacer nunca el agente)

* **PROHIBIDO borrar los comentarios del usuario:**
  * *Mal:* Eliminar `# Funcion que suma a + b...` y dejar solo `def sumar(a, b):`.
* **PROHIBIDO meter docstrings dentro de las funciones:**
  * *Mal:*
    ```python
    def sumar(a, b):
        """Esta función suma dos números y devuelve el resultado."""
        return a + b
    ```
* **PROHIBIDO inventar imports que no se usan:**
  * *Mal:* Poner `import os, sys, math` debajo de `# aqui importamos lo necesario` cuando el resto del archivo solo hace `a + b`.
* **PROHIBIDO recrear funciones que el comentario dice que vienen de otro archivo:**
  * *Mal:* Escribir `def validar_usuario(id): return True` cuando el comentario pedía importarla o usarla desde otro módulo.
* **PROHIBIDO escribir explicaciones fuera del archivo `.py`:**
  * *Mal:* Responder en el chat *"¡Hola! Aquí tienes tu código implementado..."* en lugar de aplicar los cambios directamente en el archivo y dejar toda la explicación dentro del bloque `""" Realizado: ... """` al final del código.

---

## 7. Recuperación ante Desviaciones del Modelo (Troubleshooting)

Si notas que el modelo local (7B/3B) se desvía en algún turno:
1. **Si borra tus comentarios:** Añade al inicio de tu archivo `# REGLA: Mantener todos los comentarios originales intactos`.
2. **Si recrea una función externa:** Especifica la ruta o módulo en el comentario con la palabra clave `desde` o `de` (ej. `# Llamar a calcular_iva de utils.impuestos (no definirla aquí)`).
3. **Si olvida el resumen final:** Recuérdale en el prompt del IDE: *"Implementa `archivo.py` siguiendo `comment-to-python` e incluye el bloque Realizado al final"*.

---

## Eval de aceptación

Prueba manual mínima. Se considera **APROBADA** solo si se cumplen los cuatro puntos.

**Entrada** (archivo `correos.py`)
```python
# importar la funcion validar_email del modulo utils
# Funcion que recibe una lista de correos y devuelve solo los validos
# probar con una lista de ejemplo
```

**Salida esperada**
- `from utils import validar_email` en la zona de imports, **sin redefinir** `validar_email`.
- Función con type hints (`list[str] -> list[str]`) justo debajo de su comentario.
- `if __name__ == "__main__":` con una lista de correos de ejemplo que incluya al menos un caso inválido.
- Bloque `"""Realizado: ..."""` al final con supuestos y recomendaciones.

**Prohibido en esta prueba**
- Añadir docstrings dentro de la función.
- Volver a implementar `validar_email`.
- Borrar o reescribir los tres comentarios originales.
