---
name: finetune-script-generator
description: >
  Guía interactiva y generador de notebooks de fine-tuning óptimos (Unsloth, QAT, TRL, ORPO) para Google Colab, Kaggle y Molab/Marimo.
  Convierte un objetivo, base_model y dataset_path en un flujo único validado y listo para exportar a Ollama (GGUF), vLLM y HuggingFace Hub.
  Úsala cuando el usuario quiera entrenar o adaptar un modelo; no la uses para construir o curar el dataset de entrenamiento (para eso usa generador-datasets),
  ni para diseñar la arquitectura del proyecto.
version: 1.4.0
language: es
annexes:
  - reference/plantilla-notebook.md
  - reference/ejemplo-few-shot.md
trigger: >
  Activar cuando el usuario solicite crear un script o notebook de fine-tuning, entrenar o adaptar un LLM/VLM para una tarea o empresa,
  o proporcione un `base_model` y un `dataset_path` para ejecutar en entornos cloud (Colab, Kaggle, Molab/Marimo).
runtime:
  cloud_environments:
    - Google Colab (Jupyter Cells `# %%`)
    - Kaggle Notebooks (Jupyter Cells `# %%`)
    - Molab / Marimo (Reactive Python Cells `@app.cell`)
  training_engines:
    - Unsloth + TRL (Predeterminado: 2x velocidad, -70% VRAM, train_on_responses_only)
    - Unsloth + TorchAO QAT / Checkpoints QAT nativos (Gemma 3 / Gemma 4 QAT)
    - HuggingFace TRL + PEFT + BitsAndBytes (Fallback de compatibilidad universal)
outputs:
  - Entrevista diagnóstica breve con recomendaciones de arquitectura, método y datos según eficiencia de Pareto.
  - Notebook único autocontenido en español (compatible 100% con Python puro y Jupyter) con hiperparámetros explicados, autodiagnóstico de dataset, validación de inferencia y subida privada por defecto a HuggingFace Hub (GGUF Q4_K_M/Q5_K_M para Ollama + Merged 16-bit para vLLM).
---

# Skill: `finetune-script-generator`

## 1. Propósito y Cuándo Activarse

Esta skill transforma al agente de IA en un **Arquitecto Senior de Fine-Tuning Eficiente en Cloud**. Su objetivo es guiar al usuario y generar un **único notebook autocontenido, listo para ejecutar en entornos cloud (Google Colab, Kaggle o Molab/Marimo)** a partir de dos insumos principales (`base_model` y `dataset_path`) o de una necesidad de negocio/tarea específica.

### Principio Rector: Eficiencia de Pareto (Máximo Resultado por Menor Cómputo/VRAM)
1. **Entorno 100% Cloud Notebook:** Esta skill **nunca** genera scripts pensados para ejecución local por consola ni para agentes locales (como OpenCode o Antigravity). Todo el código se diseña en Python puro portable para ser interpretado y ejecutado dentro de **Google Colab**, **Kaggle** o **Molab (Marimo)**.
2. **Optimización Calidad/VRAM:** Se busca siempre el punto óptimo de Pareto: ni el entrenamiento más caro y pesado (Full Fine-Tuning innecesario) ni la compresión agresiva que destruye el razonamiento. Incluye enmascaramiento de pérdida sobre las respuestas (`train_on_responses_only`) para aumentar la precisión en modelos pequeños sin gastar un solo megabyte extra de VRAM.
3. **Conciencia de Cuantización (QAT y Exportación Inteligente):** Prioriza modelos y flujos preparados para inferencia eficiente en 4 bits —como la familia **Gemma 4 QAT / Gemma 3 QAT** (Quantization-Aware Training) o el flujo **Unsloth + TorchAO QAT**—, que permiten exportar a `Q4` manteniendo casi intacta la precisión de `BF16`.
4. **Privacidad por Defecto en HuggingFace Hub:** Al estar orientado a tareas y datos de empresa, toda subida a HuggingFace Hub se configura como **privada (`HF_PRIVATE_REPO = True`) por defecto**, avisando explícitamente al usuario al finalizar de cómo hacerla pública manualmente desde la web de HuggingFace si así lo desea.

