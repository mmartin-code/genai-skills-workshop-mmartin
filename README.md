# GenAI Skills Workshop — MMARTIN

## Google Cloud Agent Development Kit (ADK) — Skills Validation Workshop

This repository contains my completed solutions for the **Agentic AI with the Google Agent Development Kit** skills validation workshop. The workshop is a two-day intensive covering agent development, multi-agent orchestration, and production deployment on Google Cloud.

---

## Architecture & Technology Stack

| Component | Technology |
|---|---|
| Agent Framework | Google Agent Development Kit (ADK) |
| Language Model | Gemini 2.0 Flash via Vertex AI |
| Content Safety | Google Cloud Model Armor |
| Search | ADK built-in Google Search |
| Weather Data | National Weather Service (NWS) API |
| Geocoding | OpenStreetMap Nominatim |
| Routing | OSRM (Open Source Routing Machine) |
| Knowledge Base | Wikidata SPARQL API |
| Country Data | REST Countries API |
| Deployment | Vertex AI Agent Engine |
| Notebooks | Colab Enterprise (Vertex AI) |
| Language | Python 3.12 |

---

## Challenge Solutions

### Challenge 1: Real-Time Weather Alerts Agent

Built a weather agent using the Google ADK with custom Python tools that retrieve real-time data from the National Weather Service API.

**Key Features:**
- `get_weather()` tool fetches current conditions via wttr.in
- `get_weather_alerts()` tool retrieves active NWS severe weather alerts by state
- 429 rate-limit retry handling with exponential backoff

---

### Challenge 2: Enhancing Agents with Callbacks

Added lifecycle callbacks for logging, content moderation, and input validation using Google Cloud Model Armor for enterprise-grade prompt sanitization.

**Key Features:**
- `before_model_callback` chains Model Armor sanitization, US location validation, and prompt logging
- `after_model_callback` logs all model responses
- Model Armor integration detecting prompt injection and malicious URIs
- Custom US location validation (NWS API only supports US locations)
- Chained callback pattern matching ADK best practices

**Callback Architecture:**

    User Input
      -> Layer 1: Model Armor (prompt injection, malicious URI)
      -> Layer 2: US Location Check (business logic)
      -> Layer 3: Prompt Logging
      -> Gemini 2.0 Flash
      -> Response Logging
      -> User Output

---

### Challenge 3: Developing Multi-Agent Systems

Built a three-agent system with a root coordinator routing to specialized weather and search sub-agents using the `AgentTool` pattern.

**Key Features:**
- Root agent as coordinator, always faces the user
- Weather agent with NWS tools (reused from Challenge 1)
- Search agent with isolated Google Search (avoids tool-mixing API error)
- `AgentTool` wrapping pattern (not `sub_agents`) ensures root retains control
- Callbacks carried forward from Challenge 2

**Architecture:**

    root_agent (coordinator)
      +-- AgentTool(weather_agent)  -> get_weather, get_weather_alerts
      +-- AgentTool(search_agent)   -> AgentTool(google_search_agent)

---

### Challenge 4: Programming an Agent Workflow

Created **GeoGuesser Bot** — an interactive location guessing game powered by a multi-agent workflow with iterative refinement across four knowledge sources.

**Key Features:**
- Chained `AgentTool` architecture: greeter -> critique -> guess -> search -> google_search
- Four knowledge sources: Google Search, Wikidata, OpenStreetMap Nominatim, REST Countries
- Search -> Guess -> Critique pipeline with refinement loop (root calls critique up to 3 times)
- Clue scorecard system with confidence ratings (HIGH / MEDIUM / LOW)
- Interactive chat mode with session persistence
- Automated test suite with difficulty levels (Easy / Medium / Hard)

**Architecture:**

    greeter (root — game host)
      +-- AgentTool(critique_agent)
            +-- AgentTool(guess_agent)
                  +-- AgentTool(search_agent)
                        +-- AgentTool(google_search_agent)
                        +-- search_wikidata()
                        +-- search_places()
                        +-- lookup_country_info()

---
### Challenge 5: Deploying Agents to Agent Engine

Deployed the GeoGuesser Bot from Challenge 4 to Vertex AI Agent
Engine as a fully managed, scalable endpoint. Extended the
deployment with a lightweight persistent web frontend hosted on
Cloud Run, generated programmatically from within the notebook.

**Key Features:**
- Full agent chain deployed (greeter -> critique -> guess -> search)
- `AdkApp` wrapper for serialization (prevents 409 Conflict error)
- Callback-free deployment copy (avoids pickle serialization issues
  with logger objects)
- IAM configuration: Vertex AI User role on Reasoning Engine
  Service Agent
- Remote testing via `stream_query` on the deployed endpoint
- Flask web frontend template generated inside the notebook
- Frontend containerized and deployed to Cloud Run
- Persistent, publicly accessible GeoGuesser chat interface

**Deployment Architecture:**

    Notebook
      -> AdkApp(root_agent)
      -> agent_engines.create()    [Vertex AI Agent Engine]
      -> Flask app template        [generated in notebook]
      -> Cloud Run deployment      [persistent web frontend]
             |
             v
      Public HTTPS endpoint
      (chat UI -> Agent Engine -> full GeoGuesser agent chain)

