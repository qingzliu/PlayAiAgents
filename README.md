## PlayAiAgents – Vibe Coding and AI Agents

This mini-tutorial is for **Prof. Veera Baladandayuthapani** (and collaborators) to quickly **play with modern AI agents and “vibe coding”** for data science and biostatistics.  
The goal is **not** to build a huge framework, but to give a few concrete patterns that are easy to try and extend.

### Who this is for

- **Biostatisticians and data scientists** who already use Python/R and want to see what “agentic AI” can really do beyond a simple chat window.
- **Developers** who want examples of using Cursor and a small Python agent on real research-style tasks.

You can skim the ideas here, then explore more formal coverage of agentic patterns in Andrew Ng’s Agentic AI course \([Agentic AI – DeepLearning.AI](https://learn.deeplearning.ai/courses/agentic-ai/information)\).

---

### 1. What is “vibe coding” and agentic AI?

- **Vibe coding** (as we use it informally) means:
  - Working inside an IDE (like Cursor) with an AI partner.
  - Letting the AI help with **planning, coding, refactoring, running tests**, and explaining results.
  - Iterating quickly based on “feel” and feedback instead of writing long design documents first.

- **Agentic AI** means:
  - The model is not just answering one prompt, but **taking multiple steps** toward a goal.
  - It can call **tools** (code, APIs, web search, file system, browser) and use their results.
  - It can **reflect on its own output**, revise, and follow a rough plan.

Traditional “ChatGPT in a browser tab”:

- Cannot see your local files by default.
- Cannot run your unit tests or open a browser or modify code directly.
- Mostly a **single message in, single response out** workflow.

Cursor (and similar IDE-native agents like Cloud Code, GitHub Copilot Agents, etc.):

- Work **inside your project** with access to the codebase.
- Can **edit files, run tests, search, and refactor** under your supervision.
- Can behave more like a **junior collaborator** than a Q&A bot.

This repository is about **playing with those agentic ideas** in the context of biostatistics and data-centric research.

---

### 2. Case study A – Talking with the Cursor agent about a biostatistics analysis

This is a **no-setup** example: you only need Cursor and some data.

#### 2.1. Scenario

Suppose you have a dataset with:

- **Patients** with baseline covariates (age, stage, biomarkers, treatment arm).
- **Outcomes** such as survival time and censoring indicator.

You want to:

- Do a **quick exploratory analysis** (EDA).
- Build a **small web UI** (e.g., Streamlit or a simple React front end) to explore survival curves by subgroup.
- Keep the code tidy and reproducible so students or collaborators can re-run it.

#### 2.2. How to use Cursor for this

1. **Open the repo** (or a folder with your data files) in Cursor.
2. **Create a new file** `eda_survival.py` and paste or sketch a minimal outline (or just describe what you want).
3. **Ask the Cursor agent**, for example:

   > “You are helping with a survival analysis EDA on `data/clinical.csv`  
   > Variables: `time`, `event`, `treatment`, `age`, `stage`.  
   > Write Python code using `pandas` and (optionally) `lifelines` to:  
   > - Load the data  
   > - Plot Kaplan–Meier curves by treatment arm  
   > - Output basic summary tables (N, events, median survival)  
   > Please write clear, reusable functions and a `main()` that I can run from the command line.”

4. Let the agent **generate the initial code**, then ask follow‑ups:
   - “Refactor this into separate modules `data_io.py`, `analysis.py`, `plots.py`.”
   - “Add a command‑line argument for which subgroup variable to stratify on.”
   - “Explain what assumptions the Cox model is making here.”

5. Optionally, ask Cursor to:
   - Scaffold a **Streamlit app** that reads the same CSV and shows:
     - Upload widget for a CSV.
     - Drop‑downs to choose outcome and covariates.
     - KM plots and hazard ratio estimates.

The key idea: you are **steering the analysis at a high level**, and Cursor is doing the repetitive coding and refactoring while staying inside your project.

---

### 3. Case study B – A tiny Python “research agent” for summarizing datasets

In this section we outline a **simple Python agent** your professor can run locally.  
It is intentionally small and focused:

- Input: a folder of `.csv` files, each representing a study or dataset.
- Task:
  - Compute simple descriptive statistics.
  - Ask an LLM to turn those numbers into **short, readable summaries**.
  - Write everything into a single **Excel workbook** for quick review.

You do not need any specialized libraries beyond `openai` and `pandas`.  
The code below can live in (for example) `agents/biostats_dataset_summarizer.py`.

#### 3.1. Example agent code (Python)

```python
import os
from pathlib import Path
from typing import List, Dict, Any

import pandas as pd
from openai import OpenAI


client = OpenAI()  # reads OPENAI_API_KEY from environment


SYSTEM_PROMPT = """
You are a senior biostatistician.
Given simple summary statistics for multiple datasets, you write
short, precise, and non-hyped descriptions in 2-4 sentences per dataset.

Audience: other statisticians and clinical researchers.
Avoid over-claiming causality; describe patterns, not certainties.
"""


def load_csv_files(folder: Path) -> Dict[str, pd.DataFrame]:
    """Load all .csv files in a folder into a dict name -> DataFrame."""
    data = {}
    for path in folder.glob("*.csv"):
        df = pd.read_csv(path)
        data[path.stem] = df
    return data


def quick_numeric_summary(df: pd.DataFrame, outcome_cols: List[str]) -> pd.DataFrame:
    """Return a compact numeric summary table for selected outcome columns."""
    numeric_df = df[outcome_cols].select_dtypes("number")
    summary = numeric_df.agg(["count", "mean", "std", "min", "max", "median"]).T
    summary.reset_index(inplace=True)
    summary.rename(columns={"index": "variable"}, inplace=True)
    return summary


def call_model_for_summary(study_name: str, summary_table: pd.DataFrame) -> str:
    """Ask the LLM for a natural-language summary of one study."""
    csv_text = summary_table.to_csv(index=False)
    user_prompt = f"""
Study name: {study_name}

Here is a table of summary statistics:
{csv_text}

Write 2–4 sentences summarizing key patterns.
Mention which variables are higher or lower on average and any
obvious differences in spread. Avoid p-values or formal claims.
"""

    response = client.chat.completions.create(
        model="gpt-4.1-mini",
        messages=[
            {"role": "system", "content": SYSTEM_PROMPT},
            {"role": "user", "content": user_prompt},
        ],
        max_tokens=250,
        temperature=0.3,
    )

    return response.choices[0].message.content.strip()


def run_agent(
    input_folder: str,
    outcome_cols: List[str],
    excel_path: str = "dataset_summaries.xlsx",
) -> None:
    folder = Path(input_folder)
    if not folder.exists():
        raise FileNotFoundError(f"Folder not found: {folder}")

    all_dfs = load_csv_files(folder)
    rows: List[Dict[str, Any]] = []

    for study_name, df in all_dfs.items():
        summary_table = quick_numeric_summary(df, outcome_cols)
        narrative = call_model_for_summary(study_name, summary_table)

        # Store one row per variable for the Excel sheet
        for _, row in summary_table.iterrows():
            rows.append(
                {
                    "study": study_name,
                    "variable": row["variable"],
                    "count": row["count"],
                    "mean": row["mean"],
                    "std": row["std"],
                    "min": row["min"],
                    "max": row["max"],
                    "median": row["median"],
                    "llm_summary": narrative,
                }
            )

    result_df = pd.DataFrame(rows)
    result_df.to_excel(excel_path, index=False)
    print(f"Wrote {len(result_df)} rows to {excel_path}")


if __name__ == "__main__":
    # Example usage:
    #
    #   export OPENAI_API_KEY=...  # or set in your environment
    #   python agents/biostats_dataset_summarizer.py
    #
    run_agent(input_folder="data", outcome_cols=["time", "event"])
```

This is intentionally minimal:

- It **stays local** except for the LLM call.
- It **summarizes multiple datasets at once** into a single Excel file.
- You can easily extend it to:
  - Add covariate summaries per treatment arm.
  - Call R scripts or other tools from Python.
  - Save extra sheets with stratified analyses.

---

### 4. How to run the example agent

After you clone this repo (or copy the files into a folder):

1. **Create and activate a virtual environment** (optional but recommended):

   ```bash
   cd PlayAiAgents
   python -m venv .venv
   source .venv/bin/activate  # On Windows: .venv\Scripts\activate
   ```

2. **Install dependencies**:

   ```bash
   pip install -r requirements.txt
   ```

3. **Set your API key** (for OpenAI or another provider compatible with the `openai` Python client):

   ```bash
   export OPENAI_API_KEY="sk-..."  # or use a .env file
   ```

4. **Prepare some CSVs** in a `data/` folder, e.g.:

   - `data/trial_a.csv`
   - `data/trial_b.csv`

   Each with numeric columns such as `time`, `event`, or other outcomes of interest.

5. **Create the `agents/` folder and save the script**:

   - In Cursor, ask the agent:

     > “Create a file `agents/biostats_dataset_summarizer.py` and paste the example agent code from the README into it.  
     > Make sure it runs as a script when I call `python agents/biostats_dataset_summarizer.py`.”

6. **Run the agent**:

   ```bash
   python agents/biostats_dataset_summarizer.py
   ```

You should see an `dataset_summaries.xlsx` file appear, which you can open in Excel or R.

---

### 5. Key concepts: tokens, APIs, skills, MCP, multi‑agent workflows

- **Tokens**:
  - Models do not think in words; they think in **tokens** (sub‑word pieces).
  - Every prompt and response consumes tokens; this affects **cost, latency, and context length**.
  - When building agents, it is useful to **keep prompts compact** and summarize or prune history.

- **APIs**:
  - The `openai` client here calls a **remote API** that runs the model.
  - Your code sends: model name, messages, options (temperature, max tokens).
  - The response includes text, usage information (tokens used), and sometimes **tool calls**.

- **Skills** (in tools like Cursor):
  - A **skill** is a reusable instruction set or behavior profile.
  - For example, “You are a biostatistics reviewer; always check proportional hazards assumptions.”
  - You can store such skills and **reuse them across projects** to keep your agents consistent.

- **MCP (Model Context Protocol)**:
  - MCP is an **open protocol** for connecting models to tools and data sources in a structured way.
  - In environments like Cursor, MCP can expose:
    - File system access.
    - Browsers (for web automation and testing).
    - Domain‑specific tools (e.g., internal APIs, databases).
  - This turns an LLM into an **operator that can call many tools** safely and in a standardized format.

- **Multi‑agent and orchestration frameworks**:
  - Beyond single agents, you can orchestrate **multiple specialized agents**:
    - One agent plans the workflow.
    - Another does data cleaning and quality checks.
    - Another focuses on model fitting and diagnostics.
  - There are many ways to orchestrate this:
    - Built‑in orchestration in providers’ SDKs.
    - Libraries like LangChain, LlamaIndex, and other open‑source orchestrators.
  - Andrew Ng’s Agentic AI course discusses **reflection, tool use, planning, and multi‑agent patterns** in more depth \([Agentic AI – DeepLearning.AI](https://learn.deeplearning.ai/courses/agentic-ai/information)\).

---

### 6. Suggested experiments for a biostatistics course or lab meeting

Here are some concrete ideas that might be interesting for Prof. Veera Baladandayuthapani and colleagues:

- **Reproducible teaching examples**:
  - Use the agent to generate multiple small synthetic datasets with different survival patterns.
  - Ask Cursor to create teaching notebooks (Jupyter or Quarto) that compare:
    - KM curves.
    - Cox models with and without violations.
    - Alternative models (e.g., AFT, flexible parametric).

- **Rapid literature triage (with caution)**:
  - Combine a simple PubMed or web search script with an LLM summarizer.
  - Use the agent to produce **structured evidence tables** (study, sample size, endpoint, key results).
  - Manually verify any claims; treat the LLM as a **drafting assistant**, not an oracle.

- **Model diagnostics assistant**:
  - Build a small agent that:
    - Reads model outputs (coefficients, residual plots, Schoenfeld tests).
    - Suggests which diagnostics to examine and why.
    - Drafts short paragraphs for methods/results sections.

Each of these can start from the simple dataset‑summarizing agent above and then be extended step by step with the help of the Cursor agent.

---

### 7. Where to go next

- **Deep dive on patterns**: Andrew Ng’s course on agentic workflows \([Agentic AI – DeepLearning.AI](https://learn.deeplearning.ai/courses/agentic-ai/information)\).
- **More tooling**:
  - Explore MCP‑enabled tools in Cursor (browser, file system, external APIs).
  - Experiment with multiple “roles” or skills (e.g., “statistical reviewer”, “clinical collaborator”).
- **Biostatistics‑specific extensions**:
  - Add R interop (call R scripts from Python and let the agent manage both).
  - Use agents to help design simulation studies, power calculations, and sensitivity analyses.

The intention of this repo is to be a **playground**: you can modify the examples, add your own agents, and use Cursor to help you push these ideas toward real research workflows.

