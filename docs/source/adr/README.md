# Architecture Decision Records (ADRs)

This directory contains Architecture Decision Records for the caldav project.

## What are ADRs?

An Architecture Decision Record (ADR) captures an important architectural decision made along with its context and consequences. ADRs help us:

- Document the reasoning behind major technical decisions
- Provide context for future maintainers
- Enable informed discussion before implementation
- Track the evolution of the project's architecture

## ADR Format

Each ADR includes:

- **Status**: Proposed, Accepted, Deprecated, or Superseded
- **Context**: The situation motivating the decision
- **Decision**: The choice we made and why
- **Consequences**: The results of applying the decision (both positive and negative)
- **Alternatives**: Options we considered but didn't choose

## Index

### Proposed

- [ADR 0001: Async-First Architecture with Sync Wrapper](0001-async-first-architecture.md) - Proposal to adopt async-first design for caldav v3.0

### Accepted

_None yet_

### Deprecated

_None yet_

## Contributing

When proposing a new architectural decision:

1. Create a new ADR file: `docs/source/adr/NNNN-short-title.md`
2. Use the next sequential number
3. Start with status "Proposed"
4. Open a GitHub discussion or issue to gather feedback
5. Update status to "Accepted" once consensus is reached

## References

- [ADR GitHub Organization](https://adr.github.io/) - Templates and examples
- [Documenting Architecture Decisions](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions) - Original ADR proposal by Michael Nygard
