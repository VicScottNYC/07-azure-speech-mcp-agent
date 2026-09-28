# Azure Speech MCP-Enabled AI Agent

An MCP-enabled AI speech agent built with **Microsoft Foundry**, **Azure Speech**, and Python. The project demonstrates how a Python client can interact with a Microsoft Foundry agent that has access to speech capabilities through the **Model Context Protocol (MCP)**.

> **Portfolio Project 07 — Intermediate Track**
>
> This project advances the portfolio from locally registered Agent Framework tools into standardized external tool integration through MCP.

---

## Project Overview

The application provides a command-line client that sends natural-language requests to an existing Microsoft Foundry agent.

The Python client connects to the Foundry project using `AIProjectClient`, obtains an OpenAI-compatible client, and sends user prompts to the configured agent through an `agent_reference`.

The referenced Foundry agent is configured with access to Azure Speech capabilities through an MCP server. This allows speech-related operations to be exposed to the agent without implementing those capabilities directly inside the Python client.

During portfolio validation, the application successfully:

1. Loaded the Foundry endpoint and agent name from local environment configuration.
2. Authenticated to Azure using `DefaultAzureCredential`.
3. Created an `AIProjectClient`.
4. Obtained an OpenAI-compatible client from the Foundry project.
5. Accepted a natural-language request from the command line.
6. Routed the request to the configured Foundry agent using `agent_reference`.
7. Successfully exercised the agent's configured Azure Speech MCP capability.
8. Returned the resulting agent response to the client.

The MCP server itself is **not implemented by `speech-client.py`**. The Python application acts as a client to a Foundry agent whose speech capability is configured separately through Microsoft Foundry.

---

## Architecture

```text
User
 │
 ▼
Python CLI
speech-client.py
 │
 ▼
AIProjectClient
 │
 ▼
OpenAI-Compatible Client
 │
 ▼
responses.create()
 │
 │ agent_reference
 ▼
Microsoft Foundry Agent
 │
 ▼
MCP Integration
 │
 ▼
Azure Speech MCP Server
 │
 ▼
Azure Speech Capability
 │
 ▼
Agent Response
 │
 ▼
Python CLI
```

The architecture separates the **client application**, **agent**, and **external capability integration**.

The Python application is responsible for:

- Loading local configuration
- Authenticating to Azure
- Connecting to the Foundry project
- Accepting user prompts
- Referencing the configured agent
- Displaying the resulting response

The Foundry agent is responsible for interpreting the request and using its configured capabilities when appropriate.

The MCP integration provides a standardized mechanism through which external speech functionality can be made available to the agent.

For a deeper technical walkthrough, see [`docs/architecture.md`](docs/architecture.md).

---

## Technologies & Services

| Technology | Purpose |
|---|---|
| Microsoft Foundry | Hosts and manages the configured AI agent |
| Azure Speech | Provides speech-related capabilities used by the agent |
| Model Context Protocol (MCP) | Provides standardized external capability integration |
| Python | Implements the command-line client |
| Azure AI Projects SDK | Connects the application to the Foundry project |
| `AIProjectClient` | Creates the authenticated Foundry project client |
| OpenAI-compatible client | Sends requests to the configured Foundry agent |
| `agent_reference` | Identifies the existing Foundry agent that should process the request |
| Azure Identity | Provides Azure authentication support |
| `DefaultAzureCredential` | Obtains an Azure identity without embedding credentials in source code |
| `python-dotenv` | Loads local environment configuration |
| `httpx` | HTTP client dependency used by the application environment |

---

## Foundry Agent Integration

The application creates an authenticated project client:

```python
project_client = AIProjectClient(
    endpoint=foundry_endpoint,
    credential=DefaultAzureCredential(),
)
```

It then obtains an OpenAI-compatible client:

```python
openai_client = project_client.get_openai_client()
```

User requests are routed to the configured Foundry agent through:

```python
response = openai_client.responses.create(
    input=[{"role": "user", "content": prompt}],
    extra_body={
        "agent_reference": {
            "name": agent_name,
            "type": "agent_reference"
        }
    },
)
```

This design keeps the Python client lightweight. The client does not need to recreate the agent definition or directly implement its external speech capability.

---

## MCP Integration

The key engineering concept introduced by this project is **Model Context Protocol integration**.