### Cuándo Activarse
* Cuando el usuario entrega un `base_model` y un `dataset_path` (URL, lista de archivos, ID de HuggingFace o ruta subida al entorno cloud).
* Cuando el usuario pide adaptar un modelo pequeño o mediano para una tarea específica de empresa (extracción de datos, soporte técnico, SQL, clasificación, generación de código, estilo corporativo) para luego consumirlo en **Ollama**, **vLLM** o **HuggingFace Hub**.

---

## 2. Contratos de Entrada y Salida

### 2.1. Contrato de Entrada

| Parámetro | Tipo | Carácter | Descripción y Comportamiento por Defecto |
| :--- | :--- | :--- | :--- |
| `base_model` | `str` | **Obligatorio** *(o definido en entrevista)* | ID del modelo en HuggingFace Hub (ej. `"unsloth/gemma-4-E4B-it-qat-q4_0-unquantized"`, `"Qwen/Qwen2.5-Coder-7B-Instruct"`). Si el usuario no sabe cuál elegir, el agente recomienda el óptimo según la tarea. |
| `dataset_path` | `str` \| `list[str]` | **Obligatorio** *(o definido en entrevista)* | ID de dataset en HuggingFace Hub (`"usuario/dataset"`), URL directa o lista de rutas/URLs (`.jsonl`, `.json`, `.parquet`, `.csv`, `.txt`) en el entorno cloud. |
| `cloud_platform` | `str` | **Consultado en entrevista** | Plataforma destino: `"colab"` (Google Colab), `"kaggle"` (Kaggle Notebooks) o `"molab"` (Marimo reactivo). |
| `engine` | `str` | Opcional *(Default: `"unsloth"`)* | `"unsloth"` (predeterminado por ahorro de VRAM y velocidad), `"unsloth_qat"` (para entrenamiento con Quantization-Aware Training vía TorchAO) o `"trl_standard"` (si la arquitectura no está soportada por Unsloth). |
| `training_method` | `str` | Opcional *(Default: `"qlora_sft"`)* | `"qlora_sft"` (SFT en 4-bit, predeterminado), `"qat_lora_sft"` (SFT en 16-bit con simulación Q4), `"orpo"` (alineamiento de preferencias con pares `chosen`/`rejected` sin modelo de referencia extra), o `"lora_16bit"`. |
| `hf_repo_id` | `str` | Opcional *(Default: `"tu-usuario/mi-modelo-finetuned"`)* | Repositorio destino en HuggingFace Hub donde se publicará el modelo tras validarlo. |
| `hf_private_repo` | `bool` | Opcional *(Default: `True`)* | Crea los repositorios en HuggingFace Hub en modo **Privado** por defecto para proteger modelos corporativos, informando al final de cómo cambiarlo a Público en la web de HF. |
| `export_format` | `str` | Opcional *(Default: `"auto_optimal"`)* | Selección automática del formato ideal para producción local (Ollama) y servidor (vLLM/HF): `q4_k_m` para modelos $\ge$ 4B o modelos QAT, `q5_k_m` para modelos diminutos $\le$ 3B no-QAT, más `merged_16bit` (`safetensors`). |

### 2.2. Contrato de Salida

El modelo destino debe estructurar su respuesta en **dos turnos** (o un turno directo solo si el usuario ya proporcionó todas las preferencias de la entrevista en su prompt inicial):

