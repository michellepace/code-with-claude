---
what: response from the prompt it sits alongside — as run in claude.ai
---

# Anthropic Eval Workshops — Taste vs Decomposition (Compared)

## 📚 1. Primer: the words you need

**📦 An eval** is short for *evaluation*. It is a **repeatable test** for an AI system. You give the agent the same jobs every time. You score the results the same way every time. Then you can change the agent and see if the score moves.

**📝 A task** is one job in the test. Example: *"Make a 5-slide overview of how transformers work."* A group of tasks is the **test set**.

**⚖️ A grader** is the code that turns one result into one score. It answers a single question. Example: *"How many slides had more than 300 characters of text?"*

Graders come in two kinds. Both workshops use both.

| Kind | How it scores |
| --- | --- |
| **Code grader** | Plain code. No AI. |
| **LLM judge** | A second AI model scores it. |

**🤖 An LLM judge** is a model you ask to grade. You give it a **rubric** — written rules for what a good answer looks like. You use a judge when code cannot check the thing. Beauty, tone, and coherence need a judge.

**🏁 A baseline** is your first score. You save it. Every later score is compared against it. The change is called the **delta**.

**🔁 A regression** is when a fix breaks something that used to work. A test set catches this.

**🧰 A harness** (or *runner*) is the plumbing. It runs every grader on every task and prints the table.

---

## 🎯 2. The point of each workshop

### 🎨 Workshop 1 — `eval-driven-agent-development`

The presenter is Felix Becker. The session title is *"Evals for Taste."* The stated promise is to teach you how to turn "this looks bad" into a number you can move.

**🩹 The problem it solves: you cannot grade taste by eye.**

The agent makes PowerPoint files. A deck can be ugly in many ways. Ugly is not a number. You cannot improve what you cannot measure.

**🛠 The fix:** build a **scorecard** with 12 columns. Each column is one grader. Now "ugly" becomes twelve numbers. Change the agent. Watch which numbers move.

### 📦 Workshop 2 — `agent-decomposition` (StockPilot)

You inherit a real, messy agent. It manages shop inventory. It has **12 tools**, **3 hardcoded subagents**, and a **402-line system prompt**. Every part was a sensible choice at the time. The whole thing is now the problem.

**🩹 The problem it solves: a working agent that is slow, costly, and untrustworthy.**

The agent is mostly correct. But one task takes **488 seconds** and **102 tool calls**. Another loses a number on the way through.

**🛠 The fix:** rip the agent apart and rebuild it. The eval tells you if you broke anything.

---

## 🔧 3. What you learn — the actual techniques

### 🎨 Workshop 1 techniques

**🧱 Make graders a data structure, not scattered code.**
Every grader is one object with the same shape. The fields are `name`, `kind`, `description`, and a `grade()` function. Adding a metric means appending one object to a list. The harness never changes.

**🚪 Put a "did it even work" grader first.**
The `Produced result` grader returns `missing`, `invalid`, or `ok`. It checks the file exists and unzips. Every other grader can stop early if this fails. This is the cheapest possible check.

**🔍 Parse the artefact for hard facts.**
A `.pptx` file is a zip of XML. The workshop unzips it and pulls out **shape count**, **picture count**, **character count**, **font sizes**, and **emoji count** per slide. No AI needed. From those raw facts it builds seven code graders:

| Grader | What it counts |
| --- | --- |
| `Slide count` | Slides in the deck |
| `Slides with image` | % with a picture |
| `Text-heavy slides` | Slides over 300 chars |
| `Cluttered slides` | Slides over 20 shapes |
| `Small-font slides` | Any font under 14pt |
| `Emoji count` | Emoji in whole deck |

**👁 Render to pixels, then use a vision judge.**
Code cannot see overlap or bad contrast. So the harness converts each slide to a **JPG** using LibreOffice in Docker. Then it sends the image to Claude Opus. The judge scores four things, each 0–5: **text**, **image**, **layout**, and **colour**.

