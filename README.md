# OpenAPI Mock Server

**Status:** Candidate Project  
**Market Size:** $1.18B (2024) → $3.67B (2033) at 13.7% CAGR  
**Last Updated:** 2026-05-02

## Overview

An AI-native OpenAPI mock server that generates production-quality API mocks with semantically realistic data. Today's mock servers produce structurally valid but meaningless random data. This project advances the field by combining:

- **Semantic data generation** using field names and descriptions (not random Faker.js output)
- **Stateful simulation** of realistic multi-step user journeys from natural language scenarios
- **Automatic error response generation** for OWASP API Top 10 attack scenarios (RFC 7807 conformant)
- **AI-driven contract drift detection** comparing real API responses to specs in production

## The Market Gap

The API mocking market is growing at 13.7% CAGR. Existing tools fall into two categories:

1. **Open-source tools** (Prism, WireMock, Mockoon) — Strong but limited to structurally valid random data
2. **Commercial AI tools** (Beeceptor, Apidog) — AI-powered data generation exists but only in cloud SaaS, proprietary, and often vendor-locked

**Beeceptor** demonstrated that AI-aware data seeding is commercially viable and valued by customers. Yet the open-source ecosystem remains completely unserved—no widely adopted open-source tool offers field-aware realistic data generation.

## Core Features

### MVP (Must-Have)
- Parse OpenAPI 3.x (and 2.x) specs; auto-generate mock endpoints with no manual stub authoring
- Structurally valid response body generation honoring types, formats, enums, and required fields
- **AI-assisted semantically realistic data generation** using field names and schema descriptions (the key differentiator)
- Request validation against the spec with informative 4xx error responses
- Docker-compatible deployment with CLI invocation for CI/CD integration
- Request log with matched-rule display for developer debugging

### Should-Have (v1.1)
- Stateful session simulation: maintain in-memory state across multi-request sequences, configured via natural language scenario descriptions
- Proxy mode with live contract-drift detection: compare real API responses to spec and flag divergences
- RFC 7807-conformant error body generation for all 4xx/5xx responses, with OWASP API Top 10 scenario coverage
- Spec-gap detection: identify missing response codes and suggest or auto-generate definitions
- OpenAPI 3.1 full support including JSON Schema 2020-12 keywords

### Nice-to-Have (Backlog)
- AsyncAPI / event-driven mock support (Kafka, WebSocket) for parity with Microcks
- Visual web UI for non-developer stakeholders to browse and test the mock
- Bidirectional spec-mock sync: edits to the running mock update the source OpenAPI file
- Contract drift PR/issue auto-creation: when drift is detected in staging, open a GitHub/GitLab issue or PR
- gRPC/Protobuf mock generation from `.proto` files

## AI-Native Opportunities

1. **Semantically realistic data generation from field names and descriptions**
   - Current tools generate random values: `"username": "xKq7r3"` instead of `"username": "sarah.chen"`
   - An LLM layer reading field context can produce demo-quality data aligned with real-world patterns

2. **Stateful mock simulation from natural language scenarios**
   - Existing mocks are stateless; every request returns the same randomised response
   - An AI-native server could maintain in-memory state and simulate realistic workflows: "user registers → confirms email → places order → ships" without manual stub scripting

3. **Automatic edge-case and error response generation**
   - OpenAPI specs define happy paths well but rarely include comprehensive error definitions
   - An AI layer could infer realistic error payloads, generate RFC 7807 objects, and surface OWASP-aligned error scenarios

4. **Spec-gap detection and auto-completion**
   - Large organisations often have incomplete or outdated OpenAPI specs
   - An AI model could detect missing operations, infer implications from naming conventions, and suggest definitions before mock generation

5. **AI-driven contract drift alerting**
   - As a living mock runs alongside a real API in staging, an AI component could detect divergences, cluster them by type, and proactively open issues or PRs

## Competitive Landscape

| Tool | Type | AI Data Gen? | Statefulness | Cost Entry |
|------|------|-------------|--------------|-----------|
| **Prism (Stoplight)** | OSS CLI | ❌ | ❌ | Free |
| **WireMock** | OSS server | ❌ | ✓ (with extension) | Free |
| **Mockoon** | OSS desktop | ❌ | ❌ | Free |
| **Beeceptor** | Cloud SaaS | ✓ (AI Magic) | ✓ | $199+/mo |
| **This Project** | OSS CLI/server | ✓ (AI-native) | ✓ (stateful) | Free |

## Technical Design Considerations

- **Architecture:** Embeddable as a Node.js package or standalone Docker container; CLI-first for CI/CD
- **Data generation:** Integrate LLM API (Claude, GPT) for field-name-aware generation; cache results to avoid per-request latency
- **Statefulness:** In-memory session store per mock instance; optional persistence to Redis for distributed deployments
- **Specification support:** OpenAPI 2.x, 3.0, 3.1 with full JSON Schema 2020-12 compliance
- **Validation:** Request/response validation via `@apidevtools/swagger-parser` or equivalent; configurable strictness
- **Error handling:** RFC 7807 Problem Details support; OWASP API Top 10 scenario templates

## Market Validation

- **Market size:** API mocking tools market growing at 13.7% CAGR ($1.18B → $3.67B by 2033)
- **Customer personas:** Frontend/full-stack developers, QA engineers, platform teams, API product advocates
- **Buyer pain points:** 
  - Prism users: random data is not convincing for demos
  - Enterprise teams: SaaS pricing ($100K+/yr) is prohibitive for self-hosted needs
  - CI/CD teams: need fast, lightweight mock generation that fits in pipelines

## Why Build This

1. **Market timing:** Beeceptor proved AI data generation is valued; open-source space is completely unserved
2. **Open-source advantage:** Free, self-hosted alternative to $15.7B+ in commercial BI/API tooling
3. **AI-native differentiation:** Statefulness + semantic data generation + contract drift detection are not offered together anywhere
4. **Platform leverage:** Build on proven tools (Prism's OpenAPI engine, Anthropic Claude for data generation)

## Success Metrics

- **Adoption:** 1K+ GitHub stars within 12 months; featured as Prism/WireMock alternative in industry guides
- **Feature parity:** Cover 80%+ of Beeceptor's AI data generation use cases at zero cost
- **Community:** Active issue triage; 2+ contributors beyond core team
- **Deployment:** Used in CI/CD pipelines of 50+ companies (inferred from GitHub insights)

---

**Resources:**
- [OpenAPI 3.1 Specification](https://spec.openapis.org/oas/v3.1.0)
- [Prism Documentation](https://stoplight.io/open-source/prism/)
- [WireMock Documentation](https://wiremock.org/)
- [RFC 7807 Problem Details](https://tools.ietf.org/html/rfc7807)
