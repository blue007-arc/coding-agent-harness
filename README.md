# Coding Agent Harness 🛠️🤖

[![Python 3.10+](https://img.shields.io/badge/python-3.10+-blue.svg)](https://www.python.org/downloads/)
[![LangGraph](https://img.shields.io/badge/Orchestrator-LangGraph%201.0-orange)](https://github.com/langchain-ai/langgraph)
[![E2B Sandbox](https://img.shields.io/badge/Sandbox-E2B-purple)](https://e2b.dev)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Author](https://img.shields.io/badge/Maintainer-blue007--arc-blue)](https://github.com/blue007-arc)

An autonomous software engineering agent built as a robust engineering harness: multi-agent planning, repository exploration, code generation with mechanical diffs, strict human approval gates, isolated sandboxed test execution (E2B), and closed-loop iterative repair.

Developed and maintained by **[Sakshi Pandey](https://github.com/blue007-arc)** (`231FA04H01@gmail.com`).

---

## 🏗️ Architectural Overview

```
                          ┌────────────────────────┐
                          │     User Objective     │
                          └───────────┬────────────┘
                                      ▼
                          ┌────────────────────────┐
                          │     Planner Agent      │  (Drafts phased repair plan)
                          └───────────┬────────────┘
                                      ▼
                          ┌────────────────────────┐
                          │     Explorer Agent     │  (Inspects AST / file trees)
                          └───────────┬────────────┘
                                      ▼
                          ┌────────────────────────┐
                          │      Coder Agent       │  (Mechanical difflib generation)
                          └───────────┬────────────┘
                                      ▼
                      ┌────────────────────────────────┐
                      │  Diff Review Gate (LangGraph)  │ ⏸️ PAUSE (interrupt)
                      └───────┬────────────────┬───────┘
                     Approved │                │ Rejected with feedback
                              ▼                └───────────────────────┐
                      ┌────────────────┐                               │
                      │  Apply Diffs   │ (Writes to disk)              │
                      └───────┬────────┘                               │
                              ▼                                        │
                      ┌────────────────┐                               │
                      │   E2B Sandbox  │ (Executes pytest in VM)       │
                      └───────┬────────┘                               │
                              │                                        │
                 Tests Pass?  ├─────────────► Tests Fail ──────────────┘
                              │               (Feeds stderr back into coder)
                              ▼
                      ┌────────────────┐
                      │  Complete (✅)  │
                      └────────────────┘
```

---

## 🚀 Key Features

- **🛡️ Human-Gated File Edits**: The coder model never directly writes to the workspace. It calls `propose_edit`, generating unified diffs. LangGraph's native `interrupt()` halts execution until you explicitly approve or reject changes.
- **🔄 Closed-Loop Feedback on Rejection**: Rejection reasons are fed directly into the coder's state, preventing blind retries and enforcing iterative improvement.
- **📦 Isolated Sandboxed Execution**: Test suites execute inside ephemeral [E2B](https://e2b.dev) micro-VMs—zero untrusted code runs on your host machine.
- **🧠 Role-Separated Architecture**: Distinct agents for Planning, Code Exploration, Code Generation, and Test Execution.
- **🖥️ Dual Interfaces**: High-speed interactive terminal CLI (via Rich) and a visual patch reviewer web application (via Streamlit).
- **🧪 Seeded Regression Scenario**: Comes pre-packaged with a reproducer workspace (`workspace/cart.py`) containing 3 real failing test cases.

---

## 🛠️ Tech Stack

- **Orchestration**: [LangGraph](https://github.com/langchain-ai/langgraph) (`StateGraph`, `interrupt()`, `MemorySaver`)
- **Agent Framework**: [LangChain](https://python.langchain.com/)
- **LLM Inference**: OpenAI / [Nebius Token Factory](https://tokenfactory.nebius.com) / Anthropic
- **Execution Sandbox**: [E2B Cloud Sandboxes](https://e2b.dev)
- **Terminal UI**: [Rich](https://rich.readthedocs.io/)
- **Web UI**: [Streamlit](https://streamlit.io/)

---

## 📁 Repository Structure

```text
coding-agent-harness/
├── app.py                  # Streamlit visual diff reviewer UI
├── cli.py                  # Rich terminal CLI runner + interactive walkthrough
├── demo_data.py            # Seeded ticket and baseline failing patch
├── graph.py                # LangGraph StateGraph & agent node definitions
├── tools.py                # propose_edit, read_file, list_dir tools
├── sandbox.py              # E2B cloud sandbox pytest integration
├── workspace/              # Working repository inspected and modified by agents
│   ├── cart.py             # Target codebase with bugs
│   └── tests/test_cart.py  # Pytest suite
├── pyproject.toml          # Project dependencies & packaging
├── .gitignore              # Git ignore rules
└── .env.example            # Environment variable template
```

---

## ⚡ Quick Start

### 1. Prerequisites
- Python 3.10 or higher
- [uv](https://github.com/astral-sh/uv) (recommended) or `pip`
- An LLM API key ([Nebius Token Factory](https://tokenfactory.nebius.com) or [OpenAI](https://platform.openai.com))
- An [E2B](https://e2b.dev) API key for sandboxing

### 2. Installation

```bash
# Clone the repository
git clone https://github.com/blue007-arc/coding-agent-harness.git
cd coding-agent-harness

# Install dependencies with uv
uv sync

# Or with pip
pip install -e .
```

### 3. Environment Configuration

```bash
cp .env.example .env
```

Edit `.env` with your credentials:

```env
NEBIUS_API_KEY="your_nebius_api_key"
NEBIUS_MODEL="Qwen/Qwen3-30B-A3B"
E2B_API_KEY="your_e2b_api_key"
MAX_ITERATIONS="4"
```

---

## 💻 Usage

### Terminal Mode (Interactive CLI)
Run the LangGraph crew directly in the terminal:

```bash
uv run python cli.py
```

Execution will stream each agent's reasoning, print the unified diffs, and prompt for human review:
```text
Approve this patch? [y/n]
```
If you reject (`n`), enter your feedback (e.g. *"handle zero division silently instead of raising an error"*), and watch the coder revise its patch.

### Web Dashboard Mode (Streamlit)
Launch the visual diff review application:

```bash
uv run streamlit run app.py
```

Open `http://localhost:8501` to view pending changes side-by-side, inspect test status, and approve patches with one click.

---

## 🚢 Deployment & CI/CD

### Docker Deployment
You can package and run the Streamlit reviewer via Docker:

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY . /app
RUN pip install --no-cache-dir -e .
EXPOSE 8501
CMD ["streamlit", "run", "app.py", "--server.port=8501", "--server.address=0.0.0.0"]
```

Build and run:
```bash
docker build -t coding-agent-harness .
docker run -p 8501:8501 --env-file .env coding-agent-harness
```

---

## 👤 Author & Maintainer

**Sakshi Pandey**
- GitHub: [@blue007-arc](https://github.com/blue007-arc)
- Email: [231FA04H01@gmail.com](mailto:231FA04H01@gmail.com)

---

## 📄 License
This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
