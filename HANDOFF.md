# HANDOFF — devtown

## Last Session

Landed #257 (activate manifest-driven agent routing) — removed `io.casehub.platform.agent.router.**` CDI exclusions from both main and test `application.properties`, removed 6 stale `Llm*Agent` exclusions for deleted classes, added `casehub-platform-agent-config` and `casehub-platform-agent-simulation-core` as runtime dependencies. `RoutingAgentProvider`, `BackendInstanceCoordinator`, and `AgentConfigBeans` are now active CDI beans. 757 tests pass, full build green. Landed as `f0cd2e3` on main.

Closed epic #223 (Agents post findings on GitHub PRs) — all three children (#218, #217, #209) were already landed. Branch `issue-223-agents-post-findings-github` was an empty coordination branch, stamped and closed.

Filed casehub-mosaic#281 for missing `@casehubio/mosaic-graph-stencil-htn` tarball after blocks→mosaic rename — fixed same day by another session, landed as `987d5d9`.

## Immediate Next Step

#258 (Integration test: simulated review pipeline end-to-end) — M/Med, validates the pipeline activated by #257. Part of epic #247 (Wire real LLM agents). Recommended session order: #258 → #259 → #260.

## Key Context for #258

- `agent-config.yaml` (project root) defines three alias chains (`reasoning-heavy`, `standard-review`, `cheap-fast`), all terminating at `backendKey: simulation`
- `SimulatedAgentBackend` (platform `agent-simulation-core`) implements `AgentBackend`, returns canned responses — but is NOT `@ApplicationScoped`, needs a producer or factory registration
- `BackendInstanceCoordinator` discovers backends via `@Any Instance<AgentBackend>` at startup
- `DescriptorDrivenReviewerAgentProducer` and `DescriptorDrivenCodeAnalysisAgent` inject `AgentProvider` — now satisfied by `RoutingAgentProvider`
- `RouterBeans` produces `RoutingAgentProvider`, `InMemoryBackendInstanceRegistry` (@DefaultBean), `ChatModel`, `EmbeddingModel`
- `AgentConfigBeans` loads `agent-config.yaml` and produces `ManifestResult`

## Epic #247 Status

| # | Title | Scale | Status |
|---|-------|-------|--------|
| #257 | Activate manifest-driven agent routing | M | Done |
| #258 | Integration test: simulated review pipeline e2e | M | Next |
| #259 | Add real LLM backend deps — Claude and Ollama | S | Queued |
| #260 | Enable GitHub write-back observers | M | Queued |

## References

- Platform agent router: `platform/agent-router/src/main/java/io/casehub/platform/agent/router/`
- Simulation backend: `platform/agent-simulation-core/src/main/java/io/casehub/platform/agent/simulation/`
- Agent config beans: `platform/agent-config/src/main/java/io/casehub/platform/agent/config/quarkus/AgentConfigBeans.java`
- Devtown agent consumers: `app/src/main/java/io/casehub/devtown/app/agents/DescriptorDriven*.java`
