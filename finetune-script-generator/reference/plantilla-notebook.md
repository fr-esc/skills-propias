# Anexo 4 — Plantilla interna del notebook a generar

> Anexo de la skill `finetune-script-generator`. Se consulta solo cuando hay que escribir el notebook.
> Sustituye los placeholders (`{{BASE_MODEL}}`, `{{DATASET_PATH}}`, `{{HF_REPO_ID}}`, `{{CLOUD_PLATFORM}}`, `{{ENGINE}}`) por los valores acordados en la entrevista.

## 4. Plantilla Interna del Notebook a Generar

El modelo destino debe usar la siguiente arquitectura base para construir el notebook, rellenando los placeholders (`{{BASE_MODEL}}`, `{{DATASET_PATH}}`, `{{HF_REPO_ID}}`, etc.) con los valores acordados con el usuario:

```python
# %% [markdown]
# # 🚀 Pipeline Óptimo de Fine-Tuning en Cloud ({{CLOUD_PLATFORM}})
# - **Modelo Base:** `{{BASE_MODEL}}`
# - **Dataset:** `{{DATASET_PATH}}`
# - **Motor:** `{{ENGINE}}` (Optimizado por relación Calidad / VRAM)
# - **Destino de Exportación:** Ollama (GGUF inteligente) + vLLM / HuggingFace Hub (`{{HF_REPO_ID}}` - Privado por defecto)

# %%
# ==============================================================================
# CELDA 1: VERIFICACIÓN DE GPU Y ENTORNO CLOUD + INSTALACIÓN DE DEPENDENCIAS
# ==============================================================================
import os
import sys
import subprocess
import torch

assert torch.cuda.is_available(), (
    "❌ ERROR CRÍTICO: No se detectó ninguna GPU activa en este entorno cloud. "
    "En Google Colab ve a 'Entorno de ejecución -> Cambiar tipo de entorno de ejecución -> GPU (T4/L4)'. "
    "En Kaggle activa 'Accelerator -> GPU T4 x2 / P100' en el panel derecho."
)

gpu_stats = torch.cuda.get_device_properties(0)
max_vram_gb = round(gpu_stats.total_memory / 1024 / 1024 / 1024, 2)
bf16_supported = torch.cuda.is_bf16_supported()
print(f"✅ GPU Detectada: {gpu_stats.name} | VRAM Total: {max_vram_gb} GB | Soporte BF16: {bf16_supported}")

# Instalación limpia en Python puro (100% compatible con Google Colab, Kaggle y Molab/Marimo)
print("⏳ Instalando y verificando motor Unsloth, TRL y dependencias...")
subprocess.check_call([
    sys.executable, "-m", "pip", "install", "-q", "--upgrade",
    "unsloth", "trl>=0.12.0", "peft>=0.13.0", "accelerate>=1.0.0",
    "bitsandbytes>=0.44.0", "torchao>=0.10.0", "datasets>=3.0.0", "huggingface_hub>=0.26.0"
])
print("✅ Entorno preparado correctamente.")

# %%
# ==============================================================================
# CELDA 2: PANEL CENTRAL DE CONFIGURACIÓN E HIPERPARÁMETROS (PARETO ÓPTIMO)
# ==============================================================================
# Modifica únicamente este bloque para ajustar el comportamiento del entrenamiento.
# Cada parámetro incluye su valor recomendado y qué ocurre si decides cambiarlo.

BASE_MODEL = "{{BASE_MODEL}}"
# -> Modelo recomendado para tu tarea.
# -> Alternativas: "unsloth/gemma-4-E4B-it-qat-q4_0-unquantized" (óptimo para Q4 sin perder calidad),
#    "Qwen/Qwen2.5-Coder-7B-Instruct" (código/SQL/JSON), "meta-llama/Llama-3.2-3B-Instruct" (ultraligero).

DATASET_PATH = {{DATASET_PATH}}
# -> Acepta un ID de HuggingFace Hub ("usuario/dataset"), una URL directa ("https://.../datos.jsonl"),
#    un archivo subido al entorno cloud ("/content/mis_datos.jsonl") o una lista de archivos ["part1.jsonl", "part2.jsonl"].

HF_REPO_ID = "{{HF_REPO_ID}}"
# -> Nombre del repositorio en HuggingFace Hub donde se publicará el modelo validado.

HF_PRIVATE_REPO = True
# -> Recomendado: True (crea los repositorios en modo PRIVADO en HuggingFace Hub para proteger datos/modelos de empresa).
# -> Alternativa: False (publica el modelo abierto a cualquier usuario de internet; también puedes cambiarlo
#    manualmente más tarde desde la web de HuggingFace en 'Settings -> Make public').

# --- Hiperparámetros de Memoria y Arquitectura ---
MAX_SEQ_LENGTH = {{MAX_SEQ_LENGTH}}
# -> Recomendado: 2048.
# -> Alternativas: 1024 (ahorra ~35% de VRAM si tus prompts son cortos) | 4096 u 8192 (para contextos largos;
#    consume más VRAM por batch y reduce ligeramente la velocidad de entrenamiento).

LOAD_IN_4BIT = True
# -> Recomendado: True (QLoRA 4-bit). Reduce el uso de VRAM en un 70% con pérdida de precisión casi nula.
# -> Alternativa: False (LoRA 16-bit). Nota: Si activas USE_QAT = True abajo, el script forzará automáticamente
#    16-bit durante la carga porque TorchAO QAT necesita pesos sin cuantizar para insertar los nodos QAT.

USE_QAT = {{USE_QAT}}
# -> Recomendado: False si ya usas un checkpoint nativo "-qat" (como Gemma 3/4 QAT) o haces QLoRA estándar en T4.
# -> Alternativa: True si quieres aplicar Quantization-Aware Training (TorchAO QAT int4) en un modelo normal
#    para que no pierda calidad al pasarlo a Q4 (requiere más VRAM porque desactiva LOAD_IN_4BIT).

TRAIN_ON_RESPONSES_ONLY = True
# -> Recomendado: True (enmascara los prompts del usuario para que el modelo solo calcule el error sobre las
#    respuestas del asistente; aumenta un 10-20% la precisión en modelos pequeños sin gastar nada de VRAM extra).
# -> Alternativa: False (entrena sobre preguntas y respuestas; úsalo solo si tu dataset es texto plano sin formato chat).

# --- Hiperparámetros de Adaptadores LoRA ---
LORA_R = {{LORA_R}}
# -> Recomendado: 16 (punto óptimo calidad/cómputo para tareas de dominio y formato).
# -> Alternativas: 8 (ideal si tu dataset tiene menos de 500 ejemplos para evitar que memorice literal) |
#    32 o 64 (úsalo si necesitas inyectar mucho vocabulario técnico nuevo; aumenta el uso de VRAM y peso del adaptador).

LORA_ALPHA = {{LORA_ALPHA}}
# -> Recomendado: 16 (igual a LORA_R) o 32 (el doble de LORA_R).
# -> Efecto: Controla la fuerza con la que los nuevos pesos sobreescriben al modelo base. Subirlo a 2*LORA_R
#    acelera el aprendizaje de estilos nuevos, pero puede causar inestabilidad si el LEARNING_RATE es alto.

LORA_DROPOUT = 0.0
# -> Recomendado: 0.0 (obligatorio en 0.0 para activar los kernels rápidos de Triton en Unsloth).
# -> Alternativa: 0.05 (solo si desactivas Unsloth y notas sobreajuste severo en más de 5 épocas).

# --- Hiperparámetros del Optimizador y Entrenamiento ---
PER_DEVICE_BATCH_SIZE = {{BATCH_SIZE}}
# -> Recomendado: 2 para GPUs de 15-16 GB (Colab T4 / Kaggle P100).
# -> Alternativas: 1 (si sufres error 'CUDA Out of Memory') | 4 u 8 (si usas GPU L4 24GB o A100 40GB; acelera cada época).

GRADIENT_ACCUMULATION_STEPS = {{GRAD_ACCUM}}
# -> Recomendado: 4 (para un Batch Efectivo = 2 * 4 = 8).
# -> Alternativas: 8 o 16 (simula un batch más grande sin gastar VRAM extra; estabiliza la curva de loss
#    en datasets ruidosos a cambio de tardar un poco más por paso).

NUM_TRAIN_EPOCHS = {{EPOCHS}}
# -> Recomendado: 3 épocas para datasets de entre 300 y 5.000 ejemplos.
# -> Alternativas: 1 época (si tu dataset supera las 15.000 filas, más épocas causarán overfitting) |
#    5 épocas (solo si tu dataset es muy pequeño, de 100 a 250 ejemplos de alta calidad).

LEARNING_RATE = {{LEARNING_RATE}}
# -> Recomendado: 2e-4 (estándar óptimo para QLoRA SFT).
# -> Alternativas: 5e-5 o 1e-4 (más seguro si usas LORA_R >= 32, QAT o modelos >12B para no destruir capacidades previas).

SEED = 3407

# --- Configuración de Exportación Óptima (Ollama / vLLM / HF Hub) ---
GGUF_QUANT_METHOD = "{{GGUF_QUANT}}"
# -> "auto" elige "q4_k_m" para modelos >= 4B o modelos QAT, y "q5_k_m" para modelos pequeños <= 3B (para no perder coherencia).
# -> Opciones manuales: "q4_k_m" (el más ligero y rápido para Ollama), "q5_k_m" (equilibrio extra de calidad), "q8_0" (máxima fidelidad local).

PUSH_GGUF_TO_HUB = True       # Sube el archivo .gguf listo para 'ollama run hf.co/...'
PUSH_MERGED_16BIT = {{PUSH_MERGED_16BIT}}   # Sube safetensors en 16-bit listo para desplegar en servidores vLLM
PUSH_LORA_ADAPTERS = True     # Sube los adaptadores LoRA ligeros (100-300 MB) como respaldo

# %%
# ==============================================================================
# CELDA 3: CARGA EFICIENTE DEL MODELO BASE Y CONFIGURACIÓN LORA / QAT
# ==============================================================================
from unsloth import FastLanguageModel

# Resolución automática de conflicto entre QAT (requiere 16-bit base) y QLoRA 4-bit
effective_load_in_4bit = False if USE_QAT else LOAD_IN_4BIT
if USE_QAT and LOAD_IN_4BIT:
    print("ℹ️ Aviso técnico: Se ha desactivado automáticamente LOAD_IN_4BIT porque USE_QAT=True requiere cargar los pesos en 16-bit para insertar nodos QAT.")

print(f"⏳ Cargando modelo base '{BASE_MODEL}' (4-bit={effective_load_in_4bit})...")
model, tokenizer = FastLanguageModel.from_pretrained(
    model_name=BASE_MODEL,
    max_seq_length=MAX_SEQ_LENGTH,
    dtype=None,               # Autodetecta bfloat16 (Ampere+) o float16 (T4/P100)
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
    use_gradient_checkpointing="unsloth", # Reduce un 30% extra de VRAM y permite contextos más largos
    random_state=SEED,
)

if USE_QAT:
    print("🔥 Activando Quantization-Aware Training (QAT int4) para preservar calidad máxima al cuantizar...")
    peft_kwargs["qat_scheme"] = "int4"

model = FastLanguageModel.get_peft_model(model, **peft_kwargs)

# %%
# ==============================================================================
# CELDA 4: AUTODETECCIÓN, DIAGNÓSTICO Y FORMATEO DEL DATASET
# ==============================================================================
from datasets import load_dataset
from unsloth.chat_templates import standardize_sharegpt

def load_and_detect_dataset(path_or_url):
    """Detecta si la entrada es un str o list[str] de archivos/URLs, o un repo de HuggingFace Hub."""
    ext_map = {
        ".jsonl": "json",
        ".json": "json",
        ".csv": "csv",
        ".parquet": "parquet",
        ".txt": "text",
    }
    sample_path = path_or_url[0] if isinstance(path_or_url, (list, tuple)) else str(path_or_url)
    lower_sample = sample_path.lower()

    for ext, loader_type in ext_map.items():
        if lower_sample.endswith(ext) or f"{ext}?" in lower_sample:
            print(f"📂 Detectado archivo(s)/URL de tipo '{loader_type}': {path_or_url}")
            return load_dataset(loader_type, data_files=path_or_url, split="train")

    if isinstance(path_or_url, str) and os.path.exists(path_or_url):
        raise ValueError(
            f"❌ El archivo '{path_or_url}' existe pero su extensión no es estándar ({list(ext_map.keys())})."
        )

    print(f"🌐 Descargando dataset desde HuggingFace Hub: {path_or_url}")
    return load_dataset(path_or_url, split="train")

raw_dataset = load_and_detect_dataset(DATASET_PATH)
cols = raw_dataset.column_names
print(f"🔍 Columnas detectadas ({len(raw_dataset)} filas): {cols}")
print("📋 Muestra de la primera fila original:", raw_dataset[0])

# Asegurar que el tokenizador tenga un chat_template válido
if tokenizer.chat_template is None:
    from unsloth.chat_templates import get_chat_template
    tokenizer = get_chat_template(tokenizer, chat_template="chatml")

is_conversational_dataset = True

# Detección automática del esquema de columnas y conversión a texto con apply_chat_template
if "messages" in cols or "conversations" in cols:
    # Caso 1: Formato conversacional (ChatML o ShareGPT)
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

elif "instruction" in cols and ("output" in cols or "response" in cols):
    # Caso 2: Formato Instruct / Alpaca (instruction + input opcional + output/response)
    out_col = "output" if "output" in cols else "response"

    def format_alpaca_to_chat(batch):
        texts = []
        inputs = batch.get("input", [""] * len(batch["instruction"]))
        for instr, inp, out in zip(batch["instruction"], inputs, batch[out_col]):
            user_content = f"{instr}\n\nContexto:\n{inp}" if inp and str(inp).strip() else str(instr)
            messages = [
                {"role": "user", "content": user_content},
                {"role": "assistant", "content": str(out)},
            ]
            texts.append(
                tokenizer.apply_chat_template(messages, tokenize=False, add_generation_prompt=False)
            )
        return {"text": texts}

    train_dataset = raw_dataset.map(format_alpaca_to_chat, batched=True)

elif "prompt" in cols and ("completion" in cols or "response" in cols):
    # Caso 3: Formato Prompt-Completion directo
    comp_col = "completion" if "completion" in cols else "response"

    def format_prompt_completion(batch):
        texts = []
        for p, c in zip(batch["prompt"], batch[comp_col]):
            messages = [
                {"role": "user", "content": str(p)},
                {"role": "assistant", "content": str(c)},
            ]
            texts.append(
                tokenizer.apply_chat_template(messages, tokenize=False, add_generation_prompt=False)
            )
        return {"text": texts}

    train_dataset = raw_dataset.map(format_prompt_completion, batched=True)

elif "text" in cols:
    # Caso 4: Ya viene pre-formateado en una columna 'text'
    train_dataset = raw_dataset
    is_conversational_dataset = False
else:
    raise ValueError(
        f"❌ Esquema de columnas no reconocido automáticamente: {cols}. "
        "Por favor, renombra tus columnas a ('instruction', 'output') o ('messages')."
    )

print("\n✅ Ejemplo ya formateado para entrenamiento:\n" + "-" * 60)
print(train_dataset[0]["text"][:600])
print("-" * 60)

# %%
# ==============================================================================
# CELDA 5: ENTRENAMIENTO SUPERVISADO (SFT) CON MONITOREO DE VRAM
# ==============================================================================
from trl import SFTTrainer, SFTConfig

trainer = SFTTrainer(
    model=model,
    tokenizer=tokenizer,
    train_dataset=train_dataset,
    dataset_text_field="text",
    max_seq_length=MAX_SEQ_LENGTH,
    dataset_num_proc=2,
    packing=False, # Pon True si tienes miles de ejemplos muy cortos para multiplicar velocidad x3
    args=SFTConfig(
        per_device_train_batch_size=PER_DEVICE_BATCH_SIZE,
        gradient_accumulation_steps=GRADIENT_ACCUMULATION_STEPS,
        warmup_ratio=0.05,
        num_train_epochs=NUM_TRAIN_EPOCHS,
        learning_rate=LEARNING_RATE,
        fp16=not bf16_supported,
        bf16=bf16_supported,
        logging_steps=5,
        optim="adamw_8bit", # Optimizador en 8 bits: ahorra gigabytes de VRAM sin perder precisión
        weight_decay=0.01,
        lr_scheduler_type="cosine",
        seed=SEED,
        output_dir="outputs_finetune",
        report_to="none",   # Cambia a "wandb" o "tensorboard" si deseas telemetría externa
    ),
)

# Enmascaramiento automático de prompts para entrenar solo sobre las respuestas del asistente
if TRAIN_ON_RESPONSES_ONLY and is_conversational_dataset:
    try:
        from unsloth.chat_templates import train_on_responses_only
        trainer = train_on_responses_only(trainer)
        print("🎯 Activado 'train_on_responses_only': el modelo concentrará el 100% de su aprendizaje en las respuestas.")
    except Exception as e:
        print(f"ℹ️ Entrenamiento estándar sobre secuencia completa (aviso train_on_responses_only: {e})")

start_gpu_memory = round(torch.cuda.max_memory_reserved() / 1024 / 1024 / 1024, 3)
print(f"🚀 Iniciando entrenamiento... (VRAM reservada al inicio: {start_gpu_memory} GB)")
trainer_stats = trainer.train()

used_memory = round(torch.cuda.max_memory_reserved() / 1024 / 1024 / 1024, 3)
print(f"✅ Entrenamiento completado en {round(trainer_stats.metrics['train_runtime'] / 60, 2)} minutos.")
print(f"📊 Pico máximo de VRAM utilizada: {used_memory} GB de {max_vram_gb} GB ({round(used_memory/max_vram_gb*100, 1)}%).")

# %%
# ==============================================================================
# CELDA 6: VALIDACIÓN INTERACTIVA DE CALIDAD ANTES DE SUBIR A HUGGINGFACE
# ==============================================================================
# Si activaste QAT con TorchAO, primero convertimos los nodos fake-quantize a inferencia real
if USE_QAT:
    try:
        from torchao.quantization import quantize_
        from torchao.quantization.qat import QATConfig
        quantize_(model, QATConfig(step="convert"))
        print("✅ Modelo QAT convertido exitosamente para inferencia.")
    except Exception as e:
        print(f"⚠️ Aviso al convertir QAT: {e}")

FastLanguageModel.for_inference(model)

TEST_PROMPT = "{{TEST_PROMPT}}"
messages = [{"role": "user", "content": TEST_PROMPT}]
inputs = tokenizer.apply_chat_template(
    messages,
    tokenize=True,
    add_generation_prompt=True,
    return_tensors="pt",
).to("cuda")

outputs = model.generate(
    input_ids=inputs,
    max_new_tokens=512,
    temperature=0.3,
    top_p=0.9,
    use_cache=True,
)
respuesta_generada = tokenizer.batch_decode(outputs[:, inputs.shape[1]:], skip_special_tokens=True)[0]

print("🧪 PRUEBA DE VALIDACIÓN POST-ENTRENAMIENTO")
print("=" * 60)
print(f"Prompt de prueba:\n{TEST_PROMPT}\n")
print(f"Respuesta del modelo afinado:\n{respuesta_generada}")
print("=" * 60)
print("👉 Revisa junto con el agente si la respuesta cumple el objetivo antes de ejecutar la Celda 7.")

# %%
# ==============================================================================
# CELDA 7: VERIFICACIÓN DE CREDENCIALES Y SUBIDA ÓPTIMA A HUGGINGFACE HUB
# (Formatos listos para Ollama / llama.cpp, vLLM y consumo en HF)
# ==============================================================================
import re
import getpass
from huggingface_hub import login

def get_hf_token() -> str:
    """Busca el token HF_TOKEN en secretos de Colab, Kaggle o entorno; si no existe, lo solicita."""
    # 1. Variable de entorno (Molab / Marimo / Entornos configurados)
    token = os.environ.get("HF_TOKEN") or os.environ.get("HUGGINGFACE_TOKEN")
    if token:
        print("🔑 Token de HuggingFace detectado en variables de entorno.")
        return token

    # 2. Secretos de Google Colab
    try:
        from google.colab import userdata
        token = userdata.get("HF_TOKEN")
        if token:
            print("🔑 Token de HuggingFace recuperado desde Google Colab Secrets ('HF_TOKEN').")
            return token
    except Exception:
        pass

    # 3. Secretos de Kaggle
    try:
        from kaggle_secrets import UserSecretsClient
        token = UserSecretsClient().get_secret("HF_TOKEN")
        if token:
            print("🔑 Token de HuggingFace recuperado desde Kaggle Secrets ('HF_TOKEN').")
            return token
    except Exception:
        pass

    # 4. Solicitud interactiva la primera vez
    print("ℹ️ No se encontró 'HF_TOKEN' en los secretos del notebook.")
    print("Por favor, introduce tu token de HuggingFace con permisos de ESCRITURA (Write):")
    token = getpass.getpass("HF_TOKEN: ").strip()
    if not token:
        raise ValueError("❌ No se proporcionó un token válido. Subida cancelada.")
    os.environ["HF_TOKEN"] = token
    return token

def resolve_optimal_gguf_quant(model_name: str, requested_quant: str, is_qat: bool) -> str:
    """
    Aplica la regla de Pareto para elegir la cuantización GGUF ideal:
    - Si es QAT (ej. Gemma 4 QAT) o >= 4B parámetros: 'q4_k_m' (máximo ahorro de VRAM sin pérdida).
    - Si es un modelo diminuto (<= 3B parámetros) y NO es QAT: 'q5_k_m' para evitar degradación severa.
    """
    if requested_quant != "auto":
        return requested_quant

    lower_name = model_name.lower()
    if is_qat or "-qat" in lower_name:
        print("🎯 Modelo QAT detectado: seleccionando 'q4_k_m' (preparado para 4 bits sin pérdida de calidad).")
        return "q4_k_m"

    size_match = re.search(r"(\d+(?:\.\d+)?)b", lower_name)
    if size_match:
        param_b = float(size_match.group(1))
        if param_b <= 3.5:
            print(f"🎯 Modelo compacto ({param_b}B) no-QAT detectado: seleccionando 'q5_k_m' para proteger el razonamiento.")
            return "q5_k_m"

    print("🎯 Seleccionando 'q4_k_m' como punto óptimo universal calidad/peso para Ollama.")
    return "q4_k_m"

hf_token = get_hf_token()
login(token=hf_token, add_to_git_credential=False)
visibility_label = "PRIVADO 🔒" if HF_PRIVATE_REPO else "PÚBLICO 🌍"

# 1. Subida de Adaptadores LoRA ligeros (respaldo instantáneo y consumo PEFT en HF)
if PUSH_LORA_ADAPTERS:
    print(f"☁️ [1/3] Subiendo adaptadores LoRA ({visibility_label}) a '{HF_REPO_ID}'...")
    model.push_to_hub(HF_REPO_ID, token=hf_token, private=HF_PRIVATE_REPO)
    tokenizer.push_to_hub(HF_REPO_ID, token=hf_token, private=HF_PRIVATE_REPO)

# 2. Subida en formato Merged 16-bit (óptimo para motores de producción como vLLM)
if PUSH_MERGED_16BIT:
    print(f"☁️ [2/3] Fusionando y subiendo pesos en 16-bit para vLLM ({visibility_label}) a '{HF_REPO_ID}-merged-16bit'...")
    model.push_to_hub_merged(
        f"{HF_REPO_ID}-merged-16bit",
        tokenizer,
        save_method="merged_16bit",
        token=hf_token,
        private=HF_PRIVATE_REPO,
    )

# 3. Conversión y subida en formato GGUF (óptimo para ejecución local en Ollama y llama.cpp)
if PUSH_GGUF_TO_HUB:
    final_quant = resolve_optimal_gguf_quant(BASE_MODEL, GGUF_QUANT_METHOD, USE_QAT)
    print(f"☁️ [3/3] Cuantizando a GGUF ({final_quant}) y subiendo ({visibility_label}) a '{HF_REPO_ID}-GGUF'...")
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
        f"   Tu modelo se ha subido como PRIVADO en HuggingFace Hub (no es visible para nadie más).\n"
        f"   👉 Si deseas hacerlo PÚBLICO para compartirlo o descargarlo en Ollama sin autenticar,\n"
        f"      entra en [https://huggingface.co/](https://huggingface.co/){HF_REPO_ID}-GGUF/settings y pulsa en 'Make public'\n"
        f"      (o cambia HF_PRIVATE_REPO = False en la Celda 2)."
    )
print(f"\n👉 Para ejecutar tu modelo en local con Ollama:\n   ollama run hf.co/{HF_REPO_ID}-GGUF")
if PUSH_MERGED_16BIT:
    print(f"👉 Para servir tu modelo en producción con vLLM:\n   vllm serve {HF_REPO_ID}-merged-16bit")
```

