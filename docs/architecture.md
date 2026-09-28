# Architecture — Azure Speech MCP-Enabled AI Agent

## 1. Architecture Overview

Project 07 demonstrates a Python client application communicating with a Microsoft Foundry agent that has access to Azure Speech capabilities through a Model Context Protocol (MCP) integration.

The architecture separates the system into three primary layers:

1. **Client Application Layer** — accepts user input, loads configuration, authenticates to Azure, and communicates with Microsoft Foundry.
2. **Agent Layer** — the configured Microsoft Foundry agent interprets the user's request and determines how its available capabilities should be used.
3. **MCP Capability Layer** — exposes Azure Speech functionality to the configured agent through the MCP integration.

The Python client does not directly implement the speech service or MCP server. Its responsibility is to establish authenticated Foundry connectivity and submit requests to the existing agent.

---

## 2. High-Level Architecture

```text
┌───────────────────────────────┐
│             User              │
└───────────────┬───────────────┘
                │
                │ Natural-language request
                ▼
┌───────────────────────────────┐
│      Python CLI Client        │
│      speech-client.py         │
└───────────────┬───────────────┘
                │
                ├── .env configuration
                │
                └── DefaultAzureCredential
                │
                ▼
┌───────────────────────────────┐
│       AIProjectClient         │
│     Microsoft Foundry         │
└───────────────┬───────────────┘
                │
                ▼
┌───────────────────────────────┐
│   OpenAI-Compatible Client    │
│  project_client.get_openai_   │
│           client()            │
└───────────────┬───────────────┘
                │
                │ responses.create()
                │ + agent_reference
                ▼
┌───────────────────────────────┐
│  Configured Foundry Agent     │
└───────────────┬───────────────┘
                │
                │ MCP capability
                ▼
┌───────────────────────────────┐
│    MCP Integration Boundary   │
└───────────────┬───────────────┘
                │
                ▼
┌───────────────────────────────┐
│ Azure Speech MCP Capability   │
└───────────────┬───────────────┘
                │
                │ Result
                ▼
┌───────────────────────────────┐
│       Foundry Agent           │
└───────────────┬───────────────┘
                │
                │ Agent response
                ▼
┌───────────────────────────────┐
│       Python CLI Client       │
└───────────────────────────────┘
```

---

## 3. Architectural Pattern

The application follows an **agent-mediated external capability integration** pattern.

The Python application does not contain direct Azure Speech execution logic. Instead, it communicates with an existing Foundry agent:

```text
Client Application
        │
        ▼
Foundry Agent
        │
        ▼
MCP Integration
        │
        ▼
External Capability
```

This separates application concerns from agent capabilities.

The client is responsible for communication and identity. The agent is responsible for processing the request. The configured MCP integration provides access to the external speech capability.

This differs from directly embedding service-specific execution logic into the client application.

---

## 4. Client Application Layer

The application entry point is:

```text
speech-client.py
```

The client performs the following responsibilities:

```text
Application Startup
        │
        ▼
Load Environment Configuration
        │
        ▼
Establish Azure Identity
        │
        ▼
Create AIProjectClient
        │
        ▼
Create OpenAI-Compatible Client
        │
        ▼
Accept User Input
        │
        ▼
Reference Existing Foundry Agent
        │
        ▼
Submit Request
        │
        ▼
Display Agent Response
```

The client intentionally remains lightweight.

It does not define the Foundry agent, implement the MCP server, or contain the Azure Speech execution logic.

---

## 5. Configuration Flow

Application configuration is loaded using:

```python
load_dotenv()

foundry_endpoint = os.getenv('FOUNDRY_ENDPOINT')
agent_name = os.getenv('AGENT_NAME')
```

Two environment variables are required:

```text
FOUNDRY_ENDPOINT
AGENT_NAME
```

`FOUNDRY_ENDPOINT` identifies the Microsoft Foundry project endpoint.

`AGENT_NAME` identifies the existing Foundry agent that should receive the request.

The local `.env` file contains environment-specific values and is excluded from source control.

The repository provides `.env.example` containing sanitized placeholders so another developer can understand the required configuration without exposing the original environment.

---

## 6. Identity and Authentication

The project uses Azure Identity:

