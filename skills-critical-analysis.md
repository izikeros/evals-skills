# Critical Analysis of AI Eval Skills

A thorough review of the 7 eval skills, identifying flaws, gaps, and concrete improvement recommendations. Based on skill-by-skill analysis and comparison against current industry best practices (2025-2026 research and production experience).

---

## Executive Summary

The skills form a solid, opinionated foundation for evaluating single-turn LLM outputs. They are grounded in real consulting experience, enforce good defaults (binary Pass/Fail, error-analysis-first, code-before-judges), and guard against common mistakes. The methodology is internally consistent and well-referenced.

However, the skills have significant blind spots:

1. **Agentic evaluation is absent.** The industry has moved to multi-step, tool-using agents, but every skill assumes a single input-output evaluation model.
2. **Known LLM-judge biases are not addressed.** Position bias, self-preference bias, and verbosity bias are well-documented in 2025 research but never mentioned.
3. **The entire production lifecycle is missing.** No CI/CD integration, no production monitoring, no regression testing, no A/B testing guidance.
4. **Safety and adversarial evaluation do not exist** across any skill.
5. **Inter-annotator agreement is never measured**, despite multiple skills relying on human labels as ground truth.

The skills are best understood as a "Phase 1" toolkit: excellent for bootstrapping an eval practice from zero, but incomplete for teams operating at production scale or building agentic systems.

---

## Per-Skill Analysis

### 1. error-analysis

**Strengths:**
- The "observe before you measure" philosophy is well-founded and widely validated
- The triage framework (fix first, then decide if an evaluator is worth it) prevents premature evaluator construction
- Trace sampling strategies table is practical and covers the main approaches
- Anti-patterns list catches the most common mistakes (brainstorming categories, generic scores)

**Flaws:**
- **The ~100 trace target is a magic number without formal justification.** The skill says "roughly where new traces stop revealing new kinds of failures" but provides no method to actually measure saturation. A team at trace #80 has no way to know if they should stop or continue to 200. A simple saturation curve (new categories discovered per N traces) would make this actionable.
- **"Focus on the first thing that went wrong" is sometimes wrong.** In agentic traces with parallel tool calls, there may be no single "first" failure. In multi-turn conversations, a late-stage failure may be independent of earlier issues. The heuristic works for linear pipelines but breaks for complex architectures.
- **No guidance on trace complexity affecting sample size.** A 2-step prompt-response trace is fundamentally different from a 50-step agent trace with 15 tool calls. The former might saturate at 50 traces; the latter might need 300+.
- **LLM-assisted clustering is relegated to an afterthought.** Modern embedding-based clustering (e.g., cluster failure descriptions, then review cluster representatives) could significantly accelerate the grouping step, especially beyond 50 traces.