**📐 Force the judge into a fixed shape.**
The judge call uses **structured outputs**. That means you attach a schema — a strict description of the reply format. Here it is a Zod schema demanding four integers from 0 to 5, plus a comment. The model cannot ramble. You get typed data with no parsing.

**📊 Fight score clustering.**
Judges tend to give everything a 4. The rubric ends with a direct order: *give scores across the full spectrum (0-5) instead of only good ones (3-5)*. This is a real, learnable prompt trick.

**💰 Share one model call across many graders.**
One vision call returns all four criteria. The call is **memoised** — the result is cached. So the four judge graders cost one batch of calls, not four.

**📄 Judge text without pixels too.**
`Title-body coherence` is a separate judge. It sends only the title and body text. It asks: *does the body deliver what the title promises?* Same 0–5 scale. No image attached.

**🌡 Give each number a direction.**
Each grader can declare a `scale` with `min`, `max`, and `good`. `good` can be `"high"`, `"low"`, or an **exact number**. Slide count uses `good: 5`, because every task asked for five slides. The table then colours cells red to green automatically.

**📌 Pin the baseline. Do not diff against the last run.**
You run `npm run eval -- --all --baseline` once. That writes `baseline.score.json`. All later runs show the delta against **that pinned point**, not against the previous run. The code comments explain why: diffing against the last run shows you noise, not progress.

**🎛 Test four different levers, not just one.**
The `solutions/` folder holds four agent versions. Each pulls a different lever.

| Round | The lever |
| --- | --- |
| `01-polish` | Add style rules to prompt |
| `02-diagram` | Add a capability (matplotlib) |
| `03-qa-loop` | Agent checks its own work |
| `04-model-swap` | Naive prompt, bigger model |

**🔬 Round 3 is the clever one.** The agent must render its own deck to images, look at them, find bugs, fix them, and re-render. The prompt says *"Assume there are problems. Your job is to find them."* This is the eval loop, moved inside the agent.

**⚗️ Round 4 is the control experiment.** It takes the *original naive prompt* and runs it on Opus instead of Sonnet. This separates the **prompt lever** from the **model lever**. Without it you would never know which one earned the gain.

---

### 📦 Workshop 2 techniques

**🧮 Compute the right answer at grading time.**
The expected answers are **not hardcoded**. The grader reads the source CSV files and recomputes the truth every run. Stock levels come from the latest date in `stock_levels.csv`. The reorder quantity is recalculated by formula. If the seed data changes, the eval still works.

**🪞 Make the grader mirror the policy.**
The reorder formula in `graders.py` is identical to the formula in the `reorder-policy` skill file. The eval and the agent's instructions state the same rule, in two places, in two languages. This is deliberate.

**👣 Grade side effects, not talk.**
The agent does not say *"I placed the order."* It appends a line to `purchase_orders.jsonl`. Three of these **sink files** exist: purchase orders, notifications, and ERP writes. The grader reads the files. Each run gets its own isolated folder, so parallel tasks never mix.

**🧩 Ten grader types, matched to what is being asked.**

| Grader | Checks |
| --- | --- |
| `exact_match` | One exact number |
| `set_match` | Right set of IDs |
| `numeric_tolerance` | Close enough, ±% |
| `action_taken` | A real side effect |
| `wall_budget` | Finished in time |
| `ranked_mention` | Item in top 3 rows |
| `regex_present` | A pattern appears |
| `efficiency` | Correct *and* cheap |
| `llm_judge` | Rubric, PASS/FAIL |
| `composite` | ANDs several checks |

**🟡 Add a third result: "correct but too expensive."**
Most evals are pass or fail. This one has **PASS**, **FAIL**, and **PASS-SLOW**. The score formula is `(PASS + ½·PASS-SLOW) / 12`. A right answer that burnt 5,000 tokens gets half credit. Cost becomes part of quality.