```python
DefaultAzureCredential()
```

The credential is supplied when `AIProjectClient` is initialized:

```python
project_client = AIProjectClient(
    endpoint=foundry_endpoint,
    credential=DefaultAzureCredential(),
)
```

This design avoids embedding authentication secrets directly in the Python application.

Conceptually:

```text
Python Application
        │
        ▼
DefaultAzureCredential
        │
        ▼
Available Azure Identity
        │
        ▼
Microsoft Foundry
```

`DefaultAzureCredential` represents a credential chain and can evaluate supported identity sources available in the execution environment.

The portfolio implementation was validated in an authenticated local Azure development environment.

No API key, password, access token, or client secret is embedded in `speech-client.py`.

---

## 7. Foundry Project Client

After configuration and identity initialization, the application creates an `AIProjectClient`:

```python
project_client = AIProjectClient(
    endpoint=foundry_endpoint,
    credential=DefaultAzureCredential(),
)
```

`AIProjectClient` establishes the application-level connection to the configured Microsoft Foundry project.

The client application therefore does not need to embed Foundry credentials directly in source code.

Conceptually:

```text
Environment Configuration
        │
        ├── FOUNDRY_ENDPOINT
        │
        └── Azure Identity
        │
        ▼
AIProjectClient
        │
        ▼
Microsoft Foundry Project
```

---

## 8. OpenAI-Compatible Client

The application obtains an OpenAI-compatible client from the Foundry project client:

```python
openai_client = project_client.get_openai_client()
```

This client is then used to submit requests through the Responses API.

The architecture therefore contains two distinct client abstractions:

```text
AIProjectClient
      │
      │ provides
      ▼
OpenAI-Compatible Client
      │
      │ submits
      ▼
Responses API Request
```

`AIProjectClient` establishes Foundry project connectivity, while the OpenAI-compatible client provides the interface used by the application to send the user's request.

---

## 9. Agent Reference

The Python application does not recreate the Foundry agent for every request.

Instead, it references an existing configured agent:

```python
extra_body={
    "agent_reference": {
        "name": agent_name,
        "type": "agent_reference"
    }
}
```

The complete request is submitted using:

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

This creates the following request path:

```text
User Prompt
     │
     ▼
Responses API Request
     │
     ├── User message
     │
     └── Agent reference
             │
             ▼
     Existing Foundry Agent
```

The agent configuration is therefore maintained separately from the Python client.

---

## 10. MCP Integration Boundary

The Model Context Protocol capability exists beyond the Python client's direct implementation boundary.

The client communicates with the configured Foundry agent:

```text
speech-client.py
       │
       ▼
Foundry Agent
```

The agent has access to its configured MCP capability:

```text
Foundry Agent
       │
       ▼
MCP Integration
       │
       ▼
Azure Speech Capability
```

Combined:

```text
speech-client.py
       │
       ▼
Foundry Agent
       │
       ▼
MCP Integration Boundary
       │
       ▼
Azure Speech Capability
```

This distinction is important.

`speech-client.py` does **not**:

- Implement an MCP server
- Host an MCP server
- Define the Azure Speech MCP tools
- Directly execute Azure Speech operations
- Recreate the configured Foundry agent

Instead, the application communicates with an agent that has been configured with access to the external capability.

---

## 11. Request Execution Flow

A complete request follows this sequence:

```text
1. User launches speech-client.py
                │
                ▼
2. Application loads .env
                │
                ▼
3. FOUNDRY_ENDPOINT and AGENT_NAME are read
                │
                ▼
4. DefaultAzureCredential is initialized
                │
                ▼
5. AIProjectClient is created
                │
                ▼
6. OpenAI-compatible client is obtained
                │
                ▼
7. User enters a natural-language request
                │
                ▼
8. responses.create() submits the request
                │
                ▼
9. agent_reference identifies the Foundry agent
                │
                ▼
10. Foundry agent processes the request
                │
                ▼
11. Configured MCP capability is available to the agent
                │
                ▼
12. Azure Speech capability is exercised when appropriate
                │
                ▼
13. Agent produces a response
                │
                ▼
14. response.output_text is returned
                │
                ▼
15. Python client displays the result
```

