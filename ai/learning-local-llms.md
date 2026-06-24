# Learning to Run LLMs Locally: Beginner to Expert

A structured learning path for running, understanding, and customizing LLMs on an Ubuntu PC with an RTX 5080 (16GB VRAM).

---

## Phase 1: Foundations — What Are LLMs?

### Goal
Understand what an LLM actually is before touching any tools.

### Key Concepts
- **Tokens**: LLMs don't read words — they read tokens (subword pieces). "understanding" might be 2-3 tokens.
- **Parameters**: The "weights" learned during training. A 7B model has 7 billion parameters.
- **Context window**: How many tokens the model can "see" at once (input + output combined).
- **Transformer architecture**: The neural network design behind all modern LLMs (attention mechanism, feed-forward layers, embeddings).
- **Training vs inference**: Training = learning the weights. Inference = using those weights to generate text.

### Reading & Videos
- [3Blue1Brown — "But what is a GPT?"](https://www.youtube.com/watch?v=wjZofJX0v4M) — Visual intuition for transformers
- [3Blue1Brown — "Attention in Transformers"](https://www.youtube.com/watch?v=eMlx5fFNoYc) — How attention works
- [Andrej Karpathy — "Let's build GPT from scratch"](https://www.youtube.com/watch?v=kCc8FmEb1nY) — 2hr hands-on video building a tiny GPT
- [The Illustrated Transformer](https://jalammar.github.io/illustrated-transformer/) — Jay Alammar's visual guide
- [Hugging Face NLP Course, Chapter 1](https://huggingface.co/learn/nlp-course/chapter1) — Free, interactive

### Exercise 1.1: Tokenization
```bash
pip install tiktoken
python3 -c "
import tiktoken
enc = tiktoken.encoding_for_model('gpt-4')
text = 'The RTX 5080 has 16GB of VRAM'
tokens = enc.encode(text)
print(f'Text: {text}')
print(f'Tokens: {tokens}')
print(f'Token count: {len(tokens)}')
for t in tokens:
    print(f'  {t} -> \"{enc.decode([t])}\"')
"
```
**What to notice**: How words get split, how common words are single tokens, how rare words get broken up.

### Exercise 1.2: Build a Tiny Language Model
Follow Karpathy's video and build a character-level model on a small text corpus. You don't need a GPU for this — it trains on CPU in minutes. The goal is to see the training loop: forward pass, loss calculation, backpropagation, weight update.

```bash
git clone https://github.com/karpathy/nanoGPT.git
cd nanoGPT
pip install torch numpy transformers datasets tiktoken wandb tqdm
# Train on Shakespeare (tiny, runs on CPU)
python data/shakespeare_char/prepare.py
python train.py config/train_shakespeare_char.py
```

---

## Phase 2: Running Your First Local Model

### Goal
Get a model running on your RTX 5080 and understand the inference pipeline.

### Key Concepts
- **Quantization**: Reducing precision of weights (e.g., 16-bit -> 4-bit) to fit larger models in less VRAM. Your 16GB can run ~13B parameter models at 4-bit quantization.
- **GGUF format**: A file format for quantized models, used by llama.cpp and Ollama.
- **Inference engine**: Software that loads the model and generates text (llama.cpp, vLLM, Ollama).
- **Sampling parameters**: temperature, top_p, top_k — control randomness of output.

### Tool: Ollama (Simplest Starting Point)

```bash
# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh

# Pull and run a model (fits easily in 16GB VRAM)
ollama pull llama3.1:8b
ollama run llama3.1:8b

# Try a larger quantized model
ollama pull deepseek-coder-v2:16b-lite-instruct-q4_K_M
ollama run deepseek-coder-v2:16b-lite-instruct-q4_K_M
```

### Exercise 2.1: Compare Model Sizes
Run the same prompt through different model sizes and notice quality differences:
```bash
# Small (3B) - fast but less capable
ollama run phi3:3.8b "Explain what a linked list is and when to use one"

# Medium (8B) - good balance
ollama run llama3.1:8b "Explain what a linked list is and when to use one"

# Large (14B, quantized) - better quality, slower
ollama run qwen2.5:14b "Explain what a linked list is and when to use one"
```
**What to notice**: Response quality, speed differences, how quantization affects output.

### Exercise 2.2: Monitor GPU Usage
```bash
# In one terminal, watch GPU memory:
watch -n 1 nvidia-smi

# In another, load models and observe VRAM usage
ollama run llama3.1:8b
```
**What to notice**: How much VRAM each model uses. Calculate: parameters * bits-per-weight / 8 = approximate size in bytes.

### Exercise 2.3: Use the API
Ollama exposes an OpenAI-compatible API:
```python
import requests
import json

response = requests.post("http://localhost:11434/api/generate", json={
    "model": "llama3.1:8b",
    "prompt": "Write a Python function that reverses a linked list",
    "stream": False,
    "options": {
        "temperature": 0.7,
        "top_p": 0.9,
    }
})
print(json.loads(response.text)["response"])
```

### Exercise 2.4: Experiment with Sampling Parameters
```python
import requests, json

prompt = "Write a haiku about debugging code"

for temp in [0.0, 0.5, 1.0, 1.5]:
    resp = requests.post("http://localhost:11434/api/generate", json={
        "model": "llama3.1:8b",
        "prompt": prompt,
        "stream": False,
        "options": {"temperature": temp}
    })
    print(f"\n--- Temperature {temp} ---")
    print(json.loads(resp.text)["response"])
```
**What to notice**: Low temperature = deterministic/repetitive. High temperature = creative/chaotic.

### Reading
- [Ollama documentation](https://github.com/ollama/ollama/blob/main/docs/README.md)
- [GGUF format explanation](https://github.com/ggerganov/ggml/blob/master/docs/gguf.md)
- [Quantization explained (TheBloke's guide)](https://huggingface.co/TheBloke/Llama-2-7B-GGUF#provided-files) — read the table explaining Q4_K_M, Q5_K_S, etc.

---

## Phase 3: Understanding Inference in Depth

### Goal
Understand what happens between "prompt goes in" and "text comes out."

### Key Concepts
- **Prefill vs decode**: Prefill processes your entire prompt in parallel (fast). Decode generates one token at a time (slow, sequential).
- **KV cache**: Stores computed attention values so the model doesn't recompute them for every new token.
- **Tokens per second (tok/s)**: Your primary performance metric. Prefill tok/s >> decode tok/s.
- **Batch size**: Processing multiple requests simultaneously to improve GPU utilization.
- **Speculative decoding**: Using a small fast model to "guess" tokens, then verifying with the large model.

### Tool: llama.cpp (Lower Level Than Ollama)

```bash
# Build llama.cpp with CUDA support
git clone https://github.com/ggerganov/llama.cpp.git
cd llama.cpp
cmake -B build -DGGML_CUDA=ON
cmake --build build --config Release -j$(nproc)

# Download a model (GGUF format) from Hugging Face
# Example: Llama 3.1 8B at Q4_K_M quantization
# Go to https://huggingface.co/bartowski/Meta-Llama-3.1-8B-Instruct-GGUF
# Download the Q4_K_M file

# Run inference with verbose stats
./build/bin/llama-cli \
  -m ~/models/Meta-Llama-3.1-8B-Instruct-Q4_K_M.gguf \
  -p "Explain how a hash table works" \
  -n 256 \
  --gpu-layers 99 \
  --ctx-size 4096
```

### Exercise 3.1: Measure Inference Performance
```bash
# Benchmark the model
./build/bin/llama-bench \
  -m ~/models/Meta-Llama-3.1-8B-Instruct-Q4_K_M.gguf \
  -p 512 -n 128 -ngl 99
```
**What to notice**: Prefill speed vs generation speed. The ratio tells you about memory bandwidth vs compute bottleneck.

### Exercise 3.2: Observe KV Cache Impact
```bash
# Short context
./build/bin/llama-cli -m ~/models/your-model.gguf \
  -p "Hello" -n 100 --gpu-layers 99 --ctx-size 512

# Long context (notice slower generation as context fills)
./build/bin/llama-cli -m ~/models/your-model.gguf \
  -p "$(cat some-long-file.txt)" -n 100 --gpu-layers 99 --ctx-size 8192
```
**What to notice**: Generation slows as context length grows because the KV cache gets larger.

### Exercise 3.3: Compare Quantization Levels
Download the same model at different quantization levels (Q2_K, Q4_K_M, Q8_0, F16) and compare:
1. File size
2. VRAM usage
3. Tokens per second
4. Output quality (same prompt, temperature 0)

### Reading
- [llama.cpp README](https://github.com/ggerganov/llama.cpp/blob/master/README.md)
- [Efficient Inference on a Single GPU (Hugging Face)](https://huggingface.co/docs/transformers/perf_infer_gpu_one)
- [LLM Inference Explained (YouTube - Umar Jamil)](https://www.youtube.com/watch?v=mYRqvB1_gRk)

---

## Phase 4: Serving Models for Code Generation

### Goal
Set up a local inference server that tools (like VS Code extensions) can connect to.

### Key Concepts
- **OpenAI-compatible API**: A REST API format that most tools support. Your local server mimics OpenAI's API.
- **System prompts**: Instructions that shape model behavior (e.g., "You are a Python expert").
- **Chat templates**: How messages get formatted into the raw token stream the model expects.
- **FIM (Fill-in-the-Middle)**: A special mode for code completion where the model fills a gap between prefix and suffix.

### Exercise 4.1: Set Up a Code Completion Server
```bash
# Using Ollama with a code-specialized model
ollama pull qwen2.5-coder:14b

# Start serving (already running by default on port 11434)
# Test with a code completion request:
curl http://localhost:11434/api/chat -d '{
  "model": "qwen2.5-coder:14b",
  "messages": [
    {"role": "system", "content": "You are an expert Python programmer. Write clean, well-structured code."},
    {"role": "user", "content": "Write a Python class for a thread-safe LRU cache with TTL expiration"}
  ],
  "stream": false
}'
```

### Exercise 4.2: Connect to VS Code
Install the [Continue](https://continue.dev/) extension in VS Code and configure it to use your local Ollama:
```json
// ~/.continue/config.json
{
  "models": [
    {
      "title": "Local Qwen Coder",
      "provider": "ollama",
      "model": "qwen2.5-coder:14b"
    }
  ],
  "tabAutocompleteModel": {
    "title": "Local Autocomplete",
    "provider": "ollama",
    "model": "qwen2.5-coder:7b"
  }
}
```
Use it for a real coding task and compare the experience to Claude CLI.

### Exercise 4.3: Build a Simple Chat Interface
```python
# chat.py - A minimal chat loop using your local model
import requests
import json

def chat(model="qwen2.5-coder:14b"):
    messages = [
        {"role": "system", "content": "You are a helpful coding assistant."}
    ]
    
    print(f"Chatting with {model}. Type 'quit' to exit.\n")
    
    while True:
        user_input = input("You: ")
        if user_input.lower() == 'quit':
            break
            
        messages.append({"role": "user", "content": user_input})
        
        response = requests.post("http://localhost:11434/api/chat", json={
            "model": model,
            "messages": messages,
            "stream": True
        }, stream=True)
        
        print("Assistant: ", end="", flush=True)
        full_response = ""
        for line in response.iter_lines():
            if line:
                chunk = json.loads(line)
                content = chunk.get("message", {}).get("content", "")
                print(content, end="", flush=True)
                full_response += content
        print("\n")
        
        messages.append({"role": "assistant", "content": full_response})

if __name__ == "__main__":
    chat()
```

### Reading
- [Continue.dev docs](https://docs.continue.dev/)
- [Ollama OpenAI compatibility](https://github.com/ollama/ollama/blob/main/docs/openai.md)
- [Chat templates (Hugging Face)](https://huggingface.co/docs/transformers/chat_templating)

---

## Phase 5: Fine-Tuning — Making a Model Your Own

### Goal
Understand how to customize a model's behavior for specific tasks.

### Key Concepts
- **Fine-tuning vs prompting**: Prompting = changing input. Fine-tuning = changing weights. Fine-tuning is permanent but requires data and compute.
- **LoRA (Low-Rank Adaptation)**: Only trains a small number of new parameters (adapters) instead of all weights. Fits on consumer GPUs.
- **QLoRA**: LoRA applied to a quantized base model. Lets you fine-tune 13B+ models on 16GB VRAM.
- **Training data format**: Instruction/response pairs in JSONL format.
- **Overfitting**: Training too long on too little data — the model memorizes instead of generalizing.
- **Loss curve**: Tracks how well the model is learning. Should decrease then plateau.

### Tool: Unsloth (Optimized Fine-Tuning)

```bash
pip install "unsloth[colab-new] @ git+https://github.com/unslothai/unsloth.git"
pip install --no-deps trl peft accelerate bitsandbytes
```

### Exercise 5.1: Prepare Training Data
Create a dataset of instruction/response pairs for a specific task (e.g., generating code reviews):
```python
# prepare_data.py
import json

training_data = [
    {
        "instruction": "Review this Python code for issues:\n```python\ndef get_user(id):\n    return db.query(f'SELECT * FROM users WHERE id = {id}')\n```",
        "output": "**SQL Injection vulnerability**: The `id` parameter is interpolated directly into the SQL string. Use parameterized queries instead:\n```python\ndef get_user(id):\n    return db.query('SELECT * FROM users WHERE id = %s', (id,))\n```"
    },
    {
        "instruction": "Review this Python code for issues:\n```python\ndef process_items(items):\n    results = []\n    for i in range(len(items)):\n        results.append(items[i].upper())\n    return results\n```",
        "output": "**Non-idiomatic iteration**: Use a list comprehension and iterate directly over items instead of indexing:\n```python\ndef process_items(items):\n    return [item.upper() for item in items]\n```"
    },
    # Add 50-200+ more examples for a meaningful fine-tune
]

with open("training_data.jsonl", "w") as f:
    for item in training_data:
        f.write(json.dumps(item) + "\n")
```

### Exercise 5.2: Fine-Tune with QLoRA
```python
# finetune.py
from unsloth import FastLanguageModel
import torch
from trl import SFTTrainer
from transformers import TrainingArguments
from datasets import load_dataset

# Load base model in 4-bit
model, tokenizer = FastLanguageModel.from_pretrained(
    model_name="unsloth/llama-3-8b-Instruct-bnb-4bit",
    max_seq_length=2048,
    dtype=None,  # auto-detect
    load_in_4bit=True,
)

# Add LoRA adapters
model = FastLanguageModel.get_peft_model(
    model,
    r=16,              # LoRA rank (higher = more capacity, more VRAM)
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj",
                    "gate_proj", "up_proj", "down_proj"],
    lora_alpha=16,
    lora_dropout=0,
    bias="none",
    use_gradient_checkpointing="unsloth",
)

# Load your dataset
dataset = load_dataset("json", data_files="training_data.jsonl", split="train")

# Format into chat template
def formatting_func(example):
    return f"""<|begin_of_text|><|start_header_id|>system<|end_header_id|>
You are an expert code reviewer.<|eot_id|><|start_header_id|>user<|end_header_id|>
{example['instruction']}<|eot_id|><|start_header_id|>assistant<|end_header_id|>
{example['output']}<|eot_id|>"""

# Train
trainer = SFTTrainer(
    model=model,
    tokenizer=tokenizer,
    train_dataset=dataset,
    formatting_func=formatting_func,
    args=TrainingArguments(
        per_device_train_batch_size=2,
        gradient_accumulation_steps=4,
        warmup_steps=5,
        max_steps=60,
        learning_rate=2e-4,
        fp16=not torch.cuda.is_bf16_supported(),
        bf16=torch.cuda.is_bf16_supported(),
        logging_steps=1,
        output_dir="outputs",
    ),
)

trainer.train()

# Save the LoRA adapter
model.save_pretrained("code-reviewer-lora")
```

### Exercise 5.3: Test Your Fine-Tuned Model
```python
# test_finetuned.py
from unsloth import FastLanguageModel

model, tokenizer = FastLanguageModel.from_pretrained(
    model_name="code-reviewer-lora",
    max_seq_length=2048,
    load_in_4bit=True,
)
FastLanguageModel.for_inference(model)

prompt = """<|begin_of_text|><|start_header_id|>system<|end_header_id|>
You are an expert code reviewer.<|eot_id|><|start_header_id|>user<|end_header_id|>
Review this Python code for issues:
```python
def connect():
    password = "admin123"
    return mysql.connect(host="prod-db.internal", password=password)
```<|eot_id|><|start_header_id|>assistant<|end_header_id|>
"""

inputs = tokenizer(prompt, return_tensors="pt").to("cuda")
outputs = model.generate(**inputs, max_new_tokens=256, temperature=0.7)
print(tokenizer.decode(outputs[0], skip_special_tokens=True))
```

### Exercise 5.4: Export to GGUF and Run in Ollama
```python
# After training, export to GGUF
model.save_pretrained_gguf("code-reviewer-gguf", tokenizer, quantization_method="q4_k_m")
```
```bash
# Create an Ollama modelfile
cat > Modelfile <<EOF
FROM ./code-reviewer-gguf/unsloth.Q4_K_M.gguf
TEMPLATE """<|begin_of_text|><|start_header_id|>system<|end_header_id|>
{{ .System }}<|eot_id|><|start_header_id|>user<|end_header_id|>
{{ .Prompt }}<|eot_id|><|start_header_id|>assistant<|end_header_id|>
{{ .Response }}<|eot_id|>"""
SYSTEM "You are an expert code reviewer."
EOF

ollama create code-reviewer -f Modelfile
ollama run code-reviewer "Review this: def f(x): return eval(x)"
```

### Reading
- [Unsloth documentation](https://docs.unsloth.ai/)
- [LoRA paper (Hu et al.)](https://arxiv.org/abs/2106.09685) — the original paper, readable
- [QLoRA paper](https://arxiv.org/abs/2305.14314)
- [Hugging Face PEFT docs](https://huggingface.co/docs/peft)
- [Sebastian Raschka — "Finetuning LLMs" (YouTube)](https://www.youtube.com/watch?v=eC6Hd1hFvos)

---

## Phase 6: RAG — Giving Models Access to Your Data

### Goal
Understand Retrieval-Augmented Generation: letting models answer questions about documents they weren't trained on.

### Key Concepts
- **Embeddings**: Dense vector representations of text. Similar texts have similar vectors.
- **Vector database**: Stores embeddings and retrieves the most similar ones to a query.
- **Chunking**: Splitting documents into pieces small enough for the context window.
- **Retrieval pipeline**: Query -> embed -> search vectors -> inject top results into prompt -> generate.

### Exercise 6.1: Build a Simple RAG System
```python
# rag.py
import chromadb
from sentence_transformers import SentenceTransformer
import requests
import json
import os

# 1. Set up embedding model (runs locally!)
embedder = SentenceTransformer("all-MiniLM-L6-v2")

# 2. Set up vector store
client = chromadb.PersistentClient(path="./chroma_db")
collection = client.get_or_create_collection("my_docs")

# 3. Index some documents (e.g., your project's docs)
def index_file(filepath):
    with open(filepath, 'r') as f:
        content = f.read()
    
    # Simple chunking by paragraphs
    chunks = [c.strip() for c in content.split('\n\n') if c.strip()]
    
    for i, chunk in enumerate(chunks):
        collection.add(
            documents=[chunk],
            ids=[f"{filepath}_{i}"],
            metadatas=[{"source": filepath}]
        )

# Index your project docs
for f in os.listdir("./docs"):
    if f.endswith(".md"):
        index_file(f"./docs/{f}")

# 4. Query with RAG
def ask(question, model="llama3.1:8b"):
    # Retrieve relevant chunks
    results = collection.query(query_texts=[question], n_results=3)
    context = "\n\n---\n\n".join(results['documents'][0])
    
    # Generate with context
    prompt = f"""Use the following context to answer the question. If the context doesn't contain the answer, say so.

Context:
{context}

Question: {question}

Answer:"""
    
    response = requests.post("http://localhost:11434/api/generate", json={
        "model": model,
        "prompt": prompt,
        "stream": False
    })
    return json.loads(response.text)["response"]

print(ask("How does the notification dedup system work?"))
```

### Exercise 6.2: Evaluate Retrieval Quality
Test whether the right chunks come back for different questions. Try different chunk sizes (256, 512, 1024 tokens) and overlap strategies.

### Reading
- [ChromaDB docs](https://docs.trychroma.com/)
- [Sentence Transformers](https://www.sbert.net/)
- [RAG paper (Lewis et al.)](https://arxiv.org/abs/2005.11401)
- [LangChain RAG tutorial](https://python.langchain.com/docs/tutorials/rag/)

---

## Phase 7: Agents and Tool Use

### Goal
Understand how LLMs use tools (code execution, web search, APIs) — the pattern behind Claude CLI, GitHub Copilot, etc.

### Key Concepts
- **Tool/function calling**: The model outputs structured JSON describing which function to call and with what arguments. The runtime executes it and feeds the result back.
- **ReAct pattern**: Reasoning + Acting. The model thinks step-by-step, decides on an action, observes the result, repeats.
- **System prompt engineering**: Describing available tools so the model knows when and how to use them.
- **Agentic loop**: prompt -> model -> tool call -> execute -> result -> model -> (repeat or final answer).

### Exercise 7.1: Build a Minimal Agent
```python
# agent.py
import requests
import json
import subprocess

TOOLS = {
    "run_python": {
        "description": "Execute Python code and return the output",
        "parameters": {"code": "string"}
    },
    "read_file": {
        "description": "Read the contents of a file",
        "parameters": {"path": "string"}
    },
    "list_files": {
        "description": "List files in a directory",
        "parameters": {"path": "string"}
    }
}

def execute_tool(name, args):
    if name == "run_python":
        result = subprocess.run(
            ["python3", "-c", args["code"]],
            capture_output=True, text=True, timeout=10
        )
        return result.stdout + result.stderr
    elif name == "read_file":
        with open(args["path"]) as f:
            return f.read()
    elif name == "list_files":
        return "\n".join(os.listdir(args["path"]))

SYSTEM_PROMPT = f"""You are a coding assistant with access to tools.

Available tools:
{json.dumps(TOOLS, indent=2)}

To use a tool, respond with EXACTLY this format:
TOOL: tool_name
ARGS: {{"param": "value"}}

After seeing tool results, continue reasoning or give a final answer.
If you don't need a tool, just respond normally."""

def agent_loop(question):
    messages = [
        {"role": "system", "content": SYSTEM_PROMPT},
        {"role": "user", "content": question}
    ]
    
    for _ in range(5):  # max iterations
        response = requests.post("http://localhost:11434/api/chat", json={
            "model": "llama3.1:8b",
            "messages": messages,
            "stream": False
        })
        
        reply = json.loads(response.text)["message"]["content"]
        print(f"Assistant: {reply}\n")
        
        if "TOOL:" in reply:
            # Parse tool call
            lines = reply.split("\n")
            tool_name = None
            tool_args = None
            for line in lines:
                if line.startswith("TOOL:"):
                    tool_name = line.split("TOOL:")[1].strip()
                if line.startswith("ARGS:"):
                    tool_args = json.loads(line.split("ARGS:")[1].strip())
            
            if tool_name and tool_args:
                result = execute_tool(tool_name, tool_args)
                print(f"[Tool {tool_name} returned: {result[:200]}]\n")
                messages.append({"role": "assistant", "content": reply})
                messages.append({"role": "user", "content": f"Tool result:\n{result}"})
        else:
            return reply
    
    return "Max iterations reached"

agent_loop("What Python files are in the current directory? Read the largest one and summarize what it does.")
```

### Exercise 7.2: Add Structured Tool Calling
Upgrade the agent to use Ollama's native tool calling (available with some models):
```python
import ollama

response = ollama.chat(
    model='llama3.1:8b',
    messages=[{'role': 'user', 'content': 'What files are in /tmp?'}],
    tools=[{
        'type': 'function',
        'function': {
            'name': 'list_files',
            'description': 'List files in a directory',
            'parameters': {
                'type': 'object',
                'properties': {
                    'path': {'type': 'string', 'description': 'Directory path'}
                },
                'required': ['path']
            }
        }
    }]
)
print(response['message']['tool_calls'])
```

### Reading
- [ReAct paper](https://arxiv.org/abs/2210.03629)
- [Anthropic tool use docs](https://docs.anthropic.com/en/docs/build-with-claude/tool-use)
- [LangChain Agents](https://python.langchain.com/docs/how_to/#agents)
- [Simon Willison — "Building an AI Agent"](https://simonwillison.net/tags/agents/)

---

## Phase 8: Training From Scratch (Advanced)

### Goal
Understand the full training pipeline — not to train GPT-4, but to grasp what's actually happening when you read about model releases.

### Key Concepts
- **Pre-training**: Training on massive internet text to learn language. Costs millions of dollars at scale.
- **Supervised Fine-Tuning (SFT)**: Teaching the model to follow instructions using curated examples.
- **RLHF / DPO**: Aligning the model to human preferences. RLHF uses a reward model; DPO skips it.
- **Data pipeline**: Collection -> filtering -> deduplication -> tokenization -> shuffling.
- **Scaling laws**: Predictable relationship between compute, data, parameters, and performance.

### Exercise 8.1: Train a Tiny Model From Scratch
Using nanoGPT, train a small transformer on a custom dataset:
```bash
cd nanoGPT

# Prepare your own dataset (e.g., all Python files from a project)
find ~/your-project -name "*.py" -exec cat {} \; > data/custom/input.txt
python data/custom/prepare.py

# Train (will take hours even at small scale on your 5080)
python train.py \
  --dataset=custom \
  --n_layer=6 \
  --n_head=6 \
  --n_embd=384 \
  --batch_size=32 \
  --block_size=256 \
  --max_iters=5000 \
  --device=cuda
```

### Exercise 8.2: Implement DPO Alignment
```python
# dpo_training.py (conceptual - uses trl library)
from trl import DPOTrainer, DPOConfig
from transformers import AutoModelForCausalLM, AutoTokenizer
from datasets import Dataset

# DPO needs pairs: (prompt, chosen_response, rejected_response)
dpo_data = [
    {
        "prompt": "How do I read a file in Python?",
        "chosen": "```python\nwith open('file.txt', 'r') as f:\n    content = f.read()\n```\nUsing a context manager ensures the file is properly closed.",
        "rejected": "Just use open('file.txt').read() lol"
    },
    # More pairs...
]

dataset = Dataset.from_list(dpo_data)
model = AutoModelForCausalLM.from_pretrained("your-sft-model")
tokenizer = AutoTokenizer.from_pretrained("your-sft-model")

trainer = DPOTrainer(
    model=model,
    ref_model=None,  # uses implicit reference
    args=DPOConfig(
        per_device_train_batch_size=2,
        learning_rate=5e-7,
        num_train_epochs=1,
        output_dir="dpo-output",
    ),
    train_dataset=dataset,
    tokenizer=tokenizer,
)
trainer.train()
```

### Reading
- [Andrej Karpathy — "Reproducing GPT-2" (YouTube)](https://www.youtube.com/watch?v=l8pRSuU81PU)
- [Chinchilla scaling laws paper](https://arxiv.org/abs/2203.15556)
- [InstructGPT paper (RLHF)](https://arxiv.org/abs/2203.02155)
- [DPO paper](https://arxiv.org/abs/2305.18290)
- [The Llama 3 paper](https://arxiv.org/abs/2407.21783) — comprehensive look at a modern training pipeline
- [Sebastian Raschka — "Build an LLM From Scratch" (book)](https://www.manning.com/books/build-a-large-language-model-from-scratch)

---

## Phase 9: Advanced Topics

### Mixture of Experts (MoE)
Models like Mixtral have many "expert" sub-networks but only activate a few per token. This gives you a bigger model's quality at a smaller model's inference cost.

### Speculative Decoding
Use a small draft model to generate candidate tokens, then verify them in batch with the large model. Can give 2-3x speedup.

### Structured Output
Force the model to output valid JSON, code in a specific language, etc. using grammar-constrained decoding:
```bash
# llama.cpp supports GBNF grammars
./build/bin/llama-cli -m model.gguf \
  --grammar-file json.gbnf \
  -p "Output a JSON object with name, age, and city fields for a fictional person"
```

### Model Merging
Combine weights from multiple fine-tuned models without additional training:
```bash
pip install mergekit
# mergekit allows SLERP, TIES, DARE merging strategies
```

### Continuous Batching & Production Serving
For serving multiple users, use vLLM:
```bash
pip install vllm
python -m vllm.entrypoints.openai.api_server \
  --model meta-llama/Llama-3.1-8B-Instruct \
  --dtype auto \
  --max-model-len 8192
```

### Reading
- [vLLM docs](https://docs.vllm.ai/)
- [Speculative decoding paper](https://arxiv.org/abs/2211.17192)
- [MergeKit](https://github.com/arcee-ai/mergekit)
- [Mixture of Experts explained](https://huggingface.co/blog/moe)

---

## Hardware Reference: Your RTX 5080

| Spec | Value |
|------|-------|
| VRAM | 16 GB GDDR7 |
| Memory bandwidth | ~960 GB/s |
| CUDA cores | 10,752 |
| Tensor cores | 336 (5th gen) |

### What fits in 16GB VRAM

| Model Size | Quantization | Approx VRAM | Quality |
|-----------|--------------|-------------|---------|
| 7-8B | FP16 | 16 GB (tight) | Best for size |
| 7-8B | Q4_K_M | ~5 GB | Very good |
| 13-14B | Q4_K_M | ~9 GB | Good |
| 22-27B | Q4_K_M | ~14-16 GB | Usable, tight |
| 70B | Q2_K | Won't fit | - |

**Rule of thumb**: Parameters * bits / 8 = model size in bytes. Add ~1-2GB for KV cache and overhead.

---

## Recommended Learning Order

1. **Week 1-2**: Phase 1 (concepts) + Phase 2 (Ollama, first models)
2. **Week 3-4**: Phase 3 (inference details) + Phase 4 (serving for code)
3. **Week 5-6**: Phase 5 (fine-tuning with LoRA)
4. **Week 7-8**: Phase 6 (RAG) + Phase 7 (agents)
5. **Month 3+**: Phase 8 (training from scratch) + Phase 9 (advanced)

---

## Glossary

| Term | Definition |
|------|-----------|
| **Inference** | Using a trained model to generate text (as opposed to training it) |
| **VRAM** | GPU memory — the bottleneck for model size |
| **Quantization** | Reducing weight precision (FP16 -> INT4) to fit bigger models in less memory |
| **LoRA** | Training a small adapter instead of the full model |
| **GGUF** | File format for quantized models (used by llama.cpp/Ollama) |
| **KV Cache** | Memory storing computed attention values during generation |
| **Token** | The atomic unit of text for an LLM (subword piece) |
| **Context window** | Maximum tokens the model can process at once |
| **Temperature** | Controls randomness: 0 = deterministic, >1 = creative/chaotic |
| **RAG** | Retrieval-Augmented Generation — injecting retrieved documents into the prompt |
| **RLHF** | Reinforcement Learning from Human Feedback — aligning models to preferences |
| **DPO** | Direct Preference Optimization — simpler alternative to RLHF |
| **SFT** | Supervised Fine-Tuning — teaching a model to follow instructions |
| **FIM** | Fill-in-the-Middle — code completion mode |
| **MoE** | Mixture of Experts — activating only some model parameters per token |
