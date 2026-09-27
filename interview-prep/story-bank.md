# Story Bank

Accumulated STAR+R stories for interview prep. See AGENTS.md → "Source-of-Truth Boundary" for how this file is trusted relative to `cv.md`: quantified claims here should trace to a primary file or carry a `**Provenance:**` marker (see `story-provenance-check.mjs` header for the full convention). Run `node story-provenance-check.mjs --summary` before citing a figure from here as settled.

### [Ownership / Reliability] Emergency Shutdown Feature
**Provenance:** user-stated 2026-09-24
**Situation:** A customer flagged a data-loss risk: when a cluster started losing node capabilities (typically drive failures across nodes, whether from service failures causing node evictions or aging drives dying outright), there was no automated safety measure — only a snapshot-restore feature already in the IP pipeline. On an 8x16 cluster, losing just 4 nodes is enough to threaten cluster integrity. Cluster failure of this kind is rare, which is exactly why an automated stopgap mattered: it has to be there for the moment nobody expects it.
**Task:** Design, build, and own an automated emergency shutdown feature end to end — including getting buy-in from three different teams whose systems it touched.
**Action:** Designed a shutdown workflow and ran design reviews to gather feedback and iterate toward sign-off from all three teams. One team (data mapping) pushed back because the new shutdown design broke the existing drive-recovery path — it no longer waited for in-flight recovery tasks to finish. Addressed that by drafting sequence designs for alternative recovery resolutions until that team was comfortable approving. Also designed the recovery path back from a shutdown (restarting impacted services and re-adding them to the cluster), mapped out the test plan for the feature, and served as on-call support for it in both production and dogfood.
**Result:** Because the cluster lands in a predictable, known state during a safety shutdown, it can still gather logs and signal between nodes — cutting a typical 2-3 hour manual investigation (physically accessing the downed machine, diagnosing it, restarting it) down to 1 hour or less. Since release, the feature has been valuable for debugging because the cluster can still form support bundles even while down, and recovery from a safety shutdown is now tracked and visible. It has become the default state most cluster failures fall into, unless the management service itself is also down.
**Reflection:** Keeping the impacted teams updated and aligned throughout the project turned out to matter as much as finishing the project itself.
**Best for questions about:** ownership, cross-team design reviews / getting buy-in, handling pushback on a design, incident response, reliability/safety-critical systems, on-call ownership

### [Coachability / Design Iteration] NDU Failure-Handling Redesign
**Provenance:** user-stated 2026-09-24
**Situation:** While building NDU (Non-Disruptive Update), needed a way to handle update failures gracefully. The initial design was a cluster rollback to the previous version, which required tracking exactly where in the update NDU had failed.
**Task:** Land a failure-handling approach for NDU that was correct and maintainable, not just the first design that worked.
**Action:** A senior coworker — someone with prior experience on similar systems before joining Samsung — suggested a different approach: let the failed update fail cleanly and supplement it with a new update, reusing the existing update flow instead of building a separate rollback path. Weighed the tradeoff explicitly: this meant discarding the rollback design and its tests and redoing that work, but the resulting failure path was cleaner, centralized the failure case, and had much less implementation friction since it reused the update flow rather than introducing a new one. Decided to go with the coworker's approach based on that tradeoff and his relevant experience.
**Result:** Finished the NDU failure-handling work and ended up with a simpler, centralized failure path (part of the broader NDU effort that achieved zero-downtime cluster updates and 326 days of continuous uptime for a partner customer).
**Reflection:** Get experienced input on a design early, right after drafting it and before committing — before sinking more time into implementation and testing. It catches problems earlier and saves rework.
**Best for questions about:** taking feedback / coachability, weighing design tradeoffs, deciding to discard your own work, learning from more senior engineers, failure handling / resilience design

### [Application Lifecycle / Zero-Downtime] NDU Update Workflow
**Provenance:** source: cv.md
**Situation:** Storage clusters needed software updates without taking the cluster down.
**Task:** Own development, maintenance, and on-call for the Non-Disruptive Update (NDU) functionality.
**Action:** Built an NDU workflow that cycled cluster nodes offline, updated them, and reintegrated them. Hardened it with version-check utilities, script execution, and bundle validation for update fallback, cleanup, and post-upgrade flows.
**Result:** Zero-downtime cluster updates; 326 days of continuous uptime for a partner customer.
**Reflection:** Update paths need fallback and cleanup designed in from the start, not added after the first failed upgrade.
**Best for questions about:** application life cycle / upgrades, zero-downtime deploys, reliability, owning a feature end to end

### [Performance / Fault Tolerance] Startup Bottleneck from Incomplete Drive Recoveries
**Provenance:** source: cv.md
**Situation:** Incomplete drive recoveries created a startup bottleneck and reduced cluster fault tolerance.
**Task:** Remove the bottleneck without losing recovery progress.
**Action:** Detected incomplete drive recoveries and enabled them to resume post startup instead of blocking it.
**Result:** Increased cluster fault tolerance and eliminated the startup bottleneck; validated with 93 days of I/O uptime under resilience testing.
**Reflection:** Look for recovery work sitting on the critical path; moving it off often fixes both speed and resilience.
**Best for questions about:** performance/scale, debugging a bottleneck, fault tolerance, resilience testing
