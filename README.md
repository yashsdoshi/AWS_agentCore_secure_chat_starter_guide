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

### 2.1 Build the code zip

```bash
agentcore build
```
This creates a zip file with `main.py` at the root level

### 2.2 Upload the zip from step 2.1 to `S3`

- [] Create a new bucket and name it something appropriate.
- [] Keep the default settings and hit upload!
- [] Copy the S3 URI

### 2.3 Create a AgentCore Runtime

#### 2.3.1 Go to AgentCore then in `Agents` in leftSidebar and then to `host agents`
<img width="1919" height="942" alt="image" src="https://github.com/user-attachments/assets/5e49e612-6b2e-4ddf-8d76-330b14849435" />

#### 2.3.2 Create a runtime

## AWS AgentCore Runtime Configuration

| Setting | Value / Selection | Explanation |
|---|---|---|
| **Name** | `gigachadAgent_runtime` | Use letters, numbers, and underscores only. |
| **Runtime platform** | **V1** | Select V1. |
| **Code source** | **S3** | Paste the S3 URI of the ZIP file uploaded in Step 2.2. |
| **Runtime** | **Python 3.14** | Must match `runtimeVersion` in `agentcore.json`. |
| **Entry point** | `opentelemetry-instrument main.py` | Starts the agent and enables tracing in CloudWatch Observability. Plain `main.py` also works but may not produce traces. |
| **IAM role** | **Create new role** | Or select an existing AgentCore execution role with the required permissions. |
| **Network** | **Public** | Allows the agent to access external services such as OpenAI. |
| **Environment variables** | `OPENAI_API_KEY` = your OpenAI API key | Used by `main.py` to authenticate with OpenAI. |
| **Filesystem configuration** | **Leave empty** | Not required. |
| **Inbound Auth** | **JSON Web Tokens (JWT)** | Enables JWT authentication for incoming requests. |
| **Identity provider configuration** | **Use existing Identity provider configurations** | Reuses your existing Cognito User Pool. Do not select **Quick create**, as it creates a new pool. |
| **Discovery URL** | `https://cognito-idp.<region>.amazonaws.com/<user-pool-id>/.well-known/openid-configuration` | Replace `<region>` and `<user-pool-id>` with your Cognito details. |
| **Allowed clients** | **Your Cognito App Client ID** | Specifies which Cognito app client is allowed to access the runtime. |
| **Allowed audiences** | **Leave empty** | Cognito access tokens use `client_id` instead of `aud`. The **Allowed clients** setting checks this value. |

Then hit the `Create Runtime` button!!

#### 2.3.3 Copy the runtime ARN
<img width="1919" height="956" alt="image" src="https://github.com/user-attachments/assets/701430e1-7fc4-493b-b8fe-87a5829be78e" />

# Step 3 - Test the Agent

### 3.1 Copy your cognito Access token 

### 3.2 Create a python test script to test the deployed agent!

1. Create a test.py file (Replace the line 2 value with your actual ARN)

```bash
import base64, getpass, json, urllib.error, urllib.parse, urllib.request, uuid

REGION = "us-west-2"
RUNTIME_ARN = "<your runtime ARN>"

token = getpass.getpass("Paste your Cognito ACCESS token (it stays hidden), then press Enter: ").strip()
prompt = input("Type your message, then press Enter: ").strip()

# Quick check: AgentCore needs the access token (has client_id), not the ID token.
claims = json.loads(base64.urlsafe_b64decode(token.split(".")[1] + "=="))
print("token_use:", claims.get("token_use"), "| client_id:", claims.get("client_id"))

url = (f"https://bedrock-agentcore.{REGION}.amazonaws.com/runtimes/"
       f"{urllib.parse.quote(RUNTIME_ARN, safe='')}/invocations?qualifier=DEFAULT")
req = urllib.request.Request(url, method="POST",
    data=json.dumps({"prompt": prompt}).encode(),
    headers={
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json",
        "Accept": "text/event-stream",
        "X-Amzn-Bedrock-AgentCore-Runtime-Session-Id": str(uuid.uuid4()),
    })
try:
    with urllib.request.urlopen(req) as resp:
        for line in resp:
            print(line.decode().rstrip())
except urllib.error.HTTPError as e:
    print("HTTP", e.code, e.read().decode())

```

2. Run the test.py file

```bash
python test.py
```
Common error you may face and how to resolve them

| Error | Cause | Fix |
|---|---|---|
| `Claim 'client_id' value mismatch` | You used the ID token (`token_use: id`) | Use the access token |
| `Ineffectual token, will expire within the next minute` | The token is about to expire | Log in again and get a fresh token |
| `HTTP 401` with any other message | Wrong Discovery URL or app client ID | Check the settings from 2.4 |

# Step 3 - Precautionary Step (Hide your ARN!)

You can now connect this agent to a website and talk to it once the user is logged in using the same cognito user pool but the ARN will be exposed to the end user which will cause unnecessory information disclosure.

### Step 3.1 - Create a new cloudfront function
<img width="1902" height="945" alt="image" src="https://github.com/user-attachments/assets/ae0149ff-1610-4a10-b161-65a50211cd2e" />

### Step 3.2 - Add code to hide the ARN
paste this code in the build tab once you create the function (Update the code as per your need)

```bash
function handler(event) {
    var request = event.request;

    // Replace with your own Runtime ARN (AgentCore console → your agent → Runtime ARN).
    // It lives only here, at the CloudFront edge. The browser never sees it.
    var RUNTIME_ARN = "arn:aws:bedrock-agentcore:<region>:<account-id>:runtime/<runtime-id>";

    // Send every request on this path to the AgentCore invoke endpoint.
    request.uri = "/runtimes/" + encodeURIComponent(RUNTIME_ARN) + "/invocations";
    request.querystring = { qualifier: { value: "DEFAULT" } };
    return request;
}
```

And then test the agent through postman!

# Step 4 - The most important step! 
# Give this repo a well deserving starrr ⭐
