# ADR-0001: Strands agent on Bedrock AgentCore Runtime (S3-zip deploy)

- Status: Accepted
- Date: 2026-07-09

## Context

The shop needs a customer support chat agent that answers product/hardware
questions grounded in the live DynamoDB catalog, deployed cheaply, running the
same code locally and in the cloud.

## Decision

- Build the agent loop with Strands Agents, grounded by two read-only DynamoDB
  tools (`search_products`, `list_categories`).
- Wrap it with `bedrock_agentcore` and deploy to Amazon Bedrock AgentCore Runtime
  (eu-west-2) via direct code deployment (zip through S3, no container).
- Provision it through the project's Terraform module/stack. It is invoked by the
  backend's `/chat` proxy, which keeps the runtime private and reuses the API's
  CORS.

## Consequences

- The same `agent.py` runs locally and in the cloud (the `bedrock_agentcore` HTTP
  wrapper).
- No container image or registry to manage (S3 zip).
- The frontend never calls the agent directly; the backend proxy fronts it.

## Alternatives considered

- Container-image deploy to AgentCore: heavier (build + ECR) for no gain at this
  size.
- A plain Lambda + custom HTTP: would reimplement the agent-runtime contract that
  AgentCore already provides.
