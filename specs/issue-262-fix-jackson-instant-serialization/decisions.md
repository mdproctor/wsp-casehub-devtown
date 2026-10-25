## D1: Fix scope — targeted vs systemic

**Choice:** Systemic fix — all 22+ static ObjectMapper instances across runtime-core and common-core
**Alternatives:**
- Targeted fix (WorkerScheduleEventHandler only) — minimal change but leaves 21 other instances vulnerable
- Targeted + follow-up issue — unblocks now, defers systemic fix (risk: issue never picked up)
**Rationale:** The same pattern exists in 22+ classes; any handler that encounters Java 8 date/time types will fail the same way. Fixing once prevents recurrence.
**Trade-offs:** Larger initial change spanning two engine modules instead of one class
**Sources:** engine `runtime-core/src/main/java/**/handler/*.java`, `common-core/src/main/java/**/*.java` — 22+ `new ObjectMapper()` instances found via ide_search_text
**Exploration:** quick
**Status:** captured

## D2: Implementation approach — shared factory utility

**Choice:** Create `EngineMappers.defaultMapper()` in `common-core` that returns a pre-configured ObjectMapper with JavaTimeModule. Replace all `new ObjectMapper()` with the factory call.
**Alternatives:**
- Static initializer blocks per class — zero new classes but 22+ identical boilerplate additions, no prevention of recurrence
- CDI-injected ObjectMapper — full Quarkus integration but disproportionate constructor/API changes across 22+ classes, RuntimeBeans, RuntimeManualConfig, and all tests
**Rationale:** Single configuration point; prevents recurrence (new handlers use the factory); no constructor/API changes; minimal structural impact.
**Trade-offs:** Introduces one new utility class in common-core. Static instances still exist (not CDI-managed), but this is appropriate for a POJO module.
**Sources:** `jackson-datatype-jsr310` already a dependency in `api` module (transitive to runtime-core). No existing shared ObjectMapper utility found.
**Exploration:** quick
**Status:** captured
