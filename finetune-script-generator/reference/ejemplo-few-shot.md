# Anexo 8 — Ejemplo few-shot completo

> Anexo de la skill `finetune-script-generator`. Referencia de tono y nivel de detalle, no requisito.

## 8. Ejemplo Few-Shot Completo

A continuación se muestra un ejemplo íntegro de cómo debe comportarse el modelo destino al recibir una invocación con `base_model="Qwen/Qwen2.5-Coder-7B-Instruct"` y `dataset_path="mis-datos.jsonl"`.

### 8.1. Entrada del Usuario
> Quiero hacer fine-tuning con `base_model="Qwen/Qwen2.5-Coder-7B-Instruct"` y `dataset_path="mis-datos.jsonl"`.

### 8.2. Respuesta del Agente — Turno 1 (Entrevista Guiada con Recomendaciones)
> Para generarte el notebook único optimizado al máximo en relación calidad/VRAM para `Qwen/Qwen2.5-Coder-7B-Instruct` y `mis-datos.jsonl`, confírmame rápidamente estos **4 puntos** (puedes responder "todo por defecto" si te encajan mis recomendaciones):
>
> 1. **Objetivo del fine-tuning y validación de modelo:**
>    `Qwen/Qwen2.5-Coder-7B-Instruct` es excelente si tu objetivo es generar código, consultas SQL, extracción JSON estricta o llamadas a herramientas. *(Nota: si tu prioridad máxima fuera ejecutar un modelo multimodal/generalista en Q4 ocupando el mínimo de VRAM sin perder nada de precisión, te recomendaría valorar también `unsloth/gemma-4-E4B-it-qat-q4_0-unquantized`, que ya viene entrenado con Quantization-Aware Training para 4 bits).* ¿Mantenemos `Qwen2.5-Coder-7B-Instruct` y qué tarea concreta quieres que aprenda de `mis-datos.jsonl`?
> 2. **Entorno Cloud donde lo vas a ejecutar:**
>    * **[Predeterminado]** **Google Colab** (GPU T4 16GB o L4)
>    * **Kaggle Notebooks** (GPU T4x2 / P100)
>    * **Molab / Marimo** (Celdas reactivas `@app.cell`)
> 3. **Motor y Método de Entrenamiento:**
>    * **[Predeterminado]** **Unsloth + QLoRA 4-bit (SFT)** con `train_on_responses_only` (consume solo ~6.5 GB de VRAM para este 7B y entrena el doble de rápido).
>    * **Unsloth + QAT (Quantization-Aware Training)**: añade simulación de 4 bits durante el entrenamiento para mejorar aún más la precisión al exportarlo a `Q4`.
>    * **Unsloth + ORPO** (si tu `.jsonl` tiene pares de respuestas elegidas/rechazadas) o **HuggingFace TRL estándar**.
> 4. **Repositorio destino en HuggingFace Hub (Privado por defecto) y consumo posterior:**
>    Tras validar juntos con una prueba de inferencia que el modelo responde bien, el notebook comprobará tus credenciales `HF_TOKEN` (o te las pedirá la primera vez) y exportará el modelo en modo **Privado** en formato **`GGUF q4_k_m`** (óptimo para **Ollama** en local) y/o **`merged_16bit`** (para **vLLM**). ¿Qué nombre de repositorio quieres dejar configurado (ej. `tu-usuario/qwen2.5-coder-7b-empresa`)?

### 8.3. Confirmación del Usuario
> Todo por defecto en Google Colab con Unsloth. El objetivo es que convierta peticiones en lenguaje natural a consultas SQL internas de nuestra empresa. El repo será `mi-empresa/qwen-sql-7b`.

### 8.4. Respuesta del Agente — Turno 2 (Notebook Estructurado Completo Listo para Ejecutar)