**🧷 Bundle checks with `composite`.**
Task F1 must pass three things at once: the purchase order was placed, the whole thing finished inside **270 seconds**, and the urgent SKU appeared in the **top 3 rows** of the output table. Any fail sinks the task. This is how you stop the agent trading speed for correctness.

**🎯 Use a regex to catch a specific known bug.**
Task F2 demands a numeric confidence score. The pattern is `confidence[^A-Za-z]{0,10}0?\.\d{1,2}`. It only matches a decimal like `0.41`.

**Why:** the old subagent was told to reply in prose with *"low / medium / high"* confidence. The orchestrator then read that prose and dropped the number. The regex catches exactly that failure. The task even carries its own error message: *"confidence stated qualitatively, not as a number."*

**📜 Write the `why`, not just the score.**
Every grader returns a pair: a status **and a reason**. Failures say things like `+43% vs target (anchored on mean?)` or `SKU-0183 not in top-3 of ranked output`. The reason is a hypothesis about the cause. The README says the **Why** column is the thing to read.

**🤝 Fix handoffs with a contract.**
The cure for F2 is not a better prompt. It is a **contract**: the forecaster must return JSON with `{forecast_qty, confidence, method, flags}`. The skill says to **parse it strictly** — malformed JSON is an error, not something to guess around. Structured handoff beats prose handoff.

**🧭 Learn the decision framework.**
For each of the 12 tools you must choose: keep it, replace with code, replace with a skill, or delete it.

| Option | When |
| --- | --- |
| **Tool call** | Stateless, deterministic |
| **Skill** | Rules read on demand |
| **Subagent** | Needs own context window |

**👃 Three smell tests** tell you when to move something:
- 🔸 Tool returns over **2k tokens**? → use code execution instead.
- 🔸 Writing *"always do X before Y"* in the prompt? → make it a **skill**.
- 🔸 Subagent output is **one number**? → it should not be a subagent.

**📉 Accept variance and design for it.**
`reference_scores.json` stores a range, not a target: `0.63`–`0.75`. The summary line tells you if your score is normal wobble or a real problem. Run variance is about **±8 points**. You can also pass `--trials 3` to run each task three times and take the **majority vote**.

**↔️ Compare two agents side by side.**
`--compare` runs the old agent and the new agent over the same tasks. The table shows `Before`, `After`, an arrow, and the token change. You see exactly which task flipped.

**📈 The measured result:**

| Metric | Before → After |
| --- | --- |
| Score | 71% → 92% |
| F1 time | 488s → ~100s |
| F1 tool calls | 102 → 3 scripts |
| Prompt lines | 402 → 15 |
| Subagents | 3 → 0 |

The README makes the key point: **the knowledge did not shrink**. 402 prompt lines became 400 skill lines. The difference is **when** they enter the context.

---

## ⚖️ 4. How they differ

**🎨 Workshop 1 grades an artefact. Workshop 2 grades behaviour.**
One inspects a file. The other watches what the agent did, how long it took, and what it wrote to disk.

**❓ Workshop 1 has no right answer. Workshop 2 has one.**
No correct 5-slide deck about sourdough exists. But the stock level of SKU-0042 is a fact in a CSV. This single difference drives everything else.

**🛠 Workshop 1 teaches you to *build* an eval. Workshop 2 hands you one and teaches you to *use* it.**
In Workshop 1, `graders.ts` starts with **one** grader and a comment saying *"more graders..."*. You write the rest. In Workshop 2, all 10 graders and 12 tasks exist on day one. Your job is to change the agent.

**🎛 They pull different levers.**

| Workshop 1 levers | Workshop 2 levers |
| --- | --- |
| Prompt wording | Tools vs skills |
| Add a capability | Code exec vs tool call |
| Self-QA loop | Subagent or not |
| Swap the model | Prompt vs on-demand |

**💸 Only Workshop 2 treats cost as quality.**
Workshop 1 never measures time or tokens. Workshop 2 makes them grading criteria: `wall_budget`, `budget_turns`, `budget_tokens`, and the PASS-SLOW status.