1. **Salida de Fase A (Entrevista Consultiva):** Un cuestionario breve y directo (máximo 4 puntos) que asesora al usuario sobre su objetivo, recomienda modelos/métodos óptimos y confirma si ejecutará en **Colab**, **Kaggle** o **Molab/Marimo** y si usará **Unsloth** (predeterminado).
2. **Salida de Fase B (Notebook Único Autocontenido):** Un único bloque de código estructurado por celdas comentadas al 100% en español, que incluye:
   * **Celda 1:** Verificación de GPU/VRAM e instalación limpia de dependencias mediante `subprocess` (100% compatible con Colab, Kaggle y Marimo sin errores de sintaxis por comandos mágicos `!pip`).
   * **Celda 2:** Panel centralizado de configuración e hiperparámetros, donde **cada variable documenta el valor recomendado, valores alternativos viables y el impacto exacto de modificarlos**.
   * **Celda 3:** Carga del modelo base y tokenizador con resolución automática de colisiones entre `LOAD_IN_4BIT` y `USE_QAT`.
   * **Celda 4:** Autodetección de `dataset_path` (acepta tanto un `str` como una lista `list[str]` de URLs o archivos, o un repo de HF Hub), diagnóstico de esquema (ShareGPT, ChatML, Alpaca, QA, texto plano o preferencias ORPO) y formateo con `apply_chat_template`.
   * **Celda 5:** Configuración del entrenador (`SFTTrainer` con `SFTConfig` y `train_on_responses_only` automático, o `ORPOTrainer` si eligió preferencias) y ejecución con métricas de VRAM y tiempo.
   * **Celda 6:** Validación interactiva con el agente/usuario (inferencia de prueba antes de dar por bueno el modelo).
   * **Celda 7:** Gestión inteligente de credenciales de HuggingFace (`HF_TOKEN`), subida privada por defecto (`private=HF_PRIVATE_REPO`) en formatos óptimos (GGUF para Ollama + Safetensors/LoRA para vLLM y HF Hub) y recordatorio explícito sobre la visibilidad privada del repositorio.

---

## 3. Flujo Paso a Paso que Debe Seguir el Modelo Destino

> **Anexos de esta skill (no se cargan solos, léelos cuando toque):**
> - `reference/plantilla-notebook.md` — plantilla completa de las 7 celdas (§4). **Obligatorio antes de escribir el notebook.**
> - `reference/ejemplo-few-shot.md` — ejemplo íntegro de entrevista + notebook (§8). Consulta opcional.
>
> Ambos están en el directorio de esta skill: `~/.config/opencode/skills/finetune-script-generator/`.

Cuando se active esta skill, sigue estrictamente esta secuencia de razonamiento y ejecución:

### Paso 1: Entrevista Guiada y Recomendación Estratégica
Antes de generar código a ciegas, evalúa qué información ha proporcionado el usuario. Si falta definir el entorno cloud, el motor o el objetivo del entrenamiento, realiza una **entrevista breve y guiada**:

1. **Objetivo y Recomendación de Modelo Base:** Pregunta para qué tarea o función empresarial se entrenará el modelo y recomienda la mejor opción calidad/VRAM:
   * *Uso general / razonamiento compacto listo para Q4 sin pérdida:* Recomendar **`unsloth/gemma-4-E4B-it-qat-q4_0-unquantized`** o **`unsloth/gemma-3-4b-it-qat`** (entrenados con Quantization-Aware Training para conservar la precisión de 16 bits ocupando solo 3–5 GB de VRAM al exportarse a Q4).
   * *Código, SQL, JSON estricto y llamadas a herramientas (Tool Calling):* Recomendar **`Qwen/Qwen2.5-Coder-7B-Instruct`** o **`Qwen/Qwen3-8B`**.
   * *Agentes ultraligeros o ejecución rápida en CPU/Edge:* Recomendar **`unsloth/gemma-4-E2B-it-qat-q4_0-unquantized`** o **`meta-llama/Llama-3.2-3B-Instruct`**.
2. **Confirmación del Entorno Cloud:** Pregunta dónde se va a ejecutar (solo opciones cloud):
   * **Google Colab** (GPU T4 16GB gratuita o L4/A100 Pro).
   * **Kaggle Notebooks** (GPU P100 16GB o doble T4 16GBx2 gratuita).
   * **Molab / Marimo** (Notebook reactivo moderno en cloud).
3. **Confirmación del Motor y Paradigma:**
   * Propón **Unsloth** como predeterminado (ahorra hasta un 70% de VRAM, duplica la velocidad frente a TRL estándar e incluye `train_on_responses_only`).
   * Propón **SFT con QLoRA 4-bit** como predeterminado (o **QAT + LoRA** si quiere simular cuantización de 4 bits durante el fine-tuning en un modelo que no sea nativo QAT), ofreciendo **ORPO** como alternativa si su dataset tiene respuestas buenas (`chosen`) y malas (`rejected`) para alinear tono o políticas sin gastar el doble de VRAM que DPO.

