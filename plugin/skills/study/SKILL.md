---
name: study
description: Use this skill when the user wants to read, study, analyze, or deeply understand a research paper (PDF).
disable-model-invocation: false
allowed-tools: Bash, Write, Edit, Read, WebSearch
---

# Paper Study Workflow

Invoke this skill with a paper PDF path.

**Language Detection**: Detect the user's language from their input and generate ALL materials in that language.
- Example: User says "我们学习一下这篇论文吧" → Generate materials in Chinese
- Example: User says "Let's study this paper" → Generate materials in English

---

# Core Philosophy

Primary Objective:
Facilitate deep conceptual understanding and research-level thinking.

Secondary Objective:
Create a structured, reusable paper knowledge system.

**Detail Level**: You must produce textbook-level detailed output. Never summarize briefly — the goal is that someone reading your output should NOT need to read the original paper to understand every technical detail.

---

# Step 0: Check Dependencies (First Run Only)

```bash
if [ ! -f "${CLAUDE_PLUGIN_ROOT}/.installed" ]; then
  echo "First run - installing dependencies..."
  cd "${CLAUDE_PLUGIN_ROOT}"
  npm install || exit 1
  touch "${CLAUDE_PLUGIN_ROOT}/.installed"
  echo "Dependencies installed!"
fi
```

Recommended:

* Node >= 18

---

# Step 1: Download and Parse PDF

Supports multiple input formats:

* **Local path**: `~/Downloads/paper.pdf`
* **Direct PDF URL**: `https://arxiv.org/pdf/1706.03762.pdf`
* **arXiv URL**: `https://arxiv.org/abs/1706.03762`

## Step 1a: Check input type and download if URL

```bash
USER_INPUT="<user-input>"

# Check if input is a URL (starts with http:// or https://)
if [[ "$USER_INPUT" =~ ^https?:// ]]; then
  # Download PDF from URL
  INPUT_PATH=$(node ${CLAUDE_PLUGIN_ROOT}/skills/study/scripts/download-pdf.cjs "$USER_INPUT")
else
  # Use local path directly
  INPUT_PATH="$USER_INPUT"
fi
```

For URLs, the download script will:
* Download PDFs to `/tmp/claude-paper-downloads/`
* Convert arXiv `/abs/` URLs to PDF URLs automatically
* Validate that URLs point to PDF files
* Return the local file path for processing

For local paths, use the path directly without downloading.

## Step 1b: Parse PDF

Extract structured information:

```bash
node ${CLAUDE_PLUGIN_ROOT}/skills/study/scripts/parse-pdf.js "$INPUT_PATH"
```

Output includes:

* title
* authors
* abstract
* full content
* githubLinks
* codeLinks
* tags (generated in Step 2.5)

Save to:

```
~/claude-papers-data/papers/{paper-slug}/meta.json
```

Copy original PDF:

```bash
cp <pdf-path> ~/claude-papers-data/papers/{paper-slug}/paper.pdf
```

Fallback:
If structured parsing fails, extract raw text and continue with degraded structure.

---

# Step 2: Assess Paper Before Generating Materials

Before generating any files, evaluate:

1. Difficulty Level

   * Beginner
   * Intermediate
   * Advanced
   * Highly Theoretical

2. Paper Nature

   * Theoretical
   * Architecture-based
   * Empirical-heavy
   * System design
   * Survey

3. Methodological Complexity

   * Simple pipeline
   * Multi-stage training
   * Novel architecture
   * Heavy mathematical derivation

This assessment determines explanation depth and the level of detail in generated materials.

---

# Step 2.5: Generate Exactly 2 Semantic Tags (Mandatory)

Before generating files, infer exactly 2 tags from semantic understanding of the paper.

Rules:

* Generate exactly 2 tags, no more and no less
* Tags must be distinct
* Each tag should be short (1-3 words)
* Avoid generic tags: `paper`, `research`, `ai`, `ml`
* Prefer one tag for problem/domain and one for method/core idea

Examples:

* `machine translation`, `self-attention`
* `3d detection`, `bev transformer`
* `protein folding`, `structure prediction`

Persist these 2 tags in both locations:

* `~/claude-papers-data/papers/{paper-slug}/meta.json` as `tags`
* `~/claude-papers-data/index.json` entry as `tags`

---

# Step 3: Generate Detailed Paper Review (`paper-review.md`)

**This is the most important step.** Create a comprehensive, textbook-level detailed review of the paper.

**Output path:**
```
~/claude-papers-data/papers/{paper-slug}/paper-review.md
```

## CRITICAL: Incremental Writing Strategy

