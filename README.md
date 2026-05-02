# Adversarial AI: Indirect Prompt Injection → S3 Exfiltration

A live demonstration of how a single poisoned document inside a retrieval-augmented generation (RAG) corpus can turn a tool-using LLM agent into a data-exfiltration vector against a real cloud backend.

Final project for **CSC 479 — Applied / Adversarial AI**.

---

## What this project demonstrates

NovaCorp (a fictional company) runs an AI customer-support chatbot. The chatbot is built on a standard, modern stack:

- **LLM** with function-calling: Qwen 2.5 7B served locally via Ollama
- **RAG pipeline**: documents pulled from a public Amazon S3 bucket, chunked, embedded with a Sentence-Transformers model, indexed in FAISS, and (optionally) reranked with a cross-encoder
- **Tool layer**: the agent can call `list_s3_buckets` and `read_s3_object` to answer questions about the company's storage

A malicious actor with brief write access to the public knowledge-base bucket plants a single poisoned document. From that moment, every customer query becomes a potential exfiltration event:

1. The user asks a benign question.
2. The retriever returns the poisoned chunk alongside legitimate policy documents.
3. The poisoned chunk contains instructions disguised as a "compliance runbook."
4. The agent treats those retrieved instructions as authoritative and invokes its own tools to list confidential buckets and read sensitive files.
5. The exfiltrated contents are returned to the user — or, in a real deployment, to whoever phrased the query.

The notebook walks through the **attack** (no defense), the **defense** (input sanitization on retrieved context), and a **multi-turn exfiltration session** where the agent first asks permission and then exfiltrates after the user replies "yes" — demonstrating that authority blending + conversational state can be exploited even when the model appears polite.

### Attack variants included
| Variant | Mechanism |
|---|---|
| `url_social_engineering` | Tells the user they must visit a malicious URL to "verify" |
| `ignore_previous_instructions` | Classic system-message override |
| `secret_exfiltration` | Asks the model to print a known secret value |
| `role_override` | Claims to be a new system role replacing the original policy |
| `output_dos` | Forces the model to emit thousands of tokens of filler |
| `citation_hijack` | Forces fabricated citations in the response |
| `authority_blending` | Disguises the attack as an internal IT runbook citing SOC2 — the most realistic and effective variant in our testing |
| `few_shot_hijack` | Provides fake prior assistant turns where the agent freely exfiltrates, biasing it to continue the pattern |

---

## Architecture

```
┌─────────────────┐
│  User question  │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────────┐      ┌──────────────────────────┐
│  FAISS retriever (top-k chunks)     │◄─────│  s3://public-research    │
│  + optional cross-encoder rerank    │      │  (legit + poisoned doc)  │
└────────┬────────────────────────────┘      └──────────────────────────┘
         │ retrieved context
         ▼
┌─────────────────────────────────────┐
│  Qwen 2.5 7B  (Ollama, local)       │
│  System prompt: "only public bucket"│
│  Tools: list_s3_buckets,            │
│         read_s3_object              │
└────────┬────────────────────────────┘
         │ tool_calls
         ▼
┌─────────────────────────────────────┐      ┌──────────────────────────┐
│  boto3 — executes tool calls        │─────►│  s3://confidential-hr    │
│  against real AWS S3                │      │  (employees.csv,         │
└────────┬────────────────────────────┘      │   api_keys.txt)          │
         │                                    └──────────────────────────┘
         ▼
   Exfiltrated content returned to the user
```

---

## Repository layout

```
.
├── CSC479_E1_P3_P5_RAG_FAISS_Reranker.ipynb   # main demo notebook (29 cells)
├── build_notebook.py                           # generator script for the notebook
├── cloud_access_policy.txt                     # legit corpus doc (NC-SEC-POL-014)
├── s3_usage_guidelines.txt                     # legit corpus doc (NC-CLD-GDL-007)
├── onboarding_checklist.txt                    # legit corpus doc (NC-HR-CHK-002)
├── .env                                        # bucket names, model, injection variant (gitignored)
├── .gitignore
└── README.md
```

