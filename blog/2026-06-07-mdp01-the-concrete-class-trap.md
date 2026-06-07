---
layout: post
title: "The Concrete Class Trap"
date: 2026-06-07
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [cdi, testing, gdpr, ledger]
---

## The Concrete Class Trap

The previous session filed three follow-up issues from the CaseMemoryStore work. This session was supposed to be straightforward — knock them off, move on. Two of the three were. The third pulled a thread worth documenting.

### The fix that wasn't

`CaseMemoryIntegrationTest` was `@Disabled` because `LedgerSequenceAllocator` — introduced in a recent casehub-ledger SNAPSHOT — calls `MERGE INTO ledger_subject_sequence`, a table created by Flyway. Devtown's `@QuarkusTest` uses `drop-and-create` with Flyway disabled. Hibernate never creates the table because it isn't a JPA `@Entity`. Familiar territory — the garden already had GE-20260607-ad3d62 covering this exact pattern.

The garden entry says: switch to `InMemoryLedgerEntryRepository` via `selected-alternatives`. I traced the call chain and stopped. That fix addresses `LedgerEntryRepository` — the interface. But `CaseLedgerEventCapture` doesn't inject `LedgerEntryRepository`. It injects `CaseLedgerEntryRepository` — a concrete class that *extends* `JpaLedgerEntryRepository`. CDI `@DefaultBean` only yields when another bean of the same type exists. The in-memory alternative implements a different type entirely. It would never be selected for the concrete injection point.

The real fix turned out to be simpler: `casehub.ledger.enabled=false` in test properties. Both event captures already check `ledgerConfig.enabled()` and return early. One line, no new dependencies. The flag is write-side only — it gates `CaseLedgerEventCapture` and `WorkerDecisionEventCapture` but doesn't touch `TrustGateService` or `TrustWeightedAgentStrategy`. The three trust tests pass unchanged.

The architectural issue is real, though. `CaseLedgerEntryRepository` inheriting from a concrete JPA class means every downstream consumer of casehub-engine-ledger will hit this wall. If it injected `LedgerEntryRepository` via composition, the in-memory alternative would substitute naturally. Filed engine#436 for the design fix.

### GDPR logging and the void problem

The erasure endpoint needed audit trail logging — GDPR Art.5(2) requires demonstrating compliance. Before the erasure attempt, after success, and on failure. The interesting constraint: `eraseEntity()` returns `void`. I can log that erasure was requested and that it completed without exception, but I can't log how many records were actually deleted. For a compliance audit trail, "something happened" is weaker than "47 records were deleted."

Filed platform#72 to change the return type to `int`. Breaking change — three implementations need updating — but the migration is mechanical.

The constructor injection refactor on `MemoryAdminResource` was the right call. Field injection made the class untestable without CDI, and the `UnsupportedOperationException` path needed a Mockito mock to exercise — the active `InMemoryMemoryStore` supports erasure and won't throw naturally. With constructor injection, the unit test constructs the resource directly. No CDI, no `@InjectMock` dependency.

### The annotation that does nothing (yet)

`@RolesAllowed("admin")` on `MemoryAdminResource`. Currently inert — no security extension on the classpath. But the platform's RBAC infrastructure is already built: `CurrentPrincipal.roles()` delegates to `groups()`, and `OidcCurrentPrincipal` reads roles from `SecurityIdentity.getRoles()`. The annotation activates automatically when `casehub-platform-oidc` lands.

The role name "admin" is the first named role in any CaseHub harness. Filed parent#189 to document the convention before it becomes folklore.
