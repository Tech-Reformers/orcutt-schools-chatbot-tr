# Code Review — Orcutt Schools Chatbot

Date: 2026-09-04
Scope: `lambda/chatbot/lambda_function.py`, `infrastructure/orcutt_chatbot_stack.py`
Git state at review: clean, up to date with `origin/main` (HEAD `136cdb2`)

The latest commit swapped the Bedrock model to the `claude-sonnet-4-5` inference
profile and dropped `top_p` (which conflicts with `temperature` on Claude 4.5).
That change is correct.

Findings below are ordered by priority.

## Security issues

### 1. Hardcoded AWS account ID and resource IDs in the CDK stack
File: `infrastructure/orcutt_chatbot_stack.py`

- Certificate ARN with account `785054116835` is hardcoded.
- `KNOWLEDGE_BASE_ID: "GCERPWLGOK"` is hardcoded in the chatbot Lambda env.
- Domain `orcutt-ai.techreformers.com` is hardcoded.

These belong in `config.py` / `config.yaml` per environment. This also directly
conflicts with the `dev-prod-environments` spec — hardcoded IDs will block clean
environment separation. Note the stack creates a `CfnKnowledgeBase` (`kb`) but the
Lambda points at a *different* hardcoded KB ID, so the provisioned KB is never used
by the chatbot. Either wire the Lambda to `kb.ref` or remove the created KB.

### 2. Over-broad IAM permissions
- Chatbot Lambda gets `bedrock:*` on `*`. Should be scoped to `bedrock:InvokeModel`,
  `bedrock:Retrieve`, `bedrock:ApplyGuardrail` on the specific model and KB ARNs.
- `index_creator_role` grants `es:*` and `opensearch:*` on `*`. The `"*"` resource
  wildcard alongside the domain ARNs makes the domain-scoping pointless.
- `bedrock_role` uses broad managed policies (`AmazonBedrockFullAccess`,
  `AmazonOpenSearchServiceFullAccess`). Least-privilege would be tighter.

### 3. CORS is fully open
`get_cors_headers()` returns `Access-Control-Allow-Origin: *` while API Gateway uses
`self.config.API_CORS_ALLOW_ORIGINS`. The Lambda headers override the intent of the
restricted config. Lock to the known frontend domain `orcutt-ai.techreformers.com`.

## Correctness / robustness

### 4. Error responses leak internals
`create_error_response(500, f"Internal server error: {str(e)}")` returns raw exception
text to the client. Log the detail, return a generic message.

### 5. `parse_response` can miscount sources
It only keeps digits via `isdigit()`, so model output like `[1, 2, 3.]` or `[one]`
silently drops entries. Add a guard.

### 6. Sequential message IDs are racy
`get_next_message_id` does a `COUNT` query then `+1`. Concurrent requests in the same
session can collide on the same `convN` id. The item still saves (sort key is
`timestamp`), but two messages could share a `message_id`, which then breaks feedback
updates (`update_conversation_with_feedback` takes `Items[0]`). Low probability for
this traffic, but real.

### 7. School-index parsing is fragile
`int(query_type.split("_")[-1]) - 1` trusts the Nova classifier output. A malformed
`knowledge_base_x` raises `ValueError` and falls into the generic error path. Validate
the index range.

## Minor / cleanup

### 8. Inconsistent logging
Mixes `logger.error(...)` and `logging.error(...)`. The module-level `logger` is
configured; the bare `logging.*` calls bypass it.

### 9. Dead code
`rerank_sources`, `is_date_query`, `prioritize_future_dates`, `generate_presigned_url`
appear unused now that reranking is disabled and the web crawler replaced S3 PDFs.
Remove or mark clearly.

### 10. Full prompt logged at INFO
`logger.info(prompt)` logs KB context and user data — high CloudWatch volume and
potential PII exposure. Drop to DEBUG or redact.

## What's good
- Clean separation of chat vs feedback routing.
- Guardrails fail open deliberately (reasonable for availability — worth a conscious
  decision).
- Config-driven infra for most resources via `config.py`.
- Thoughtful date-aware prompting and source-prioritization instructions.

## Suggested starting point
1. Move hardcoded IDs to config (#1) — aligns with the `dev-prod-environments` spec.
2. Scope the `bedrock:*` IAM policy (#2).

Both can be done without changing behavior.
