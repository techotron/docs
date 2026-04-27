# LLM Learning Path: From Zero to Running & Tweaking Models Locally

**Target hardware:** Ubuntu PC with NVIDIA RTX 5080 (16GB VRAM)
**Goal:** Understand, run, fine-tune, and deploy local LLMs for personal use and agent workflows

---

## Phase 1: Foundations — How Neural Networks and Language Models Work

### Learning Objectives
- Understand what a neural network is and how it learns
- Understand tokenization and how text becomes numbers
- Understand the transformer architecture at a conceptual level
- Know what "parameters," "weights," "layers," and "attention" mean in practice

### Material

| Resource | Type | Notes |
|----------|------|-------|
| [3Blue1Brown — Neural Networks](https://www.youtube.com/playlist?list=PLZHQObOWTQDNU6R1_67000Dx_ZCJB-3pi) | Video series | Best visual introduction to neural networks. Watch all 4 videos. |
| [Andrej Karpathy — Let's build GPT from scratch](https://www.youtube.com/watch?v=kCc8FmEb1nY) | Video (2hr) | Builds a small transformer from scratch in Python. Pause and follow along. |
| [Andrej Karpathy — Let's build the GPT tokenizer](https://www.youtube.com/watch?v=zduSFxRajkE) | Video (2hr) | Demystifies how text becomes tokens and why it matters. |
| [Jay Alammar — The Illustrated Transformer](https://jalammar.github.io/illustrated-transformer/) | Article | Visual walkthrough of the transformer paper. |
| [Jay Alammar — The Illustrated GPT-2](https://jalammar.github.io/illustrated-gpt2/) | Article | How autoregressive generation works in GPT-style models. |
| [Hugging Face NLP Course — Chapter 1](https://huggingface.co/learn/nlp-course/chapter1) | Interactive course | Practical intro with code. Covers pipelines, models, tokenizers. |

### Key Concepts to Lock In
- **Tokens** are the atomic units (subwords) the model operates on, not characters or whole words
- **Parameters/weights** are the numbers the model learned during training — they encode its "knowledge"
- **Attention** is the mechanism that lets the model relate different parts of the input to each other
- **Autoregressive generation** means the model predicts one token at a time, feeding its output back as input
- **Context window** is the maximum number of tokens the model can "see" at once

### Project 1: Watch a tiny model think

**Goal:** See token-by-token generation happen in real time to internalise how LLMs actually produce text.

1. Install Python 3.10+ on your Ubuntu machine
2. `pip install transformers torch`
3. Run this script:

```python
from transformers import AutoTokenizer, AutoModelForCausalLM
import torch

model_name = "gpt2"  # 124M params — will run on CPU easily
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForCausalLM.from_pretrained(model_name)

prompt = "The capital of France is"
input_ids = tokenizer.encode(prompt, return_tensors="pt")

print(f"Prompt tokens: {tokenizer.convert_ids_to_tokens(input_ids[0])}")
print(f"Generating token by token:\n")

for i in range(20):
    with torch.no_grad():
        outputs = model(input_ids)
    next_token_logits = outputs.logits[0, -1, :]
    next_token_id = torch.argmax(next_token_logits).unsqueeze(0).unsqueeze(0)
    next_token = tokenizer.decode(next_token_id[0])
    print(f"  Step {i+1}: predicted '{next_token}' (token id {next_token_id.item()})")
    input_ids = torch.cat([input_ids, next_token_id], dim=1)

print(f"\nFull output: {tokenizer.decode(input_ids[0])}")
```

4. Try different prompts and observe how the model's "confidence" varies
5. Modify the script to print the top-5 candidate tokens at each step (use `torch.topk`)

**What you'll learn:** That generation is a sequential sampling process, that the model assigns probabilities to every possible next token, and that "creativity" is just sampling from those probabilities.

---

## Phase 2: Your GPU, Drivers, and the Local Inference Stack

### Learning Objectives
- Set up NVIDIA drivers and CUDA on Ubuntu for ML workloads
- Understand VRAM as the primary constraint for local LLM work
- Understand model quantization and why it exists
- Know the major formats: GGUF, safetensors, GPTQ, AWQ

### Material

| Resource | Type | Notes |
|----------|------|-------|
| [NVIDIA CUDA Installation Guide — Linux](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/) | Docs | Official guide. Follow it carefully for your Ubuntu version. |
| [Hugging Face — Quantization Overview](https://huggingface.co/docs/transformers/main/en/quantization/overview) | Docs | Explains all quantization methods and when to use each. |
| [TheBloke's Quantization Explanation (Hugging Face)](https://huggingface.co/TheBloke/Llama-2-7B-GGUF) | Model card | Read the model card — it explains quant levels (Q4_K_M, Q5_K_S, etc.) and their tradeoffs. |
| [llama.cpp README](https://github.com/ggerganov/llama.cpp) | GitHub | The foundational project for running quantised models on consumer hardware. |

### Key Concepts to Lock In
- **VRAM (16GB on your 5080)** determines the maximum model size you can load. Rough guide:
  - 7B parameter model at Q4 quantization: ~4-5GB VRAM
  - 13B parameter model at Q4: ~8-9GB VRAM
  - 7B at full FP16 precision: ~14GB VRAM (tight fit)
  - 70B+ models: won't fit without offloading to RAM (slow)
- **Quantization** reduces precision of weights (e.g., from 16-bit to 4-bit) to shrink models with minimal quality loss
- **GGUF** is the format used by llama.cpp — CPU + GPU inference, widely supported
- **safetensors** is the standard format on Hugging Face — used for full-precision or GPU-native inference
- **GPTQ/AWQ** are GPU-optimised quantization formats — faster inference than GGUF on NVIDIA GPUs

### Setup Tasks

1. **Install NVIDIA drivers:**
```bash
sudo apt update
sudo apt install nvidia-driver-560  # or latest available
sudo reboot
nvidia-smi  # verify — should show your RTX 5080 and VRAM
```

2. **Install CUDA toolkit:**
```bash
sudo apt install nvidia-cuda-toolkit
nvcc --version  # verify
```

3. **Install PyTorch with CUDA support:**
```bash
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu124
python -c "import torch; print(torch.cuda.is_available(), torch.cuda.get_device_name(0))"
```

### Project 2: Run your first local model with llama.cpp

**Goal:** Get a quantised model running locally on your GPU and have a conversation with it.

1. Build llama.cpp with CUDA support:
```bash
git clone https://github.com/ggerganov/llama.cpp
cd llama.cpp
cmake -B build -DGGML_CUDA=ON
cmake --build build --config Release -j$(nproc)
```

2. Download a quantised model (start small):
```bash
# Install huggingface-hub CLI
pip install huggingface-hub
# Download Mistral 7B at Q4_K_M quantization (~4.4GB)
huggingface-cli download TheBloke/Mistral-7B-Instruct-v0.2-GGUF \
  mistral-7b-instruct-v0.2.Q4_K_M.gguf --local-dir ./models
```

3. Run interactive chat:
```bash
./build/bin/llama-cli \
  -m ./models/mistral-7b-instruct-v0.2.Q4_K_M.gguf \
  -n 512 \
  -ngl 99 \
  --chat-template mistral \
  -cnv
```

4. Note the speed (tokens/second) printed in the output.

5. Now download the Q8_0 version of the same model and compare:
   - Is the output quality noticeably different?
   - How does the speed change?
   - How much more VRAM does it use? (check with `nvidia-smi`)

**What you'll learn:** The direct relationship between quantization level, VRAM usage, inference speed, and output quality. This intuition will guide every model choice you make going forward.

---

## Phase 3: Inference Servers and the OpenAI-Compatible API

### Learning Objectives
- Run a model as a persistent local server with an API
- Understand the OpenAI-compatible API format (it's the de facto standard)
- Call your local model programmatically from Python
- Understand inference parameters: temperature, top_p, top_k, repetition_penalty

### Material

| Resource | Type | Notes |
|----------|------|-------|
| [Ollama](https://ollama.com/) | Tool | Simplest way to run a local model as a server. One command to start. |
| [Ollama API docs](https://github.com/ollama/ollama/blob/main/docs/api.md) | Docs | The API endpoints available. |
| [llama.cpp server docs](https://github.com/ggerganov/llama.cpp/tree/master/examples/server) | Docs | More control than Ollama, OpenAI-compatible endpoints. |
| [vLLM](https://github.com/vllm-project/vllm) | Tool | High-performance inference engine. Better throughput for serving multiple requests. |
| [Text Generation Inference — Hugging Face](https://github.com/huggingface/text-generation-inference) | Tool | Production-grade server from Hugging Face. |
| [Hugging Face — Generation Parameters](https://huggingface.co/docs/transformers/main/en/generation_strategies) | Docs | Explains temperature, sampling, beam search, etc. |

### Key Concepts to Lock In
- **Temperature** controls randomness: 0 = deterministic (always pick the highest probability token), 1 = sample according to the learned distribution, >1 = more random
- **top_p (nucleus sampling)** limits the token pool to the smallest set whose cumulative probability exceeds p (e.g., 0.9)
- **top_k** limits the token pool to the k most probable tokens
- **Repetition penalty** discourages the model from repeating itself
- **System prompt** sets the model's persona/instructions — it's just text prepended to the conversation in a specific format
- **Chat templates** format conversations into the token structure the model was trained to expect (e.g., `[INST]...[/INST]` for Mistral)

### Project 3: Local API server with parameter experiments

**Goal:** Run a local model as a server, call it from Python, and observe how parameters affect output.

1. Install and start Ollama:
```bash
curl -fsSL https://ollama.com/install.sh | sh
ollama pull mistral:7b-instruct-q4_K_M
ollama serve  # starts on port 11434
```

2. Create a Python script to experiment with parameters:
```python
import requests
import json

def generate(prompt, temperature=0.7, top_p=0.9, system="You are a helpful assistant."):
    response = requests.post(
        "http://localhost:11434/api/chat",
        json={
            "model": "mistral:7b-instruct-q4_K_M",
            "messages": [
                {"role": "system", "content": system},
                {"role": "user", "content": prompt},
            ],
            "options": {
                "temperature": temperature,
                "top_p": top_p,
            },
            "stream": False,
        },
    )
    return response.json()["message"]["content"]

prompt = "Write a one-sentence description of the ocean."

print("=== Temperature comparison ===")
for temp in [0.0, 0.3, 0.7, 1.0, 1.5]:
    result = generate(prompt, temperature=temp)
    print(f"  temp={temp}: {result}")

print("\n=== System prompt comparison ===")
personas = [
    "You are a marine biologist.",
    "You are a pirate from the 1700s.",
    "You are a minimalist poet who uses no more than 8 words.",
]
for persona in personas:
    result = generate(prompt, system=persona)
    print(f"  [{persona[:30]}...]: {result}")
```

3. Run it multiple times and observe the variance at each temperature
4. Try the OpenAI-compatible endpoint (Ollama supports it at `/v1/chat/completions`) with the official `openai` Python package pointed at localhost

**What you'll learn:** How inference parameters shape output, why the same model can behave very differently with different settings, and how the local API mirrors the OpenAI API format — meaning tools built for OpenAI can point at your local server instead.

---

## Phase 4: Understanding Model Architecture and Selection

### Learning Objectives
- Read a model card on Hugging Face and understand what it tells you
- Distinguish between base models and instruction-tuned (chat) models
- Understand model families and their tradeoffs (Llama, Mistral, Qwen, Phi, Gemma)
- Know how to evaluate whether a model fits your hardware and use case

### Material

| Resource | Type | Notes |
|----------|------|-------|
| [Hugging Face Open LLM Leaderboard](https://huggingface.co/spaces/open-llm-leaderboard/open_llm_leaderboard) | Benchmark | Compare model performance across tasks. |
| [Andrej Karpathy — Intro to Large Language Models (1hr)](https://www.youtube.com/watch?v=zjkBMFhNj_g) | Video | High-level overview of the LLM landscape and how models relate to each other. |
| [Hugging Face Model Hub](https://huggingface.co/models?pipeline_tag=text-generation&sort=trending) | Hub | Browse available models, read their cards. |
| [r/LocalLLaMA](https://www.reddit.com/r/LocalLLaMA/) | Community | Active community for local LLM discussion. Good for real-world experience reports. |
| [LLM Comparison — Willison's Blog](https://simonwillison.net/tags/llms/) | Blog | Simon Willison's practical observations on different models. |

### Key Concepts to Lock In
- **Base model** (e.g., Llama-3-8B): trained on raw text, good at completing text, not good at following instructions
- **Instruct/Chat model** (e.g., Llama-3-8B-Instruct): fine-tuned on instruction-following data, aligned to be helpful
- **Model size vs quality is not linear**: a well-tuned 8B model can outperform a mediocre 70B model on specific tasks
- **Your 16GB VRAM sweet spot**: 7B-14B parameter models at Q4-Q8, or 30B models at Q3-Q4 (will be tight)
- **Context length** varies by model: 4K, 8K, 32K, 128K tokens. Longer context = more VRAM usage during inference

### Model families worth knowing (as of 2025):

| Family | Sizes available | Strengths |
|--------|----------------|-----------|
| Llama 3 (Meta) | 8B, 70B, 405B | Strong general-purpose, large community |
| Mistral/Mixtral | 7B, 8x7B, 8x22B | Excellent efficiency, good coding |
| Qwen 2.5 (Alibaba) | 0.5B–72B | Strong multilingual, good reasoning |
| Phi-3/4 (Microsoft) | 3.8B, 14B | Surprisingly capable for their size |
| Gemma 2 (Google) | 2B, 9B, 27B | Good quality at small sizes |

### Project 4: Model comparison on your hardware

**Goal:** Build personal intuition for what different models can and can't do on your hardware.

1. Pull 3 different models via Ollama:
```bash
ollama pull llama3.1:8b-instruct-q4_K_M
ollama pull qwen2.5:14b-instruct-q4_K_M
ollama pull phi3:14b-instruct-q4_K_M
```

2. Create a benchmark script that sends the same prompts to each model and records:
   - Response quality (your subjective rating 1-5)
   - Tokens per second
   - VRAM usage (`nvidia-smi`)
   - Ability to follow specific instructions (e.g., "respond in JSON", "respond in exactly 3 bullet points")

3. Test across categories:
   - General knowledge questions
   - Code generation (write a Python function)
   - Instruction following (structured output)
   - Creative writing
   - Reasoning (simple logic puzzles)

4. Write up your findings — which model would you choose for which use case?

**What you'll learn:** That model selection is task-dependent, that benchmarks don't tell the whole story, and what "good enough" feels like on your specific hardware.

---

## Phase 5: Fine-Tuning with LoRA/QLoRA

### Learning Objectives
- Understand what fine-tuning does and when to use it vs prompting
- Understand LoRA and QLoRA — why they make fine-tuning possible on consumer GPUs
- Prepare a training dataset
- Run a fine-tuning job on your RTX 5080
- Evaluate whether your fine-tune actually improved things

### Material

| Resource | Type | Notes |
|----------|------|-------|
| [Hugging Face PEFT Documentation](https://huggingface.co/docs/peft/main/en/index) | Docs | The library you'll use for LoRA/QLoRA fine-tuning. |
| [Hugging Face TRL (Transformer Reinforcement Learning)](https://huggingface.co/docs/trl/main/en/index) | Docs | High-level trainer for SFT (supervised fine-tuning). |
| [Sebastian Raschka — Practical Tips for Finetuning LLMs](https://magazine.sebastianraschka.com/p/practical-tips-for-finetuning-llms) | Article | Battle-tested advice from a researcher. |
| [Unsloth](https://github.com/unslothai/unsloth) | Tool | 2-5x faster fine-tuning with 80% less memory. Excellent for single-GPU setups. |
| [Axolotl](https://github.com/axolotl-ai-cloud/axolotl) | Tool | Flexible fine-tuning framework with many config options. |
| [Maxime Labonne — Fine-Tune Llama 3 with Unsloth](https://mlabonne.github.io/blog/posts/2024-07-29_Finetune_Llama31.html) | Tutorial | Step-by-step walkthrough using Unsloth. |
| [Hugging Face — Creating a dataset for fine-tuning](https://huggingface.co/docs/datasets/main/en/create_dataset) | Docs | How to structure and upload training data. |

### Key Concepts to Lock In
- **Full fine-tuning** updates all model parameters — requires enormous VRAM (multiple A100s for a 7B model)
- **LoRA (Low-Rank Adaptation)** freezes the base model and trains small adapter matrices that modify its behavior — 100-1000x fewer trainable parameters
- **QLoRA** loads the frozen base model in 4-bit quantization, then trains LoRA adapters on top — this is what makes fine-tuning possible on 16GB VRAM
- **SFT (Supervised Fine-Tuning)** means training on example input/output pairs — "when given this input, produce this output"
- **Epochs** = how many times the model sees each training example. For LLM fine-tuning, 1-3 epochs is typical; more risks overfitting
- **Learning rate** for LoRA is typically 1e-4 to 2e-4; too high = catastrophic forgetting, too low = no learning
- **LoRA rank (r)** controls adapter capacity: r=8 is minimal, r=16-32 is common, r=64+ for complex adaptations
- **Overfitting** means the model memorises your training data instead of learning the pattern. Watch for training loss dropping while validation loss rises.

### When to fine-tune vs when to prompt:
- **Prompt** when: you need the model to follow a format, use a persona, or apply knowledge it already has
- **Fine-tune** when: you need the model to consistently write in a specific style, learn domain terminology it doesn't know, or behave in ways that are hard to express in a prompt

### Project 5a: Fine-tune a model to respond in your personal writing style

**Goal:** Create a LoRA adapter that makes a model respond as if it were you.

1. **Prepare your dataset** — collect 50-200 examples of your writing (emails, messages, documents). Format as instruction/response pairs:
```json
[
  {
    "instruction": "Write a reply to a colleague asking about the project status",
    "output": "Hey — things are moving well. We shipped the auth fix on Tuesday and the metrics look clean. Main blocker now is the API rate limiting issue, should have a fix in by Thursday. Let me know if you need anything else."
  }
]
```

2. **Fine-tune with Unsloth** (memory-efficient, designed for single GPU):
```python
from unsloth import FastLanguageModel
from trl import SFTTrainer
from transformers import TrainingArguments
from datasets import load_dataset

# Load base model in 4-bit (QLoRA)
model, tokenizer = FastLanguageModel.from_pretrained(
    model_name="unsloth/Meta-Llama-3.1-8B-Instruct",
    max_seq_length=2048,
    load_in_4bit=True,
)

# Add LoRA adapters
model = FastLanguageModel.get_peft_model(
    model,
    r=16,
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj",
                    "gate_proj", "up_proj", "down_proj"],
    lora_alpha=16,
    lora_dropout=0,
    bias="none",
)

# Load your dataset
dataset = load_dataset("json", data_files="my_writing_style.json", split="train")

# Format into chat template
def format_example(example):
    return tokenizer.apply_chat_template(
        [
            {"role": "user", "content": example["instruction"]},
            {"role": "assistant", "content": example["output"]},
        ],
        tokenize=False,
    )

dataset = dataset.map(lambda x: {"text": format_example(x)})

# Train
trainer = SFTTrainer(
    model=model,
    train_dataset=dataset,
    dataset_text_field="text",
    max_seq_length=2048,
    args=TrainingArguments(
        per_device_train_batch_size=2,
        gradient_accumulation_steps=4,
        warmup_steps=5,
        num_train_epochs=3,
        learning_rate=2e-4,
        fp16=True,
        logging_steps=1,
        output_dir="outputs",
    ),
)

trainer.train()

# Save the adapter
model.save_pretrained("my-style-adapter")
```

3. **Test the adapter:**
```python
FastLanguageModel.for_inference(model)
inputs = tokenizer.apply_chat_template(
    [{"role": "user", "content": "Write a reply declining a meeting invitation"}],
    return_tensors="pt",
    add_generation_prompt=True,
).to("cuda")
outputs = model.generate(input_ids=inputs, max_new_tokens=200, temperature=0.7)
print(tokenizer.decode(outputs[0], skip_special_tokens=True))
```

4. **Compare** the output with and without the adapter (reload the base model without the adapter and generate from the same prompt)

**What you'll learn:** How a small amount of data can measurably shift model behavior, what LoRA adapters actually are (small weight matrices you can swap in/out), and the difference between the base model's generic style and your personalised one.

### Project 5b: Fine-tune for structured output (JSON mode)

**Goal:** Make a model reliably output valid JSON for a specific schema, even when the base model is inconsistent at it.

1. **Define your schema** (e.g., extracting entities from text):
```json
{
  "people": ["string"],
  "locations": ["string"],
  "dates": ["string"],
  "sentiment": "positive|negative|neutral"
}
```

2. **Generate training data** — write 100+ examples of input text paired with the correct JSON output. You can use a larger cloud model to help generate these, then manually verify them.

3. **Fine-tune** using the same Unsloth/TRL approach as 5a, but with your JSON dataset.

4. **Evaluate** by testing on held-out examples:
   - Does the output parse as valid JSON?
   - Are the extracted entities correct?
   - Compare accuracy before/after fine-tuning

**What you'll learn:** Fine-tuning for reliability and format compliance — one of the most practical uses of LoRA for agent workflows where you need predictable structured output.

---

## Phase 6: Merging, Exporting, and Serving Your Fine-Tuned Model

### Learning Objectives
- Merge a LoRA adapter back into a base model
- Quantise your merged model for efficient serving
- Convert between formats (safetensors -> GGUF)
- Serve your custom model via Ollama or llama.cpp

### Material

| Resource | Type | Notes |
|----------|------|-------|
| [llama.cpp — Model conversion](https://github.com/ggerganov/llama.cpp/blob/master/docs/development/HOWTO-add-model.md) | Docs | Converting models to GGUF format. |
| [Unsloth — Saving and exporting](https://docs.unsloth.ai/basics/saving-and-export) | Docs | Export to GGUF, merged model, or push to Hub. |
| [Ollama — Import a model](https://github.com/ollama/ollama/blob/main/docs/import.md) | Docs | How to import a GGUF into Ollama for serving. |

### Project 6: Package your fine-tuned model for daily use

**Goal:** Take your Project 5 adapter, merge it, quantise it, and serve it as a permanent local model.

1. **Merge the adapter into the base model:**
```python
from unsloth import FastLanguageModel

model, tokenizer = FastLanguageModel.from_pretrained(
    model_name="unsloth/Meta-Llama-3.1-8B-Instruct",
    max_seq_length=2048,
    load_in_4bit=True,
)
# Load your trained adapter
model.load_adapter("my-style-adapter")

# Save merged model as GGUF (Unsloth handles conversion)
model.save_pretrained_gguf("my-model-gguf", tokenizer, quantization_method="q4_k_m")
```

2. **Import into Ollama:**
Create a `Modelfile`:
```
FROM ./my-model-gguf/unsloth.Q4_K_M.gguf
TEMPLATE """{{ if .System }}<|start_header_id|>system<|end_header_id|>
{{ .System }}<|eot_id|>{{ end }}<|start_header_id|>user<|end_header_id|>
{{ .Prompt }}<|eot_id|><|start_header_id|>assistant<|end_header_id|>
"""
SYSTEM "You are a helpful assistant."
PARAMETER temperature 0.7
PARAMETER top_p 0.9
```

```bash
ollama create my-custom-model -f Modelfile
ollama run my-custom-model
```

3. Your model is now permanently available via `ollama serve` at `http://localhost:11434`

**What you'll learn:** The full pipeline from training to serving. Your custom model is now accessible the same way any other Ollama model is — any tool that speaks the Ollama/OpenAI API can use it.

---

## Phase 7: Agent Frameworks and Custom Workflows

### Learning Objectives
- Understand how LLM-powered agents work (tool calling, reasoning loops)
- Connect your local model to an agent framework
- Build a custom workflow that uses your local model
- Understand the limitations of smaller local models for agentic tasks

### Material

| Resource | Type | Notes |
|----------|------|-------|
| [LangChain — Introduction](https://python.langchain.com/docs/get_started/introduction) | Docs | Popular agent framework. Verbose but comprehensive. |
| [LangGraph](https://langchain-ai.github.io/langgraph/) | Docs | LangChain's graph-based agent framework. Better for complex workflows. |
| [CrewAI](https://github.com/crewAIInc/crewAI) | Tool | Multi-agent framework. Good for defining agent teams. |
| [Haystack](https://github.com/deepset-ai/haystack) | Tool | Framework for building RAG and agent pipelines. |
| [OpenAI function calling docs](https://platform.openai.com/docs/guides/function-calling) | Docs | The standard for tool-use that local models increasingly support. |
| [Ollama — OpenAI compatibility](https://github.com/ollama/ollama/blob/main/docs/openai.md) | Docs | How to use Ollama as a drop-in OpenAI replacement. |

### Key Concepts to Lock In
- **Agents** are loops: prompt the model, parse its response for tool calls, execute tools, feed results back, repeat
- **Tool calling** means the model outputs a structured request (usually JSON) saying "call function X with arguments Y"
- **RAG (Retrieval-Augmented Generation)** means fetching relevant documents and injecting them into the prompt before generation
- **Smaller models (7-14B) are worse at agentic tasks** than large frontier models. They struggle with multi-step reasoning, tool selection, and error recovery. Plan for simpler, more constrained workflows.
- **Structured output fine-tuning (Project 5b) directly helps here** — a model that reliably outputs valid JSON tool calls is much more useful as an agent

### Project 7: Build a local agent workflow

**Goal:** Create a simple agent that uses your local model to complete a multi-step task with tool use.

1. **Install dependencies:**
```bash
pip install langchain langchain-community langchain-openai
```

2. **Build a local agent with tools:**
```python
from langchain_openai import ChatOpenAI
from langchain.agents import tool, AgentExecutor, create_tool_calling_agent
from langchain_core.prompts import ChatPromptTemplate

# Point at your local Ollama server
llm = ChatOpenAI(
    base_url="http://localhost:11434/v1",
    api_key="not-needed",
    model="my-custom-model",  # your fine-tuned model
    temperature=0.1,  # low temp for reliable tool use
)

@tool
def get_weather(city: str) -> str:
    """Get the current weather for a city."""
    # In reality, call a weather API here
    return f"Weather in {city}: 22C, partly cloudy"

@tool
def calculate(expression: str) -> str:
    """Evaluate a mathematical expression."""
    try:
        return str(eval(expression))  # simplified — use a safe parser in production
    except Exception as e:
        return f"Error: {e}"

tools = [get_weather, calculate]

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful assistant. Use tools when needed."),
    ("human", "{input}"),
    ("placeholder", "{agent_scratchpad}"),
])

agent = create_tool_calling_agent(llm, tools, prompt)
executor = AgentExecutor(agent=agent, tools=tools, verbose=True)

result = executor.invoke({"input": "What's the weather in London, and what's 32 * 47?"})
print(result["output"])
```

3. **Observe** the agent's reasoning in verbose mode — you'll see it decide which tools to call and when
4. **Compare** how your fine-tuned model handles tool calling vs the base model
5. **Iterate** — if the model fails to call tools correctly, this tells you what to add to your fine-tuning dataset

**What you'll learn:** How agents use LLMs as a "brain" for decision-making, what makes a model good or bad at agentic tasks, and how your fine-tuning work in Phase 5 directly improves agent reliability.

---

## Phase 8: RAG — Giving Your Model Access to Your Own Documents

### Learning Objectives
- Understand embeddings and vector similarity search
- Set up a local vector database
- Build a RAG pipeline that retrieves context from your documents
- Understand chunking strategies and their impact on retrieval quality

### Material

| Resource | Type | Notes |
|----------|------|-------|
| [ChromaDB](https://github.com/chroma-core/chroma) | Tool | Simple local vector database. Good for getting started. |
| [LangChain — RAG tutorial](https://python.langchain.com/docs/tutorials/rag/) | Tutorial | End-to-end RAG walkthrough. |
| [Sentence Transformers](https://www.sbert.net/) | Library | Local embedding models. |
| [Hugging Face — MTEB Leaderboard](https://huggingface.co/spaces/mteb/leaderboard) | Benchmark | Compare embedding models. |
| [Ollama — Embedding models](https://ollama.com/library?q=embed) | Models | Embedding models you can run locally via Ollama. |

### Key Concepts to Lock In
- **Embeddings** convert text into dense numerical vectors where similar meanings are close together in vector space
- **Vector search** finds the most similar documents to a query by computing distance between their embedding vectors
- **Chunking** splits documents into smaller pieces for embedding. Chunk size affects retrieval: too small = missing context, too large = diluted relevance
- **RAG pipeline**: user query -> embed query -> find similar chunks -> inject chunks into prompt -> generate answer

### Project 8: Personal knowledge base with RAG

**Goal:** Build a system where your local model can answer questions about your own documents.

1. **Set up the vector database and embeddings:**
```python
import chromadb
from sentence_transformers import SentenceTransformer

embedding_model = SentenceTransformer("all-MiniLM-L6-v2")  # runs locally, fast
client = chromadb.PersistentClient(path="./my_knowledge_base")
collection = client.get_or_create_collection("docs")
```

2. **Ingest your documents:**
```python
import os

def chunk_text(text, chunk_size=500, overlap=50):
    words = text.split()
    chunks = []
    for i in range(0, len(words), chunk_size - overlap):
        chunk = " ".join(words[i:i + chunk_size])
        chunks.append(chunk)
    return chunks

docs_dir = "./my_documents"
for filename in os.listdir(docs_dir):
    with open(os.path.join(docs_dir, filename)) as f:
        text = f.read()
    chunks = chunk_text(text)
    embeddings = embedding_model.encode(chunks).tolist()
    collection.add(
        documents=chunks,
        embeddings=embeddings,
        ids=[f"{filename}_chunk_{i}" for i in range(len(chunks))],
        metadatas=[{"source": filename}] * len(chunks),
    )
```

3. **Query with RAG:**
```python
import requests

def ask_with_rag(question, n_results=3):
    query_embedding = embedding_model.encode([question]).tolist()
    results = collection.query(query_embeddings=query_embedding, n_results=n_results)

    context = "\n\n---\n\n".join(results["documents"][0])

    response = requests.post(
        "http://localhost:11434/api/chat",
        json={
            "model": "my-custom-model",
            "messages": [
                {"role": "system", "content": f"Answer based on the following context. If the context doesn't contain the answer, say so.\n\nContext:\n{context}"},
                {"role": "user", "content": question},
            ],
            "stream": False,
        },
    )
    return response.json()["message"]["content"]

print(ask_with_rag("What was the decision about the API migration?"))
```

**What you'll learn:** How to extend your model's knowledge beyond its training data without fine-tuning, the tradeoffs in chunking and retrieval, and when RAG is preferable to fine-tuning (answer: when the information changes frequently or is too large to bake into weights).

---

## Phase 9: Advanced Topics and Continued Learning

### Learning Objectives
- Understand RLHF/DPO and how alignment training works
- Explore model merging techniques
- Monitor model performance and detect degradation
- Stay current with the rapidly evolving ecosystem

### Material

| Resource | Type | Notes |
|----------|------|-------|
| [Hugging Face — DPO training with TRL](https://huggingface.co/docs/trl/main/en/dpo_trainer) | Docs | Preference optimisation — the step after SFT. |
| [mergekit](https://github.com/arcee-ai/mergekit) | Tool | Merge multiple models/adapters into one. |
| [Sebastian Raschka — Build an LLM from Scratch (book, open)](https://github.com/rasbt/LLMs-from-scratch) | Book/Code | Complete walkthrough of building an LLM from scratch. Best for deep understanding. |
| [Maxime Labonne — LLM Course](https://github.com/mlabonne/llm-course) | Course | Comprehensive free course covering the full LLM stack. |
| [The Little Book of Deep Learning (free PDF)](https://fleuret.org/public/lbdl.pdf) | Book | Concise 175-page overview of deep learning fundamentals. |
| [Weights & Biases — LLM monitoring](https://wandb.ai/site) | Tool | Track training runs, compare experiments. |
| [Open WebUI](https://github.com/open-webui/open-webui) | Tool | Self-hosted ChatGPT-like interface for your local models. |

### Project 9: DPO preference tuning

**Goal:** Go beyond "teach the model what to say" to "teach the model which of two responses is better."

1. Create a preference dataset — pairs of (chosen, rejected) responses for the same prompt:
```json
[
  {
    "prompt": "Explain recursion",
    "chosen": "Recursion is when a function calls itself...",
    "rejected": "Recursion is a computer science concept that involves..."
  }
]
```

2. Use TRL's DPOTrainer to train a preference adapter on top of your SFT model from Phase 5
3. Compare outputs: base model vs SFT model vs SFT+DPO model on the same prompts

**What you'll learn:** How alignment training shapes model preferences without explicit "right answers" — the same technique used to make frontier models helpful and harmless.

---

## Suggested Timeline

This is a self-paced path. Rough estimates assuming a few hours per week:

| Phase | Estimated time | Depends on |
|-------|---------------|------------|
| Phase 1: Foundations | 1-2 weeks | — |
| Phase 2: GPU + Local Inference | 1 week | Phase 1 |
| Phase 3: API Server + Parameters | 1 week | Phase 2 |
| Phase 4: Model Selection | 1 week | Phase 3 |
| Phase 5: Fine-Tuning | 2-3 weeks | Phase 4 |
| Phase 6: Export + Serve | 1 week | Phase 5 |
| Phase 7: Agents | 1-2 weeks | Phase 3 (Phase 6 optional) |
| Phase 8: RAG | 1-2 weeks | Phase 3 |
| Phase 9: Advanced | Ongoing | Phase 5 |

Phases 7 and 8 can be done in parallel with Phase 5-6 since they only require a working local inference server (Phase 3).

---

## Quick Reference: Your RTX 5080 Capabilities

| Task | What fits in 16GB VRAM |
|------|----------------------|
| Inference (Q4) | Up to ~30B parameters comfortably |
| Inference (FP16) | Up to ~7B parameters |
| QLoRA fine-tuning | Up to ~14B parameters (with gradient checkpointing) |
| Full fine-tuning | Not practical — use QLoRA instead |
| Embedding + LLM simultaneously | Embedding model (~0.5GB) + 7B Q4 model (~5GB) fits easily |

---

## Community and Staying Current

- [r/LocalLLaMA](https://www.reddit.com/r/LocalLLaMA/) — the most active community for local LLM work
- [Hugging Face Discord](https://huggingface.co/join/discord) — direct access to library maintainers
- [Unsloth Discord](https://discord.gg/unsloth) — fine-tuning help
- [Simon Willison's blog](https://simonwillison.net/) — practical LLM observations and tool reviews
- [The Batch (Andrew Ng)](https://www.deeplearning.ai/the-batch/) — weekly AI news digest
