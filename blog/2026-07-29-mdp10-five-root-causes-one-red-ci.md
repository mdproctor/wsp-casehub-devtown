---
layout: post
title: "Five Root Causes Hiding Behind One Red CI"
date: 2026-07-29
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [ci, platform-compat, preferences, debugging]
---

## Five Root Causes Hiding Behind One Red CI

CI had been red since the casehub-pages sibling checkout was introduced. The visible symptom was a Yarn auth error. What followed was a dig through five distinct root causes, each hiding behind the one before it.

**The Yarn layer.** The casehub-pages `.yarnrc.yml` expects `GH_PACKAGES_TOKEN` but CI was passing `NODE_AUTH_TOKEN`. Easy fix — wrong env var name. Except Yarn 4 also defaults `enableImmutableInstalls=true` when it detects `CI=true`, regardless of whether `--immutable` is on the command line. Removing the flag does nothing. The only fix is `YARN_ENABLE_IMMUTABLE_INSTALLS=false` in the step's environment. The error message gives zero indication that an env var is involved.

**The monorepo build.** Even with auth and immutability sorted, building casehub-pages from source in devtown CI was fragile — TypeScript cross-package resolution failures, workspace hash mismatches. The published npm packages have `file:` deps baked in, so installing from the registry doesn't work either. I replaced the whole approach with a frontend stub for CI: the Java tests don't need the frontend, and Quinoa skips cleanly if `dist/` exists.

**The platform API migration.** Under the frontend failures, the Java build had its own problems. The platform SNAPSHOT had moved `startCase()` and `signal()` from `CompletionStage` returns to direct returns, added `toSerializedValue()` to the `Preference` interface, and shifted `ExpressionEvaluator` to a new package. Claude handled the bulk migration across 16 test files, and I fixed the remaining edge cases — raw `WorkerResult` casts and the local `JQExpressionEvaluator` implementation class.

**The silent preference failure.** This was the interesting one. Trust gates, routing policies, and SLA thresholds were all silently using defaults — the YAML configuration existed but preferences resolved to null with no warning. The `ConfigFilePreferenceProvider` resolves by walking the `scope` path against YAML-loaded `Path` keys. The YAML scopes use tenancy-prefixed paths like `casehubio/devtown/trust-gate`. But every `SettingsScope.of()` call in devtown put the tenancy in the `tenancyId` field and left it out of the path: `SettingsScope.of("casehubio", Path.parse("devtown/trust-gate"))`. The path segments never match. All 11 callers had the same bug. No error, no warning — preferences just return null and the code falls through to defaults.

**The exception swallower.** The engine's `CatchAllExceptionMapper` implements `ExceptionMapper<RuntimeException>`. `BadRequestException` extends `WebApplicationException` extends `RuntimeException` — so the catch-all intercepts JAX-RS exceptions and returns 500 for everything. A one-line `instanceof` guard fixed it, plus a regression test to keep it fixed.

The `MergeQueueAdmissionTrustScoreTest` was a sixth, smaller issue: the test used a `BatchFormationTestProfile` that set `min-batch-size=1`, immediately moving the PR from QUEUED to IN_BATCH. The assertion checked `store.queued()` and found it empty. Removing the profile — which the test didn't need — let the PR stay queued long enough to be checked.

The preference path bug is the one worth remembering. A `SettingsScope` record with both `tenancyId` and `scope` fields naturally suggests the tenancy participates in resolution. It doesn't. The scope path must contain the full hierarchy including the tenancy prefix, and the `tenancyId` field is metadata only. When the mismatch exists, the failure is completely silent — every preference just returns its default, and the application behaves as if no configuration exists.