Rather than defining the speech capability as a local Python function inside the client, the Foundry agent is configured to access Azure Speech functionality through MCP.

Conceptually:

```text
Traditional Application Integration

Application
    │
    ├── Custom API Code
    ├── Authentication Logic
    ├── Service-Specific Request Logic
    └── External Service


MCP-Enabled Agent Integration

Application
    │
    ▼
AI Agent
    │
    ▼
MCP
    │
    ▼
External Capability
```

This creates a clearer separation between the application that communicates with the agent and the external capabilities made available to that agent.

### Project #06 vs. Project #07

Project #06 demonstrated a locally registered Python tool:

```text
ExpenseClaimAgent
       │
       ▼
@tool
       │
       ▼
submit_claim()
```

Project #07 introduces a different integration model:

```text
Python Client
       │
       ▼
Foundry Agent
       │
       ▼
MCP Integration
       │
       ▼
Azure Speech Capability
```

The progression demonstrates the difference between **application-defined local tool invocation** and **MCP-enabled external capability integration**.

---

## Repository Structure

```text
07-azure-speech-mcp-agent/
│
├── .env.example
├── .gitignore
├── README.md
├── requirements.txt
├── speech-client.py
│
└── docs/
    └── architecture.md
```

Local files and directories such as `.env`, `.venv/`, Python cache files, and generated media are intentionally excluded from source control.

---

## Prerequisites

To run the client, the environment requires:

- Python
- Azure CLI
- Access to a Microsoft Foundry project
- A configured Microsoft Foundry agent
- Azure authentication permissions for the Foundry project
- The Azure Speech MCP capability configured for the referenced agent

The portfolio implementation was validated using:

```text
Python 3.14.7
```

The project dependency set is defined in `requirements.txt`:

```text
python-dotenv
azure-identity
azure-ai-projects==2.0.0b4
openai<3
httpx
```

> `azure-ai-projects==2.0.0b4` is the SDK version used by the completed implementation and is intentionally preserved in this portfolio version.

---

## Local Environment Setup

Create a dedicated Python virtual environment:

```bash
python3 -m venv .venv
```

Activate it on macOS or Linux:

```bash
source .venv/bin/activate
```

Install the required packages:

```bash
python -m pip install -r requirements.txt
```

Validate dependency compatibility:

```bash
python -m pip check
```

The validated portfolio environment returned:

```text
No broken requirements found.
```

The required SDK imports were also validated successfully.

---

## Environment Configuration

Create a local `.env` file using `.env.example` as the template:

```bash
cp .env.example .env
```

The application requires:

```env
FOUNDRY_ENDPOINT=https://your-foundry-project-endpoint
AGENT_NAME=your-agent-name
```

### `FOUNDRY_ENDPOINT`

Identifies the Microsoft Foundry project endpoint used by `AIProjectClient`.

### `AGENT_NAME`

Identifies the existing Microsoft Foundry agent that should process requests from the Python client.

The actual `.env` file is excluded through `.gitignore` and should not be committed to source control.

---

## Azure Authentication

The application authenticates using:

```python
DefaultAzureCredential()
```

No API key, password, access token, or client secret is embedded in the Python source.

For local development, authenticate to Azure using the Azure CLI:

```bash
az login
```

The active Azure subscription can be checked with:

```bash
az account show --query "{name:name,state:state}" -o table
```

The portfolio implementation was validated with an authenticated and enabled Azure subscription.

`DefaultAzureCredential` can evaluate multiple supported credential sources. In the validated local development environment, an authenticated Azure developer identity was available for the application to use.

---

## Run the Client

Start the command-line application:

```bash
python speech-client.py
```

The application prompts for input:

```text
User prompt (or 'quit'):
```

Enter a request appropriate for the capabilities configured on the referenced Foundry agent.

For example:

```text
Convert the text "Welcome to my Microsoft AI engineering portfolio" to speech.
```

The application submits the request using:

```python
openai_client.responses.create(...)
```

with an `agent_reference` identifying the configured Foundry agent.

The response returned by the agent is then displayed in the command line.

Enter:

```text
quit
```

to exit the application.

---

## Verified Execution

The independent portfolio version was validated end to end after being separated from the Microsoft Learn lab directory.

The validation confirmed:

