# Computer Shop Support Agent

Customer support agent for the Computer Shop project. Answers product and PC hardware
questions in chat, looks up real products and categories in DynamoDB, points customers
to the Build a PC page when they want a full build, and stays politely on topic for
everything else.

## How it works

- [Strands Agents](https://strandsagents.com) provides the agent loop.
- The model is Amazon Nova Lite on Bedrock (`amazon.nova-lite-v1:0`), the same model
  the build evaluator uses, steered by a structured system prompt tuned for small
  models (explicit rules, category vocabulary, few-shot examples).
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
