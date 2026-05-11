# Docker-pipeline



# Table of Contents
Introduction
Features
Getting Started:
  1.Project Structure
  2.Setup & Installation
  3.Modules Overview
  4.API Endpoints
  5.Usage Examples
  6.Observability
Contribute

# Introduction 
Knowledge Management Agent (GBS+ Chatbot)
 
A FastAPI-based Knowledge Management Agent leveraging Azure AI Agent, Azure OpenAI, Cosmos DB, SharePoint, and ServiceNow for conversational query handling.
Includes query rewriting, intent classification, semantic search, tool integration, and observability via OpenTelemetry.

# Features
  - AI-driven chat: Conversational interface powered by Azure AI Foundry agents.
  - Intent Classification: Automatically detects whether a query should be answered using ServiceNow or SharePoint knowledge.
  - Query Rewriting: Context-aware query rewriting using user profile and chat history.
  - Tool Integration: Supports run_search (SharePoint) and search_mosaic_index (ServiceNow) tools.
  - Feedback Handling: Capture feedback with reasons and optional messages for continuous improvement.
  - Session Management: Maintains chat sessions with dynamic session titles.
  - SSE Streaming: Real-time message streaming for enhanced user experience.
  - Observability: Integrated with OpenTelemetry for tracing, logging, and metrics.
  - FAQ Endpoint: Provides top 5 frequently asked questions from the knowledge base.

# Getting Started 
1. Project Structure
 
eva-everywhere-km-agent/
├── LICENSE
├── README.md  
├── pyproject.toml
├── requirements.txt
├── .gitignore
├── Dockerfile
├── .dockerignore
├── azure-pipeline-docker.yml
├── src/
│   └── eva_km_agent/
│       ├── __init__.py
│       ├── main.py
│       ├── agent_response_handler.py
│       ├── agent_utils.py
│       ├── tools.py
│       ├── tool_handler.py
│       ├── agents/
│       │   ├── __init__.py
│       │   └── intent_classifier.py
│       ├── services/
│       │   ├── __init__.py
│       │   ├── azure_keyvault.py
│       │   ├── cosmos_client.py
│       │   └── query_rewrite.py
│       ├── models/
│       │   ├── __init__.py
│       │   └── message.py
│       └── utils/
│          ├── __init__.py
│          ├── enum_utils.py
│          ├── session_title_utils.py
│          └── opentelemetry_helper.py
├── Variables/
│    ├── __init__.py
│    ├── base.yaml
│    ├── dev.yaml
│    ├── sit.yaml
│    └── prod.yaml
├── tests/
│   ├── unit/
│   │   ├── test_agents/
│   │   ├── test_services/
│   │   ├── test_models/
│   │   └── test_utils/
│   └── integration/
│       ├── test_azure_integration.py
│       └── test_agent_flows.py
└── docs/
    ├── architecture.md
    └── usage.md
 
2. Setup & Installation
 
a.Clone the repository:
git clone "https://ecolab.visualstudio.com/EVA%20Everywhere/_git/EVA-Everywhere-KM-Agent"
 
b.Create a virtual environment and activate it:
python -m venv venv
source venv/bin/activate  # Linux/macOS
venv\Scripts\activate     # Windows
 
c.Install dependencies:
pip install -r requirements.txt
 
3. Modules Overview

I. src/eva_km_agent/
a. main.py
FastAPI entrypoint.
Handles chat requests, feedback, session management, and user profile fetch.
Integrates query rewriting, intent classification, and tool handling.
Streams agent responses via StreamingResponse.
Observability: Tracing/logging with OTEL singleton.

b. agent_utils.py
Initializes and configures the Azure AI Agent.
Sets up tools for handling different intents:
run_search → SharePoint semantic search.
search_mosaic_index → ServiceNow knowledge base search.
Manages agent instance creation and reusability to optimize performance.

