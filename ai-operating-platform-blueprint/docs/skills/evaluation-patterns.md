# Skill: Evaluation Patterns

> **Document Type:** Skills / Architectural Patterns
> **Domain:** Evaluation Plane
> **Status:** Accepted
> **Last Updated:** 2026-05-30

---

## Purpose

Reusable patterns for implementing AI quality evaluation. Every AI agent and RAG pipeline deployed on the platform must have an evaluation strategy.

---

## Pattern 1 — LLM-as-Judge Evaluation

**When to use:** Evaluating open-ended AI outputs where exact matching is insufficient.

**Judge implementation:**

```python
class LLMJudge:
    def __init__(self, judge_model: str = "claude-sonnet-4-6"):
        self.judge_model = judge_model
    
    async def evaluate(
        self,
        question: str,
        context: list[str],
        answer: str,
        criteria: list[str]
    ) -> EvaluationResult:
        
        judge_prompt = f"""
You are an expert evaluator for AI systems in a regulated financial environment.

QUESTION: {question}

CONTEXT PROVIDED TO AI:
{chr(10).join(f"[{i+1}] {ctx}" for i, ctx in enumerate(context))}

AI ANSWER: {answer}

Evaluate the answer on each criterion (score 0.0-1.0):
{chr(10).join(f"- {criterion}" for criterion in criteria)}

Return a JSON object with:
- scores: {{criterion: score}} for each criterion
- overall_score: weighted average
- issues: list of specific problems found
- recommendation: PASS / WARN / FAIL
"""
        
        response = await model_plane.invoke({
            "model_class": "evaluation",
            "model_id": self.judge_model,
            "messages": [{"role": "user", "content": judge_prompt}],
            "response_format": {"type": "json_object"}
        })
        
        return EvaluationResult.model_validate_json(response.content)
```

**Standard criteria for financial AI:**
```python
FINANCIAL_EVAL_CRITERIA = [
    "Faithfulness: Answer is grounded in provided context (not hallucinated)",
    "Regulatory Accuracy: All regulatory references are correct and current",
    "Completeness: Answer addresses all aspects of the question",
    "Caveat Appropriateness: Appropriate caveats for uncertain information",
    "Actionability: Answer provides clear, actionable guidance"
]
```

---

## Pattern 2 — RAGAS Evaluation Pipeline

**For RAG pipeline evaluation:**

```python
from ragas import evaluate
from ragas.metrics import (
    faithfulness,
    answer_relevancy,
    context_recall,
    context_precision,
)

async def evaluate_rag_pipeline(
    test_dataset: list[RAGTestCase],
    pipeline: RAGPipeline
) -> RAGEvalReport:
    
    results = []
    for case in test_dataset:
        # Run the RAG pipeline
        retrieved_contexts, answer = await pipeline.run(case.question)
        
        results.append({
            "question": case.question,
            "contexts": retrieved_contexts,
            "answer": answer,
            "ground_truth": case.expected_answer  # For recall calculation
        })
    
    # Evaluate with RAGAS
    dataset = Dataset.from_list(results)
    scores = evaluate(
        dataset=dataset,
        metrics=[faithfulness, answer_relevancy, context_recall, context_precision]
    )
    
    return RAGEvalReport(
        faithfulness=scores["faithfulness"],
        answer_relevancy=scores["answer_relevancy"],
        context_recall=scores["context_recall"],
        context_precision=scores["context_precision"],
        pass_threshold=0.85
    )
```

---

## Pattern 3 — Golden Dataset Construction

**How to build evaluation datasets:**

```python
@dataclass
class GoldenDatasetEntry:
    # Input
    question: str
    context_documents: list[str]  # Document IDs or content
    
    # Expected output
    expected_answer: str
    expected_key_points: list[str]
    
    # Metadata
    difficulty: str  # "easy", "medium", "hard"
    domain: str  # "underwriting", "compliance", "claims"
    created_by: str
    reviewed_by: str
    created_at: datetime

# Golden dataset construction process:
# 1. SME (subject matter expert) provides question and expected answer
# 2. Peer review by second SME
# 3. Legal review if regulatory questions
# 4. Version-controlled in Registry Plane
# 5. Used for: quality gates, regression testing, continuous evaluation
```

