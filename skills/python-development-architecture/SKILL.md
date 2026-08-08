---
name: python-development-architecture
description: Architectural patterns, service boundaries, API design, resilience, and observability. Use when designing systems, reviewing integration patterns, or investigating production issues.
---

## Rules
- Follow Ports and Adapters (Hexagonal Architecture). Define abstract interfaces (ports) at the package level in `interfaces.py` or `base.py`. Place concrete implementations (adapters) in dedicated sub-packages named after the infrastructure type, e.g. `database/`, `filesystem/`, `memory/`.
- When a port has multiple adapter implementations, provide a factory module (conventionally `fabric.py` in the port's package) that selects and instantiates the concrete implementation based on a type discriminator in settings. The rest of the codebase imports only the port; it must never import a concrete adapter directly.
- Follow the Monolith-over-Microservices approach. Each top-level package inside the main project folder is a bounded context and a candidate for future extraction into a standalone service.
- A package becomes an independent runnable service when it contains a `service.py` module with an async service class. The class participates in the hierarchical service tree managed by the service orchestration library.
- The root service of the application aggregates child services through a declarative `dependencies` property. The orchestration library handles the full async lifecycle of the tree: concurrent start, run, and graceful shutdown in reverse dependency order.
- Define a single central composition root (e.g. a resolver or registry class). It reads configuration, lazily instantiates adapters and services, caches resolved instances, and injects them into consumers. Domain code must never construct its own dependencies.
- Package business logic into capability modules that expose operations to higher layers. A capability module declares what it needs (required capabilities) and what it can do (provided capabilities) through class-level declarations. The execution context and accessors are injected by the calling layer at runtime based on these declarations.