The legit corpus documents are written as realistic NovaCorp internal policies — including AI-agent-specific clauses that the poisoned doc deliberately contradicts. This gives the model a "ground truth" to violate, which is what makes the attack visible.

---

## Setup (fresh clone)

### Prerequisites

- **OS:** Linux (tested on Ubuntu 24.04). macOS likely works; Windows via WSL2.
- **Python:** 3.10+ (tested on 3.12).
- **GPU:** NVIDIA GPU recommended for the embedder/reranker (CPU works but slower). Ollama itself runs on CPU or GPU.
- **AWS account:** with permission to create IAM users and S3 buckets.

### 1. Clone and create a virtual environment

```bash
git clone <your-fork-url> Adversial-AI-Cloud-Storage-Demo
cd Adversial-AI-Cloud-Storage-Demo

python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip wheel setuptools
```

### 2. Install Python dependencies

PyTorch first (CUDA build for NVIDIA, CPU build otherwise):

```bash
# NVIDIA CUDA 12.4
pip install --index-url https://download.pytorch.org/whl/cu124 torch

# OR CPU-only
pip install --index-url https://download.pytorch.org/whl/cpu torch
```

Then everything else:

```bash
pip install \
  transformers sentence-transformers faiss-cpu bitsandbytes accelerate \
  huggingface_hub safetensors sentencepiece protobuf \
  boto3 ollama \
  pyyaml python-dotenv \
  ipykernel jupyter ipywidgets
```

### 3. Install and configure the AWS CLI

The apt package is gone in Ubuntu 24.04+; install AWS CLI v2 directly:

```bash
cd /tmp
curl -fsSL "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip -q awscliv2.zip
sudo ./aws/install
aws --version
rm -rf awscliv2.zip aws
cd -
```

Create an IAM user in the AWS console, attach **AmazonS3FullAccess** (demo-only — narrow this for any real use), generate an access key pair, then:

```bash
aws configure
# Access Key ID:     <paste>
# Secret Access Key: <paste>
# Region:            us-east-1
# Output:            json

aws sts get-caller-identity   # smoke test — must succeed before continuing
```

Credentials live in `~/.aws/credentials`. **Do not put them in `.env`** — boto3 finds them automatically and `.env` files leak too easily.

### 4. Create the S3 buckets

Bucket names must be globally unique. Edit the names below if `sureh-...` is taken, then update `.env` to match.

```bash
aws s3 mb s3://sureh-public-research-data    --region us-east-1
aws s3 mb s3://sureh-confidential-hr-records --region us-east-1
```

Both buckets should already have safe defaults (Block Public Access on, Object Ownership = bucket-owner-enforced, SSE-S3 encryption). Verify in the console if you want.

### 5. Upload the corpus

Upload the three legitimate policy documents to the public bucket:

```bash
aws s3 cp cloud_access_policy.txt    s3://sureh-public-research-data/
aws s3 cp s3_usage_guidelines.txt    s3://sureh-public-research-data/
aws s3 cp onboarding_checklist.txt   s3://sureh-public-research-data/
```

Create the "sensitive" files locally, then upload to the confidential bucket:

```bash
printf 'name,ssn,salary\nJohn Doe,123-45-6789,95000\nJane Smith,987-65-4321,102000\n' \
  > employees.csv
printf 'AWS_SECRET=AKIAIOSFODNN7EXAMPLE\nDB_PASSWORD=hunter2\nSTRIPE_KEY=sk-live-abc123\n' \
  > api_keys.txt

aws s3 cp employees.csv s3://sureh-confidential-hr-records/
aws s3 cp api_keys.txt  s3://sureh-confidential-hr-records/
```

> The fake credentials and PII above are **not real**. They exist only to make the exfiltration visible in the demo output. Both files are gitignored.

### 6. Install and start Ollama

```bash
curl -fsSL https://ollama.com/install.sh | sh
ollama pull qwen2.5            # ~4.7 GB
ollama serve &                 # only if not already running as a service
ollama list                    # verify qwen2.5 is present
```

