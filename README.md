# stock-analyst
stock analyst with Claude
how to build these metrics by ourselves? binary?

Yes — binary per-atomic-unit judgment is exactly right. It's what RAGAS does internally anyway (decompose into claims/chunks, judge each, average). Building your own = same decomposition pattern, your own prompts, your own calibrated judge.

Why binary beats 1-10 scoring

LLM judges are unreliable at fine-grained numeric scales — score bunching, calibration drift between runs, low inter-rater agreement with humans. Binary (yes/no) per atomic claim/chunk gets same resolution back through counting instead of asking model to be precise about a number it can't reliably produce. Decompose first, judge binary, aggregate by fraction.

One generic function, reused 4 times

import anthropic, json

client = anthropic.Anthropic()

def judge_binary(item_text: str, context: str, criterion: str) -> dict:
    """Binary yes/no judgment with reason. Reused for every metric below."""
    resp = client.messages.create(
        model="claude-opus-5",
        max_tokens=512,
        output_config={"effort": "low", "format": {
            "type": "json_schema",
            "schema": {
                "type": "object",
                "properties": {
                    "label": {"type": "string", "enum": ["yes", "no"]},
                    "reason": {"type": "string"}
                },
                "required": ["label", "reason"],
                "additionalProperties": False
            }
        }},
        messages=[{"role": "user", "content":
            f"{criterion}\n\nITEM:\n{item_text}\n\nCONTEXT:\n{context}"}]
    )
    return json.loads(resp.content[0].text)

Skipped: four separate judge classes — one function, prompt swapped per metric. Add per-metric wrapper only if a metric genuinely needs different output shape.

Decomposition step (needed for recall + faithfulness)

def decompose_claims(text: str) -> list[str]:
    """Break answer/ground_truth into atomic claims."""
    resp = client.messages.create(
        model="claude-opus-5",
        max_tokens=1024,
        output_config={"effort": "low", "format": {
            "type": "json_schema",
            "schema": {"type": "object",
                       "properties": {"claims": {"type": "array", "items": {"type": "string"}}},
                       "required": ["claims"], "additionalProperties": False}
        }},
        messages=[{"role": "user", "content":
            f"List each distinct factual claim in this text as a separate string:\n\n{text}"}]
    )
    return json.loads(resp.content[0].text)["claims"]

The 4 metrics, built from those two primitives

def context_recall(contexts: list[str], ground_truth: str) -> float:
    claims = decompose_claims(ground_truth)
    ctx = "\n---\n".join(contexts)
    hits = [judge_binary(c, ctx, "Is this claim supported by the context? yes/no.")["label"]
            for c in claims]
    return hits.count("yes") / len(claims)

def context_precision(contexts: list[str], question: str, ground_truth: str) -> float:
    hits = [judge_binary(c, f"Q: {question}\nGold answer: {ground_truth}",
                          "Is this retrieved chunk relevant to answering the question? yes/no.")["label"]
            for c in contexts]
    return hits.count("yes") / len(contexts)

def faithfulness(answer: str, contexts: list[str]) -> float:
    claims = decompose_claims(answer)
    ctx = "\n---\n".join(contexts)
    hits = [judge_binary(c, ctx, "Is this claim supported by the context? yes/no.")["label"]
            for c in claims]
    return hits.count("yes") / len(claims)

def answer_relevancy(question: str, answer: str) -> float:
    # Single binary judgment, no decomposition needed — directness check.
    result = judge_binary(answer, question,
        "Does this answer directly and completely address the question? yes/no.")
    return 1.0 if result["label"] == "yes" else 0.0

answer_relevancy here is simpler than RAGAS's reverse-question-embedding trick — binary direct-match proxy. Good enough to start; add embedding version only if binary proves too coarse on real data.

Rank-weighting precision (add later, not now)

Plain fraction above treats all chunk positions equal. RAGAS weights by rank (early irrelevant chunks hurt more). Skip until plain version proves insufficient — YAGNI, add weight = 1/log2(rank+2) when you have evidence rank matters for your ranker specifically.

