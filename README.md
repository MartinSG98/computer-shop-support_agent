# Computer Shop Support Agent

Customer support agent for the Computer Shop project. Answers product and PC hardware
questions in chat, points customers to the Build a PC page when they want a full build,
and stays politely on topic for everything else.

Part of the Computer Shop polyrepo, alongside the FastAPI backend, the React frontend
and the Terraform infrastructure.

## How it works

- [Strands Agents](https://strandsagents.com) provides the agent loop.
- The model is Amazon Nova Lite on Bedrock (`amazon.nova-lite-v1:0`), the same model
  the build evaluator uses.
- `bedrock_agentcore` wraps the agent as the HTTP service that
  [Amazon Bedrock AgentCore Runtime](https://aws.amazon.com/bedrock/agentcore/) expects,
  so the same file runs locally and in the cloud.
- Deployment target is AgentCore Runtime in eu-west-2 using direct code deployment
  (zip via S3, no container).

The whole agent currently lives in `agent.py`: a structured system prompt tuned for
Nova Lite, plus the entrypoint that AgentCore invokes.

## Run it locally

You need Python 3.10 to 3.13 and AWS credentials with Bedrock access in eu-west-2.

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -r requirements.txt
python agent.py
```

Then from another terminal:

```powershell
Invoke-RestMethod -Uri http://localhost:8080/invocations -Method Post `
  -ContentType "application/json" -Body '{"prompt": "Do you sell graphics cards?"}'
```

## Status

Work in progress. Done so far: minimal agent with system prompt, local run. Next up:
catalog lookup tool (DynamoDB), then deployment to AgentCore Runtime and wiring into
the backend.