The paper review is a long document. You MUST write it **section by section** to avoid write failures:

1. **First call**: Use `Write` to create the file with Section 1 (Background and Motivation) + Section 2 (Related Work)
2. **Second call**: Use `Edit` to append Section 3 (Method — Detailed Explanation) at the end of the file
3. **Third call**: Use `Edit` to append Section 4 (Experimental Setup) at the end of the file
4. **Fourth call**: Use `Edit` to append Section 5 (Experimental Results) at the end of the file
5. **Fifth call**: Use `Edit` to append Section 6 (Key Findings and Observations) at the end of the file

For each Edit append, match the last line of the existing content as `old_string` and include that line plus the new section as `new_string`. For example:
```
old_string: "[last line of current content]"
new_string: "[last line of current content]\n\n---\n\n## 3. Method — Detailed Explanation\n\n[new content...]"
```

**NEVER attempt to write the entire review in a single Write call — it WILL fail.**

## Writing Guidelines

You are writing a detailed Chinese (or user's language) review that serves as a **complete replacement for reading the original paper**. Follow these principles:

- Write like a textbook chapter, not a summary
- Never skip important details — if the paper mentions it, you should cover it
- Every formula must include an intuitive explanation of what it means and why it's designed that way
- Experimental results MUST include specific numbers from the paper (tables, metrics, scores)
- The target reader should be able to fully understand every technical detail without reading the original paper

## Required Structure

### 1. Background and Motivation (Write — creates file)
- Current state of the research field
- The specific problem being addressed
- Why existing approaches are insufficient (with concrete examples from the paper)
- What gap this paper fills

### 2. Related Work (Write — same call as Section 1)
- Briefly mention key prior works referenced in the paper
- Note: A more detailed analysis of the research landscape goes in `research-context.md`

### 3. Method — Detailed Explanation (Edit — append)
- **Overall framework**: High-level overview of the proposed approach, with ASCII architecture diagram if applicable
- **Each core component/module**: Describe in detail with:
  - Design motivation (why this component is needed)
  - Formal definition (formulas with variable explanations)
  - Intuitive explanation (what the formula/algorithm is actually doing)
  - Algorithm flow / pseudocode where applicable
- **Key implementation details**: Hyperparameters, training strategies, optimization tricks
- **How components work together**: The full pipeline from input to output

If the method section itself is very long (e.g. multiple complex components), split it further into multiple Edit calls — one per major component.

### 4. Experimental Setup (Edit — append)
- Datasets used (with sizes, splits, preprocessing)
- Evaluation metrics (with definitions if non-standard)
- Baselines and competing methods (list all)
- Implementation details (hardware, training time, batch size, learning rate, etc.)

### 5. Experimental Results (Edit — append)
- **Main results table(s)**: Reproduce key tables with actual numbers from the paper
- **Ablation studies**: What each component contributes (with numbers)
- **Analysis experiments**: Any additional analysis the paper provides
- **Visualization results**: Describe qualitative results, figures, case studies
- **Comparison with SOTA**: How does it compare to the best prior work

### 6. Key Findings and Observations (Edit — append)
- Interesting phenomena discovered during experiments
- Unexpected results or behaviors
- Insights the authors highlight

---

# Step 3.5: Generate Research Context (`research-context.md`)

Create a comprehensive analysis that positions this paper within the broader research landscape. This file is **independent of the paper's specific technical details** — it focuses on the "before and after" context.

**Output path:**
```
~/claude-papers-data/papers/{paper-slug}/research-context.md
```

## CRITICAL: Incremental Writing Strategy

Same as paper-review.md — write section by section:

1. **First call**: Use `Write` to create the file with Research Lineage section
2. **Second call**: Use `Edit` to append Core Insights section
3. **Third call**: Use `Edit` to append Reflection and Extension section
4. **Fourth call**: Use `Edit` to append Mental Model section

**NEVER attempt to write the entire file in a single Write call.**

## Research Lineage (Write — creates file)

Build a chronological narrative of how this research area evolved:

- **Timeline of key works**: List the most important prior papers in chronological order
- **For each prior work**: Explain its core idea, contribution, and limitations (2-3 sentences each)
- **Evolution logic**: How each work addressed limitations of its predecessors
- **How this paper fits**: What specific gaps or limitations of prior work does this paper address

**Sources for this section:**
1. The paper's own Related Work section (primary source)
2. Use `WebSearch` to look up additional context about key referenced papers if needed
3. **If the user mentions specific related papers**, you MUST include those papers in the lineage and explicitly analyze their logical relationship to this paper

### Core Insights (Edit — append)

- What **conceptual shift** does this paper introduce?
- Why does this approach work at a deeper level (not just "experiments show...")
- Detailed comparison table with the most closely related works (dimensions: method, performance, complexity, assumptions, etc.)

### Reflection and Extension (Edit — append)

- If you were to extend this paper, what directions would you pursue? (at least 3 concrete ideas)
- Which assumptions in the paper are fragile? Under what conditions might they break?
- Where might this approach fail in practice?
- What open questions remain?

### Mental Model (Edit — append)

- What prerequisite knowledge is needed to understand this paper?
- How to position this work in the broader research map (ASCII diagram encouraged)
- Recommended follow-up reading (3-5 papers)

---

# Step 4: Generate Interactive HTML Explorer

Create a single self-contained HTML file for interactively exploring the paper's core concepts.

**Output path:**
```
~/claude-papers-data/papers/{paper-slug}/index.html
```

## CRITICAL: Incremental Writing Strategy

HTML files can be very large. Write in stages:

1. **First call**: Use `Write` to create the file with the complete HTML skeleton (`<!DOCTYPE html>` through `</html>`), including all CSS styles and the structural HTML. Put a placeholder comment `<!-- JS_CONTENT -->` where the JavaScript will go.
2. **Second call**: Use `Edit` to replace `<!-- JS_CONTENT -->` with the actual JavaScript code.

If the JavaScript is still too large, split it into multiple Edit calls by using sequential placeholder comments (`<!-- JS_PART_1 -->`, `<!-- JS_PART_2 -->`, etc.).

**NEVER attempt to write the entire HTML file in a single Write call — it WILL fail for complex visualizations.**

## Requirements

* Single HTML file, all CSS/JS inline, zero external dependencies
* Uses **real data from the paper** (actual metrics, hyperparameters, comparisons) — never invent numbers
* Must work in a sandboxed iframe (no external fetches, no localStorage)

## Guidelines

Choose the interaction pattern that best fits the paper — architecture diagrams, parameter explorers, result dashboards, formula breakdowns, comparison matrices, etc. Let the paper's content dictate the format rather than forcing a fixed layout, focusing on the core ideas of the paper.

Every interactive control (slider, toggle, dropdown) should visibly change the visualization. Include brief explanatory text alongside interactive elements to teach concepts.

---

# Step 5: Update Index

**CRITICAL**: Read existing index.json first, then append the new paper. Never overwrite the entire file.

If index.json does not exist, create:

```json
{"papers": []}
```

Append new entry to the papers array:

```json
{
  "id": "paper-slug",
  "title": "Paper Title",
  "slug": "paper-slug",
  "authors": ["Author 1", "Author 2"],
  "abstract": "Paper abstract...",
  "year": 2024,
  "date": "2024-01-01",
  "tags": ["tag-1", "tag-2"],
  "githubLinks": ["https://github.com/..."],
  "codeLinks": ["https://..."]
}
```
**IMPORTANT**: The index.json file must be located at:
```
~/claude-papers-data/index.json
```

---

# Step 6: Relaunch Web UI

Invoke:

```
/claude-paper:webui
```

---

# Step 7: Interactive Deep Learning Loop

After all files are generated:

## Present Questions to User

Generate 15 questions about the paper:
* 5 basic (test understanding of core concepts)
* 5 intermediate (test understanding of method details and design choices)
* 5 advanced (test ability to critically analyze, extend, or compare)

Present ALL questions to the user at once and ask them to pick any they want to discuss.

## How to Handle Answers

**IMPORTANT: Do NOT create separate Q&A files.**

When the user discusses a question or asks for deeper explanation:

1. Provide the detailed answer in conversation
2. Then **insert the Q&A content into `paper-review.md`** at the most relevant location
3. Use the following format for inserted content:

```markdown
> **Q: [Question text]**
>
> [Detailed answer, integrated naturally into the surrounding context]
```

For example, if a question is about a specific formula, insert the Q&A right after the formula's explanation section in `paper-review.md`.

This makes `paper-review.md` a **living document** that grows richer with each Q&A interaction.

## Continuing the Loop

After answering questions, ask the user:
* What part is still unclear?
* Do you want deeper mathematical breakdown of any component?
* Do you want comparison with another specific paper?

If the user mentions a specific paper for comparison:
* Add the comparison to `research-context.md` in the Research Lineage section
* Use WebSearch if needed to gather information about the referenced paper

All deeper discussions should be integrated back into either `paper-review.md` or `research-context.md` — never create standalone files.

---