1. The dedicated Python virtual environment activated successfully.
2. Required dependencies installed successfully.
3. `python -m pip check` reported no broken requirements.
4. Required Azure and Foundry SDK modules imported successfully.
5. Local Foundry configuration loaded successfully from `.env`.
6. Azure authentication was available to the application.
7. `AIProjectClient` connected to the configured Foundry project.
8. The OpenAI-compatible client was created successfully.
9. The application reached the interactive user prompt.
10. A natural-language speech request was routed to the referenced Foundry agent.
11. The configured Azure Speech MCP capability was exercised successfully.
12. The resulting agent response was returned to the Python client.

Conceptually, the validated execution path was:

```text
Natural-Language Request
          │
          ▼
speech-client.py
          │
          ▼
AIProjectClient
          │
          ▼
OpenAI-Compatible Client
          │
          ▼
responses.create()
          │
          ▼
agent_reference
          │
          ▼
Configured Foundry Agent
          │
          ▼
Azure Speech MCP Capability
          │
          ▼
Agent Response
          │
          ▼
Command-Line Client
```

---

## Generated Media and Sensitive URLs

Speech workflows may produce or reference generated media or temporary access locations depending on the configured agent and service behavior.

This repository intentionally excludes common generated audio formats:

```text
*.wav
*.mp3
*.m4a
```

SAS tokens, tokenized storage URLs, temporary access URLs, credentials, and environment-specific service values must not be committed to the repository.

The project documentation therefore describes the validated execution without publishing temporary or sensitive service URLs.

---

## Security Considerations

This project separates application code from environment-specific configuration and authentication material.

### Environment Configuration

The application loads configuration from environment variables:

```python
foundry_endpoint = os.getenv('FOUNDRY_ENDPOINT')
agent_name = os.getenv('AGENT_NAME')
```

The local `.env` file is excluded from Git, while `.env.example` documents only the variable names and sanitized placeholder values required by the application.

No real Foundry endpoint, credential, token, or environment-specific identifier is required to be stored in source code.

### Identity-Based Authentication

The application creates the Foundry project client using:

```python
DefaultAzureCredential()
```

This avoids embedding authentication secrets directly in the Python application.

`DefaultAzureCredential` supports a credential chain rather than representing a single authentication mechanism. The portfolio implementation was validated in a local Azure-authenticated development environment.

### Sensitive Service Data

MCP-enabled speech workflows can involve service responses containing generated media references or temporary access locations.

The repository must not contain:

- SAS tokens
- Tokenized storage URLs
- Temporary access URLs
- API keys
- Access tokens
- Client secrets
- Passwords
- Real `.env` configuration
- Generated media containing sensitive information

Common generated audio formats are excluded through `.gitignore`.

### Repository Protection

The repository excludes:

- `.env`
- Other local environment files
- `.venv/`
- Python caches
- Compiled Python files
- Generated audio
- VS Code configuration
- macOS metadata
- Test and coverage artifacts
- Build artifacts
- Logs
- Temporary files

A repository-wide sensitive-data scan should be performed before publication.

---

## Implementation Boundaries

This project demonstrates an **MCP-enabled Foundry agent integration**, but the responsibilities of each component are intentionally separated.

### Implemented in the Python Client

`speech-client.py` implements:

- Environment configuration loading
- Azure identity integration
- Foundry project connectivity
- OpenAI-compatible client creation
- Interactive command-line input
- Agent referencing
- Request submission
- Response display
- Basic exception handling

### Configured Outside the Python Client

The following capabilities are configured through the Microsoft Foundry environment rather than implemented directly inside `speech-client.py`:

- The Foundry agent definition
- The agent's model configuration
- MCP server connectivity
- Azure Speech capability exposure
- MCP tool definitions
- Speech service execution

The Python client therefore should not be interpreted as an MCP server implementation.

It is a client application that communicates with an MCP-enabled Foundry agent.

---

## Skills Demonstrated

Project 07 demonstrates hands-on experience with:

- Model Context Protocol concepts
- MCP-enabled AI agent integration
- Microsoft Foundry
- Azure Speech integration
- Azure AI Projects SDK
- `AIProjectClient`
- OpenAI-compatible Foundry clients
- Responses API usage
- `agent_reference`
- Existing-agent invocation
- `DefaultAzureCredential`
- Identity-based Azure authentication
- Environment-variable configuration
- `python-dotenv`
- Python command-line applications
- Interactive request processing
- External AI capability integration
- Dependency isolation
- Python virtual environments
- Secure configuration practices
- Git repository hygiene
- Architecture documentation
- Implementation-boundary analysis

