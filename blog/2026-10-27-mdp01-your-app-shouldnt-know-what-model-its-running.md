---
layout: post
title: "Your Application Shouldn't Know What Model It's Running"
date: 2026-10-27
entry_type: note
subtype: diary
projects: [casehubio/devtown]
tags: [agent-architecture, manifest, eidos, routing, abstraction, casehub-platform]
---

# Your Application Shouldn't Know What Model It's Running

The moment your application code imports `AnthropicClient` or `OllamaAgentBackend`, you've made a deployment decision at design time. You've said "this application runs on Claude" the same way you'd say "this application runs on PostgreSQL" by importing the JDBC driver directly into your domain layer. Every developer who's done that knows how it ends: the day you need to swap the database, you discover the driver is threaded through forty classes, and the "quick migration" becomes a quarter.

We hit this exact mistake last week on devtown, our PR review coordination system. The task was straightforward: wire up Claude and Ollama as production backends for the six AI reviewer agents that analyse pull requests. The obvious implementation — add `casehub-platform-agent-claude` and `casehub-platform-agent-ollama` as Maven dependencies, exclude a transitive MCP SDK conflict, add CDI bean exclusions for tests — compiled, passed all 765 tests, and was wrong.

It was wrong because it violated the abstraction that makes the whole agent routing system work. Here's what that abstraction looks like.

---

Three layers, each owning a different question.

**Layer 1 — Eidos descriptors: what capability does the work need?**

Each AI agent in devtown is declared in `descriptors.yaml`. A security reviewer, a code analyser, an architecture reviewer — six agents, each with a system prompt and a `modelRef` alias:

```yaml
- agentId: security-reviewer
  capabilities:
    - name: security-review
      modelRef: reasoning-heavy
      qualityHint: 0.85
```

The descriptor says "this agent needs a model classified as `reasoning-heavy`." It says nothing about Claude, nothing about Ollama, nothing about any vendor. The application owns this layer — it knows what work needs doing and what quality of model that work demands.

**Layer 2 — Manifest: how does an alias resolve?**

The `agent-config.yaml` maps each alias to a resolution strategy:

```yaml
aliases:
  reasoning-heavy:
    chain:
      - tier: FLAGSHIP
        capabilities: [reasoning]
        prefer-vendor: anthropic
      - tier: FLAGSHIP
        capabilities: [reasoning]
      - backend-key: simulation
```

This reads: "For `reasoning-heavy`, find a FLAGSHIP-tier model with reasoning capability, preferring Anthropic. If that's unavailable, try any FLAGSHIP model with reasoning. If nothing's available, fall back to simulation."

The manifest is the operator's tool, not the application's. The operator decides which vendors are preferred, which credential environment variables are set, which providers are declared. A different deployment — a government environment that can't use cloud APIs — changes the manifest, not the application.

**Layer 3 — Platform runtime: which backend answers this request?**

`RoutingAgentProvider` walks the chain at invocation time. For each entry, it queries the model registry (populated from the seed catalog — a classpath resource listing every known model with its tier, capabilities, vendor, and `backendKey`). The first entry that matches an available, registered backend wins.

`BackendInstanceCoordinator` discovers backends at startup by iterating CDI-produced `AgentBackend` beans — whatever jars the operator put on the classpath. If `casehub-platform-agent-claude` is deployed, a Claude backend registers itself. If it isn't, the chain skips it and tries the next entry.

---

The key insight is where the dependency boundary sits. The application depends on three things: `agent-api` (the `AgentProvider` interface), `agent-config` (manifest loading and alias resolution), and `agent-router` (the routing logic). That's it. These are abstractions. They know about aliases, chains, tiers, and capabilities. They know nothing about Anthropic's API or Ollama's HTTP endpoint.

The backend modules — `agent-claude`, `agent-ollama`, `agent-gemini` — are deployment artifacts, not application dependencies. They have embedded Jandex indices so Quarkus auto-discovers their CDI beans when they're on the classpath. The operator includes them in the container image. The application never imports them.

In tests, the separation is even cleaner. `TestManifestResultProducer` is a CDI `@Alternative` that maps every alias to `ModelChain.of("simulation")`. The simulation backend is always available, always deterministic, and completely decoupled from any real inference endpoint. Tests don't need to exclude Claude beans or Ollama beans because those jars aren't on the test classpath. The test doesn't know what production will run, and it doesn't need to.

---

The mistake I mentioned — adding `agent-claude` and `agent-ollama` as runtime Maven dependencies — compiled and passed tests precisely because the test override was strong enough to mask it. The beans were excluded from CDI in tests, the manifest was overridden, everything worked. But it embedded a deployment decision into the application's build. A deployment that wanted only Ollama would still pull in the Anthropic SDK transitively. A deployment that wanted neither would still resolve both jars.

The fix was to remove both dependencies entirely and change the alias definitions from hardcoded `backend-key: claude` entries to tier-based queries. Instead of saying "use Claude, then try Ollama," the manifest now says "find a FLAGSHIP model with reasoning capability, preferring Anthropic." The difference matters: the first version breaks when Claude isn't deployed. The second version gracefully resolves to whatever FLAGSHIP model IS deployed — Claude via direct API, Claude via Vertex AI, or a local model through Ollama — and falls back to simulation if nothing matches.

This is the same principle that makes JDBC work. Your application depends on `java.sql.Connection`, not on `org.postgresql.Driver`. The driver is a deployment choice. The connection interface is a design choice. Mixing the two is how you get applications that can't be moved.

For agent-based systems, the pressure to mix them is stronger. The SDKs are new, the APIs are different, the temptation to call `AnthropicClient.messages().create()` directly is real because the abstraction layers didn't exist six months ago. But the cost of coupling is the same as it always was. Your application shouldn't know what model it's running any more than it should know what database it's writing to.

The manifest is how the operator tells the runtime what's available. The descriptor is how the application tells the runtime what it needs. The runtime resolves between them. Each layer changes independently, and that independence is the whole point.
