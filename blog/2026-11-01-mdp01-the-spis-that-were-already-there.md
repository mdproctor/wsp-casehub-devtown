---
layout: post
title: "The SPIs That Were Already There"
date: 2026-11-01
entry_type: note
subtype: diary
projects: [casehubio/devtown]
tags: [spi, cdi, triage, playbook, architecture, displacement-pattern]
---

# The SPIs That Were Already There

Three SPIs shipped across the CaseHub foundation over the past few months. TriageHandler for intelligent failure recovery. PlaybookWorkerFunctionProvider for YAML-defined worker behaviour. SubCaseGroup.bindingName for richer sub-case metadata. All three landed in their respective foundation repos — engine, platform, agency — with clean contracts, default implementations, and passing tests.

None of them did anything in devtown.

That gap between "the foundation supports X" and "the application uses X" is the gap this session closed. But the interesting part isn't the wiring itself. It's what the three SPIs reveal about how layered architecture works in practice — and why the default-to-no-op pattern is the only sane way to build a multi-application platform.

## The Displacement Stack

CaseHub's CDI displacement pattern is load-bearing architecture, not a CDI trick. The platform ships a `NoOpTriageHandler` as `@DefaultBean`. It always returns `Abandon("no triage handler configured")`. If you never wire anything else, structural recovery exhausts and the case stalls. That's correct behaviour for an application that hasn't decided what failure triage means in its domain.

The agency module ships `LlmTriageHandler` as `@ApplicationScoped` — a generic implementation that serialises the failure context into an LLM prompt and parses the response into Retry or Abandon. It displaces the NoOp automatically when agency is on the classpath. No configuration. No feature flag. CDI priority does the work.

Devtown needs something different. A PR review that gets DECLINED because the agent lacks security expertise shouldn't go to an LLM for a decision — it should re-route to the next trust-ranked candidate. A reviewer that times out on the first attempt should retry. A reviewer that times out three times running should escalate to a human. These are domain rules, not LLM inference problems.

![CDI displacement chain for TriageHandler](images/spi-displacement-chain.svg)

`ReviewTriageHandler` sits at the top of the stack with `@Priority(1)`. It parses the failure summary string — the same string `DevtownFailureClassifier` already parses for structural recovery — and classifies into three buckets: declined (always retry, the engine re-routes), unrecoverable (abandon immediately), and everything else (retry while budget remains, abandon when exhausted). The engine already excludes the failed worker from future dispatch, so a retry with an empty modifications map is sufficient to trigger re-routing.

The critical insight: the handler doesn't need to know about trust scores, routing policies, or reviewer capabilities. Those live in the selection strategy. The triage handler's job is narrower — decide whether to try again or give up. The domain knowledge is in the classification, not in the resolution.

## Playbooks: Configuration Over Code

The second SPI tells a different story. `PlaybookWorkerFunctionProvider` handles YAML worker definitions with a `steps:` block — open an agent session, query it, close the session. The engine's `RuntimeBeans` collects all `WorkerFunctionProvider` CDI beans into a registry. But `PlaybookWorkerFunctionProvider` ships as a plain class with no CDI annotations. Unlike the flow, react, MCP, and A2A providers — each of which has a dedicated Quarkus extension module with a `*Beans.java` producer — playbooks have no extension module.

So devtown creates the producer:

```java
@Produces
@ApplicationScoped
PlaybookWorkerFunctionProvider playbookWorkerFunctionProvider() {
    return new PlaybookWorkerFunctionProvider();
}
```

Five lines. But those five lines are the difference between "the engine supports playbook workers" and "devtown can define review behaviour in YAML." The style-review playbook is the proof of concept — three steps that open an agent session with a style review system prompt, query it with the PR diff, and close the session. The playbook compiler turns that into a Serverless Workflow that the engine dispatches like any other worker.

The practical value isn't for style review specifically. It's for the next capability someone wants to add. Instead of writing a Java `ReviewerAgent` implementation, registering it in the agent registry, and rebuilding the module — they write a YAML file and drop it into `review/src/main/resources/devtown/playbooks/`. Configuration over code. The agent session steps handle the LLM interaction; the playbook compiler handles the workflow orchestration.

## The Field That Was Already Populated

The third SPI is the simplest and the most instructive. Engine #1260 added `bindingName` to `SubCaseGroup`. The engine's `SubCaseExecutionHandler` already passes the binding name through to the repository when creating sub-case groups. The `InMemorySubCaseGroupRepository` already stores it. The JPA repository already persists it. The field was populated end-to-end in the foundation.

Devtown's merge queue creates bisection sub-cases (`bisect-left`, `bisect-right`) through its YAML case definition. The engine populates their binding names automatically. But devtown's governance query layer — the API that feeds the merge queue workbench — never queried the sub-case groups at all. The `batchStatus()` method returned PR entries and batch metadata. Sub-case structure was invisible.

Adding `BisectionSubCase` to `BatchStatus` and querying `SubCaseGroupRepository.findAllByParentCaseId()` took three files and seventeen lines of change. The data was there. The query path existed. The field was populated. Nobody had connected the last mile.

## What This Pattern Teaches

All three SPIs follow the same arc: foundation ships a capability with a safe default. Agency adds a generic implementation. The domain application wires the domain-specific version. Each layer can function without the layers above it. Each layer above adds value without modifying the layers below.

This is the argument for the entire CaseHub architecture. A healthcare triage handler would classify differently — medication interaction failures are never retry-safe. A legal compliance handler would classify differently — regulatory deadline breaches require immediate human escalation regardless of attempt count. The TriageHandler interface doesn't know about any of these domains. The displacement pattern doesn't care. The same CDI mechanism that makes `ReviewTriageHandler` win in devtown makes `LegalTriageHandler` win in a compliance application, with zero changes to the platform, zero changes to the engine, and zero changes to the agency.

The platform's job is to ship the extension point and the safe default. The application's job is to express its domain knowledge through that extension point. The gap between them — that's just wiring. But wiring that nobody does is the same as a feature that doesn't exist.
