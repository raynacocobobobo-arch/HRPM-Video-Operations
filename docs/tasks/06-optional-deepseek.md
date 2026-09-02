# Optional post-v1: DeepSeek assistant

## Status

Deferred. Do not start until Packages 1–5 are complete and the user explicitly approves this extension.

## Allowed uses

- Project and risk summaries.
- Natural-language read queries.
- Monthly report drafts.
- Schedule explanation in plain Chinese.
- Retrospective and template suggestions.

## Prohibited uses

- Direct schedule application.
- Rate, budget, phase-gate, or template changes without confirmation.
- Browser-side API keys.
- Uploading real HR/project data to a cloud API without explicit approval.

## Recommended boundary

Implement a backend provider adapter that can target local Ollama or the official DeepSeek API. Treat all responses as untrusted suggestions and preserve audit/human-confirmation boundaries.

