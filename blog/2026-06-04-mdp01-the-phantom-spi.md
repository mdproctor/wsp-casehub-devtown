---
layout: post
title: "The Phantom SPI"
date: 2026-06-04
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-devtown]
tags: [qhorus, allowedTypes, channel-contracts, layer3]
---

The issue said the channel creation API didn't expose `allowedTypes`. That was wrong.

`ChannelService.create()` has a nine-argument overload that takes `allowedTypes` as the last parameter. `StoredMessageTypePolicy.validate()` parses it and throws `MessageTypeViolationException` at dispatch time. The enforcement infrastructure was already there — devtown just wasn't calling it. The claim about `NormativeChannelLayout` being the right enforcement boundary was worse: I brought Claude in to check the JARs, and that class doesn't exist anywhere. Not in qhorus, not in claudony, not anywhere. A phantom SPI that lived only in the issue description.

Once that was clear, the real question was simpler: what types should each channel allow?

`/work` needed COMMAND, DONE, DECLINE. The interesting decisions were what to add on top. STATUS was worth including — real out-of-process agents doing security or architecture reviews will need to extend their commitment windows, and `allowedTypes` is immutable once set. Adding STATUS now avoids a migration trap at Layer 6 that would require channel deletion. FAILURE by the same logic: the protocol `case-channel-message-signal.md` lists the four commitment-resolving types as RESPONSE, DONE, DECLINE, and FAILURE. It's first-class, not a special case.

HANDOFF was a deliberate exclusion. The devtown model is orchestrator-mediated — specialist-to-specialist delegation via HANDOFF on a shared work channel bypasses the orchestrator. If reassignment is needed, the pattern is DECLINE from the current agent, then COMMAND from the orchestrator. HANDOFF on `/work` would be the wrong architecture.

`/observe` gets EVENT only — audit and telemetry, no speech acts. Forward assumption before Layer 4 is designed; flagged explicitly as such in the spec.

`/oversight` gets COMMAND, DONE, DECLINE. Human oversight is a terminal decision gate. FAILURE on an oversight channel is a bug, not a speech act.

The harder problem was migration. `allowedTypes` is immutable — `ChannelService` has no `setAllowedTypes`. A channel created before the fix with `allowedTypes=null` would be returned unchanged by the `findOrCreate` pattern, and enforcement would be silently absent. The fix is a `requireAllowedTypes` guard: if the found channel's type contract doesn't match, fail fast with an `IllegalStateException` that names the mismatch and references qhorus#246 as the long-term path.

That qhorus#246 didn't exist yet. We filed three: `deniedTypes` has documented "denial wins" semantics in the `Channel` entity Javadoc but `StoredMessageTypePolicy` doesn't check it (qhorus#245); `setAllowedTypes` is missing entirely (qhorus#246); and the string-parsed `allowedTypes` parameter should be `Set<MessageType>` so typos fail at compile time rather than dispatch time (qhorus#247). Three foundational gaps, all found by being the first real consumer of the `allowedTypes` feature.

devtown#20 was quick — BATCH_BISECT, COORDINATED_MERGE, and COORDINATED_ROLLBACK appear nowhere in Java source, only in documentation explaining why they were excluded. Naming stays deferred to Epics 4 and 5.