### Paso 2: Análisis y Autodetección del Dataset (`dataset_path`)
Diseña el bloque de carga de datos del script soportando tanto un único `str` como una `list[str]`:
* **Si es una URL, ruta de archivo o lista de archivos (`.jsonl`, `.json`, `.csv`, `.parquet`, `.txt`):** Inspecciona la extensión del primer elemento y llama a `load_dataset("json" | "csv" | "parquet" | "text", data_files=DATASET_PATH, split="train")`.
* **Si es un identificador de HuggingFace Hub (`organizacion/dataset`):** Usa `load_dataset(DATASET_PATH, split="train")`.
* **Autodiagnóstico de Columnas:** El código generado debe inspeccionar automáticamente `dataset.column_names` y transformar las filas al formato conversacional nativo del tokenizador (`tokenizer.apply_chat_template`) cubriendo:
  1. **Formato ChatML / Messages:** Columna `messages` o `conversations` con roles `user`/`assistant`.
  2. **Formato ShareGPT:** Lista con claves `{"from": "human"|"gpt", "value": "..."}` normalizada con `standardize_sharegpt`.
  3. **Formato Instruct / Alpaca:** Columnas `instruction`, `input` (opcional) y `output` / `response`.
  4. **Formato Prompt / Completion o Question / Answer:** Pares directos convertidos a mensajes `user`/`assistant`.
  5. **Formato Texto Plano:** Columna `text` directa.

### Paso 3: Selección Heurística de Hiperparámetros y Documentación Inline
Genera un bloque central de variables de configuración donde cada hiperparámetro incluya:
1. El **valor óptimo por defecto** según el tamaño de `base_model`.
2. Un **comentario técnico claro en español** indicando qué otros valores son funcionales y **qué cambiaría exactamente en el entrenamiento** si el usuario decide cambiarlos.

| Hiperparámetro | Valor Predeterminado Óptimo | Alternativas Funcionales y Efecto Explicado en Comentarios |
| :--- | :--- | :--- |
| `MAX_SEQ_LENGTH` | `2048` | `1024` (reduce VRAM un 35% para tareas cortas) \| `4096` o `8192` (para documentos largos; incrementa VRAM linealmente). |
| `LOAD_IN_4BIT` | `True` | `False` (carga en 16 bits; se desactiva automáticamente si `USE_QAT = True` porque TorchAO QAT requiere pesos en 16 bits para insertar nodos *fake-quantize*). |
| `TRAIN_ON_RESPONSES_ONLY` | `True` | `True` (calcula el loss únicamente sobre las respuestas del asistente e ignora las instrucciones del usuario; sube un 10-20% la precisión en modelos pequeños sin gastar más VRAM) \| `False` (entrena sobre todo el texto; útil solo en preentrenamiento continuo). |
| `LORA_R` | `16` | `8` (menor riesgo de sobreajuste en datasets de menos de 500 filas) \| `32` o `64` (mayor capacidad para inyectar jerga compleja, requiere más VRAM). |
| `LORA_ALPHA` | `16` *(o `32`)* | Regla óptima: igual a `LORA_R` o el doble (`2 * LORA_R`). Subirlo a `2 * LORA_R` acelera la adaptación de los pesos nuevos. |
| `LEARNING_RATE` | `2e-4` | `1e-4` o `5e-5` (recomendado si `LORA_R >= 32`, QAT, o modelos >12B) \| `8e-6` (obligatorio si se usa `ORPOTrainer`). |
| `PER_DEVICE_BATCH_SIZE` | `2` | `1` (si salta error de falta de VRAM en GPUs de 16GB) \| `4` u `8` (en GPUs L4/A100 de 24-40GB; acelera cada época). |
| `GRADIENT_ACCUMULATION_STEPS` | `4` | Ajustar para que `BATCH_SIZE * GRAD_ACCUM = 8` o `16`. Subirlo a `8` o `16` suaviza la curva de pérdida sin gastar más VRAM. |
| `NUM_TRAIN_EPOCHS` | `3` | `1` (óptimo si el dataset supera las 10.000 filas) \| `3` a `5` (óptimo para datasets corporativos pequeños de 200 a 2.000 ejemplos). |
| `HF_PRIVATE_REPO` | `True` | `True` (crea el repositorio en HuggingFace como Privado por seguridad empresarial) \| `False` (lo publica abierto a todo el mundo). |

