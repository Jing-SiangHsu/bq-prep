# viewer-ui — Resume Content

**Role:** Full-Stack / Frontend Engineer
**Stack:** Vue 3, TypeScript, Pug, Pinia-style store, protobuf-generated API client (cross-repo submodule), SVG
**Scope:** 23 commits across 6 GitHub issues over ~6 months
**What it is:** The web frontend for a network-switch monitoring/management product ("Viewer"), white-labeled across multiple vendor brands, talking to a Go gRPC backend over a generated, versioned API client.

> Honest framing: smaller surface area than the other projects, but the bugs here are some of the cleanest examples of root-cause reasoning in reactive frontend state — useful interview material even though the codebase itself is small.

---

## Resume Bullets

- **Diagnosed and fixed a cross-cutting data-integrity bug** where frontend-side IDs had drifted out of sync with backend-assigned IDs, corrupting create/edit/delete operations on user records — traced four separate QA-reported symptoms back to one root cause and fixed it at the source rather than patching each symptom independently.
- **Found and fixed a reactive-state "clobber" bug class**: a Vue watcher meant to apply *default* values on user input was firing on initial load and silently overwriting real configuration loaded from the server; fixed by branching on the previously-persisted value instead of unconditionally overwriting, then found and fixed a second related bug where the same watcher discarded user-selected values it shouldn't have.
- **Implemented a new feature end-to-end while a backend protobuf schema was being split mid-development** (a flat config message restructured into two nested sub-configs for a new SNMP Trap-receiving capability) — updated the store, UI, and i18n in lockstep with the upstream schema change via the generated type client, with zero hand-maintained type definitions.
- **Built a device-action feature (LED control) using only generated protobuf enum types** rather than hardcoded values, so a mid-development backend change to the enum's underlying values required zero frontend code changes — caught and adapted to a live upstream contract change for free.
- **Implemented vendor-aware feature gating** for a white-labeled product (the same UI ships under multiple vendor brands with different backend capabilities) by verifying live API behavior per-vendor and wiring a feature flag to the actual vendor ID rather than leaving a feature permanently disabled for all vendors.
- **Extended ownership beyond UI code into the asset pipeline** — produced the device front-panel SVG artwork itself for new hardware models (matching real product photos) when the team's usual asset owner's output was incomplete, then wired the new assets into the panel-rendering submodule.
- Maintained consistent **root-cause/solution/side-effects documentation** for every fix, and worked a batch-operations UX bug (an action-selection step not enforcing a validation guard that a later step did enforce) by finding the correct earlier point in a multi-step flow to apply the existing check, rather than duplicating it.

## Supporting talking points (for interviews)

**Root-causing across symptoms, not patching each one:** A QA cycle reported what looked like four unrelated bugs in user management (failed adds, failed edits, failed deletes, a stale password field). Rather than write four patches, I traced all of them to one cause — the frontend was tracking users by a stale local index instead of the backend's real ID — and fixed the alignment once, which resolved all four reports simultaneously.

**Adapting to a schema change mid-flight:** While I was building the LED-control feature, the backend engineer changed the underlying values of the enum I depended on partway through the issue thread. Because I'd consumed the enum through the generated type client instead of hardcoding ordinal values, my code needed no changes — a concrete example of why I treat generated/versioned API contracts as non-negotiable even for "just a UI."

---

**Why this maps to AI engineering:** Reactive watcher bugs that silently overwrite valid state on load are the same failure mode as an agent loop overwriting good context/memory with a stale default — the fix (branch on prior persisted state, don't unconditionally reset) is the same pattern used to guard agent state stores. Vendor-aware feature gating is structurally identical to per-tenant or per-customer agent capability configuration in a multi-tenant AI product.
