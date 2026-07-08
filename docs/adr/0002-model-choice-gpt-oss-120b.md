# ADR-0002: gpt-oss-120b as the agent model

- Status: Accepted
- Date: 2026-07-09

## Context

The agent's hardest task is compatibility questions that require cross-
referencing one product against the whole catalog and naming both the fits and
the misfits. Cheap serverless Bedrock models vary widely on this.

## Decision

Use OpenAI's gpt-oss-120b on Bedrock (`openai.gpt-oss-120b-1:0`), chosen by a
bake-off on exactly that compatibility question.

## Consequences

- The only model in the bake-off that applied the right rule and produced
  complete, correct lists.
- Roughly $0.15/$0.60 per 1M tokens, about $0.002 per question with the
  full-catalog tool. Best result for the price by a wide margin.

## Alternatives considered

- Amazon Nova Lite / Nova 2 Lite: right rule but incomplete/wrong lists, or
  skipped the tool on follow-ups. Rejected.
- NVIDIA Nemotron 3 Nano: ignored the tools entirely. Rejected.
- Claude Haiku 4.5: would likely pass at several times the cost; untested because
  Anthropic models require a use-case form first.

See the agent README "Model choice".