Other models that work with the tool-calling loop: `llama3.1`, `qwen2.5:7b-instruct`. Smaller models like `llama3.2:3b` are easier to jailbreak — useful if `qwen2.5` shrugs the attack off.

### 7. Configure `.env`

Copy or edit the existing `.env`:

```ini
# Ollama
OLLAMA_MODEL=qwen2.5

# AWS
AWS_REGION=us-east-1
S3_PUBLIC_BUCKET=sureh-public-research-data
S3_PRIVATE_BUCKET=sureh-confidential-hr-records

# Demo — change INJECTION_VARIANT to swap attacks
INJECTION_VARIANT=authority_blending
INJECTION_URL=www.fakepoizzzzzzon.com/8675309
SECRET_KEY=sk-DO_NOT_LEAK-12345
```

### 8. Open the notebook

VS Code:
1. Open `CSC479_E1_P3_P5_RAG_FAISS_Reranker.ipynb`.
2. Kernel picker (top-right) → select `.venv/bin/python`.

CLI:
```bash
.venv/bin/jupyter lab
```

---

## Running the demos

Run cells top-to-bottom. The interesting cells:

| Cell | What it does |
|------|---|
| 1–2  | Imports, config (reads `.env`) |
| 3    | Initializes boto3, prints visible buckets — your AWS smoke test |
| 4    | Loads legitimate docs from the public S3 bucket |
| 5    | Defines all 8 poison variants and selects the one from `.env` |
| 6–7  | Chunks the corpus, builds the FAISS index |
| 8    | Defines the `retrieve()` function |
| 9    | (Optional) cross-encoder reranker |
| 10   | Defines `TOOLS`, `execute_tool()`, `sanitize_context()`, `run_agent()` |
| 11   | **ATTACK** — single-turn run with no sanitization |
| 12   | **DEFENSE** — same query, sanitization on |
| 11b  | **Multi-turn exfiltration** — `ChatSession` class; sends the query, then replies "yes" to whatever the agent asks |
| 13   | Cleanup helpers (delete buckets) |

### Swapping attacks

Change `INJECTION_VARIANT` in `.env` and re-run cells 5 onward (or restart the kernel).

### Tuning if the attack doesn't fire

- Make the query semantically vaguer (`"Help me with my account"`) so the poisoned chunk ranks higher.
- Increase `top_hits = candidates[:5]` in cell 11.
- Print `top_hits` titles to confirm the poisoned chunk is actually being passed to the model.
- Try `OLLAMA_MODEL=llama3.2:3b` — smaller models are more easily steered by injected instructions.

---

## Cleanup (avoid AWS charges)

After the demo, run cell 13 with the `delete_bucket(...)` lines uncommented, or:

```bash
aws s3 rb s3://sureh-public-research-data    --force
aws s3 rb s3://sureh-confidential-hr-records --force
```

If you no longer need the IAM user, deactivate and delete its access key in the IAM console as well.

---

## Security notes (read before running)

- **The "confidential" data is fake.** Do not put real PII or real secrets in the demo bucket.
- **Use a dedicated demo IAM user.** Do not run this notebook against credentials that have access to anything you care about. The whole point is that the agent will exfiltrate data you give it access to.
- **Run on a private network.** The notebook makes outbound calls to AWS and (if you configure remote Ollama) to the model server. Don't expose this on a public IP.
- **The poisoned documents are real prompt-injection payloads.** They are limited to the corpus inside this repo — do not reuse them as templates against systems you don't own.
- **Rotate any credentials that briefly appeared in `.env`.** If you ever pasted real keys into `.env` (even commented out), treat them as compromised: deactivate them in IAM and create new ones.

---

## Acknowledgements

- Course: **CSC 479 — Applied / Adversarial AI**
- Models: Qwen 2.5 (Alibaba), all-MiniLM-L6-v2 + ms-marco-MiniLM-L6-v2 (Sentence-Transformers / cross-encoder)
- Inspired by published research on indirect prompt injection in RAG systems and agentic LLM tool-use vulnerabilities.