### Paso 4: Adaptación de Sintaxis según Plataforma Cloud
* **Instalación sin comandos mágicos (`!pip` prohibido):** Usa siempre `subprocess.check_call([sys.executable, "-m", "pip", "install", ...])` en la Celda 1 para que el código sea 100% Python válido tanto en **Colab/Kaggle** como en **Molab/Marimo**.
* **Si `cloud_platform` es `"colab"` o `"kaggle"`:** Estructura el archivo con separadores de celda estándar `# %% [markdown]` y `# %%`.
* **Si `cloud_platform` es `"molab"` (Marimo):** Envuelve cada bloque lógico en una función decorada con `@app.cell` dentro de `import marimo as mo; app = mo.App()`, asegurando que **ninguna variable global se redefina en dos celdas distintas**.

### Paso 5: Estrategia de Cuantización, Privacidad y Subida Validada a HuggingFace Hub
1. **Validación Pre-Subida (Celda 6):** Ejecutar una inferencia de prueba con `FastLanguageModel.for_inference(model)` e imprimir la respuesta antes de subir nada.
2. **Verificación de Credenciales y Subida Privada (Celda 7):**
   * Buscar `HF_TOKEN` en secretos de Colab, Kaggle o variables de entorno; si no existe, solicitarlo interactivamente con `getpass.getpass()`.
   * **Selección automática del formato GGUF óptimo:**
     * **Si el modelo es Gemma 3 / Gemma 4 QAT o se activó `USE_QAT = True`:** Exportar a `q4_k_m` (sus pesos ya están preparados para 4 bits sin pérdida).
     * **Si el modelo tiene $\ge$ 4B parámetros:** Exportar a `q4_k_m` (punto óptimo universal para **Ollama** y **llama.cpp**).
     * **Si el modelo es pequeño ($\le$ 3B parámetros y NO es QAT):** Seleccionar automáticamente `q5_k_m` para proteger la coherencia sin apenas coste de VRAM (+300 MB).
   * **Aviso Final de Privacidad:** Al terminar la subida con `private=HF_PRIVATE_REPO`, imprimir un aviso destacado recordando al usuario que **el modelo ha quedado en modo PRIVADO (oculto al público)** y explicando que si desea hacerlo público debe entrar en la pestaña `Settings` de su repositorio en HuggingFace Hub y pulsar `Make public` (o cambiar `HF_PRIVATE_REPO = False`).

---

## 4. Plantilla del Notebook (anexo externo)

La arquitectura completa del notebook (**7 celdas**) vive en un anexo para no cargarla en cada conversación:

> **Lee `reference/plantilla-notebook.md` antes de escribir el notebook**, desde el directorio de esta skill (`~/.config/opencode/skills/finetune-script-generator/`). Es **obligatorio**: contiene la Celda 2 con los hiperparámetros comentados y la Celda 7 con la subida privada validada.

Qué cubre el anexo:
- **Celda 1:** verificación de GPU/entorno e instalación limpia (Colab, Kaggle y Molab).
- **Celda 2:** panel central de hiperparámetros; cada uno con valor recomendado, alternativas y efecto de cambiarlo.
- **Celda 3:** carga con `FastLanguageModel` y resolución del conflicto QAT vs QLoRA 4-bit.
- **Celda 4:** autodetección, diagnóstico y formateo del dataset.
- **Celda 5:** SFT con monitoreo de VRAM y `train_on_responses_only`.
- **Celda 6:** validación de inferencia antes de subir nada.
- **Celda 7:** credenciales y subida a HF Hub (privada) con LoRA, merged 16-bit y GGUF.
- **§4.1:** bloque intercambiable `ORPOTrainer` que sustituye la Celda 5.
- **§4.2:** reglas de adaptación a Molab/Marimo (`@app.cell`).

**No reconstruyas la plantilla de memoria.** Si el anexo no es accesible, dilo explícitamente y pide permiso antes de improvisar una arquitectura distinta.

---

## 5. Sección de Validación

El modelo destino debe aplicar dos niveles de validación: **Validación Estática (antes de entregar el script al usuario)** y **Validación Empírica (instrucciones dentro del notebook antes de publicar en HuggingFace)**.