**Deployment Steps:**
1. `vertexai.init()` with project, location, staging bucket
2. Recreate agents without callbacks for serialization safety
3. Wrap in `AdkApp(agent=root, enable_tracing=False)`
4. `agent_engines.create(app, requirements=[...])`
5. Generate Flask frontend template in notebook
6. Deploy frontend to Cloud Run
7. Test via both `remote_agent.stream_query()` and live web UI

![Challenge 5 Demo](Challenge%20Notebooks/challenge5_demo_1.gif)
![Challenge 5 Demo](Challenge%20Notebooks/challenge5_demo_2.gif)

---

### Challenge 6: ReadyNow! — Federal Emergency Management Assistant

Built **ReadyNow!** — a production-ready FEMA emergency preparedness chat assistant as the capstone case study. This is a complex multi-agent system integrating real-time weather data, internet search, evacuation routing, response validation, and enterprise content safety.

**Key Features:**
- 5 specialist agents coordinated by root via `AgentTool` pattern
- Real-time weather via NWS API with severity/urgency classification
- Evacuation routing via OSRM + OpenStreetMap Nominatim (free, no API key)
- Emergency news via isolated Google Search agent
- Response validation pipeline: refine_agent -> validate_agent chain ensures accuracy, completeness, and appropriate tone
- Model Armor integration for prompt injection and malicious URI detection
- Emergency topic enforcement rejects off-topic queries
- Full callback chain logging all user-agent interactions
- Deployed to Agent Engine as a scalable production endpoint

**Architecture:**

    User Input
      -> Model Armor (prompt injection, malicious URI)
      -> Emergency Topic Validation (business logic)
      -> Prompt Logging

      root_agent ("ReadyNow!")
        +-- AgentTool(weather_agent)   -> get_weather, get_weather_alerts
        +-- AgentTool(search_agent)    -> AgentTool(google_search_agent)
        +-- AgentTool(routes_agent)    -> get_evacuation_route (OSRM)
        +-- AgentTool(refine_agent)    -> AgentTool(validate_agent)

      -> Response Logging
      -> Agent Engine (Vertex AI)

**Requirements Tracing:**

| ID | Requirement | Implementation |
|---|---|---|
| R1 | Real-time weather + alerts | weather_agent + NWS API |
| R2 | Internet search / news | search_agent + Google Search |
| R3 | Evacuation routes | routes_agent + OSRM/Nominatim |
| R4 | Answer safety questions | search_agent (reused) |
| R5 | Root coordinator | readynow root with 4 AgentTools |
| R6 | Validate + refine workflow | refine_agent -> validate_agent |
| R7 | Log all interactions | Callbacks on all agents |
| R8 | Input validation | Model Armor + topic enforcement |
| R9 | Deploy to Agent Engine | AdkApp + agent_engines.create() |
| R10 | Test code | 7 local + 4 remote tests |
| R11 | Architecture diagram | ASCII diagram in notebook |

![Challenge 6 Demo](Challenge%20Notebooks/challenge6_demo.gif)

---

## Key Lessons Learned

| Challenge | Lesson | Impact |
|---|---|---|
| 1 | Always handle 429 rate limits with retry + backoff | Prevents silent failures during Gemini API throttling |
| 2 | Never break on first is_final_response() — take the last one | Agent needs full event loop to execute tools |
| 3 | Use Model Armor for enterprise sanitization, custom logic for business rules | Separates security concerns from domain logic |
| 4 | Use AgentTool() not sub_agents=[] | Root agent retains control; sub_agents causes handoff issues |
| 5 | Isolate google_search in its own agent | Cannot mix search tools with Python function tools |
| 6 | Chain agents as nested AgentTool calls | Avoids SequentialAgent/LoopAgent multiple sub-agent errors |
| 7 | Remove all logger-referencing callbacks before deployment | Logger objects cannot be pickled for Agent Engine serialization |
| 8 | Include requests in deployment requirements | Custom tools using requests library need it in the container |
| 9 | Model Armor template reuse across challenges | Create once in Challenge 2, reference in all subsequent challenges |

---

## Roadmap

| Item | Description |
|---|---|
| 1 | Complete difficulty assessment logic after successfully guessing a location. |

---

## Environment

| Setting | Value |
|---|---|
| GOOGLE_GENAI_USE_VERTEXAI | True |
| GOOGLE_CLOUD_LOCATION | us-central1 |
| MODEL | gemini-2.0-flash |
| Runtime | Colab Enterprise (Vertex AI Workbench) |
| Python | 3.12 |
| ADK Version | 1.23.0 |

---

## References

- [Google ADK Documentation](https://google.github.io/adk-docs/)
- [ADK Callbacks](https://google.github.io/adk-docs/callbacks/)
- [ADK Multi-Agent Systems](https://google.github.io/adk-docs/agents/multi-agents/)
- [ADK Workflow Agents](https://google.github.io/adk-docs/agents/workflow-agents/)
- [Vertex AI Agent Engine](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview)
- [Model Armor](https://cloud.google.com/security/products/model-armor)
- [Agent2Agent Protocol](https://google.github.io/A2A/)
- [National Weather Service API](https://www.weather.gov/documentation/services-web-api)
- [OSRM Routing](http://project-osrm.org/)
- [Wikidata API](https://www.wikidata.org/wiki/Wikidata:Data_access)
- [REST Countries](https://restcountries.com/)

---

*Completed March 2026 | Google Cloud Skills Validation Workshop*