### 4.1. Bloque Intercambiable para Alineamiento de Preferencias (`ORPOTrainer`)
Si durante la entrevista el usuario indica que su dataset contiene ejemplos de respuestas preferidas y rechazadas (`prompt`, `chosen`, `rejected`) y elige **ORPO**, el modelo destino debe sustituir el entrenador de la **Celda 5** por esta receta exacta (fijando `LEARNING_RATE = 8e-6` en la Celda 2):

```python
from trl import ORPOConfig, ORPOTrainer
from unsloth import PatchDPOTrainer

PatchDPOTrainer()

trainer = ORPOTrainer(
    model=model,
    train_dataset=raw_dataset, # Debe contener las columnas: 'prompt', 'chosen', 'rejected'
    tokenizer=tokenizer,
    args=ORPOConfig(
        max_length=MAX_SEQ_LENGTH,
        max_prompt_length=MAX_SEQ_LENGTH // 2,
        per_device_train_batch_size=PER_DEVICE_BATCH_SIZE,
        gradient_accumulation_steps=GRADIENT_ACCUMULATION_STEPS,
        beta=0.1,
        learning_rate=LEARNING_RATE, # Recomendado en ORPO: 8e-6
        fp16=not bf16_supported,
        bf16=bf16_supported,
        logging_steps=5,
        optim="adamw_8bit",
        lr_scheduler_type="linear",
        num_train_epochs=NUM_TRAIN_EPOCHS,
        output_dir="outputs_orpo",
        report_to="none",
        seed=SEED,
    ),
)
```

### 4.2. Regla de Adaptación para Molab / Marimo (`cloud_platform == "molab"`)
Si durante la entrevista el usuario elige **Molab / Marimo** en lugar de Google Colab o Kaggle, el modelo destino debe transformar las celdas `# %%` al formato de grafo reactivo de Marimo siguiendo estas tres reglas estrictas:
1. Encabezar el archivo con `import marimo as mo` y `app = mo.App(width="medium")`.
2. Envolver cada celda en `@app.cell` definiendo una función `def _(dependencias): ... return (variables_exportadas,)` (y gracias al uso de `subprocess.check_call` en la Celda 1 no habrá errores de sintaxis por `!pip`).
3. **Prohibido redefinir el mismo nombre de variable en dos celdas distintas** (por ejemplo, en la celda de carga usar `model_base`, en la de LoRA `model_peft` y en el entrenador no reasignar `trainer`).