### 5.1. Checklist de Validación Estática del Código Generado
Antes de imprimir el bloque de código final, comprueba internamente los siguientes 7 puntos:
1. **Cero comandos mágicos ni rutas locales:** ¿La Celda 1 instala paquetes con `subprocess.check_call([sys.executable, "-m", "pip", ...])` en lugar de `!pip` y no asume rutas de disco duro local?
2. **Prevención de conflicto `USE_QAT` vs `LOAD_IN_4BIT`:** ¿La Celda 3 incluye `effective_load_in_4bit = False if USE_QAT else LOAD_IN_4BIT`?
3. **Soporte para lista de archivos en `dataset_path`:** ¿La función `load_and_detect_dataset` comprueba `isinstance(path_or_url, (list, tuple))` para no fallar si el usuario pasa varios archivos `.jsonl`?
4. **Sincronización del `chat_template` y `train_on_responses_only`:** ¿Se usa `tokenizer.apply_chat_template` en la Celda 4 y en la Celda 6, y se envuelve el `SFTTrainer` con `train_on_responses_only` de forma segura (`try/except`)?
5. **Documentación completa de hiperparámetros:** ¿Cada variable de la Celda 2 tiene su comentario explicando qué valores alternativos existen y qué cambiaría si se usan?
6. **Privacidad por defecto y recordatorio final:** ¿Está `HF_PRIVATE_REPO = True` en la Celda 2, se pasa `private=HF_PRIVATE_REPO` en todas las funciones `push_to_hub*` de la Celda 7 y se imprime el aviso final indicando cómo pasarlo a público en HuggingFace?
7. **Prompt de prueba contextualizado:** ¿El hueco de prompt de prueba de la Celda 6 ha sido rellenado con un ejemplo realista acorde a la tarea que el usuario describió en la entrevista?

### 5.2. Validación Empírica Post-Entrenamiento (Guiada por el Agente en Cloud)
El script obliga a detenerse conceptualmente en la **Celda 6** para que el usuario y el agente interno de Colab/Kaggle/Molab revisen:
* **Curva de Training Loss:** Debe descender progresivamente (ej. de `1.8` a `0.6`). Si baja a `0.001` en pocas iteraciones, avisar al usuario de *overfitting* y recomendar bajar `NUM_TRAIN_EPOCHS` o `LORA_R`.
* **Test de Inferencia:** Verificar que el modelo respeta el formato pedido (ej. JSON válido, SQL limpio, tono corporativo) y detiene la generación correctamente con el token EOS antes de ejecutar la **Celda 7** (subida a HF Hub).

---

## 6. Guardarraíles Explícitos para el Modelo Destino

1. **Prohibido inventar IDs de modelos o librerías:** Usa únicamente identificadores reales de HuggingFace Hub (`unsloth/...`, `google/...`, `Qwen/...`, `meta-llama/...`). Nunca inventes métodos inexistentes en `FastLanguageModel` o `SFTTrainer`.
2. **Prohibido generar código para ejecución local por consola:** No generes scripts con `argparse`, `sys.argv`, ni archivos `train.py` + `requirements.txt` separados. Todo debe ir en un único notebook interactivo para **Colab, Kaggle o Molab**.
3. **Tratamiento defensivo de datasets desconocidos:** Si el usuario pasa una URL, archivo o lista de archivos `.jsonl` cuyas columnas no conoces de antemano, mantén siempre el bloque policondicional de la Celda 4 (`messages` / `conversations` / `instruction`+`output` / `prompt`+`completion` / `text`) que inspecciona las columnas en tiempo de ejecución e imprime la primera fila antes de entrenar.
4. **Avisos obligatorios de límites de VRAM y Disco Temporal en Cloud:**
   * En **Google Colab gratuito (T4 16GB VRAM / ~12GB RAM de sistema)** y **Kaggle (T4/P100)**: advierte en los comentarios que modelos de más de **14B parámetros** o activar `PUSH_MERGED_16BIT = True` junto con `PUSH_GGUF_TO_HUB = True` en modelos grandes puede agotar el espacio en disco temporal (`/content` o `/kaggle/working`) durante la fusión.
   * Para GPUs T4 o P100 (arquitecturas Turing/Pascal sin soporte nativo `bfloat16`), asegúrate de que el código mantenga `fp16=not bf16_supported` y `bf16=bf16_supported`.
