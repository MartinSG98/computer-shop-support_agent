# Computer Shop Support Agent

![CI](https://github.com/MartinSG98/computer-shop-support-agent/actions/workflows/ci.yml/badge.svg)

Customer support agent for the Computer Shop project. Answers product and PC hardware
questions in chat, looks up real products and categories in DynamoDB, points customers
to the Build a PC page when they want a full build, and stays politely on topic for
everything else.

## How it works

- [Strands Agents](https://strandsagents.com) provides the agent loop.
- The model is OpenAI's gpt-oss-120b on Bedrock (`openai.gpt-oss-120b-1:0`),
  chosen in a bake-off across several models (see Model choice below). Steered
  by a structured system prompt (explicit rules, few-shot examples).
- `bedrock_agentcore` wraps the agent as the HTTP service that
  [Amazon Bedrock AgentCore Runtime](https://aws.amazon.com/bedrock/agentcore/) expects,
  so the same file runs locally and in the cloud.
- Deployment target is AgentCore Runtime in eu-west-2 using direct code deployment
  (zip via S3, no container), provisioned through the project's Terraform module and
  stack.

The whole agent lives in `agent.py`.

## Tools

The agent is grounded in the shop's DynamoDB tables through two read-only tools:

| Tool | Answers | Source |
| --- | --- | --- |
| `search_products` | "Whats your cheapest GPU?", "Do you have AMD CPUs in stock?" | products table |
| `list_categories` | "What do you actually sell?" | categories table |

`search_products` matches a single term (category slug, brand, or product name
fragment) against name, brand, category and description. A small alias map translates
customer vocabulary ("gpu", "ram", "psu") into the category slugs that actually appear
on products. Results are compact, JSON-safe summaries, sorted into shape for the model
and capped at 20.

## Model choice

The agent's hardest job is compatibility questions ("which of your cases fit
this PSU?"), which require cross-referencing one product against every case in
the catalog and naming both the fits and the misfits. We bake-tested several
Bedrock serverless models on exactly that question:

| Model | Result |
| --- | --- |
| Amazon Nova Lite | Applied the right rule, but the lists were incomplete or wrong on every attempt |
| NVIDIA Nemotron 3 Nano | Ignored the tools entirely and improvised ("I don't store stock info") |
| Amazon Nova 2 Lite | Skipped the tool on follow-ups, then wrongly claimed every case fits |
| OpenAI gpt-oss-120b | Correct rule, all 11 fitting cases named, and the one SFX-only case called out |

gpt-oss-120b was the only model to pass, at roughly $0.15/$0.60 per million
tokens - about $0.002 per customer question with the full-catalog tool. Best
results for the price by a wide margin. (Claude Haiku 4.5 would likely pass
too at several times the cost; untested because Anthropic models require a
use-case form first.)

## API

`POST /invocations` with `{"prompt": "..."}` returns `{"reply": "..."}`.

## Run it locally

You need Python 3.10 to 3.13 and AWS credentials with Bedrock and DynamoDB read
access in eu-west-2.

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -r requirements.txt
$env:PRODUCTS_TABLE = "computer-shop-products"
$env:CATEGORIES_TABLE = "computer-shop-categories"
python agent.py
```

Then from another terminal:

```powershell
$r = Invoke-RestMethod -Uri http://localhost:8080/invocations -Method Post `
  -ContentType "application/json" -Body '{"prompt": "Whats your cheapest GPU?"}'
$r.reply
```

The agent makes real Bedrock and DynamoDB calls even when running locally, so the
env vars and credentials are required. Missing table env vars fail fast at startup.

## Status

Working locally with both tools. Next up: AgentCore Runtime resources in the
Terraform module (execution role, artifacts bucket, runtime), apply through the
stack, then wire a chat box into the frontend via the backend.

## Related

Part of the Computer Shop project:

- [computer-shop-backend](https://github.com/MartinSG98/computer-shop-backend) — FastAPI backend API
- [computer_shop_ui](https://github.com/MartinSG98/computer_shop_ui) — React/Vite/Mantine frontend
- [computer-shop-build-eval](https://github.com/MartinSG98/computer-shop-build-eval) — PC build scorer + suggestions (eval Lambda)
- [tf-module-computer_shop](https://github.com/MartinSG98/tf-module-computer_shop) — Terraform infrastructure module
- [tf-stack-computer_shop](https://github.com/MartinSG98/tf-stack-computer_shop) — Terraform deployment stack