---

## Key Engineering Takeaway

The central engineering concept demonstrated by Project 07 is that an AI application does not need to directly implement every external capability used by an agent.

Project 06 demonstrated application-defined tool execution:

```text
Agent
  │
  ▼
Registered Python Tool
  │
  ▼
Application Function
```

Project 07 introduces another integration boundary:

```text
Python Client
     │
     ▼
Foundry Agent
     │
     ▼
MCP Integration
     │
     ▼
External Speech Capability
```

This separates three major concerns:

```text
Application Layer
      │
      │ communicates with
      ▼
Agent Layer
      │
      │ accesses capabilities through
      ▼
Tool / MCP Layer
```

The Python client does not need to contain the implementation details of the speech capability itself.

Instead, it communicates with an agent that has access to the capability through its configured MCP integration.

This is an important architectural progression toward AI systems in which agents can interact with external tools and services through standardized interfaces.

---

## Training Context

This portfolio project was developed after completing the Microsoft Learn exercise:

**Develop a speech agent with the Azure Speech MCP server**

The Microsoft Learn exercise introduced the core implementation pattern used by the application, including:

- Microsoft Foundry project connectivity
- `AIProjectClient`
- `DefaultAzureCredential`
- OpenAI-compatible client access
- `agent_reference`
- Foundry agent invocation
- Azure Speech MCP integration

The independent portfolio version was maintained separately from the cloned Microsoft Learn repository.

Additional portfolio work included:

- Creating a dedicated Project 07 repository structure
- Creating an isolated Python virtual environment
- Revalidating the Microsoft Learn dependency set
- Verifying dependency compatibility
- Validating required Python imports
- Creating a sanitized `.env.example`
- Protecting the working `.env` through `.gitignore`
- Excluding generated audio artifacts
- Revalidating Azure authentication
- Revalidating the Python client outside the original lab directory
- Performing an independent end-to-end execution test
- Confirming successful routing to the configured Foundry agent
- Confirming successful use of the configured Azure Speech MCP capability
- Separating Python client responsibilities from Foundry/MCP responsibilities
- Documenting the architecture and security boundaries
- Preparing the project for independent source-control publication

Environment-specific credentials, SAS tokens, temporary access URLs, and Microsoft-provided virtual environments are not included in this repository.

---

## Portfolio Progression

This repository is the second project in the portfolio's **Intermediate Track**.

| Project | Engineering Focus |
|---|---|
| 01 | Microsoft Foundry AI Agent |
| 02 | Agent-Driven Workflow |
| 03 | Generative AI Chat Application |
| 04 | Foundry IQ Knowledge Agent / Enterprise RAG |
| 05 | Generative AI Guardrails & Evaluation |
| 06 | Microsoft Agent Framework + Custom Tools |
| **07** | **MCP-Enabled Agent + Azure Speech** |

The progression through Projects 06 and 07 is particularly important.

### Project 06

```text
Microsoft Agent Framework
          │
          ▼
Agent
          │
          ▼
Locally Registered @tool
          │
          ▼
Python Function
```

Project 06 established **agent-directed application function invocation**.

### Project 07

```text
Python Client
      │
      ▼
Microsoft Foundry Agent
      │
      ▼
MCP Integration
      │
      ▼
Azure Speech Capability
```

Project 07 advances the portfolio into **standardized external capability integration through MCP**.

Together, the projects demonstrate two different agent extensibility patterns:

```text
06 — Local Tool Integration
             │
             ▼
07 — MCP-Enabled External Integration
```

This establishes the foundation for subsequent intermediate projects involving advanced enterprise retrieval, multi-agent orchestration, GenAIOps, observability, and enterprise AI security.

---

## Detailed Architecture

For a deeper technical walkthrough covering client initialization, identity, Foundry connectivity, agent referencing, MCP boundaries, request flow, and security considerations, see:

[`docs/architecture.md`](docs/architecture.md)

---

## Disclaimer

This repository is an independent educational portfolio project developed from concepts practiced through Microsoft Learn.

The application is intended to demonstrate Microsoft Foundry agent connectivity and MCP-enabled Azure Speech integration. It is not presented as a production speech platform, production MCP server implementation, or production identity architecture.