c. agent_response_handler.py
Handles streaming agent events and responses.
Processes intermediate tool outputs and final assistant responses.
Performs document validation and fuzzy matching to ensure correct citations.
Updates CosmosDB with final responses and session data.
Manages session title updates dynamically based on conversation flow.
 
d. tool_handler.py
Handles tool calls for agent runs.
Supports:
search_mosaic_index → ServiceNow integration
run_search → SharePoint semantic search
Submits tool outputs back to Azure AI Agent.
Returns structured tool context for downstream processing.
 
e. tools.py
Implements actual tool calls.
run_search:
Azure Cognitive Search semantic queries.
Returns top-k documents with document_name, web_url, chunk_id, content.
search_mosaic_index:
Connects to Mosaic AI index.
Performs hybrid similarity search on ServiceNow knowledge base.
Returns structured search results.
 
II. src/agents/
a. intent_classifier.py
Classifies incoming user queries as either SharePoint or ServiceNow.
Uses Azure OpenAI for intent detection with predefined rules and examples.
Ensures strict and deterministic intent routing to the correct tool.

III. src/services/
a. azure_keyvault.py
Fetches and manages secrets stored in Azure KeyVault.
Provides secure access to keys, endpoints, and credentials required by:
  Azure AI Foundry,
  Azure OpenAI,
  App Insights,
  CosmosDB, etc

b. cosmos_client.py
Initializes and manages CosmosDB connection.
Provides helper functions for CRU (Create, Read and Update) operations on:
  Chat Messages: Store and retrieve assistant and user messages.
  Chat Sessions: Track conversation sessions by user.
  FAQs: Manage frequently asked questions content.

c. query_rewrite.py
Loads user CSV and fetches user profile (get_user_profile).
RewriterWithLogicApp:
Uses Azure OpenAI for query rewriting.
Integrates conversation history and user profile metadata.
Returns clean, enriched queries optimized for semantic search.

IV. src/utils/
a. enum_utils.py
Defines enumerations for system-wide constants such as:
  Feedback reasons for structured reporting.
Provides utility functions to easily access enums across modules.

b. session_title.py
Generates meaningful session titles dynamically.
Titles are based on:
  Conversation content.
  User actions.
  Key context signals from chat flow.

c. opentelemetry_helper.py
Singleton helper for OpenTelemetry observability.
Handles tracing, logging, and metrics throughout the application.
Integrates with Azure Monitor exporters for real-time telemetry.
Ensures proper startup and shutdown of observability components.
 
4. API Endpoints
 
Method     Path                Description
POST      /chat                Main chat endpoint. Handles query rewrite, intent, tool calls, streaming response.
GET   /feedback_reasons        Returns all valid feedback reasons.
POST    /feedback              Submit feedback for assistant responses.
GET   /chat_history            Fetch chat history by thread_id.
GET  /chat/sessions            Fetch all chat sessions for a user_id.
GET  /chat/faq                 Fetch top 5 FAQs from Cosmos DB.
POST /user-profile             Fetch user profile via Graph API using idToken.
 
5. Usage Examples
 
a.Run the app
uvicorn src.eva_km_agent.main:app --reload
 
b.Chat request
{
  "user_id": "user123",
  "user_input": "What is the leave policy?",
  "thread_id": null
}
 
c.Get chat history
curl http://localhost:8000/chat_history?thread_id=<thread_id>
 
d.Submit feedback
{
  "chat_id": "1234",
  "thread_id": "abcd",
  "feedback": "like",
  "feedback_reason": "correct_answer",
  "feedback_message": "This was very helpful!"
}
 
6. Observability
    OpenTelemetry integrated via open_telementary_helper.py.
    Traces, logs, and metrics exported to Azure Monitor.
    Logs include:
        Query rewrite execution time
        Intent classification time
        Tool call execution
        Cosmos DB operations

# Contribute
Contributions are welcome!
To contribute:
    Fork this repository.
    Create a feature branch.
    Commit changes with descriptive messages.
    Open a pull request.
        - Please ensure your code is well-documented and includes necessary tests before submitting a PR.
