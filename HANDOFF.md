# HANDOFF — 2026-08-03

## Last Session

Two XS follow-ups from #152 (per-capability duration breakdown): #175 adds `durationSeconds` to per-capability map in `MemoryContext.toContextMap()`, #176 adds doc comment on `SlaEstimate` explaining sample count divergence. Also fixed pre-existing SNAPSHOT API breakage across 14 files — neocortex `importance` parameter on `Memory`/`MemoryInput`, and engine routing API migration (`TrustWeightedAgentStrategy` → `TrustWeightedImplementationRoutingStrategy`). All 549 tests green.

## Known Issues

- **Quinoa npm install** fails during Quarkus augmentation — pre-existing frontend build issue
- **CaseMemoryObserver** binary incompatibility with neocortex SNAPSHOT — resolves when engine is rebuilt