**📊 Workshop 1 uses continuous scores. Workshop 2 uses verdicts.**
Workshop 1 outputs `4.2/5` and `80%`, coloured red to green. You *hill-climb* — nudge a number up. Workshop 2 outputs PASS or FAIL. You *flip* a task from red to green.

**🧪 They handle noise differently.**

| Workshop 1 | Workshop 2 |
| --- | --- |
| Pin one baseline | Store a range |
| Diff against it | Run 3 trials |
| Average judge scores | Majority vote |

**🗂 Workshop 2 splits its test set. Workshop 1 does not.**
Workshop 2 has **R1–R9 regression** tasks and **F1–F3 failure-mode** tasks. The R tasks guard what already works. The F tasks are the known bugs you are hunting. Workshop 1's five tasks are all the same shape — five topics, one job.

**👁 Only Workshop 1 teaches vision judging.** Rendering to images to score layout is unique to it.

**🤝 Only Workshop 2 teaches handoff contracts.** How to stop data being lost between two agents is unique to it.

---

## 🤝 5. What they agree on

**🔒 A fixed test set, run every time.** Same prompts, same order, every run. No cherry-picking.

**⚡ Prefer code graders. Use judges only when you must.** Both write cheap deterministic checks first. Both reach for a model only for things code cannot see — beauty in one case, report quality in the other.

**🧱 Separate the harness from the rules.** Workshop 1's `CLAUDE.md` says `eval-runner.ts` must not be modified during the workshop; the graders live elsewhere. Workshop 2 keeps the CLI in `run.py` and every rule in `graders.py`. The runner should not know what it is measuring.

**🚦 Check the cheapest thing first.** `Produced result` and `exact_match` cost nothing and catch total failure. Never run an expensive judge on a broken file.

**1️⃣ Change one thing, then re-run.** Workshop 1 has four separate rounds. Workshop 2 has three cycles with a verify step. Neither changes two things at once.

**🔍 Numbers say *what*. Transcripts say *why*.** Workshop 2 states it plainly: *the eval CLI tells you PASS/FAIL, the Console shows you why*. Workshop 1 has the judge return a free-text **comment** alongside every score, for the same reason.

**🎲 LLM output is noisy, and you must plan for it.** Neither pretends a single run is truth.

**🚫 Never trust the agent's self-report.** Workshop 1 parses the actual file. Workshop 2 reads the actual sink files. Neither asks the agent how it did.

**🧊 Keep the grader stupid and the context rich.** Both build a context object once per task, then hand the same object to every grader. Graders stay small and independent.

---

## 🥇 6. Which to do first

**👉 Do Workshop 1 first: `eval-driven-agent-development`.**

**🪜 Reason 1 — you build graders by hand.** The file starts with one grader and an invitation to add more. Nothing teaches the concept like writing eight of them. Workshop 2 gives you ten finished graders; you would be reading, not building.

**🎓 Reason 2 — it teaches the whole loop end to end.** Define tasks → run agent → parse output → score → pin baseline → change one thing → read the delta. That loop is the entire subject. Workshop 2 assumes you already have it.

**🔤 Reason 3 — smaller vocabulary.** Workshop 1 needs graders, judges, rubrics, and baselines. Workshop 2 needs all of that **plus** skills, subagents, code execution, context windows, and sinks. Learn the measuring before the architecture.

**🌡 Reason 4 — it teaches the harder judging skill.** Grading taste is harder than grading a number. If you can build a rubric for "is this slide ugly", the objective checks in Workshop 2 will feel easy. The reverse is not true.

**📦 Then do Workshop 2.** It answers the question Workshop 1 leaves open: *the score is bad — now what do I actually change?* Workshop 1's answer is mostly "edit the prompt or swap the model." Workshop 2's answer is "restructure the agent," which is what you need for real production work.

**⏭ One exception.** If your job right now is a **slow or expensive agent that already mostly works**, go straight to Workshop 2. F1 and F3 are exactly that problem. But you asked how to build better evals — so start with Workshop 1.
