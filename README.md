# McGill Course Advising Agent

A single-agent system built with the [OpenAI Agents SDK](https://openai.github.io/openai-agents-python/) that helps a McGill undergraduate choose courses, surfaces prerequisite and policy risks, and flags when human advisor approval is required.


---

## What it does

- Searches a fake 10-course McGill catalog by course code or keyword
- Checks prerequisites (and catches unknown or already-completed courses)
- Flags policy risks: credit overload, advisor-approval threshold, GPA minimum for 400-level courses, too many 400-level courses
- Returns a typed `CourseRecommendation` object
- Blocks unsafe input with an **input guardrail** and re-validates every answer with an **output guardrail**

---

## Project structure

```
mcgill_advisor_agent.ipynb   # all code: fixtures, tools, guardrails, agent, tests, evals
README.md
reflection.docx              # what failed, what improved, what is still risky
requirements.txt
.gitignore
evidence/                    # screenshots of runs
```

---

## Setup

Requires Python 3.10+ and an OpenAI API key with billing enabled ([platform.openai.com/api-keys](https://platform.openai.com/api-keys)).

### Google Colab

1. Upload `mcgill_advisor_agent.ipynb` to [colab.research.google.com](https://colab.research.google.com)
2. **Runtime -> Run all**. When Step 2 runs, you are prompted for your API key.

### Local

```bash
pip install -r requirements.txt
export OPENAI_API_KEY=sk-your-key-here      # Windows PowerShell: $env:OPENAI_API_KEY="sk-..."
jupyter notebook mcgill_advisor_agent.ipynb
```

> **Never paste your key into the notebook or commit it.** The notebook reads `OPENAI_API_KEY` from the environment or prompts for it. `.gitignore` excludes `.env` files.

| Variable | Description |
|---|---|
| `OPENAI_API_KEY` | Required for Steps 8, 10 and 11 (anything that calls the model) |

The model is set by `MODEL` at the top of Step 3 (default `gpt-4.1-mini`).

---

## Testing

There are two layers, and they answer different questions.

**Step 9 - offline tests (no API key, no network).** Plain `assert`s over the tool logic, the input-guardrail regex and the output validator. These are deterministic and should always pass. Expected output:

```
All offline tests passed.
```

**Step 10 - live evals (calls the API).** Eight cases run against the real model, one run each. Because the model is non-deterministic, a case can occasionally flip between runs; re-run before concluding something is broken. Expected output ends with:

```
RESULTS: 8 passed, 0 failed out of 8 cases
```

| ID | Type | What it checks |
|---|---|---|
| eval_01 | Happy path | Eligible course (COMP 251) is recommended, no approval needed (15 credits) |
| eval_02 | Happy path | Topic search ("software design") finds COMP 303 |
| eval_03 | Approval | COMP 251 + COMP 302 lands on exactly 18 credits -> advisor approval required |
| eval_04 | Edge | Missing prerequisites (COMP 360) -> not recommended, `missing_prereq` risk |
| eval_05 | Edge | **Input guardrail** fires on an explicit "25 credits" request |
| eval_06 | Edge | Nonexistent course (COMP 999) -> not recommended, `not_in_catalog` risk |
| eval_07 | Edge | Overload with **no credit number stated** (3 courses = 21 credits) -> stays under the cap; if the model overshoots, the **output guardrail** catches it |
| eval_08 | Edge | AI courses exist but the student lacks prerequisites -> COMP 424 not recommended |

---

## Agent design

### Tools (3 typed tools)

| Tool | Purpose |
|---|---|
| `search_courses` | Code lookup or token-based keyword search over the catalog |
| `check_prerequisites` | Course exists? Already completed? Prerequisites met? |
| `flag_policy_risk` | Credit cap, advisor-approval threshold, GPA and 400-level rules |

The logic lives in plain `*_impl` functions; the `@function_tool` wrappers only serialise to JSON. This is what makes the offline tests possible.

### Structured output

`CourseRecommendation` (Pydantic):
- `recommended_courses` - course codes safe to register
- `risks` - list of `CourseRisk` (`course_code`, `risk_type`, `description`, `blocking`)
- `advisor_approval_required` / `advisor_approval_reason`
- `summary`

`risk_type` is a `Literal` (`missing_prereq`, `credit_overload`, `gpa_too_low`, `policy_flag`, `not_in_catalog`, `already_completed`), so the model cannot invent new labels.

### Guardrails

| Name | Type | Behaviour |
|---|---|---|
| `CREDIT_OVERLOAD_BLOCK` | Input | If the request explicitly mentions more than 18 credits, the run is halted before the agent starts (`run_in_parallel=False`), so no tools are called. |
| `RECOMMENDATION_VALIDATOR` | Output | After the agent answers, plain code re-checks the recommendation: every course exists, is not already completed, has its prerequisites met, the total is within 18 credits, and `advisor_approval_required` matches policy. Any violation trips the guardrail. |

The input guardrail only sees explicit numbers ("25 credits"). The output guardrail is the backstop for requests that overload the schedule without saying so ("add these three courses").

### State strategy

| What | Where |
|---|---|
| Course catalog, prerequisite table, student profile, policy rules | In-memory dicts (fixtures). In production: a database. |
| Student profile | Embedded in the system prompt; `student_id` is passed to tools |
| Credit totals, prerequisite gaps, policy flags | Recomputed on every tool call - nothing cached |
| Conversation history | None. Every run is stateless. |

### Escalation rule

`advisor_approval_required` is true when:
- the resulting total is **17 or more credits** (and within the 18-credit cap), or
- the student has a GPA below 3.0 and the list includes a 400-level course.

The agent only *flags* this; nothing in this project routes to a human. A real system would consume that flag and open a ticket for an advisor. Over the 18-credit cap, courses are dropped rather than escalated.

---

## Evidence

`evidence/run_query_comp251.png` - a single query run.

`evidence/eval_run_before_fixes.png` - an early eval run (3/7). Most failures turned out to be bugs in the tests themselves; see `reflection.docx` for the diagnosis and what changed.

`evidence/eval_run_after_fixes.png` - the current eval run.


---

## Limitations

- Fictional single-student fixtures; no co-requisites, equivalencies, course-offering terms or real McGill rules
- No memory across sessions
- Output-guardrail failures raise an exception rather than retrying or repairing the answer
- Live evals run once per case and cover scoped questions, not open-ended advising