**Minimum dataset sizes:**
| Evaluation Type | Minimum Entries | Recommended |
|---|---|---|
| Smoke test | 10 | 25 |
| Quality gate (staging) | 50 | 100 |
| Continuous monitoring | 100 | 250 |
| Bias evaluation | 200 per group | 500 per group |

---

## Pattern 4 — Semantic Drift Detection

**Detect when AI behavior changes without intentional change:**

```python
class SemanticDriftDetector:
    def __init__(self, baseline_window_days: int = 30):
        self.baseline = baseline_window_days
    
    async def compute_drift(
        self,
        agent_id: str,
        tenant_id: str,
        current_window_days: int = 7
    ) -> DriftReport:
        
        # Sample recent outputs
        recent_outputs = await eval_store.get_outputs(
            agent_id=agent_id,
            days=current_window_days
        )
        
        # Sample baseline outputs
        baseline_outputs = await eval_store.get_outputs(
            agent_id=agent_id,
            days_offset=self.baseline,
            days=current_window_days
        )
        
        # Embed and compute distribution distance
        recent_embeddings = await embed_batch([o.output for o in recent_outputs])
        baseline_embeddings = await embed_batch([o.output for o in baseline_outputs])
        
        # Jensen-Shannon divergence between distributions
        drift_score = jensen_shannon_divergence(
            compute_distribution(recent_embeddings),
            compute_distribution(baseline_embeddings)
        )
        
        # Score interpretation: 0 = no drift, 1 = complete distribution shift
        severity = "none" if drift_score < 0.1 else "low" if drift_score < 0.3 else "high"
        
        return DriftReport(
            agent_id=agent_id,
            drift_score=drift_score,
            severity=severity,
            sample_size=len(recent_outputs),
            alert=drift_score > 0.3
        )
```

---

## Pattern 5 — A/B Model Evaluation

**Compare primary model against challenger model:**

```python
async def ab_evaluation(
    question: str,
    context: list[str],
    primary_model: str,
    challenger_model: str,
    judge_model: str,
    n_trials: int = 5
) -> ABEvaluationResult:
    
    primary_scores = []
    challenger_scores = []
    
    for _ in range(n_trials):
        primary_response = await model_plane.invoke(ModelRequest(
            model_id=primary_model, messages=build_rag_messages(question, context)
        ))
        challenger_response = await model_plane.invoke(ModelRequest(
            model_id=challenger_model, messages=build_rag_messages(question, context)
        ))
        
        # Judge evaluates both (blind, randomized order)
        judge_result = await judge.compare(
            question=question, context=context,
            response_a=primary_response.content,
            response_b=challenger_response.content
        )
        
        primary_scores.append(judge_result.score_a)
        challenger_scores.append(judge_result.score_b)
    
    return ABEvaluationResult(
        primary_model=primary_model,
        primary_mean=statistics.mean(primary_scores),
        challenger_model=challenger_model,
        challenger_mean=statistics.mean(challenger_scores),
        recommendation="PROMOTE" if statistics.mean(challenger_scores) > statistics.mean(primary_scores) * 1.05 else "KEEP"
    )
```

---

## Evaluation Quality Gate

**Template for quality gate configuration:**

```yaml
# quality-gates/loan-underwriting-agent.yaml
agent_id: loan-underwriting-agent-v2
evaluation_dataset: loan-underwriting-golden-v3

thresholds:
  faithfulness: 0.90
  answer_relevancy: 0.85
  regulatory_accuracy: 0.95
  consistency_score: 0.90  # Variance across 5 runs < 0.1

regression_check:
  enabled: true
  baseline_agent_version: v1.8.0
  max_regression_pct: 5  # Cannot degrade more than 5% from baseline

fairness_check:
  enabled: true
  protected_attributes: [age_group, geography, loan_size]
  max_disparate_impact: 0.05  # Max 5% difference across groups

gate_decision:
  PASS: All thresholds met + no regression + fairness passed
  WARN: Thresholds met but regression or fairness concern
  FAIL: Any threshold missed OR regression > 5% OR fairness breach
```
