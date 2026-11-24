# Agent instructions

## Spec-first workflow

This repository is developed spec-first. The owner decides product behavior; agents implement and verify it.

### Artifacts

- **Living specs** (`SPEC.md`, or `docs/specs/` once split) describe the product behavior in force on `main`, or the target behavior on a feature branch. They describe state, not history. No status notes.
- **Technical constraints** are a section of the spec. They list technical decisions the owner has made (language, libraries, external programs). They are binding; everything else technical is the implementing agent's choice.
- **Proposals** (`docs/proposals/`) hold ideas. They are context, never requirements.
- **Approach** (`docs/changes/<change>/APPROACH.md`): the implementing agent's intended way to deliver an approved spec change. It lists the slices in order with the spec sections each covers, technical decisions not already fixed in the spec's technical constraints, risks and assumptions, and open questions for the owner. It does not restate product rules; it refers to the spec.
- **Change records** hold the owner's approvals and remain after integration. Agents never create or edit approval entries.

### Sequence

1. **Spec change** on the feature branch. Agents may draft and revise it on the owner's request.
2. **Spec approval** by the owner.
3. **Approach.** The orchestrating agent writes the approach from the approved spec diff and the actual repository. The owner discusses it and approves it. No implementation starts before approval.
4. **Implementation in slices** (see below), on the same feature branch, following the approved approach.
5. **Independent review** of each slice and of the complete change against the spec.
6. **Owner acceptance** on a usable build.
7. **Integration.** Only the owner merges. The approach and task list are removed from the tree before merging; the approvals remain in the change record.

### Changes to an approved approach

Reordering, splitting or merging slices within the approved scope does not need new approval. Changing a technical decision listed in the approach, adding a dependency or architecture element not covered by it, or changing scope needs the owner's approval before continuing.

### Slices

- The orchestrator cuts the spec diff into **vertical slices**: each delivers observable behavior from the spec and can be verified on its own. Not layers (backend, frontend), but behavior.
- The first slice is a **walking skeleton**: thinnest end-to-end path, project structure, fakes for external programs and services, CI. It is built by one agent before any parallel work.
- Further slices may run in parallel when they touch separate parts of the code. Each agent works in its own worktree or sub-branch. Slices are merged back into the feature branch one at a time, with the full test suite after each merge.
- Each slice: tests from the spec's rules and examples first, shown to fail; then implementation; then review.
- The slice list is working material, kept as a task list on the branch or as issues. The owner does not approve it, and it may change during the work within the approved approach.
- If a slice cannot be closed with a test because the spec leaves the behavior open, stop and ask the owner. Do not decide product behavior.

### Verification

Choose the cheapest evidence that actually proves a rule:

- **Automated tests** for deterministic behavior, including error cases. External programs and services are replaced by fakes. Runs in CI.
- **Evals** for model-dependent behavior (prompts, generated text). Run on prompt or model changes, not in CI.
- **Manual checks** for what only a real environment shows (desktop integration, timing, visual result), as a short checklist derived from the spec.

Not every rule needs its own test; one test may cover several rules.

### Independent review

The reviewer works in a separate session without the implementation's context, preferably with a different model. It checks whether the tests prove the specified behavior, not merely whether they pass. Expectations in tests that differ from the approved spec are a finding.

### Small changes

Defect fixes against an approved spec, and changes that do not alter product behavior (refactoring, dependencies, tooling), skip spec change, spec approval and approach. They still need tests and independent review. If a fix turns out to require a behavior change, stop and treat it as a spec change.

### Boundaries

- Do not add requirements, limits, prohibitions or enforcement mechanisms that no spec or owner decision states. Mark ideas as proposals.
- A mismatch between code and the approved spec is a defect in the code, not a reason to change the spec.

### After acceptance

Findings from owner acceptance are either:

- a **defect**: the software contradicts the approved spec; fixed against the existing spec;
- a **product change**: the owner wants different behavior; made as an approved spec change first, then implemented.

Decisions that should apply beyond one feature go into the spec, not into change records, comments or task lists.