```python
# %% [markdown]
# # 🚀 Fine-Tuning Óptimo Text-to-SQL en Google Colab
# - **Modelo Base:** `Qwen/Qwen2.5-Coder-7B-Instruct`
# - **Dataset:** `mis-datos.jsonl` (Autodetección de esquema conversacional / instruct)
# - **Motor:** `Unsloth + TRL (QLoRA 4-bit + train_on_responses_only)` — Máximo rendimiento con ~6.5 GB de VRAM
# - **Exportación:** `mi-empresa/qwen-sql-7b` (Privado por defecto 🔒 | GGUF `q4_k_m` para Ollama + `merged_16bit` opcional para vLLM)

# %%
# ==============================================================================
# CELDA 1: VERIFICACIÓN DE GPU EN GOOGLE COLAB E INSTALACIÓN DE DEPENDENCIAS
# ==============================================================================
import os
import sys
import subprocess
import torch

assert torch.cuda.is_available(), (
    "❌ ERROR CRÍTICO: No hay GPU activa en Google Colab. "
    "Ve al menú 'Entorno de ejecución -> Cambiar tipo de entorno de ejecución' y selecciona 'T4 GPU' o 'L4 GPU'."
)

gpu_stats = torch.cuda.get_device_properties(0)
max_vram_gb = round(gpu_stats.total_memory / 1024 / 1024 / 1024, 2)
bf16_supported = torch.cuda.is_bf16_supported()
print(f"✅ GPU Detectada: {gpu_stats.name} | VRAM Total: {max_vram_gb} GB | Soporte BF16: {bf16_supported}")

print("⏳ Instalando y verificando motor Unsloth, TRL y dependencias...")
subprocess.check_call([
    sys.executable, "-m", "pip", "install", "-q", "--upgrade",
    "unsloth", "trl>=0.12.0", "peft>=0.13.0", "accelerate>=1.0.0",
    "bitsandbytes>=0.44.0", "torchao>=0.10.0", "datasets>=3.0.0", "huggingface_hub>=0.26.0"
])
print("✅ Entorno preparado correctamente.")

# %%
# ==============================================================================
# CELDA 2: PANEL CENTRAL DE CONFIGURACIÓN E HIPERPARÁMETROS (EXPLICADOS)
# ==============================================================================
# Todas las variables ajustables están centralizadas aquí con su explicación de impacto.

BASE_MODEL = "Qwen/Qwen2.5-Coder-7B-Instruct"
# -> Modelo recomendado: Especialista líder en generación de código y SQL en tamaño 7B.
# -> Alternativas: "unsloth/gemma-4-E4B-it-qat-q4_0-unquantized" (si prefieres un modelo de 4B con QAT nativo
#    que consume solo 3.5 GB de VRAM en local) o "Qwen/Qwen2.5-Coder-3B-Instruct" (para máquinas sin GPU dedicada).

DATASET_PATH = "mis-datos.jsonl"
# -> Ruta al archivo subido al panel lateral de archivos de Google Colab (/content/mis-datos.jsonl),
#    lista de archivos ["datos1.jsonl", "datos2.jsonl"], URL directa (https://...) o repo en HuggingFace Hub.

HF_REPO_ID = "mi-empresa/qwen-sql-7b"
# -> Repositorio destino en HuggingFace Hub donde se subirá el modelo una vez validado.

HF_PRIVATE_REPO = True
# -> Recomendado: True (crea los repositorios en modo PRIVADO en HuggingFace Hub para que las tablas y consultas SQL
#    de tu empresa no sean visibles públicamente).
# -> Alternativa: False (lo publica como repositorio público; también puedes pasarlo a público manualmente más tarde
#    entrando en la pestaña 'Settings' de tu modelo en HuggingFace Hub).

# --- Hiperparámetros de Contexto y Precisión ---
MAX_SEQ_LENGTH = 2048
# -> Recomendado: 2048 tokens (cubre de sobra esquemas de tablas DDL + consulta SQL).
# -> Alternativas: 1024 (ahorra un 35% de VRAM si tus consultas son cortas) | 4096 (si inyectas el DDL completo
#    de decenas de tablas en cada prompt; consume más VRAM por paso).

LOAD_IN_4BIT = True
# -> Recomendado: True (activa cuantización QLoRA NF4 para que el modelo de 7B quepa holgadamente en la T4 de 16GB).
# -> Alternativa: False (carga en 16-bit; se fuerza automáticamente a False si activas USE_QAT = True abajo).

USE_QAT = False
# -> Recomendado: False para QLoRA rápido en T4.
# -> Alternativa: True (activa Quantization-Aware Training con TorchAO para simular cuantización int4 durante
#    el entrenamiento; desactiva automáticamente LOAD_IN_4BIT y requiere más VRAM).

TRAIN_ON_RESPONSES_ONLY = True
# -> Recomendado: True (hace que el modelo solo calcule la pérdida sobre la consulta SQL generada y no sobre
#    la pregunta del usuario; mejora notablemente la exactitud sintáctica SQL sin gastar más VRAM).
# -> Alternativa: False (entrena sobre toda la conversación).

# --- Hiperparámetros LoRA ---
LORA_R = 16
# -> Recomendado: 16 (capacidad ideal para aprender el esquema SQL y convenciones de tu empresa).
# -> Alternativas: 8 (si 'mis-datos.jsonl' tiene menos de 300 ejemplos, evita que memorice consultas de memoria) |
#    32 (si tienes más de 3.000 ejemplos con funciones SQL propietarias muy complejas).

LORA_ALPHA = 32
# -> Recomendado: 32 (2 * LORA_R para que adopte con firmeza la sintaxis SQL corporativa).
# -> Alternativa: 16 (igual a LORA_R; aprendizaje más suave y conservador).

LORA_DROPOUT = 0.0
# -> Recomendado: 0.0 (permite a Unsloth activar los parches rápidos de Triton sin sobrecoste de memoria).

# --- Hiperparámetros del Entrenador ---
PER_DEVICE_BATCH_SIZE = 2
# -> Recomendado: 2 en GPU Colab T4 (16GB).
# -> Alternativas: 1 (si usas MAX_SEQ_LENGTH=4096 y te quedas sin VRAM) | 4 (si estás en Colab Pro con GPU L4/A100).

GRADIENT_ACCUMULATION_STEPS = 4
# -> Recomendado: 4 (Batch efectivo = 2 * 4 = 8 ejemplos por actualización de pesos).
# -> Alternativa: 8 (Batch efectivo = 16; hace que el descenso del loss sea más estable si hay ejemplos ruidosos).

NUM_TRAIN_EPOCHS = 3
# -> Recomendado: 3 épocas (óptimo para datasets empresariales de entre 300 y 3.000 pares pregunta-SQL).
# -> Alternativas: 1 época (si tu archivo supera las 10.000 filas) | 4-5 épocas (si tienes menos de 250 ejemplos).

LEARNING_RATE = 2e-4
# -> Recomendado: 2e-4 (velocidad de convergencia óptima para adaptadores QLoRA en Qwen 2.5).
# -> Alternativa: 1e-4 (si subes LORA_R a 32 o ves que el modelo olvida cómo escribir SQL estándar).

SEED = 3407

# --- Configuración de Exportación Post-Validación ---
GGUF_QUANT_METHOD = "auto"
# -> "auto" seleccionará automáticamente "q4_k_m" para este modelo de 7B (ocupa ~4.4 GB en disco/VRAM,
#    listo para 'ollama run' y llama.cpp conservando el 99% de precisión en generación SQL).
# -> Alternativas manuales: "q5_k_m" (~5.2 GB, un plus de seguridad en queries anidadas) o "q8_0" (~7.5 GB).

PUSH_GGUF_TO_HUB = True       # Genera y sube el GGUF cuantizado listo para Ollama
PUSH_MERGED_16BIT = False     # Pon True si además vas a levantar un clúster vLLM en servidor (consume ~15GB de disco temporal)
PUSH_LORA_ADAPTERS = True     # Sube los adaptadores LoRA (~160 MB) a HuggingFace Hub

# %%
# ==============================================================================
# CELDA 3: CARGA DEL MODELO BASE Y APLICACIÓN DE ADAPTADORES LORA
# ==============================================================================
from unsloth import FastLanguageModel

effective_load_in_4bit = False if USE_QAT else LOAD_IN_4BIT
if USE_QAT and LOAD_IN_4BIT:
    print("ℹ️ Aviso técnico: Se ha desactivado LOAD_IN_4BIT automáticamente porque USE_QAT=True requiere cargar pesos en 16-bit.")

print(f"⏳ Descargando y preparando '{BASE_MODEL}' (4-bit={effective_load_in_4bit})...")
model, tokenizer = FastLanguageModel.from_pretrained(
    model_name=BASE_MODEL,
    max_seq_length=MAX_SEQ_LENGTH,
    dtype=None,
    load_in_4bit=effective_load_in_4bit,
)

peft_kwargs = dict(
    r=LORA_R,
    target_modules=[
        "q_proj", "k_proj", "v_proj", "o_proj",
        "gate_proj", "up_proj", "down_proj",
    ],
    lora_alpha=LORA_ALPHA,
    lora_dropout=LORA_DROPOUT,
    bias="none",
    use_gradient_checkpointing="unsloth",
    random_state=SEED,
)

if USE_QAT:
    peft_kwargs["qat_scheme"] = "int4"

model = FastLanguageModel.get_peft_model(model, **peft_kwargs)

# %%
# ==============================================================================
# CELDA 4: AUTODETECCIÓN Y PREPROCESAMIENTO DE 'mis-datos.jsonl'
# ==============================================================================
from datasets import load_dataset
from unsloth.chat_templates import standardize_sharegpt

def load_and_detect_dataset(path_or_url):
    ext_map = {".jsonl": "json", ".json": "json", ".csv": "csv", ".parquet": "parquet", ".txt": "text"}
    sample_path = path_or_url[0] if isinstance(path_or_url, (list, tuple)) else str(path_or_url)
    lower_sample = sample_path.lower()

    for ext, loader_type in ext_map.items():
        if lower_sample.endswith(ext) or f"{ext}?" in lower_sample:
            if isinstance(path_or_url, str) and not path_or_url.startswith(("http://", "https://")) and not os.path.exists(path_or_url):
                raise FileNotFoundError(
                    f"❌ No se encontró el archivo '{path_or_url}' en el entorno de Colab. "
                    "Súbelo arrastrándolo a la carpeta izquierda ('Archivos') de Google Colab."
                )
            print(f"📂 Cargando archivo(s)/URL '{path_or_url}' como tipo '{loader_type}'...")
            return load_dataset(loader_type, data_files=path_or_url, split="train")
    return load_dataset(path_or_url, split="train")

raw_dataset = load_and_detect_dataset(DATASET_PATH)
cols = raw_dataset.column_names
print(f"🔍 Columnas detectadas ({len(raw_dataset)} ejemplos): {cols}")

SYSTEM_SQL_PROMPT = (
    "Eres un experto en bases de datos de la empresa. Convierte la petición del usuario "
    "en una consulta SQL válida, limpia y optimizada sin añadir explicaciones innecesarias."
)

is_conversational_dataset = True

if "messages" in cols or "conversations" in cols:
    if "conversations" in cols:
        raw_dataset = standardize_sharegpt(raw_dataset)
    msg_col = "messages" if "messages" in raw_dataset.column_names else "conversations"

    def format_chat(batch):
        texts = [
            tokenizer.apply_chat_template(conv, tokenize=False, add_generation_prompt=False)
            for conv in batch[msg_col]
        ]
        return {"text": texts}

    train_dataset = raw_dataset.map(format_chat, batched=True)

elif "instruction" in cols and ("output" in cols or "response" in cols or "sql" in cols):
    out_col = "output" if "output" in cols else ("sql" if "sql" in cols else "response")

    def format_instruct_sql(batch):
        texts = []
        inputs = batch.get("input", [""] * len(batch["instruction"]))
        for instr, inp, out in zip(batch["instruction"], inputs, batch[out_col]):
            user_msg = f"{instr}\n\nEsquema / Contexto:\n{inp}" if inp and str(inp).strip() else str(instr)
            messages = [
                {"role": "system", "content": SYSTEM_SQL_PROMPT},
                {"role": "user", "content": user_msg},
                {"role": "assistant", "content": str(out)},
            ]
            texts.append(tokenizer.apply_chat_template(messages, tokenize=False, add_generation_prompt=False))
        return {"text": texts}

    train_dataset = raw_dataset.map(format_instruct_sql, batched=True)

elif "question" in cols and ("sql" in cols or "query" in cols):
    sql_col = "sql" if "sql" in cols else "query"

    def format_qa_sql(batch):
        texts = []
        for q, s in zip(batch["question"], batch[sql_col]):
            messages = [
                {"role": "system", "content": SYSTEM_SQL_PROMPT},
                {"role": "user", "content": str(q)},
                {"role": "assistant", "content": str(s)},
            ]
            texts.append(tokenizer.apply_chat_template(messages, tokenize=False, add_generation_prompt=False))
        return {"text": texts}

    train_dataset = raw_dataset.map(format_qa_sql, batched=True)

elif "text" in cols:
    train_dataset = raw_dataset
    is_conversational_dataset = False
else:
    raise ValueError(f"❌ No se pudo mapear automáticamente las columnas {cols}. Usa ('instruction', 'output') o ('messages').")

print("\n✅ Muestra del primer ejemplo formateado con el Chat Template de Qwen:\n" + "-" * 60)
print(train_dataset[0]["text"][:600])
print("-" * 60)

# %%
# ==============================================================================
# CELDA 5: EJECUCIÓN DEL FINE-TUNING (SFTTrainer + SFTConfig)
# ==============================================================================
from trl import SFTTrainer, SFTConfig

trainer = SFTTrainer(
    model=model,
    tokenizer=tokenizer,
    train_dataset=train_dataset,
    dataset_text_field="text",
    max_seq_length=MAX_SEQ_LENGTH,
    dataset_num_proc=2,
    packing=False,
    args=SFTConfig(
        per_device_train_batch_size=PER_DEVICE_BATCH_SIZE,
        gradient_accumulation_steps=GRADIENT_ACCUMULATION_STEPS,
        warmup_ratio=0.05,
        num_train_epochs=NUM_TRAIN_EPOCHS,
        learning_rate=LEARNING_RATE,
        fp16=not bf16_supported,
        bf16=bf16_supported,
        logging_steps=5,
        optim="adamw_8bit",
        weight_decay=0.01,
        lr_scheduler_type="cosine",
        seed=SEED,
        output_dir="outputs_qwen_sql",
        report_to="none",
    ),
)

if TRAIN_ON_RESPONSES_ONLY and is_conversational_dataset:
    try:
        from unsloth.chat_templates import train_on_responses_only
        trainer = train_on_responses_only(trainer)
        print("🎯 Activado 'train_on_responses_only': entrenando exclusivamente sobre las consultas SQL de salida.")
    except Exception as e:
        print(f"ℹ️ Entrenamiento estándar sobre secuencia completa (aviso: {e})")

print("🚀 Entrenando modelo Text-to-SQL...")
trainer_stats = trainer.train()
used_vram = round(torch.cuda.max_memory_reserved() / 1024 / 1024 / 1024, 2)
print(f"✅ Entrenamiento finalizado en {round(trainer_stats.metrics['train_runtime']/60, 2)} min | Pico VRAM: {used_vram} GB")

# %%
# ==============================================================================
# CELDA 6: VALIDACIÓN CON EL AGENTE ANTES DE SUBIR A HUGGINGFACE HUB
# ==============================================================================
if USE_QAT:
    try:
        from torchao.quantization import quantize_
        from torchao.quantization.qat import QATConfig
        quantize_(model, QATConfig(step="convert"))
    except Exception as e:
        print(f"⚠️ Aviso QAT: {e}")

FastLanguageModel.for_inference(model)

TEST_PROMPT = (
    "Obtén el total de ventas netas por departamento de los últimos 30 días "
    "para clientes activos, ordenando de mayor a menor."
)

messages = [
    {"role": "system", "content": SYSTEM_SQL_PROMPT},
    {"role": "user", "content": TEST_PROMPT},
]
inputs = tokenizer.apply_chat_template(
    messages,
    tokenize=True,
    add_generation_prompt=True,
    return_tensors="pt",
).to("cuda")

outputs = model.generate(
    input_ids=inputs,
    max_new_tokens=256,
    temperature=0.1, # Temperatura baja para máxima precisión sintáctica en SQL
    top_p=0.9,
    use_cache=True,
)
sql_generado = tokenizer.batch_decode(outputs[:, inputs.shape[1]:], skip_special_tokens=True)[0]

print("🧪 VALIDACIÓN DE CALIDAD SQL PRE-SUBIDA")
print("=" * 60)
print(f"Petición:\n{TEST_PROMPT}\n")
print(f"SQL Generado:\n{sql_generado}")
print("=" * 60)
print("👉 Si el SQL utiliza correctamente las tablas de tu empresa, ejecuta la Celda 7 para publicarlo.")

# %%
# ==============================================================================
# CELDA 7: CREDENCIALES DE HUGGINGFACE Y EXPORTACIÓN ÓPTIMA (PRIVADO POR DEFECTO)
# ==============================================================================
import re
import getpass
from huggingface_hub import login

def get_hf_token() -> str:
    token = os.environ.get("HF_TOKEN") or os.environ.get("HUGGINGFACE_TOKEN")
    if token:
        return token
    try:
        from google.colab import userdata
        token = userdata.get("HF_TOKEN")
        if token:
            print("🔑 Token recuperado automáticamente desde Google Colab Secrets ('HF_TOKEN').")
            return token
    except Exception:
        pass
    print("ℹ️ No se detectó el secreto 'HF_TOKEN' en Google Colab.")
    print("Introduce tu token de HuggingFace con permisos 'Write' (solo se pide esta primera vez):")
    token = getpass.getpass("HF_TOKEN: ").strip()
    if not token:
        raise ValueError("❌ Token vacío. Subida abortada.")
    os.environ["HF_TOKEN"] = token
    return token

def resolve_optimal_gguf_quant(model_name: str, requested_quant: str, is_qat: bool) -> str:
    if requested_quant != "auto":
        return requested_quant
    lower_name = model_name.lower()
    if is_qat or "-qat" in lower_name:
        return "q4_k_m"
    size_match = re.search(r"(\d+(?:\.\d+)?)b", lower_name)
    if size_match and float(size_match.group(1)) <= 3.5:
        return "q5_k_m"
    return "q4_k_m"

hf_token = get_hf_token()
login(token=hf_token, add_to_git_credential=False)
visibility_label = "PRIVADO 🔒" if HF_PRIVATE_REPO else "PÚBLICO 🌍"

if PUSH_LORA_ADAPTERS:
    print(f"☁️ Subiendo adaptadores LoRA ({visibility_label}) a '{HF_REPO_ID}'...")
    model.push_to_hub(HF_REPO_ID, token=hf_token, private=HF_PRIVATE_REPO)
    tokenizer.push_to_hub(HF_REPO_ID, token=hf_token, private=HF_PRIVATE_REPO)

if PUSH_MERGED_16BIT:
    print(f"☁️ Subiendo modelo fusionado en 16-bit para vLLM ({visibility_label}) a '{HF_REPO_ID}-merged-16bit'...")
    model.push_to_hub_merged(
        f"{HF_REPO_ID}-merged-16bit",
        tokenizer,
        save_method="merged_16bit",
        token=hf_token,
        private=HF_PRIVATE_REPO,
    )

if PUSH_GGUF_TO_HUB:
    final_quant = resolve_optimal_gguf_quant(BASE_MODEL, GGUF_QUANT_METHOD, USE_QAT)
    print(f"☁️ Exportando y subiendo GGUF ({final_quant}) para Ollama ({visibility_label}) a '{HF_REPO_ID}-GGUF'...")
    model.push_to_hub_gguf(
        f"{HF_REPO_ID}-GGUF",
        tokenizer,
        quantization_method=final_quant,
        token=hf_token,
        private=HF_PRIVATE_REPO,
    )

print("\n🎉 ¡PROCESO COMPLETADO CON ÉXITO!")
if HF_PRIVATE_REPO:
    print(
        "\n🔒 AVISO IMPORTANTE DE PRIVACIDAD:\n"
        f"   Tu modelo ha quedado subido en modo PRIVADO en HuggingFace Hub (no es visible públicamente).\n"
        f"   👉 Si deseas hacerlo PÚBLICO, entra manualmente en:\n"
        f"      [https://huggingface.co/](https://huggingface.co/){HF_REPO_ID}-GGUF/settings\n"
        f"      y en la sección 'Change repository visibility' pulsa 'Make public' (o usa HF_PRIVATE_REPO = False)."
    )
print(f"\n👉 Ejecútalo en local con Ollama usando:\n   ollama run hf.co/{HF_REPO_ID}-GGUF")
```