Calibration — mandatory before trusting any number

def calibrate(gold_labels: list[str], predicted_labels: list[str]) -> float:
    """Agreement rate against human labels. Run on 50+ items before trusting the judge."""
    agree = sum(g == p for g, p in zip(gold_labels, predicted_labels))
    return agree / len(gold_labels)

# self-check
assert calibrate(["yes","no","yes"], ["yes","no","no"]) == pytest_approx if False else 2/3

Run every metric's binary judge against 30-50 human-labeled items first (ties back to judge-calibration step from before). If agreement <85%, fix prompt wording before trusting scores — vague criterion text is the usual cause, not the model.

What this buys over RAGAS library

Full control over prompt wording per domain (SharePoint docs, your terminology), swap judge model freely, no library version lock, same decomposition logic you can inspect/debug line by line. Cost: you own prompt tuning instead of library's defaults — worth it once library scores stopped making sense to you.

✻ Cogitated for 39s · done Saturday 8:03 AM

❯ how about addign correctness and complenetess

Reuse exact same two primitives — zero new infra needed. Correctness = precision of answer claims against truth. Completeness = recall of truth claims in answer. Textbook P/R pair.

def correctness(answer: str, ground_truth: str) -> float:
    """Of what the answer claims, how much is actually true? (precision)"""
    claims = decompose_claims(answer)
    hits = [judge_binary(c, ground_truth,
            "Is this claim consistent with / supported by the ground truth? yes/no.")["label"]
            for c in claims]
    return hits.count("yes") / len(claims)

def completeness(answer: str, ground_truth: str) -> float:
    """Of what ground truth requires, how much did the answer cover? (recall)"""
    claims = decompose_claims(ground_truth)
    hits = [judge_binary(c, answer,
            "Is this claim covered/present in the answer? yes/no.")["label"]
            for c in claims]
    return hits.count("yes") / len(claims)

def correctness_f1(answer: str, ground_truth: str) -> dict:
    p = correctness(answer, ground_truth)
    r = completeness(answer, ground_truth)
    f1 = 2 * p * r / (p + r) if (p + r) else 0.0
    return {"correctness": p, "completeness": r, "f1": f1}

Note difference from correctness vs completeness being low:

- Low correctness, high completeness → answer says everything expected plus wrong/extra stuff. Hallucination or stale-context problem.
- High correctness, low completeness → answer true but partial. Truncation, max_tokens cutoff, retrieval missing some gold chunks, or model stopping early.
- Both low → broken retrieval + generation both, check context_recall first (earlier debug order still applies).

One gap in binary version: contradiction vs omission

Plain yes/no can't tell "claim not mentioned" from "claim actively contradicted" — second is worse (confident wrong fact vs just missing). If correctness scores confusing later, upgrade label enum:

"label": {"type": "string", "enum": ["supported", "contradicted", "not_mentioned"]}

Skipped for now — binary first, add 3-way only if debugging shows you need to separate hallucination-by-contradiction from hallucination-by-omission. YAGNI until data demands it.

Where this plugs into existing plan

Run correctness_f1 alongside faithfulness/context_recall/context_precision per gold item, same calibration check (30-50 human-labeled items, check judge agreement before trusting). Segment by matrix cell from earlier — correctness/completeness split is especially useful on multi-hop and aggregation cells, where partial answers are the common failure shape.

✻ Cooked for 20s · done Saturday 8:06 AM

❯ context precision and contrxt recall are same as that of completness and coreecteness?

No — same math shape (precision/recall), different object compared. That's the point, not a redundancy.