5. **Preservación de calidad en cuantización (Foco Gemma 4 QAT / Modelos Pequeños):**
   * Cuando el usuario busque el máximo rendimiento en `Q4` local (Ollama), recomienda activamente checkpoints **QAT** (como la familia **Gemma 4 QAT**: `unsloth/gemma-4-E2B-it-qat-q4_0-unquantized`, `unsloth/gemma-4-E4B-it-qat-q4_0-unquantized`, `unsloth/gemma-4-12B-it-qat-q4_0-unquantized`) o activar `USE_QAT = True`, explicando que el entrenamiento consciente de cuantización evita la caída de precisión al pasar a 4 bits.

---

## 7. Protocolo de Autorreparación ("Si el Modelo Destino se Desvía")

Si durante la generación o revisión del código dentro del entorno cloud detectas alguna de estas desviaciones, aplica inmediatamente la corrección indicada:

* **Desviación 1: El modelo destino genera `AutoModelForCausalLM` + `BitsAndBytesConfig` verboso cuando el usuario eligió Unsloth.**
  * *Corrección:* Reemplazar inmediatamente por `FastLanguageModel.from_pretrained` y `FastLanguageModel.get_peft_model` de `unsloth`, ya que `AutoModelForCausalLM` estándar no aplica los kernels Triton de Unsloth y consumirá el doble de VRAM.
* **Desviación 2: El modelo destino omite los comentarios explicativos en las variables de configuración o fija hiperparámetros dentro de `SFTTrainer` sin exponerlos arriba.**
  * *Corrección:* Extraer todos los hiperparámetros a la **Celda 2** y añadir el comentario obligatorio debajo de cada uno detallando: *(1) valor recomendado, (2) valores alternativos viables y (3) qué cambia exactamente si se modifican*.
* **Desviación 3: El modelo destino usa `!pip install` causando un `SyntaxError` en Molab/Marimo.**
  * *Corrección:* Sustituir cualquier comando mágico `!pip` por `subprocess.check_call([sys.executable, "-m", "pip", "install", "-q", ...])`.
* **Desviación 4: El script intenta subir el modelo a HuggingFace como público o en la misma celda del entrenamiento sin validar antes ni comprobar si existe `HF_TOKEN`.**
  * *Corrección:* Separar estrictamente la **Celda 6 (Validación de inferencia)** de la **Celda 7 (Autenticación inteligente + Subida con `private=HF_PRIVATE_REPO` + Aviso de cómo cambiar a público)** para garantizar tanto la calidad del modelo como la privacidad de los datos corporativos.

---

## 8. Ejemplo Few-Shot (anexo externo)

El ejemplo completo (entrevista en 2 turnos + notebook Text-to-SQL) vive en `reference/ejemplo-few-shot.md`, en el directorio de esta skill.

**Cuándo leerlo:**
- El dataset tiene un esquema ambiguo y hay que justificar la autodetección de columnas.
- El usuario pide ver un ejemplo de entrevista antes de decidir hiperparámetros.
- Dudas sobre el tono o el nivel de detalle esperado en las recomendaciones.

**Cuándo NO leerlo:** si ya tienes claro el contrato de salida (§2.2) y la plantilla (§4). Es referencia, no requisito.

---

## Eval de aceptación

Prueba manual mínima. Se considera **APROBADA** solo si se cumplen todos los puntos.

**Entrada**
> `base_model: Qwen2.5-7B-Instruct`, `dataset_path: ./data/sft.jsonl`, sin plataforma indicada.

**Salida esperada**
- Entrevista diagnóstica breve antes de generar el notebook.
- **Un único** notebook autocontenido, con celdas `# %%` y ejecutable en Colab y Kaggle.
- Hiperparámetros explicados inline y justificados por eficiencia de Pareto (VRAM/tiempo).
- Autodiagnóstico del dataset y validación de inferencia al final.
- Subida a HuggingFace Hub **privada por defecto**, con GGUF `Q4_K_M`/`Q5_K_M` para Ollama y *merged* 16-bit para vLLM.

**Prohibido en esta prueba**
- Inventar requisitos de VRAM o versiones de librerías sin justificación.
- Subir el repositorio como público por defecto.
- Fragmentar el entregable en varios notebooks.
