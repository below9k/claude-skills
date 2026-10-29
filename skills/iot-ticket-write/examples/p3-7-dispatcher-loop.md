# Example: MFB-1607 · P3.7 Dispatcher loop

A **Story** rewritten to the standard. Facts were checked against `~/Develop/m3s` on
2026-10-29. This shows the target level of detail; it is not yet published to Jira.

## Before (as in Jira)

> **Pattern:** `inmarsat-downloader/src/loop/loop-orchestrator.ts`
>
> **Acceptance criteria**
> - [ ] Connect: create, insert `Pending` service, connect; immediate `OPN` settles to `Succeeded`.
> - [ ] Disconnect: call Pivotel, move service `Active → Disconnecting`; no active service settles as a no-op success.
> - [ ] Backoff 4, 8, 16, 32 min; 5 attempts; then `Failed` with the dead-letter metric.
> - [ ] A failing intent never blocks other modems.
>
> ID: P3.7 · Size: L · Depends on: P1.7, P2.2, P3.6 · Phase 3

**What the critic flagged**
- No Why or Behaviour, so an implementer can't tell what "settles" means without R2.
- *Claim*, *lease* and *idle poll* (R4) aren't mentioned, though the dispatcher owns them.
- Three disconnect branches in M3S aren't covered (scout: `MutationConsumerLoop.cs:241-294`).
- Kill-switch interaction with P3.9 is unstated, and so is the pacer token from P3.6.
- The duplicate-create finding from PR #448 lives only in a comment.
- Size L with four loosely related outcomes: a split candidate.
- Dependencies are text only, with no Jira links.

## After

**Summary:** P3.7 Dispatcher loop

### Why
Slice 3 ("operator actions end to end"): an approved connect or disconnect reaches Pivotel
at the paced rate. This is the only code path in DPS that mutates a Pivotel service.

### Behaviour
The dispatcher claims one ready `Queued` intent at a time, runs it against Pivotel, and
settles it. Settings it owns (parity with M3S):

| Setting | Value | Source |
| --- | --- | --- |
| Claim | Oldest ready `Queued` row per modem, `FOR UPDATE SKIP LOCKED`. An older row still backing off blocks newer rows **for that modem only** | `ServiceActionRequestRepository.cs` |
| Lease | 15 min | `ServiceActionProcessorOptions.cs:60` |
| Idle poll | 5 s when nothing is claimable | `ServiceActionProcessorOptions.cs:53` |
| Retry backoff | `min(240 s × 2^(attempt−1), 1 h)` → 4, 8, 16, 32 min | `RetryBackoff.cs`, `ServiceActionProcessorOptions.cs:74-79` |
| Max attempts | 5, including the first | `ServiceActionProcessorOptions.cs:67` |

Before every create, connect or disconnect it checks the `egress-pivotel-service-mutation`
gate (MFB-1609) and takes a pacer token (MFB-1606).

### Acceptance criteria
- [ ] **Connect** (parity): create service → insert `Pending` service row → connect. An
  immediate `OPN` moves the service to `Active` and the intent to `Succeeded`; anything
  else leaves `Pending` for the observer. — `dispatcher.feature: connect`
- [ ] **Disconnect** (parity), by service state:
  `Active` → call Pivotel, then `Active → Disconnecting`;
  already `Disconnecting` → hand off without calling Pivotel;
  no non-terminal service → settle `Succeeded` as a no-op;
  `Pending` → release for retry;
  lost the `Active → Disconnecting` race → release for retry.
  — `dispatcher.feature: disconnect` (one scenario per branch)
- [ ] **Retry** (parity): any Pivotel error releases the intent with the backoff above. On
  the 5th failure it goes to `Failed` and the `DeadLettered` metric is emitted. — unit test
  with an injected clock
- [ ] **Isolation** (parity): a failing or backing-off intent never delays another modem's
  intents. — integration test with two modems against the Pivotel mock (MFB-1600)
- [ ] **Gate closed** (new): no Pivotel mutation is sent, and intents stay `Queued`.
  — integration test. *Critic question: should a closed gate count as an attempt? The doc
  doesn't say. If it does, a long closure dead-letters every queued intent.*

### Out of scope
- Expired-lease reclaim → MFB-1608
- Pacer implementation → MFB-1606; gate registration → MFB-1609
- Detecting a create that reached Pivotel but lost its response → follow-up (see Notes);
  parity keeps the retry

### Notes
- **Pattern:** `apps/inmarsat-downloader/src/loop/loop-orchestrator.ts` (loop and drain
  shape); `apps/mkv-packet-processor/src/db/webhook-outbox.ts` →
  `claimPendingOutboxBatch` (claim + lease).
- **Source (parity):** `m3s/src/Farmbot.IoT.M3S.ServiceActionProcessor/Consumers/MutationConsumerLoop.cs`
  — create/connect `:353-384`, disconnect branches `:241-294`, dead-letter `:307`.
- **Carried from MFB-1598 (PR #448):** `PivotelApiError` doesn't say whether a mutation
  reached Pivotel. A retried **create** after a lost response makes a second attempt;
  Pivotel rejects the duplicate SIM, the intent dead-letters, and the first service is
  orphaned with no `PIVOTEL_SERVICE_ID`. Decision: keep parity; the runbook (MFB-1626)
  covers recovery; follow-up ticket for persist-before-create.
- **Split?** If this lands L, split as *P3.7a connect + retry* and *P3.7b disconnect
  branches*. Each leaves a verifiable path.

---
ID: P3.7 · Slice: 3 · Size: L · Blocked by: MFB-1597, MFB-1599, MFB-1606 (Jira links) ·
Source: Epic doc §7 P3.7, §8 R2/R4