┌───────────────────┬───────────────────────────────────┬─────────────────────────────────────────────────────┐
│      Metric       │             Compares              │                      Isolates                       │
├───────────────────┼───────────────────────────────────┼─────────────────────────────────────────────────────┤
│ Context recall    │ contexts vs ground_truth          │ retrieval — did search pull needed info at all      │
├───────────────────┼───────────────────────────────────┼─────────────────────────────────────────────────────┤
│ Context precision │ contexts vs question/ground_truth │ retrieval — is pulled stuff relevant, or noise      │
├───────────────────┼───────────────────────────────────┼─────────────────────────────────────────────────────┤
│ Completeness      │ answer vs ground_truth            │ generation — did model use/state what was available │
├───────────────────┼───────────────────────────────────┼─────────────────────────────────────────────────────┤
│ Correctness       │ answer vs ground_truth            │ generation — is what model stated actually true     │
└───────────────────┴───────────────────────────────────┴─────────────────────────────────────────────────────┘

Context metrics judge what retrieval handed over, before generation touches it. Completeness/correctness judge what model actually wrote. Different pipeline stage, same P/R formula reused.

Why both needed — cases where they diverge tell you exactly where bug lives:

- High context recall, low completeness → retrieval did its job, model had needed info in context but didn't use it. Generation/prompt problem, not retrieval. Don't touch chunking.
- Low context recall, high correctness → model answered correctly without needed context present. Means model guessed from parametric/pretrained knowledge, not retrieved docs. Looks fine on correctness, but faithfulness would catch this as hallucination risk — answer right today, wrong next time docs update, model won't know.
- Low context recall, low completeness → retrieval missing the info, model correctly couldn't state what it never got. Root cause in retrieval, fix corpus/chunking, not prompt.
- High context recall, low correctness → model got the right info and still stated it wrong. Pure generation bug — grounding instruction, reasoning error, or model just answering badly despite good context.

That's why stage-labeling matters (from debug order given earlier): same formula at two stages lets you triangulate instead of guessing. One pair without the other leaves you blind to which side broke.

✻ Baked for 16s · done Saturday 8:09 AM

❯ can theh be binary as well?

Yes — already built binary above. Confirm what's binary in each:

- Context recall: atomic unit = ground_truth claim. Binary judge per claim: "supported by any retrieved context? yes/no." Recall = yes-count / total claims.
- Context precision: atomic unit = retrieved chunk. Binary judge per chunk: "relevant to answering question? yes/no." Precision = yes-count / total chunks.

Different atomic unit (claim vs chunk) because they're checking different direction — recall walks ground_truth outward to contexts, precision walks contexts outward to question. Both come out as plain yes/no judgments counted into a fraction, same pattern as faithfulness/correctness/completeness.

All six metrics now share one judge function + one decomposer. No metric needs anything beyond judge_binary + decompose_claims already written.

✻ Cooked for 8s · done Saturday 8:09 AM

❯ compelteness and correctenss also bianry?

Yes — already binary too, same pattern.

- Correctness: atomic unit = answer claim. Binary judge per claim: "consistent with ground truth? yes/no." Correctness = yes-count / total answer claims.
- Completeness: atomic unit = ground_truth claim. Binary judge per claim: "covered in the answer? yes/no." Completeness = yes-count / total ground_truth claims.

All 6 metrics (context recall, context precision, faithfulness, correctness, completeness) now same shape: decompose → binary judge per atomic unit → count yes / total. Only two things change per metric: which text gets decomposed into claims (answer vs ground_truth vs none — precision uses chunks directly), and which text each claim gets checked against. One judge function, one decomposer, six metrics.

✻ Baked for 4s · done Saturday 8:10 AM

❯ I was also trying to look fro possible questions that I can ask myself and see where the agentic version fails in order to improve it

Agentic-specific failure modes — different from plain RAG failures, specific to tool-calling loop itself. Check real traces against these:

Tool selection

- Does it call search when it should? (skips search, answers from parametric knowledge — faithfulness risk)
- Does it call search when it shouldn't? (simple query, single search sufficient, but it loops anyway — latency/cost waste)
- Given multiple tools, does it pick right one? (e.g. calls doc-search for a question needing table-lookup tool, or vice versa)

Query formulation into tool call

- Does search query it generates actually match query intent, or garbled/over-literal copy of user's raw question?
- Does it reformulate query after a bad first search, or send same query twice?
