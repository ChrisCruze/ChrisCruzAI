# Personalized Event Notification Microservices Blueprint

![Python](https://img.shields.io/badge/Python-3.11+-blue.svg) ![Flask](https://img.shields.io/badge/API-Flask-green.svg) ![MongoDB Atlas](https://img.shields.io/badge/Database-MongoDB%20Atlas-brightgreen.svg) ![License](https://img.shields.io/badge/License-MIT-black.svg)

> Reference implementation for crafting highly personalized event notifications with modular Python microservices, vector search, and prompt-engineered messaging.

## Table of Contents
- [Overview](#overview)
- [Architecture](#architecture)
- [Key Features](#key-features)
- [Technology Stack](#technology-stack)
- [Getting Started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Installation](#installation)
  - [Environment Variables](#environment-variables)
- [Configuration](#configuration)
- [Usage](#usage)
  - [Run the Flask API](#run-the-flask-api)
  - [API Endpoints](#api-endpoints)
  - [Sample Requests](#sample-requests)
- [Vector Search Workflow](#vector-search-workflow)
- [Testing](#testing)
- [Project Structure](#project-structure)
- [Extensibility Roadmap](#extensibility-roadmap)
- [Contributing](#contributing)
- [License](#license)
- [Additional Resources](#additional-resources)

## Overview
This blueprint demonstrates how to orchestrate personalized event recommendations by pairing user profiles with a retrieval-augmented messaging engine. The system ingests user data, selects the most relevant upcoming events via MongoDB Atlas Vector Search, and generates natural language notifications that reflect a custom tone and personality defined in YAML prompt templates.

## Architecture
```
┌────────────────────────────────────────────────────────┐
│                        Client Apps                     │
└──────────────┬─────────────────────────────────────────┘
               │HTTP/JSON
┌──────────────▼───────────────┐
│        Flask API Layer        │
│  (Lightweight testing proxy)  │
└──────────────┬───────────────┘
               │Delegates to services
┌──────────────▼──────────────┐   ┌──────────────────────────────┐
│ User Recommendation Service │   │ Event Recommendation Service │
│ - Profile enrichment        │   │ - Event ranking              │
│ - Preference scoring        │   │ - Context assembly           │
└──────────────┬──────────────┘   └───────────┬──────────────────┘
               │                             │
               │                             │
        ┌──────▼────────┐           ┌────────▼────────────┐
        │ Vector Search │◄─────────►│ MongoDB Atlas Index │
        │ Microservice  │  Embeds   │ (event embeddings)  │
        └──────┬────────┘           └─────────────────────┘
               │
        ┌──────▼───────────────┐
        │ Prompt Orchestrator  │
        │ - YAML config        │
        │ - Tone & style rules │
        │ - Message assembly   │
        └──────────────────────┘
```

## Key Features
- **User → Message Flow**: Submit only a user profile to generate a personalized notification with the most relevant upcoming event selected automatically.
- **User + Event → Message Flow**: Provide a user profile alongside a specific event payload to produce a hyper-tailored message for that event context.
- **MongoDB Atlas Vector Search**: Retrieve and rank events using vector similarity over description, category, and metadata embeddings.
- **Prompt Configuration (YAML)**: Control tone, personality, and styling guidelines centrally to ensure consistent messaging across channels.
- **Flask API Layer**: Lightweight façade that exposes testing endpoints for rapid iteration and Postman-ready interactions.
- **Modular Microservices**: Independent services for user scoring, event ranking, and vector retrieval enable focused iteration and scaling.
- **Extensible Architecture**: Ready for scheduling, multi-channel delivery integrations, and rules engines without refactoring the core domain logic.
- **Testable Setup**: Pytest-friendly project layout with fixture placeholders and factories to make automated validation straightforward.
- **Pythonic Development**: Uses pip-based dependency management and familiar tooling for fast onboarding.

## Technology Stack
| Layer | Tools |
|-------|-------|
| Language | Python 3.11+
| API | Flask, Gunicorn (production ready optional)
| Data | MongoDB Atlas (Vector Search index)
| Embeddings | OpenAI, Cohere, or Azure OpenAI (pluggable)
| Messaging | Prompt-template YAML rendered via Jinja2 or Pydantic models
| Testing | Pytest, Faker, Responses (HTTP mocking)
| Packaging | `pip`, `pip-tools` (optional), `python-dotenv`

## Getting Started
### Prerequisites
- Python 3.11 or newer
- pip or pipx for dependency management
- MongoDB Atlas cluster with a vector-enabled collection (e.g., `events`)
- Embedding provider API key (OpenAI, Cohere, etc.)
- Optional: Docker & Docker Compose for containerized deployment

### Installation
```bash
# 1. Clone the repository
$ git clone https://github.com/your-org/personalized-event-notifications.git
$ cd personalized-event-notifications

# 2. (Optional) Create a virtual environment
$ python3 -m venv .venv && source .venv/bin/activate

# 3. Install dependencies
$ pip install -r requirements.txt

# 4. Install the project as a package for local development
$ pip install -e .
```

### Environment Variables
Create a `.env` file in the project root or export the variables in your shell:

```
FLASK_ENV=development
MONGODB_URI=mongodb+srv://<user>:<password>@cluster.mongodb.net
MONGODB_DB=notifications
MONGODB_COLLECTION=events
EMBEDDING_PROVIDER=openai
EMBEDDING_MODEL=text-embedding-3-small
EMBEDDING_API_KEY=sk-...
DEFAULT_PROMPT_CONFIG=config/prompts/default.yaml
```

## Configuration
Prompt and delivery behavior is managed through YAML files stored in `config/prompts/`. Each file is version-controlled to make experimentation auditable.

```yaml
# config/prompts/default.yaml
metadata:
  persona: "Strategic but friendly concierge"
  tone: "Empathetic, forward-looking, concise"
  style_guide:
    - "Open with one-sentence summary of the event value"
    - "Use second-person voice"
    - "Close with a proactive next step"
templates:
  subject: "{{ user.first_name }}, this event is tailor-made for your goals"
  body: |
    Hey {{ user.first_name }},

    {{ event.highlight }}

    {{ recommendation.reasoning }}

    RSVP by {{ event.registration_deadline }} to secure your spot.

    — {{ persona_signature }}
```

Swap configurations by updating `DEFAULT_PROMPT_CONFIG` or by passing the `X-Prompt-Config` header to the API.

## Usage
### Run the Flask API
```bash
$ flask --app app.api:create_app run --debug
# or production ready
$ gunicorn 'app.api:create_app()'
```

### API Endpoints
| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/v1/messages/from-user` | Generates a notification by matching the best event for the provided user profile. |
| `POST` | `/v1/messages/from-user-event` | Generates a notification using both user and event payloads. |
| `GET` | `/v1/events/recommendations` | Returns the ranked event list for a user, useful for UI previews. |
| `POST` | `/v1/events/ingest` | (Optional) Adds or updates event records and regenerates embeddings. |

### Sample Requests
#### User → Message
```bash
curl -X POST http://localhost:5000/v1/messages/from-user \
  -H "Content-Type: application/json" \
  -d '{
        "user": {
          "id": "user_123",
          "first_name": "Chris",
          "interests": ["ai", "community-building", "product"],
          "goals": "Launch a local founder mastermind",
          "availability": ["2025-03-05T18:00:00Z", "2025-03-06T18:00:00Z"]
        }
      }'
```

#### User + Event → Message
```bash
curl -X POST http://localhost:5000/v1/messages/from-user-event \
  -H "Content-Type: application/json" \
  -d '{
        "user": {"id": "user_123", "first_name": "Chris"},
        "event": {
          "id": "event_987",
          "name": "AI Founders Night",
          "highlight": "An invite-only salon for builders shipping AI-powered products.",
          "registration_deadline": "2025-03-02",
          "location": {
            "city": "Austin",
            "format": "in-person"
          }
        }
      }'
```

## Vector Search Workflow
1. **Ingest** events through `/v1/events/ingest` or direct database writes.
2. **Embed** event descriptions using the configured embedding provider; store vectors alongside metadata.
3. **Query** MongoDB Atlas Vector Search with a blended query built from user interests, goals, and location filters.
4. **Rank** events with a hybrid scoring function that mixes vector similarity, metadata boosts, and business rules.
5. **Assemble** the top match into the prompt template to produce the final notification.

## Testing
```bash
# Run all tests
$ pytest

# Run specific domain tests
$ pytest tests/services/test_event_recommendations.py -vv
```

Use the `tests/fixtures/` directory for reusable user and event payloads. Mock external APIs (embedding provider, email delivery, etc.) using the `responses` or `pytest-httpx` libraries.

## Project Structure
```
personalized-event-notifications/
├── app/
│   ├── api.py                # Flask app factory and routes
│   ├── schemas.py            # Pydantic models for validation
│   └── services/
│       ├── users.py          # User preference enrichment
│       ├── events.py         # Event ranking logic
│       └── vector_search.py  # MongoDB Atlas Vector Search client
├── config/
│   └── prompts/
│       ├── default.yaml
│       └── enterprise.yaml
├── scripts/
│   ├── ingest_events.py      # Batch load events and embeddings
│   └── refresh_embeddings.py # Scheduled embedding refresh
├── tests/
│   ├── conftest.py
│   ├── fixtures/
│   │   ├── users.json
│   │   └── events.json
│   └── services/
│       └── test_vector_search.py
├── requirements.txt
├── README.md
└── pyproject.toml
```

## Extensibility Roadmap
- [ ] **Scheduling Engine**: Cron-based reminders and send windows per channel.
- [ ] **Delivery Integrations**: Plug-ins for email, SMS, push notifications, and in-app messaging.
- [ ] **Rules Engine**: Declarative business rules to override ranking or enforce guardrails.
- [ ] **A/B Experimentation**: Track notification performance by persona, tone, or channel.
- [ ] **Real-Time Feedback Loop**: Capture engagement signals to retrain ranking heuristics.
- [ ] **Admin Dashboard**: Visualize event performance and manage prompt templates.

## Contributing
1. Fork the repository and create your feature branch (`git checkout -b feature/amazing-feature`).
2. Run tests to ensure regressions are avoided.
3. Submit a pull request that describes your changes, testing, and context.
4. Ensure commits follow Conventional Commits (e.g., `feat: add sms delivery driver`).

## License
Distributed under the MIT License. See `LICENSE` for more information.

## Additional Resources
- [MongoDB Atlas Vector Search](https://www.mongodb.com/docs/atlas/atlas-search/vector-search/)
- [Flask Application Factory Pattern](https://flask.palletsprojects.com/en/latest/patterns/appfactories/)
- [Prompt Engineering Guide](https://www.promptingguide.ai/)
- [pytest Documentation](https://docs.pytest.org/en/stable/)
