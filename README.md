# A simple step-by-step guide for deploying an Agent to `AWS AgentCore` with `Cognito` Authentication

# Note from Owner
- [x] <b>I have kept the agent very simple because the main focus of this documentation is on deployment and auth.</b>

# Step 0 - prerequisites

### Accounts & Keys
- **AWS Account** with Amazon Bedrock AgentCore access (guide uses `us-west-2`)
- **Amazon Cognito User Pool** with an **App Client** and an active user (save the **User Pool ID** and **App Client ID**)
- **OpenAI API Key** from [platform.openai.com](https://platform.openai.com)

### Local Tools
- **Python 3.12+** & **uv** — local testing and dependency builds
- **Node.js 20+ & npm** — AgentCore CLI (`npm install -g @aws/agentcore`)
- **AWS CLI v2** — authenticated via `aws login` or environment credentials
- **Postman or `curl`** *(optional)* — endpoint testing

### AWS Permissions
Ensure your IAM user/role can:
- Manage **AgentCore runtimes** and upload code bundles to **S3**
- Create or pass an **IAM execution role** (`iam:CreateRole`, `iam:PassRole`)
- *(Optional)* Assume **CDK bootstrap roles** (`cdk-hnb659fds-*`) for CLI deployments
- *(Optional)* Manage **CloudFront** functions and distributions

# Step 1 - Local Agent setup
### 1.1 Create the AgentCore project
Install the AgentCore CLI and create an empty project:
```bash
  npm install -g @aws/agentcore
  agentcore create --name GigaChadAgent --template empty
  cd GigaChadAgent
```
and then add a runtime (your agent) with Cognito login turned on:
```bash
agentcore add runtime --name chatagent --language python --authorizer-type CUSTOM_JWT --authorizer-configuration "{\"customJwtAuthorizer\":{\"discoveryUrl\":\"https://cognito-idp.<region>.amazonaws.com/<user-pool-id>/.well-known/openid-configuration\",\"allowedClients\":[\"<app-client-id>\"]}}"
```
<div align="center">👆 Fill in the placeholder text with actual values.</div>

<div align="center">👇 After the commands the project will be scaffolded with the following folder structure</div>

```text
openai-agentcore/
├── GigaChadAgent/                     # Made by `agentcore create`
│   ├── .gitignore                  # Keeps .env.local, build files and node_modules out of git
│   ├── agentcore/
│   │   ├── agentcore.json          # Runtime config (auth, env vars)
│   │   ├── aws-targets.json        # AWS account + region to deploy to (starts empty)
│   │   ├── .env.local              # Local-only env vars for `agentcore dev` (git-ignored)
│   │   └── cdk/                    # Deploy code the CLI uses (don't edit)
│   └── app/
│       └── GigaChadAgent/              # Made by `agentcore add runtime`
│           ├── README.md           # Help file (made by the CLI)
│           ├── AGENTS.md           # Help for AI coding assistants (made by the CLI)
└── test_agent.py                   # Deployed agent test script

```

The AgentCore command have also created AGENTS.md which has instructions for your coding agents.

### 1.2 Add `main.py` and `pyproject.toml` in app\GigaChadAgent.

```bash
type nul >> main.py
type nul >> pyproject.toml
```

### 1.3 Add the code for the agent, and populate pyproject.toml

Add this in main.py

```bash
import os
from openai import AsyncOpenAI
from bedrock_agentcore.runtime import BedrockAgentCoreApp
from bedrock_agentcore.identity.auth import requires_api_key

app = BedrockAgentCoreApp()

_openai_client = None


# Deployed: fetches the OpenAI key from AgentCore Identity (added with `agentcore add credentials`).
@requires_api_key(provider_name=os.environ.get("CREDENTIAL_OPENAI_NAME", "openai"))
async def _fetch_openai_key(*, api_key: str) -> str:
    return api_key


async def get_openai_client() -> AsyncOpenAI:
    global _openai_client
    if _openai_client is None:
        # Local: use OPENAI_API_KEY if set. Deployed: fetch from AgentCore Identity.
        key = os.environ.get("OPENAI_API_KEY") or await _fetch_openai_key()
        _openai_client = AsyncOpenAI(api_key=key)
    return _openai_client


@app.entrypoint
async def invoke(payload: dict, context):
    prompt = payload.get("prompt", "")
    model = payload.get("model", "gpt-4o-mini")

    if not prompt:
        yield {"error": "Prompt is required"}
        return

    try:
        openai_client = await get_openai_client()
        stream = await openai_client.chat.completions.create(
            model=model,
            messages=[
                {"role": "system", "content": "You are a helpful assistant."},
                {"role": "user", "content": prompt}
            ],
            stream=True
        )
        # Yielding dicts streams SSE chunks: data: {"delta": "..."}\n\n
        async for chunk in stream:
            delta = chunk.choices[0].delta.content if chunk.choices else None
            if delta:
                yield {"delta": delta}

        yield {"done": True, "session_id": context.session_id}
    except Exception as e:
        yield {"error": str(e)}


if __name__ == "__main__":
    app.run()
```

And this in pyproject.toml

```bash
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "GigaChadAgent"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "bedrock-agentcore >= 1.26.0, < 2.0.0",
    "aws-opentelemetry-distro ~= 0.21.0",
    "openai >= 3.51.0, < 4.0.0",
]

[tool.hatch.build.targets.wheel]
packages = ["."]
```

### 1.4 Stay in the `app\chatagent` folder and run this

```bash
uv sync
```

### 1.5 Run the agent Locally
Put your actual openAI API key in the command below
```bash
set OPENAI_API_KEY=sk-proj-...
uv run main.py
```

Open a second cmd window and leave the first one running and put this command in it
```bash
curl -N -X POST http://localhost:8080/invocations -H "Content-Type: application/json" -d "{\"prompt\": \"Pray to god that I will make this world a better place.\"}"
```
You must see something like this as output `data: {"delta": ...}`

# Step 2 - Deploying the Agent to agentCore