This demonstrates an end-to-end agent interaction while maintaining separation between client logic and externally configured capabilities.

---

## 12. Response Handling

The client displays the text returned by the agent:

```python
print(f"{agent_name}: {response.output_text}")
```

The Python application does not perform additional parsing or transformation of `response.output_text`.

The response path can therefore be represented as:

```text
Azure Speech Capability
        │
        ▼
Foundry Agent
        │
        ▼
Response
        │
        ▼
response.output_text
        │
        ▼
Python CLI
        │
        ▼
User
```

---

## 13. Interactive Application Loop

The application runs continuously until the user enters:

```text
quit
```

or submits an empty prompt.

The relevant logic is:

```python
while True:
    prompt = input("User prompt (or 'quit'): ")

    if prompt == "quit" or len(prompt) == 0:
        break
```

This provides a lightweight interactive client for submitting multiple requests to the configured agent during the same application session.

---

## 14. Validated Portfolio Execution

The independent portfolio implementation was validated outside the original Microsoft Learn lab directory.

Validation confirmed:

- Python 3.14.7 executed the application successfully.
- A dedicated `.venv` isolated Project 07 dependencies.
- The required dependency set installed successfully.
- `python -m pip check` reported no broken requirements.
- `DefaultAzureCredential` imported successfully.
- `AIProjectClient` imported successfully.
- OpenAI, HTTPX, and dotenv dependencies imported successfully.
- The local `.env` configuration loaded successfully.
- Azure authentication was available.
- The application reached the interactive prompt.
- The request was routed to the configured Foundry agent.
- The configured Azure Speech MCP capability was exercised successfully.
- The resulting agent response was returned to the Python client.

The validated architecture was therefore:

```text
Local Python Environment
          │
          ▼
Azure Authentication
          │
          ▼
Microsoft Foundry Project
          │
          ▼
Configured Foundry Agent
          │
          ▼
MCP-Enabled Speech Capability
          │
          ▼
Agent Response
          │
          ▼
Python Client
```

---

## 15. Security Boundaries

The architecture deliberately separates configuration, authentication, application logic, and external capability execution.

### Local Configuration

The working `.env` file contains environment-specific configuration and is excluded from Git.

Only `.env.example` is intended for source control.

### Authentication

Authentication is handled through:

```python
DefaultAzureCredential()
```

No authentication secret is hard-coded into `speech-client.py`.

### Sensitive URLs and Tokens

SAS tokens, tokenized storage URLs, temporary access URLs, and other environment-specific service values are not required in the repository.

Any such values produced or used during a speech workflow must remain outside source control.

### Generated Media

The repository excludes common generated audio formats:

```text
*.wav
*.mp3
*.m4a
```

This prevents locally generated speech artifacts from being added unintentionally.

---

## 16. Exception Handling

The application wraps its primary execution logic in:

```python
try:
    ...
except Exception as ex:
    print(ex)
```

This provides basic error visibility during interactive execution.

The current implementation does not include production-grade error classification, retry policies, structured logging, telemetry, or centralized exception handling.

Those capabilities remain outside the scope of this portfolio implementation.

---

## 17. Architectural Boundaries

Project 07 demonstrates:

```text
Python Client Development
        +
Azure Identity Integration
        +
Microsoft Foundry Connectivity
        +
Existing Agent Invocation
        +
MCP-Enabled External Capability Access
```

It does not demonstrate:

```text
Custom MCP Server Development
Production Speech Platform Engineering
Persistent Conversation Storage
Production Observability
Custom Retry Architecture
Production Secret Management
Infrastructure as Code
CI/CD Deployment
```

These distinctions ensure that the documented architecture remains aligned with the functionality actually implemented and validated.

---

## 18. Engineering Progression

Project 06 demonstrated a local tool integration pattern:

```text
Agent
  │
  ▼
Registered @tool
  │
  ▼
Python Function
```

Project 07 advances the architecture to:

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
External Capability
```

The progression represents a shift from an application-defined tool toward an agent accessing an externally configured capability through a standardized integration boundary.

This provides the architectural foundation for more advanced agent systems involving multiple external tools, enterprise retrieval systems, orchestration, evaluation, observability, and security controls.