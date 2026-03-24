# Project Architecture: Clean Architecture (Strict)

All code generated, refactored, or suggested by Copilot in this repository **must** follow Clean Architecture. No exceptions. If a user request conflicts with these rules, warn and propose a compliant alternative first.

## Project Structure

```
<service>/
  src/
    domain/
      entities/         # Business objects, aggregates, value objects
      services/         # Domain logic that doesn't belong to a single entity
      exceptions/       # Domain-specific errors
      interfaces/       # ABCs for domain-level contracts (repositories, core service boundaries)
    application/
      use_cases/        # One class per use case; orchestrates domain objects
      interfaces/       # ABCs for technical integration ports (email, storage, external APIs)
      dtos/             # Input/output data transfer objects for use cases
    infraestructure/
      adapters/         # Implementations of domain/application ABCs (repos, API clients, controllers)
      api/              # HTTP routes, request/response handling (inbound adapters)
      config/           # Framework bootstrapping, settings, environment
      persistence/      # DB sessions, migrations, ORM models
      llm_providers/    # LLM client implementations (outbound adapters)
  tests/
    unit/
    integration/
```

## Layer Definitions

### Domain (`src/domain/`)
The innermost layer. Contains pure business logic with **zero external dependencies** — no frameworks, no ORMs, no third-party libraries.

- **entities/**: Business objects and aggregates. Pure Python classes. No SQLAlchemy, no Pydantic BaseModel for DB, no framework types.
- **services/**: Cross-entity business logic that doesn't naturally belong to one entity.
- **exceptions/**: Domain-specific error classes. Raise these from domain logic, never generic Python exceptions for business rules.
- **interfaces/**: Abstract base classes (ABCs) defining contracts the domain needs. Example: `UserRepository(ABC)` — the domain says "I need to persist/retrieve users" without knowing how.

### Application (`src/application/`)
Orchestration layer. Coordinates domain objects to fulfill use cases. Depends only on domain.

- **use_cases/**: One class per use case. Receives input via DTOs, calls domain entities/services, uses ports (interfaces) to interact with external concerns. Never contains infrastructure details.
- **interfaces/**: ABCs for technical integrations the use cases need from outside. Example: `EmailSender(ABC)`, `FileStorage(ABC)`, `PaymentGateway(ABC)`. These are application-owned ports, not domain concepts.
- **dtos/**: Data transfer objects defining the shape of input/output for use cases. Use Python dataclasses or Pydantic for validation at this boundary.

### Infraestructure (`src/infraestructure/`)
The outermost layer. Contains all technical implementations, frameworks, and I/O.

- **adapters/**: Concrete implementations of ABCs defined in domain/interfaces/ and application/interfaces/. This is where the translation between external formats and internal objects happens.
- **api/**: HTTP route handlers (FastAPI routers, Flask blueprints, etc.). These are inbound adapters — they receive external requests and delegate to use cases.
- **config/**: App configuration, dependency injection setup, environment variable loading.
- **persistence/**: ORM models, DB session management, migrations. ORM models live here, never in domain.
- **llm_providers/**: LLM client implementations (outbound adapters for AI/ML services).

## Dependency Rules (Non-Negotiable)

```
domain  ←  application  ←  infraestructure
(inner)                     (outer)
```

1. **Domain imports nothing** from application or infraestructure. Zero external dependencies.
2. **Application imports only from domain.** Never from infraestructure.
3. **Infraestructure imports from both** domain and application to implement their interfaces.
4. **Dependencies always point inward.** Outer layers depend on inner layers, never the reverse.
5. **Dependency Inversion Principle**: Inner layers define ABCs (ports). Outer layers provide concrete implementations (adapters).

## Python Conventions

- **Type hints required** on all function signatures and return types.
- **One class per file** for entities, use cases, and interfaces.
- **Naming**: `snake_case` for modules and functions, `PascalCase` for classes.
- **ABCs**: All interface files use `from abc import ABC, abstractmethod`.
- **Tests mirror src structure**: `tests/unit/domain/`, `tests/unit/application/`, `tests/integration/`.
- **Each package has `__init__.py`**.

## Anti-Patterns (Forbidden)

Copilot must **never** generate code that does any of the following:

1. **Business logic in routes/controllers.** Route handlers must only parse input, call a use case, and return output.
2. **Direct DB/ORM calls from domain or application layers.** All persistence goes through repository interfaces.
3. **Framework types leaking into domain.** No FastAPI `Request`, no SQLAlchemy `Session`, no Pydantic `BaseModel` in domain entities.
4. **God classes.** No single class handling multiple unrelated responsibilities.
5. **Importing infraestructure from domain or application.** This violates the dependency rule.
6. **Skipping the use case layer.** Routes must not call domain services directly — always go through a use case.
7. **Concrete dependencies in use cases.** Use cases must depend on ABCs (interfaces), never on concrete implementations.

## Copilot Behavior Rules

1. When generating new features, always produce the full layer stack: domain entity/service → application use case + port interface → infraestructure adapter implementation.
2. When a request would violate architecture rules, **warn first** and propose the compliant alternative.
3. When refactoring, move code toward proper layer boundaries — never away from them.
4. When adding tests, place them in the correct mirror directory under `tests/`.
5. Keep explanations of architectural tradeoffs brief but present when generating boundary-sensitive code.