**Gaps:**
- No guidance on versioning failure categories over time (how to track category evolution across pipeline changes)
- No structured method for severity/impact scoring of categories beyond "most frequent = highest priority" (frequency and business impact don't always correlate, as the skill itself acknowledges, but offers no framework for combining them)

**Recommendations:**
- Add a saturation curve method: plot cumulative new categories vs. traces reviewed; stop when the curve flattens for 20+ traces
- Add guidance for complex/agentic traces: "for traces with 10+ steps, consider both sequential root cause analysis and independent evaluation of parallel branches"
- Provide scaling guidance: simple pipelines ~50-100, complex agents ~150-300

---

### 2. generate-synthetic-data

**Strengths:**
- The dimension/tuple/query three-step separation is a clever design that produces more diverse outputs than naive generation
- Requiring user validation of tuples before scaling up prevents unrealistic test cases
- The anti-patterns list (especially "unstructured generation" and "single-step generation") captures real failure modes

**Flaws:**
- **LLM-generated queries follow LLM distributions, not human distributions.** The skill never addresses this fundamental limitation. LLMs tend to generate "clean" queries with proper grammar, clear intent, and reasonable complexity. Real users produce typos, ambiguous phrasing, code-switching between languages, copy-pasted content, and contextually dependent references. No mitigation or even acknowledgment of this distribution gap is provided.
- **The Likert-scale filtering (Step 5) contradicts the framework's own anti-Likert stance.** The skill acknowledges this with "Likert scoring is appropriate here, since the goal is fuzzy ranking for dataset curation" -- but this undermines the otherwise clean binary-only philosophy. A binary "realistic enough? yes/no" filter would be more consistent.
- **No deduplication or diversity measurement.** After generating 100+ queries, there is no method to verify that the set is actually diverse. Embedding-based similarity analysis or coverage metrics over the dimension space would catch cases where generation clusters around a few popular patterns.

**Gaps:**
- No method to measure distribution match between synthetic and real data (when both exist)
- No guidance on generating multi-turn conversations (only single queries)
- No guidance on generating adversarial or edge-case inputs specifically targeting known failure modes (the dimensions approach is more exploratory than adversarial)
- No mention of using real user data as few-shot examples to make synthetic queries more realistic

**Recommendations:**
- Add a distribution comparison step: when real data exists, compute embedding similarity distributions and flag synthetic clusters that have no real-data counterpart
- Add a "realistic noise injection" directive: instruct the LLM to include typos, abbreviated language, incomplete sentences for a fraction of generated queries
- Replace Likert filtering with binary "Is this query something a real user would type? Yes/No"
- Add multi-turn generation for conversational systems

---

### 3. write-judge-prompt

**Strengths:**
- The four-component structure (criterion, definitions, examples, structured output) is clear and reproducible
- "Critique before verdict" is a well-validated technique for improving judge accuracy
- The "choosing what to pass to the judge" table is practical and prevents information overload
- The strong stance against Likert scales with clear reasoning (calibration difficulty, annotator disagreement)

**Flaws:**
- **Zero mention of known LLM-judge biases.** This is the most significant flaw in the entire skill set. Research from 2024-2025 has established multiple systematic biases in LLM judges:
  - **Position bias:** LLMs systematically prefer the first or last option when comparing (ACL 2025: "A Systematic Study of Position Bias in LLM-as-a-Judge")
  - **Self-preference bias:** LLMs rate their own outputs higher than equivalent outputs from other models (ICLR 2025: "Self-Preference Bias in LLM-as-a-Judge")
  - **Verbosity bias:** Longer responses are rated higher regardless of quality
  - **Authority/confidence bias:** More assertive language is rated higher even when incorrect
  - **Anchoring bias:** Few-shot example ordering influences judgments
  None of these biases are mentioned, and no mitigation strategies are provided.
- **No temperature or sampling guidance.** Running judges at temperature 0 improves consistency. Running at T>0 with majority vote over 3-5 runs can improve accuracy. Neither technique is mentioned.
- **No multi-judge or consensus strategies.** Using multiple models or multiple prompts and aggregating (majority vote, unanimous agreement) is a proven technique for reducing individual judge bias. Not covered.

**Gaps:**
- No pairwise comparison mode. Many real-world eval tasks are comparative ("Is prompt A's output better than prompt B's?") rather than absolute ("Does this output pass?"). Pairwise comparison is often more reliable than absolute judgment for subjective criteria.
- No guidance on judge prompt versioning or A/B testing different judge prompts
- No mention of chain-of-thought variations beyond the single "critique then verdict" pattern

**Recommendations:**
- P0: Add a "Judge Bias Mitigation" section covering position bias (randomize ordering), self-preference bias (avoid using the same model family as generator and judge when possible), verbosity bias (include explicit length-neutrality instructions), and anchoring (vary few-shot order across runs)
- Add temperature guidance: "Use temperature 0 for consistency. For borderline cases, run 3 times at T=0.3 and take majority vote."
- Add a pairwise comparison template for A/B evaluation use cases
- Add multi-judge consensus as an option for high-stakes evaluations

---

### 4. validate-evaluator

**Strengths:**
- The train/dev/test split methodology is rigorous and prevents data leakage
- TPR/TNR over raw accuracy is a strong, well-justified choice
- The Rogan-Gladen bias correction is a sophisticated technique rarely seen in practitioner guides
- Bootstrap confidence intervals add statistical rigor
- The "if alignment stalls" troubleshooting table is practical

**Flaws:**
- **The 90% TPR/TNR target is presented without justification.** Why 90% and not 85% or 95%? The answer depends on the cost of false passes vs. false fails, which varies by use case. A safety-critical system might need 98% TNR; a content quality check might accept 80%.
- **No statistical power analysis.** The skill says "use ~100 examples" but doesn't discuss the statistical power this provides. With 50 Fail examples and 90% TNR, the 95% CI for TNR is approximately [79%, 96%] -- wide enough that you cannot reliably distinguish 85% from 95% TNR. This limitation is never discussed.
- **The bootstrap CI code is correct but the minimum sample size for reliable CIs is not stated.** Below ~30 examples per class, bootstrap CIs become unreliable due to insufficient resampling diversity.
- **No guidance on what to do when labeled data is expensive.** The skill assumes you can get 100 labeled examples relatively easily. In domains like legal, medical, or compliance, each label may cost $50-200 in expert time. No cost-efficient alternatives (active learning, uncertainty sampling for labeling priority) are offered.

**Gaps:**
- No discussion of judge stability over time (re-running the same judge on the same data weeks later -- does it drift?)
- No guidance on handling edge cases where human labels themselves are wrong or inconsistent
- No mention of using multiple human annotators and measuring inter-annotator agreement before treating any labels as ground truth

**Recommendations:**
- Replace the flat 90%/80% targets with a framework: "Choose your target based on the cost of errors. Safety-critical: TNR > 95%. Quality optimization: TPR/TNR > 85%. Exploratory: TPR/TNR > 75%."
- Add a statistical power note: "With 50 examples per class, the 95% CI for TPR/TNR is approximately +/- 8-10 percentage points. For tighter bounds, increase to 100+ per class."
- Add IAA guidance: "Before treating human labels as ground truth, have two annotators independently label 20-30 examples. If Cohen's Kappa < 0.7, refine the rubric before proceeding."

---

### 5. evaluate-rag

**Strengths:**
- The "evaluate retrieval and generation separately" principle is sound and widely validated
- The diagnostic matrix (context relevance x faithfulness x answer relevance) is a useful debugging tool
- The chunking grid search is practical and actionable
- Multi-hop retrieval evaluation with Two-hop Recall@k addresses a real gap in most RAG eval guides

**Flaws:**
- **Missing modern RAG patterns (circa 2025).** The skill covers basic vector search + chunking but does not address:
  - **Agentic RAG:** Where the agent decides whether, when, and what to retrieve (evaluation shifts from "did retrieval return good docs?" to "did the agent decide to retrieve at the right time?")
  - **Query rewriting/expansion evaluation:** Modern RAG systems rewrite queries before retrieval. The skill never evaluates query transformation quality.
  - **Hybrid search:** Dense + sparse (BM25) retrieval is standard in 2025. No guidance on evaluating hybrid configurations.
  - **Routing:** Systems that route queries to different indexes/sources. No evaluation guidance.
- **The chunking optimization section is circa-2023 thinking.** Modern approaches like contextual retrieval (prepend document context to chunks), late interaction models (ColBERT), and hypothetical document embeddings (HyDE) are not mentioned. The grid search over fixed-size chunks is a useful starting point but presented as the primary approach.
- **Faithfulness evaluation is underspecified.** The skill says "check for hallucinations, omissions, misinterpretations" but provides no concrete method. It should reference write-judge-prompt for building a faithfulness judge, or provide a concrete evaluation prompt template.

**Gaps:**
- No evaluation of retrieval latency or cost (important for production RAG)
- No guidance on evaluating citation/attribution accuracy (linking claims to sources)
- No coverage of evaluating RAG with structured data (SQL, knowledge graphs)
- No mention of evaluating context window utilization (are retrieved chunks actually being used by the generator?)

**Recommendations:**
- Add an "Agentic RAG" section: evaluate retrieval decisions (should-retrieve/should-not-retrieve accuracy) separately from retrieval quality
- Add query rewriting evaluation: compare original vs. rewritten query retrieval metrics
- Explicitly reference write-judge-prompt for building faithfulness and relevance judges
- Add citation accuracy as a first-class evaluation dimension
- Mention hybrid search and routing evaluation even if briefly

---

### 6. build-review-interface

**Strengths:**
- The data display guidelines are excellent and highly specific (color-coding, collapsing, native rendering)
- Keyboard shortcuts are a practical accelerator for annotation speed
- The Playwright testing section ensures the interface actually works
- The design checklist provides a concrete quality bar

**Flaws:**
- **Always builds from scratch.** The skill instructs the agent to build a custom HTML page every time. It never mentions existing annotation platforms (Label Studio, Prodigy, Argilla, Doccano) that could be configured faster and provide features like multi-annotator support, conflict resolution, and IAA measurement out of the box. For many teams, configuring an existing tool is more practical than building custom.
- **No multi-annotator support.** The interface is designed for a single reviewer. There is no concept of assigning the same trace to multiple annotators, measuring agreement, or resolving conflicts. This is a fundamental gap for any team that needs to validate label quality.
- **No annotation quality control mechanisms.** Gold standard items (pre-labeled traces mixed into the annotation queue to catch annotator drift or carelessness) are a standard practice not mentioned anywhere.
- **No annotation efficiency tracking.** Time per trace, traces per hour, and reviewer fatigue patterns are useful operational metrics for managing annotation campaigns.

**Gaps:**
- No guidance on managing annotation campaigns (batching, assignment, progress tracking across multiple annotators)
- No export format standardization (how to structure labeled data for downstream use by other skills)
- No mention of active learning integration (prioritizing traces that would be most informative to label)

**Recommendations:**
- Add a decision framework: "For teams of 1-2 reviewers doing <200 traces, build a custom interface. For larger campaigns or multi-annotator needs, consider configuring Label Studio or Argilla."
- Add multi-annotator support: separate annotation queues, overlap percentage setting, agreement measurement
- Add gold standard item injection: "Mix 5-10% pre-labeled traces into the queue and alert if reviewer accuracy drops below 90% on gold items."

---

### 7. eval-audit

**Strengths:**
- The six diagnostic areas provide comprehensive coverage of common eval problems
- Each finding links to a specific skill or fix, making the audit actionable
- The "No Eval Infrastructure" path is a practical entry point for teams starting from zero
- The report format is clear and structured

**Flaws:**
- **Does not check if evals are actually being used to drive decisions.** The most common problem with eval systems in practice is not that they are poorly designed, but that they are ignored. Teams build evals, get results, and then ship anyway. The audit should check: "When was the last time an eval result blocked a deploy or changed a decision?"
- **No cost or throughput audit.** An eval pipeline that takes 4 hours and $200 to run will not be run frequently. The audit should assess whether the eval pipeline is fast and cheap enough to be used regularly.
- **The six areas are all about eval correctness, not eval utility.** A perfectly correct eval that measures something nobody cares about is useless. The audit should also check: "Do the failure categories being measured map to actual business metrics or user complaints?"

**Gaps:**
- No check for eval freshness (when were eval datasets last updated? do they reflect current usage patterns?)
- No check for eval coverage (what percentage of the system's functionality is covered by evals?)
- No check for eval pipeline reliability (do evals produce consistent results across runs? is there flakiness?)

**Recommendations:**
- Add a "7. Eval Utilization" diagnostic area: "Are eval results being used to make decisions? When was the last time an eval result changed a development decision?"
- Add an "8. Eval Efficiency" check: "How long does the full eval suite take to run? What does it cost? Is it fast enough to run in CI?"
- Add eval coverage assessment: "What percentage of known failure modes have automated evaluators? What is not covered?"

---

## Cross-Cutting Gaps

These are capabilities missing across the entire skill set that represent fundamental blind spots.

### Critical: Agentic / Multi-Step Evaluation

**The gap:** Every skill assumes a simple input-output evaluation model. But the industry has moved to multi-step, tool-using agents (Anthropic's "Demystifying evals for AI agents", Jan 2026; AWS agent eval framework, Feb 2026; tau-bench and tau2-bench benchmarks). The skills provide no guidance for:

- **Trajectory evaluation:** Was the agent's path reasonable? Did it use the right tools in a sensible order? An agent that arrives at the correct answer via 50 unnecessary tool calls is different from one that takes 3 efficient steps.
- **Step-level/span-level grading:** Evaluating individual tool calls, retrieval decisions, or reasoning steps -- not just the final output. The error-analysis skill mentions traces with tool calls but evaluation is always at the trace level.
- **Non-determinism handling:** Agents produce different trajectories on repeated runs. The concepts of pass@k (probability of at least one success in k attempts) and pass^k (probability of all k attempts succeeding) from Anthropic's guide are not present.
- **Environment state verification:** Checking whether the agent actually changed the intended state (database record, file system, API call) rather than just producing text claiming it did.
- **Simulated user interactions:** Multi-turn agents need a simulated user to test against. No skill addresses how to build or evaluate user simulators.
- **Partial credit:** An agent that correctly identifies the problem and calls the right tool but provides the wrong parameters is better than one that fails immediately. The current binary Pass/Fail model does not capture this.

**Impact:** Teams building agents with these skills will evaluate only the final output, missing the majority of failure modes that occur mid-trajectory.

**Recommendation:** Create a new `evaluate-agent` skill covering trajectory grading, step-level evaluation, pass@k/pass^k for non-determinism, environment state checks, and simulated user design.

### High: LLM Judge Bias Mitigation

**The gap:** The write-judge-prompt skill has zero mention of known LLM-judge biases despite extensive 2024-2025 research documenting them:

- **Position bias** -- LLMs prefer options presented first or last (ACL 2025, IJCNLP 2025)
- **Self-preference bias** -- LLMs rate their own outputs higher (ICLR 2025, OpenReview)
- **Verbosity bias** -- Longer responses get higher scores regardless of quality
- **Authority/confidence bias** -- More assertive language scores higher even when wrong
- **Anchoring bias** -- Few-shot example ordering influences judgments

**Impact:** Teams following write-judge-prompt will build judges with undetected systematic biases. validate-evaluator may catch some of this through TPR/TNR measurement, but teams will waste iteration cycles on bias issues they could have prevented.

**Recommendation:** Add a dedicated "Bias Mitigation" section to write-judge-prompt with concrete strategies: randomized few-shot ordering, length-neutrality instructions, using different model families for generation and judging, and multi-judge consensus.

### High: Safety and Adversarial Evaluation

**The gap:** No skill covers safety evaluation, red teaming, prompt injection testing, guardrail evaluation, or harmful output detection. The eval-audit skill checks for various eval quality issues but never asks "is the system safe?"

**Impact:** Teams following these skills can build a well-validated eval pipeline that completely ignores safety. For production systems, this is a liability.

**Recommendation:** Create a new `evaluate-safety` skill covering: guardrail testing, prompt injection resistance, harmful output detection, PII leakage checks, and adversarial input generation. At minimum, add safety as a diagnostic area in eval-audit.

### High: CI/CD Integration

**The gap:** The README mentions "wiring evals into CI/CD" is covered in the course but not in any skill. There is no guidance on:

- Running evals as part of a CI pipeline
- Distinguishing capability evals (hill-climbing, low pass rate expected) from regression evals (should be ~100% pass rate)
- When to block a deploy vs. warn
- How to handle non-deterministic eval results in CI (flaky tests)
- Eval result storage and trend tracking

**Impact:** Teams build offline evals but never integrate them into their development workflow. Evals become a periodic manual activity rather than an automated quality gate.

**Recommendation:** Create a new `integrate-eval-ci` skill or add CI/CD guidance as a section in eval-audit.

### Medium: Production Monitoring

**The gap:** All skills operate pre-production or offline. There is no guidance on:

- Online evaluation (running judges on production traffic)
- Drift detection (corrected pass rates trending downward)
- A/B testing integration (using evals to compare production variants)
- User feedback loop integration (using thumbs-up/down signals to prioritize trace review)
- Alert thresholds for eval metric degradation

**Impact:** Teams build good offline evals but have no way to detect when production quality degrades between eval runs.

**Recommendation:** Add production monitoring guidance, possibly as a new skill or as an extension to eval-audit.

### Medium: Pairwise / Comparative Evaluation

**The gap:** Every skill uses absolute evaluation (Pass/Fail). But many practical tasks are comparative:

- "Is the new prompt better than the old one?"
- "Which model produces better outputs for our use case?"
- "Did this code change improve or degrade quality?"

Pairwise comparison is often more reliable than absolute judgment for subjective criteria because it is easier for both humans and LLMs to determine "A is better than B" than to assign an absolute quality score.

**Impact:** Teams have no structured approach for the most common eval use case: comparing two versions of something.

**Recommendation:** Add a pairwise comparison mode to write-judge-prompt, including Elo-style ranking aggregation for comparing multiple variants.

### Medium: Inter-Annotator Agreement

**The gap:** Multiple skills rely on human labels as ground truth (error-analysis, validate-evaluator, build-review-interface) but none measure whether human annotators agree with each other. Cohen's Kappa is mentioned only to say "don't use it for judge validation" but never recommended for its actual purpose: measuring annotator consistency.

**Impact:** If two experts disagree 30% of the time on Pass/Fail labels, a judge aligned 90% with Expert A's labels might be aligned only 70% with Expert B's. The "ground truth" has unquantified noise.

**Recommendation:** Add IAA measurement wherever human labels are collected. Require two independent annotators on at least 20-30 traces. Flag rubrics with Kappa < 0.7 for refinement before using labels as ground truth.

### Medium: Cost and Latency Optimization

**The gap:** No skill addresses the cost or speed of running evaluations:

- LLM judge costs per evaluation (a GPT-4o judge on 1000 traces costs $X)
- Latency budgets for real-time vs. batch evaluation
- When to use smaller/cheaper models as judges
- Cost-accuracy tradeoffs in judge model selection

**Impact:** Teams build eval pipelines with the most expensive model (as recommended by write-judge-prompt: "start with the most capable model") and never optimize, making frequent evaluation prohibitively expensive.

**Recommendation:** Add cost optimization guidance to write-judge-prompt and validate-evaluator: "After validating with the most capable model, test whether a cheaper model achieves acceptable TPR/TNR. Many criteria can be judged by smaller models at 1/10th the cost."

### Low-Medium: Multi-Agent Evaluation

**The gap:** No coverage of evaluating systems where multiple agents collaborate, delegate, or coordinate. This is an emerging need per 2025-2026 research (MultiAgentBench, ACL 2025) but not yet mainstream in production.

**Impact:** Limited current impact, but growing as multi-agent architectures become more common.

**Recommendation:** Not urgent. Monitor the field and add when multi-agent production systems become more prevalent.

---

## Prioritized Recommendations

| Priority | Action | Skill Affected | Effort |
|----------|--------|----------------|--------|
| P0 | Add LLM-judge bias mitigation section | write-judge-prompt | Small (add ~50 lines) |
| P0 | Create `evaluate-agent` skill for agentic/multi-step eval | New skill | Large (new skill) |
| P1 | Add CI/CD integration guidance | New skill or eval-audit addendum | Medium |
| P1 | Add safety/adversarial testing | New skill or eval-audit addendum | Medium-Large |
| P1 | Add IAA measurement guidance | build-review-interface, validate-evaluator | Small (add to existing) |
| P1 | Add saturation analysis method | error-analysis | Small |
| P2 | Add pairwise comparison mode | write-judge-prompt | Medium |
| P2 | Add production monitoring guidance | New skill | Medium |
| P2 | Modernize for 2025-era RAG patterns | evaluate-rag | Medium |
| P2 | Add "build vs. buy" decision for annotation tools | build-review-interface | Small |
| P2 | Add eval utilization and efficiency checks | eval-audit | Small |
| P3 | Add cost optimization guidance | write-judge-prompt, validate-evaluator | Small |
| P3 | Add distribution comparison for synthetic data | generate-synthetic-data | Small |
| P3 | Add statistical power discussion | validate-evaluator | Small |

---

## Sources

Key references informing this analysis:

- Anthropic, "Demystifying evals for AI agents" (Jan 2026) -- comprehensive framework for agent evaluation including trajectory grading, pass@k/pass^k, capability vs regression evals
- "A Systematic Study of Position Bias in LLM-as-a-Judge" (ACL/IJCNLP 2025) -- documents position bias in LLM judges
- "Self-Preference Bias in LLM-as-a-Judge" (ICLR 2025) -- documents self-preference bias
- "Trust or Escalate: LLM Judge Confidence vs. Human Agreement" (ICLR 2025) -- LLM judges are overconfident in their agreement estimates
- AWS, "Evaluating AI agents: Real-world lessons from building agentic systems" (Feb 2026) -- production agent evaluation framework
- "Evaluation and Benchmarking of LLM Agents: A Survey" (KDD 2025) -- comprehensive taxonomy of agent evaluation
- tau2-bench (Sierra Research, 2025) -- multi-turn agent evaluation benchmark with simulated users
- "The Alternative Annotator Test for LLM-as-a-Judge" (ACL 2025) -- framework for validating LLM judges against annotator agreement
- "Counting on Consensus: Selecting the Right Inter-annotator Agreement Metric" (2026) -- IAA metric selection guidance
- Confident AI, "Definitive AI Agent Evaluation Guide" (Apr 2026) -- span-level evaluation and agent metrics
- Langfuse, "Agent Evaluation: How to Evaluate LLM Agents" (Mar 2026) -- trajectory accuracy and tool selection evaluation
- "Pairwise Comparisons Amplify Biased Preferences of LLM Evaluators" (BlackboxNLP 2025) -- bias amplification in pairwise settings
- Hamel Husain, "LLM Evals: Everything You Need to Know" (Jan 2026) -- the foundational reference for these skills
