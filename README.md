# cloud-itonami-isic-1393: Manufacture of carpets and rugs

Open Business Blueprint for **ISIC Rev.5 1393**: manufacture of carpets and rugs — an autonomous "actor" (LLM advisor behind an independent Governor, langgraph-clj StateGraph, append-only audit ledger) that coordinates back-office carpet/rug-plant **operations**: production-batch data logging (tufting/weaving/backing output, area, and output quality via pile-density and tuft-bind-strength testing), tufting-machine/weaving-loom/backing-line maintenance scheduling, equipment-safety/quality-defect concern flagging, and outbound carpet/rug shipment coordination.

This repository designs a forkable OSS business for carpet/rug-plant
operations: run by a qualified operator so a plant keeps its own operating
records instead of renting a closed SaaS.

## What this actor does

Proposes **plant operations coordination**, not machine operation:
- `:log-production-batch` — tufting/weaving/backing batch, area, and output-quality (pile-density / tuft-bind-strength test) data logging (administrative, not an operational decision)
- `:schedule-maintenance` — tufting-machine, weaving-loom, or backing-line maintenance scheduling proposal
- `:flag-safety-concern` — surface an equipment-safety/quality-defect concern (always escalates)
- `:coordinate-shipment` — outbound carpet/rug shipment coordination proposal

## What this actor does NOT do

**CRITICAL SCOPE BOUNDARY** (tufting machines, weaving looms, backing lines; materials-handling and equipment-safety hazards):

- Does NOT control tufting, weaving, or backing-line equipment directly
- Does NOT make plant-safety, labor-safety, or materials-safety decisions (that's the plant supervisor's exclusive human authority)
- Does NOT directly operate tufting/weaving/backing-line equipment under any proposal (permanently blocked, see Architecture)
- ONLY proposes/coordinates operations back-office; all actuation requires explicit human approval
- Safety-concern flagging ALWAYS escalates — never auto-decided, no confidence threshold or phase below escalation

## Architecture

Classic governed-actor pattern (`carpetops.operation/build`, a langgraph-clj StateGraph):
1. **`carpetops.advisor`** (sealed intelligence node, `CarpetAdvisor`): proposes decisions only, never commits
2. **`carpetops.governor`** (independent, `Carpet & Rug Plant Operations Governor`): validates against domain rules, re-derived from `carpetops.registry`'s pure functions and `carpetops.store`'s SSoT -- never trusts the advisor's own self-report
   - HARD invariants (always `:hold`, no override):
     - Plant/batch record must be independently verified/registered (`:verified?` AND `:registered?`) before any action is taken against it (equipment before maintenance scheduling, batch before shipment coordination)
     - The request's own `:effect` must be `:propose` (never a direct-write bypass)
     - `:op` must be in the closed four-op allowlist
     - The proposal's own `:effect` must be one of the four propose-shaped effects (no direct tufting/weaving/backing-line-equipment control)
     - Directly operating tufting/weaving/backing-line equipment (`:direct-operate? true`) is a PERMANENT, unconditional block
     - A shipment may not push a batch's own recorded shipped area past its own logged production area (independently recomputed)
     - No double-scheduling the same maintenance record
     - No fabricated `:quality-grade` value on a production-batch patch
     - No physically implausible `:pile-density-tufts-per-sqm` value on a production-batch patch
     - No physically implausible `:tuft-bind-strength-lbf` value on a production-batch patch
     - No physically implausible `:defect-rate-percent` value on a production-batch patch
   - ESCALATE (always human sign-off, overridable by a human):
     - `:flag-safety-concern` always escalates, regardless of confidence
     - Low-confidence proposals
3. **`carpetops.phase`** (Phase 0->3 rollout): `:schedule-maintenance`/`:flag-safety-concern`/`:coordinate-shipment` are NEVER in any phase's `:auto` set (permanent, matching the governor's own posture); only `:log-production-batch` may auto-commit at phase 3 when clean
4. **`carpetops.store`** (append-only audit ledger + SSoT): a single `MemStore` backend behind a `Store` protocol (see ns docstring for why a second Datomic-backed backend is out of scope for this build)

## Development

```bash
# Run tests (top-level deps.edn already pins langgraph+langchain local/root)
clojure -M:test

# Run tests via the workspace :dev override alias (equivalent, kept for sibling-repo parity)
clojure -M:dev:test

# Run the demo
clojure -M:dev:run

# Lint
clojure -M:lint
```

## Status

`:implemented` — `governor.cljc`/`store.cljc`/`advisor.cljc`/`registry.cljc` + `deps.edn` complete the module set; tests green, demo runnable, langgraph-clj integration verified.

## License

AGPL-3.0-or-later
