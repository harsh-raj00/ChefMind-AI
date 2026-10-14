# 📚 ChefMind AI Documentation

Technical documentation for ChefMind AI. For a project overview and quick start, see the [main README](../README.md).

| Document | What's inside | Read it when you… |
|---|---|---|
| 🏗 [**Architecture**](architecture.md) | System, component, middleware and data-model diagrams; request lifecycles; security model; design decisions | …want to understand how ChefMind AI works |
| 🔌 [**API Reference**](api.md) | Every endpoint, with request and response examples, validation rules, rate limits and error codes | …are building a client or debugging requests |
| ☁️ [**Deployment**](deployment.md) | Hosting setup, environment variables, hardening, a verification script and troubleshooting | …are shipping to production |

## Diagram index

| Diagram | Location |
|---|---|
| System architecture | [architecture.md §2](architecture.md#2-system-architecture) |
| Frontend components | [architecture.md §3](architecture.md#3-frontend-component-architecture) |
| Middleware pipeline | [architecture.md §4](architecture.md#4-backend-middleware-pipeline) |
| User journey | [architecture.md §5](architecture.md#5-end-to-end-user-journey) |
| Recipe generation sequence | [architecture.md §6](architecture.md#6-request-lifecycle-recipe-generation) |
| Image detection sequence | [architecture.md §7](architecture.md#7-image-ingredient-detection-flow) |
| Recipe chat flow | [architecture.md §8](architecture.md#8-recipe-chat-flow) |
| Gemini fallback chain | [architecture.md §9](architecture.md#9-gemini-fallback-chain) |
| Entity-relationship model | [architecture.md §10](architecture.md#10-data-model) |
| Auth sequence | [architecture.md §11](architecture.md#11-authentication-flow) |
| Security layers | [architecture.md §12](architecture.md#12-security-model) |
| API client flow | [api.md](api.md#typical-client-flow) |
| Deployment topology & pipeline | [deployment.md §1](deployment.md#1-deployment-topology) |
| Server startup | [deployment.md §3](deployment.md#startup-sequence) |

> All diagrams are written in [Mermaid](https://mermaid.js.org/), which GitHub draws as pictures automatically. To edit one, change the text inside its code block.
