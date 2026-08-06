# Eliq Drinks App — Technical Interview Playbook

This is a speaking guide for the approximately 60-minute Eliq technical interview. It is deliberately more detailed than a slide deck, but it is not a script to memorize. The goal is to make the reasoning, orchestration, and verification process visible while keeping every technical claim grounded in the submitted repository.

The technical baseline for this guide is submitted commit `f22f9be`. The live ingredient-filter feature is deliberately absent from that baseline and must be built transparently during the interview.

## The central message

> I built the application as a runtime-configured white-label product with clear boundaries between configuration, external data, application state, and UI. I used specialized AI-agent roles with explicit ownership, behavior-first tests, independent read-only review, and command-backed verification. The important outcome is not only that the application works; it is that every important claim is traceable to source, tests, a real API check, or an explicitly stated limitation.

The three ideas to repeat naturally throughout the interview are:

1. **One build, multiple runtime brands.** Branding and behavior are selected by validated JSON rather than application rebuilds.
2. **Untrusted data stays untrusted until a boundary validates it.** This applies to both brand configuration and CocktailDB responses.
3. **AI output is a proposal until evidence supports it.** An agent report is not proof; diffs, tests, builds, browser journeys, and live-contract checks are.

Ali, as an engineer, is likely to probe concrete boundaries, Angular choices, tests, and request semantics. Hadi, as an engineering manager, is likely to probe prioritization, trade-offs, coordination, and whether the process could work safely in a team. Answer both levels: start with the outcome, explain the smallest relevant mechanism, name the evidence, and state the limitation before they have to discover it.

## Interview timeline

| Time | Section | Intended outcome |
|---|---|---|
| 00:00–10:00 | Walkthrough | Demonstrate the product, architecture, white-label model, orchestration, and evidence. |
| 10:00–45:00 | Live build | Add ingredient filtering through an agent while narrating requirements, tests, review, and verification. |
| 45:00–55:00 | Curveball | Show that a changed requirement causes controlled replanning rather than uncontrolled patching. |
| 55:00–60:00 | Reflection | Explain where plausible AI output was wrong or incomplete and how evidence exposed it. |

If the conversation is more fluid, preserve the sequence of reasoning rather than forcing the exact minute marks.

---

# Pre-interview preparation

## Repository and environment checklist

- [ ] Start from a known clean branch or worktree based on submitted commit `f22f9be`.
- [ ] Run `git status --short` and be able to explain every visible path.
- [ ] Use Node `22.22.3` or another version accepted by `package.json`. The Angular CLI in this workspace rejects Node 20.
- [ ] Run `npm run quality` with the supported Node version.
- [ ] Run `npm run e2e`.
- [ ] Run the external contract separately with `NODE_USE_SYSTEM_CA=1 npm run verify:cocktaildb`.
- [ ] Open both production brand URLs and verify the list and one detail before the call.
- [ ] Keep deterministic terminal output available so an external outage does not erase the evidence.
- [ ] Disable notifications, close unrelated tabs, and use a readable terminal font size.

## Tabs to prepare

1. `https://eliq-drinks-app-marius.vercel.app/?brand=citrus#/drinks`
2. `https://eliq-drinks-app-marius.vercel.app/?brand=midnight#/drinks`
3. `README.md`
4. `docs/architecture.md`
5. `public/config/brand-manifest.json`
6. `public/config/brands/citrus.json`
7. `public/config/brands/midnight.json`
8. `src/app/core/config/validators/runtime-config.validator.ts`
9. `src/app/features/drinks/data-access/cocktail-db-response.ts`
10. `src/app/features/drinks/data-access/cocktail-db.mapper.ts`
11. `docs/ai-orchestration.md`
12. A terminal at the repository root

Do not open every file during the walkthrough. Keep them available for deeper questions.

## Fresh evidence ledger

These results were checked against submitted commit `f22f9be` on 2026-08-06:

| Evidence | Result | What it proves |
|---|---|---|
| Formatting | exit `0` | Configured source and workspace files conform to Prettier. |
| Angular ESLint | exit `0` | Source and templates pass configured lint rules. |
| Vitest | 31/31 files, 276/276 tests | The submitted unit and component behavior suite passes. |
| Production build | exit `0` | Angular production compilation and bundling succeed. |
| Playwright | 4/4 Chromium scenarios | Deterministic mobile/desktop journeys pass with intercepted external I/O. |
| CocktailDB contract | exit `0`; 100 list entries and detail ID `15423` | The two submitted API operations remain compatible at check time. |
| Vercel root | HTTP `200` | The production root document is currently served. |
| Vercel manifest | HTTP `200` with the two-brand manifest | Runtime brand configuration is currently served. |

The richer deployed browser smoke—both brands, list to detail, ingredients, languages, remote images, and no page errors—was recorded on 2026-07-21. Do not change that date unless the full browser smoke is repeated.

## Claims discipline

Use these labels when the distinction matters:

- **Verified:** supported by a command, source inspection, or recorded external check.
- **Observed:** directly seen in a browser or tool output.
- **Inferred:** a reasoned explanation that still needs confirmation.
- **Not verified:** no suitable evidence was collected.
- **Decision:** a deliberate choice and trade-off, not an objective fact.

Do not overstate the following:

- Deterministic tests do not prove permanent CocktailDB availability.
- A source review is not a complete screen-reader audit.
- Capacitor readiness is not a generated or tested native application.
- GitHub Pages is not the active deployment; Vercel is.
- The brands differ in colors, logos, radius, localization, and feature flags, but currently use the same typography and grid/comfortable layout values.
- Prep time, calories, occasion, tint, and fallback-glass values are deterministic presentation data, not CocktailDB facts.
- Feature flags control presentation, not server-side authorization.

---

# Part 1 — Ten-minute walkthrough

## 00:00–01:00 — Problem, priorities, and scope

### Show

- The Citrine Sips list.
- The two main routes: `#/drinks` and `#/drinks/:id`.

### Say

> The user's goal is to discover a drink and get a recipe complete enough to make it without leaving the screen. I prioritized the required list and detail experiences, complete request states, multilingual instructions, runtime white-label configuration, and evidence that the AI-generated implementation is trustworthy.

> Ingredient filtering was a hard scope boundary. The submitted code has no endpoint operation, feature flag, query state, control, fixture, test, or generic abstraction prepared for it. The architecture is extensible, but the feature is not prebuilt.

### Why it matters

This establishes product focus and respects the assignment's most important negative requirement. Extra polish such as Saved and the desktop drawer should not become the main story.

### If they ask about Saved

> Saved is a local projection of the already-loaded list and browser-local IDs. It makes no additional CocktailDB request and does not prepare the live feature.

## 01:00–03:00 — User journey

### Show

1. Browse Citrine Sips at mobile width.
2. Save one drink and briefly open Saved.
3. Return Home and open a recipe.
4. Point to image, name, ingredients/measures, and preparation steps.
5. Select Spanish instructions when available.
6. Open Midnight Mix and show metadata and language-selector differences.
7. If time permits, show the desktop drawer and keyboard close.

### Say

> The UI never consumes CocktailDB DTOs. Cards receive `DrinkSummary`; the recipe view receives `DrinkDetails`. Irregular external fields are validated and mapped before reaching application state.

> Both brands run the same JavaScript build. The outer `?brand=` query selects an allow-listed runtime JSON file, and that selection survives hash navigation and reload.

### Evidence if asked

- Mobile journeys are parameterized for both brands.
- The browser harness asserts exact requests and counts and rejects unexpected traffic.
- Desktop coverage verifies focus entry, wrapping, Escape restoration, and favorite-action isolation.

## 03:00–05:00 — Architecture and data flow

### Show

```text
brand-manifest.json / brand JSON (unknown)
  -> exact runtime validators
  -> ConfigurationService
  -> typed config state
  -> fixed CSS tokens + layout + localization + feature flags

CocktailDB JSON (unknown)
  -> response validators
  -> DTO mapper
  -> DrinkSummary / DrinkDetails
  -> feature stores
  -> routed containers
  -> presentational components
```

### Say

> `core/config` owns selection, validation, bootstrap state, theming, and query persistence. `data-access` alone knows CocktailDB names such as `strIngredient1`. Services own `httpResource` GET boundaries. Stores translate resource mechanics into explicit application states. Routed components coordinate interaction; focused components render inputs and emit intent.

> This prevents malformed external data from silently reaching a template merely because a TypeScript interface claimed it was safe.

### Request-state model

```text
List:   idle -> loading -> success | empty | error
Detail: idle -> loading -> success | not-found | error
```

> Empty list and missing detail have different meanings. A shared union originally made impossible states representable; review led to separate types.

### Resource lifetime

> The alcoholic list service and list store are root-scoped so results survive list/detail navigation. Detail data and state are component-scoped so a routed page and desktop drawer cannot share or cancel each other's selected ID.

### Route and instruction behavior

- Detail IDs accept ASCII digits only; invalid IDs suppress the request and produce not-found state.
- The mapper extracts meaningful translations for the finite supported language union.
- The page intersects available translations with brand-allowed languages, then prefers the configured default, English, or the first available option.

### Why `httpResource`

> These are reactive GET reads driven by signals. `httpResource` exposes status, value, error, reload, and replacement behavior through Angular HTTP. I would not automatically use it for mutations, which need explicit command and rollback semantics.

## 05:00–07:00 — White-label configuration

### Show

1. `brand-manifest.json`
2. One brand JSON
3. `runtime-config.validator.ts`
4. Optionally `theme.service.ts`

### Say

> The manifest defines the default brand and allow-listed configuration paths. A known query override selects its entry. An unknown value becomes a controlled error and never becomes an arbitrary fetch path.

> Each brand has exact schema-version, brand, theme, layout, localization, and feature sections. JSON enters as `unknown`; unexpected keys are rejected, so a typo becomes a visible deployment failure rather than ignored drift.

| Section | Responsibility | Validation examples |
|---|---|---|
| `schemaVersion` | Contract evolution | Exactly version `1`. |
| `brand` | ID, name, local wordmark, alt text | Selected ID must match; asset remains under `branding/`. |
| `theme.light/dark` | Semantic colors | Exact twelve-token sets; six-digit hex values. |
| `theme.typography` | Brand/body/mono stacks | Finite allow-list. |
| `theme.cornerRadius` | Shape language | Bounded `px` or `rem`. |
| `layout` | List layout and density | `grid/list`, `comfortable/compact`. |
| `localization` | Allowed/default instructions | Finite union, unique entries, default must be allowed. |
| `features` | Product presentation switches | Exact known booleans. |

> `ThemeService` maps validated values to fixed CSS custom-property names. JSON cannot invent CSS property names or an arbitrary stylesheet surface.

### Bootstrap behavior

> Configuration loads before the normal routed shell is published. Expected load or validation failures resolve into typed error state rather than rejecting Angular bootstrap, so the root can render and retry a controlled diagnostic.

### Scaling to 30+ brands

> The scaling unit is a validated JSON file and assets, not a code fork. I would validate every delivered config in CI, split a growing validator by versioned section, publish migrations, and keep semantic tokens instead of component-specific CSS. A production selector would be derived from the manifest; the current two-brand switch is a demo affordance.

### Honest limitation

> The brand files prove different identity, palette, wordmark, radius, language allowance, and features. The schema supports different typography, list layout, and density, but the final files use the same font stacks and grid/comfortable values. I would make the examples diverge and add browser assertions if maximizing configuration demonstration were the priority.

## 07:00–09:00 — AI orchestration

### Opening

> I began with a larger specialist roster, then reduced it because more agents do not automatically create more throughput. The final model has four core roles and two optional checks. The central rule is one writing agent at a time, with read-only research or review only where it adds independent evidence.

| Role | Responsibility | Why it exists |
|---|---|---|
| Team lead | Plan, approval, shared files, integration, gates, evidence | Single accountability point. |
| Researcher | Current Angular/API/deployment/dependency facts | Separates primary-source facts from model memory. |
| Angular implementer | One approved scope and behavior-first tests | Keeps behavior and its first tests in one red-green loop. |
| Reviewer | Findings-first stable-diff review; never edits | Challenges plausible code without changing the target. |
| Unit tester | Optional independent coverage mapping | Finds missing scenarios without production ownership. |
| Security checker | Optional reachable-risk review | Focuses on risk-sensitive paths, not general style. |

```text
lead inspection and checkpoint
  -> human approval
  -> implementer: public red -> minimum green -> refactor
  -> stable diff
  -> read-only review and optional focused checks
  -> confirmed findings return to the owner
  -> lead diff review, integration gate, evidence record
```

### Why one writer

> In a small repository, concurrent writers on shared contracts can create incompatible assumptions faster than useful code. Read-only research and review can be parallel because they do not destabilize the diff. Parallel writers become appropriate only with frozen interfaces and disjoint ownership.

### Guardrails

- Exact scope, acceptance criteria, constraints, and verification before writes.
- Inspection of current files before planning.
- External JSON typed as `unknown`.
- Public-behavior tests rather than private implementation tests.
- Reviewer findings require path, impact, violated behavior, and smallest fix.
- Bounded correction loops and human escalation.
- Lead inspection and rerun of commands; agent prose is not proof.
- Only the lead edits the orchestration record.
- Only mistakes that actually occurred are recorded.

### Examples where review mattered

1. Separate list/detail state unions removed impossible states.
2. A concurrency test exposed overlapping configuration initialization.
3. Live API verification was strengthened beyond checking one convenient item.
4. Removing an E2E workaround exposed real outer-query loss with hash routing.
5. Independent accessibility review found drawer and Saved focus-lifecycle defects.

## 09:00–10:00 — Verification and transition

### Show

- `npm run quality`
- Playwright output
- Separate live-contract output
- Vercel deployment

### Say

> Deterministic tests use reviewed fixtures and do not depend on CocktailDB availability. The separate live command checks present contract compatibility. Focused tests prove only their scope; application-level health requires the complete quality gate, while Playwright and deployment smoke add narrower browser and external evidence.

> The live feature will cross the same boundaries: validated config, verified external response, explicit state, feature-gated accessible UI, behavior tests, stable-diff review, and an honest statement of what fit within the timebox.

---

# Responding to the recruiter feedback

The feedback was positive about functionality, edge cases, intentional UI/UX, clean structures, and tool selection. The discussion points were tighter workflow configuration and deeper documentation. Do not become defensive or guess at an unspecified defect; show that you inspected concrete improvements.

## Thirty-second answer

> I agree both areas can improve. The workflow has strong human-readable controls—approval gates, one writer, read-only reviewers, structured handoffs, bounded correction loops, and lead-owned evidence—but some ownership rules are enforced by prompts rather than mechanically. I would tighten actual permissions and automate changed-path checks. The documentation has a very detailed audit trail, but depth is not the same as usability. I would add a one-page executive summary, a requirement-to-evidence index, and a concise configuration schema and migration runbook.

## What is already strong

- `AGENTS.md` centralizes source precedence, scope, Angular rules, ownership, evidence language, and Git constraints.
- The team workflow requires approval before non-trivial writes.
- The normal route is deliberately small: lead → implementer → reviewer → lead.
- Optional roles are used only for actual research, coverage, or security risk.
- Reviewers cannot edit, and the unit tester cannot patch production.
- Correction rounds are bounded rather than becoming an endless agent loop.
- Handoffs use structured evidence formats.
- Only the lead owns root integration and the orchestration record.

## How workflow configuration could be tighter

| Current observation | Risk | Next improvement |
|---|---|---|
| Implementer permissions broadly allow `src/**`, `public/**`, and `e2e/**`. | Exact phase ownership is mainly prompt-enforced. | Generate a task-specific allow-list or fail handoff when changed paths exceed scope. |
| Roles can request a broad `npm run *` family. | A future script could have unrelated side effects. | Allow-list named check/build scripts and keep deployment lead-only. |
| OpenCode and Codex definitions express similar rules separately. | The copies can drift. | Maintain one canonical workflow specification and generate or validate adapters. |
| “One writer” is a convention. | A mistaken dispatch can create an unstable diff. | Add a workflow lock or lead assertion before starting a writer. |
| Evidence schemas are return templates. | A handoff can omit required fields. | Validate handoffs against a machine-readable schema. |
| Scope is checked after writing. | Expansion is detected later than ideal. | Run preflight/postflight changed-path checks against the approved set. |

> These are improvements I would make; I am not claiming they are already implemented. Prompt-level guardrails were effective for this repository, but a production organization should turn high-value conventions into enforceable controls.

## How documentation could go deeper

The orchestration record preserves actual prompts and corrections across more than two thousand lines. Its weakness is navigation and traceability, not lack of raw detail.

| Current observation | Reader need | Next improvement |
|---|---|---|
| Chronological session log | A fast entry point | Add a one-page summary with phases, roles, major corrections, and final evidence. |
| Architecture/decision docs | Direct traceability | Map requirement → decision → implementation path → test/evidence. |
| README config table | Operational schema depth | Add full field reference, examples, failure codes, and asset constraints. |
| `schemaVersion: 1` | Evolution policy | Document compatibility, migration, deprecation, rollout, and rollback. |
| Brand creation steps | Production onboarding | Add ownership, preview, accessibility, validation, approval, deployment, and rollback. |
| Test commands | Evidence boundaries | Explain unit, component, deterministic E2E, live contract, and deployed smoke claims. |
| Known limitations | Prioritization | Add impact, likelihood, mitigation, and ownership. |

If asked why the log is so long:

> The assignment asks for actual prompts, hallucinations, corrections, and evidence from the beginning. I optimized first for auditability and truthfulness. The next improvement is an index over that evidence, not replacing it with a polished story that cannot be audited.

---

# Part 2 — Live build, 35 minutes

## Operating principle

Make the process observable. Narrate facts, observations, inferences, unknowns, and decisions. Do not leave the interviewers watching a silent agent.

Useful narration sequence:

1. “Here is what the requirement says.”
2. “Here is what the repository proves.”
3. “This is ambiguous, so I will clarify it.”
4. “This is the smallest design that fits the existing boundaries.”
5. “This test should fail for this behavioral reason.”
6. “The agent proposes more than needed, so I am narrowing it.”
7. “This is focused evidence; the full gate is pending.”

## 10:00–13:00 — Restate and clarify

> I see three mandatory outcomes: a new white-label feature flag, a UI control shown only when enabled, and a request to the new ingredient endpoint. Before choosing state or UI, I want to resolve product behavior that the assignment intentionally leaves open.

Ask one question at a time:

1. Free text, fixed select, or chips from a provided list?
2. Explicit submit, selection, or debounced input?
3. What should an empty value do?
4. Should clear restore the default alcoholic list?
5. Should selection survive reload in the URL?
6. Is one ingredient sufficient or is multi-select expected?
7. What copy is expected for no results?
8. Should Retry repeat the last ingredient request?
9. Which brand enables and which disables the feature?

Do not spend the interview only asking questions. Resolve material ambiguity, state minimal remaining assumptions, and convert answers into acceptance criteria:

- Flag false: no control and no ingredient request.
- Flag true plus valid submission: exact request and validated results.
- Loading, no-results, malformed response, network error, and retry are explicit.
- Clear follows the agreed reset behavior.
- A replaced request cannot publish stale results.
- Existing list, Saved, detail, brand, keyboard, and responsive behavior remain intact.

## 13:00–16:00 — Inspect before planning

Ask the agent to inspect read-only:

- `features-config.interface.ts`
- `runtime-config.validator.ts` and configuration specs
- both delivered brand JSON files
- `alcoholic-drinks-data.service.ts`
- `drinks-list.store.ts`
- `drinks-list-page/**`
- CocktailDB list validation, mapper, and fixtures
- relevant service/store/page specs
- `e2e/drinks-critical-journeys.spec.ts`
- exact `package.json` commands

Say:

> I am not deciding in advance whether to extend the existing service or introduce a focused read boundary. I want its public contract and lifetime first. I also do not want a generic filtering framework for one accepted operation.

Verify the real endpoint before typing its response. Check exact path/query key, encoding, success shape, no-results representation, malformed payload behavior, and the fields actually provided. Shape similarity is an inference until verified.

## 16:00–19:00 — Approval checkpoint

```text
Objective:
Add the agreed ingredient-filter behavior without changing detail behavior.

Context:
Angular 22, standalone, signals/httpResource, runtime-validated config,
validated CocktailDB boundaries, root list state, component-scoped detail.

Scope:
Exact config, data, list-state, list-UI, fixture, and test paths found by inspection.

Constraints:
- Smallest vertical change; no generic framework or unrelated refactor.
- JSON and API payloads remain unknown until validation.
- Default alcoholic behavior remains intact.
- Flag-off cannot trigger the new operation.
- Reset, no-results, error/retry, and request replacement are explicit.
- Semantic controls, keyboard use, focus, and touch behavior remain intact.

Acceptance criteria:
The agreed Given/When/Then behaviors.

Verification:
Observed red, focused green, diff inspection, read-only review, then the largest
integration gate that fits.
```

### Implementer prompt

```text
Read AGENTS.md and inspect every approved file before editing. Return a concise
plan tied to the agreed acceptance criteria, then implement one behavior at a
time with red-green-refactor.

Extend the existing Angular 22 configuration, validated CocktailDB read boundary,
explicit list state, and list UI only as required for the behavior agreed with
the interviewers. Preserve the default alcoholic list and existing list/detail,
Saved, brand, keyboard, responsive, loading, empty, error, retry, and navigation
behavior.

Treat the new response as unknown and verify its real contract before reusing a
mapper. Do not add a generic filtering framework, dependency, unrelated refactor,
unsafe HTML/style/URL flow, or unrequested URL persistence.

For each slice, write a public-behavior test first and report the exact expected
red reason. Add the minimum implementation and run the focused green test. Do not
test private methods and do not commit. Return changed paths, covered criteria,
commands and exit codes, actual corrections, and remaining risks.
```

This is stronger than “Implement ingredient filtering” because it supplies current architecture, observable scope, exclusions, ownership, TDD sequence, regression expectations, and evidence format.

## 19:00–31:00 — Vertical red-green sequence

### Slice 1 — Runtime feature contract

1. Add a failing validator/delivered-config behavior test.
2. Observe failure because the new exact key is not accepted.
3. Extend the typed contract and exact validator minimally.
4. Add explicit values to both brand JSON files.
5. Run focused config tests green.

> Both compile-time and runtime contracts must change. Updating only TypeScript does not validate deployment JSON; updating only one brand makes delivered configuration inconsistent.

### Slice 2 — Verified data boundary

1. Inspect a real response and no-results behavior.
2. Add a failing test for exact endpoint and encoded query.
3. Reuse the list parser/mapper only if the verified shape satisfies its contract.
4. Keep fixed URL and parameter construction in data access.
5. Map failures to payload-free application errors.
6. Run focused data tests green.

> Reuse is earned by contract compatibility, not visual similarity. Even if the parser is reusable, request construction should remain explicit rather than becoming an all-purpose filtering API.

### Slice 3 — Explicit list state

1. Write public store tests for selection, reset, no-results, retry, and replacement behavior required by the agreed contract.
2. Add the minimum owned signal state.
3. Derive request state with `computed()`, not effect-driven signal copying.
4. Ensure Retry repeats current semantics.
5. Verify the later selection owns visible results.
6. Run focused store tests green.

Diff questions:

- Is there one source of truth for the active ingredient?
- Does empty input mean exactly what was agreed?
- Can a disabled brand reach the operation?
- Does reset restore the original list?
- Can an older request overwrite current results?
- Is navigation retention/reset consistent with the decision?

### Slice 4 — Feature-gated accessible UI

1. Write failing component tests for feature-on and feature-off DOM behavior.
2. Add the smallest semantic control matching the decision.
3. Connect public store intent; keep HTTP out of the component.
4. Add accessible name, keyboard operation, focus, status/error messaging, and touch behavior.
5. Test action, clear, no-results, retry, and relevant focus behavior.
6. Run focused component tests green.

### Slice 5 — Critical browser integration

If time permits, prove enabled/disabled brands, exact request timing, detail navigation from a result, reset semantics, and blocked unexpected traffic. Never weaken the harness merely to make the test pass.

## 31:00–38:00 — Review the stable diff

Run `git status --short`, `git diff --check`, and `git diff`. Inspect:

- approved paths only;
- exact endpoint and query construction;
- response parsed as `unknown`;
- no DTO leakage;
- no request when disabled;
- reset and replacement behavior;
- no `any`, unsafe HTML/style/URL, speculative API, or dependency;
- public-behavior tests;
- no accidental detail/Saved changes.

### Reviewer prompt

```text
Perform a read-only, findings-first review of the stable live-feature diff. Do
not edit. Check exact runtime feature validation, both delivered configs, verified
request shape, unknown-response validation, DTO/domain separation, flag-off
behavior, reset, no-results, error/retry, replaced requests, accessibility,
regressions, test quality, and scope.

Reject speculative abstractions and private-implementation tests. Every blocker
must include severity, path/line, impact, violated behavior, and smallest fix.
Independently run focused deterministic checks when useful; implementer prose is
not proof. Return CLEAN or NEEDS_CORRECTION plus residual unverified risks.
```

Reproduce blocking findings rather than accepting them by title. Return confirmed issues to the owning writer, rerun the proving test, and avoid unrelated cleanup.

## 38:00–45:00 — Largest honest gate

Preferred sequence:

```bash
npm run quality
npm run e2e
git diff --check
git status --short
```

If time is short:

1. Finish one coherent vertical behavior.
2. Run direct focused tests.
3. Run lint/full quality only if it fits.
4. Inspect the entire diff.
5. State exactly what remains.

Full-gate handoff:

> Focused config, data, state, and component tests passed; the full quality and browser gates also passed. The diff contains only approved scope. Future CocktailDB availability remains external.

Focused-only handoff:

> The focused tests passed and I inspected the diff. The full quality/browser gate did not fit, so I am not calling this merge-ready. Those commands are the next action before commit.

---

# Part 3 — Curveball, 10 minutes

## 45:00–47:00 — Stop and restate

Do not let the agent continue a plan whose requirement changed.

> The new request changes acceptance criterion X while behavior Y remains valid. I am pausing implementation so we do not carry an outdated assumption into the next patch. The current unknown is Z.

Confirm whether the new requirement replaces or adds to the previous behavior.

## 47:00–49:00 — Read-only impact analysis

```text
Pause all edits. The interviewer changed the requirement as described below.
Inspect the current stable diff and return only:
- acceptance criteria that change;
- behavior that remains unchanged;
- affected public tests;
- exact owned files likely affected;
- regression risks;
- the smallest verification sequence.

Do not edit or generalize the architecture.

Changed requirement: [repeat the interviewer's words accurately]
```

## 49:00–52:00 — Change the behavior test first

- Update or add the public test expressing the new behavior.
- Run it and confirm it fails for the curveball reason.
- If failure comes from setup, stale mocks, or a brittle selector, correct the harness before calling it valid red evidence.
- Preserve tests for unaffected behavior.

## 52:00–54:00 — Smallest correction

- Restrict edits to impacted paths.
- Decline a generic refactor unless genuinely necessary.
- Inspect the diff before rerunning tests.
- Run the updated test green plus the smallest relevant regression set.

## 54:00–55:00 — Communicate proof and risk

> The changed behavior is proved by test X. Existing behavior Y remains covered by A and B. I ran command Z. I did not have time for W, so I would not merge yet.

## Common curveball patterns

| Curveball | First question | Boundary to inspect |
|---|---|---|
| Update while typing | Is debounce duration product-defined? | UI event stream and request replacement. |
| Survive reload | Must state be shareable in the URL? | Router parsing and validated initialization. |
| Multiple ingredients | AND or OR, and does the API support it? | Product/API contract before UI generalization. |
| Disabled brand preserves hidden state | Should hidden state ever issue requests? | Flag semantics and data boundary. |
| Offline behavior | What are freshness and invalidation rules? | Persistence strategy; probably beyond a tiny patch. |
| Client-side sorting | Which field and locale rules? | Domain/view derivation rather than request construction. |

The goal is not a prewritten answer. It is to locate the changed contract and protect everything unaffected.

---

# Part 4 — Reflection, 5 minutes

## Primary story — Configuration initialization race

### Ninety-second answer

> One useful failure was not an obvious hallucination. It was plausible code that compiled and passed initial focused tests. Overlapping `ConfigurationService.initialize()` calls could start duplicate work and allow stale state or theme publication.

> The reviewer reported the risk, but I did not accept it merely because another model said so. I requested a public concurrency test. It invoked initialization twice and failed because two manifest requests existed. The implementer coalesced callers around one in-flight promise, cleared it after settlement, and preserved retry after failure. The same regression passed afterward, followed by the configuration suite and full gate.

> Compilation and happy-path coverage were weaker than the actual lifecycle contract. Independent review supplied the hypothesis; a reproducible behavior test supplied evidence.

### Deeper structure

- **Why plausible:** each single initialization worked.
- **Missing contract:** concurrent-call semantics and stale publication.
- **Detection:** reviewer examined lifecycle, not only types.
- **Proof before fix:** concurrent calls produced duplicate requests.
- **Correction:** one shared in-flight promise, cleared after settlement.
- **Proof after fix:** same regression, focused suite, integration gate.
- **Lesson:** convert plausible findings into executable evidence.

## Backup story — E2E masked a routing defect

> The first browser test manually restored `?brand=` before detail reload. It made the test pass but repaired the product inside the harness. Removing it showed that `queryParamsHandling: 'preserve'` does not preserve an outer query before the hash under `HashLocationStrategy`. Navigation lost the brand, so reload could select the default.

> I rejected the workaround and fixed product behavior through a focused brand-query persistence service. Both brands then retained selection through detail navigation, reload, and Back. A green E2E can be misleading when the test compensates for the application.

Why this story is strong:

- Browser evidence found behavior unit tests missed.
- The initial correction was at the wrong layer.
- The test became stricter rather than weaker.
- The fix respected the chosen routing architecture.

## Backup story — Optional route input hid Back

> Component-input binding could write `undefined` over the default of an optional input. The build was green, but browser behavior showed that Back disappeared on routed detail. I made the observation a public component test and changed the condition so only explicit `drawer` hides routed Back navigation.

Use this when asked why strict typing and builds do not replace browser behavior checks.

## Reflection close

> My strategy is not to expect AI to stop producing plausible mistakes. I reduce freedom through context, contracts, ownership, and small scopes; then challenge output with runtime validation, public behavior tests, independent review, diff inspection, and command-backed evidence. AI accelerates implementation, but responsibility for scope and correctness stays with me.

---

# Technical deep dives and likely questions

## Product and prioritization

### Why these features first?

> The assignment weights orchestration and trustworthiness above raw feature count. I prioritized required journeys, white-label runtime behavior, explicit edge states, deterministic verification, and evidence. Optional polish followed only after those boundaries worked.

### Why is Saved acceptable?

> It projects IDs over the already-loaded list and adds no external query operation or prepared path for the live feature.

### What would you cut under a shorter deadline?

> Saved, some desktop polish, and Capacitor readiness before runtime validation, request states, core tests, deployment, or evidence.

## Angular architecture

### Why Angular 22 rather than only the Angular 17 minimum?

> The assignment asks for the latest version while setting 17 as a floor. The Angular 22 generated workspace and installed APIs became the authority rather than model memory.

### Why standalone and lazy routes?

> They match current Angular architecture, reduce module ceremony, make feature entry points explicit, and split list, detail, and not-found routes.

### Is it zoneless?

> Angular 22 is zoneless by default, and the project does not reintroduce `zone.js`. Template state comes from signals, resources, signal inputs/outputs, and framework events.

### Why signals instead of a state library?

> The scope has a few read resources, explicit state adapters, and local derivations. Signals cover it without global action/reducer overhead. I would reconsider for complex cross-feature commands, audit/history, or broad server-state invalidation.

### Why avoid effects for derivation?

> Copying signals through effects creates ordering and stale-state risks. Derivation belongs in `computed`; effects are for real side effects such as theme application, persistence, or focus lifecycle.

### Why `linkedSignal` for language selection?

> It is user-writable but must reset when recipe or preferred available languages change. `linkedSignal` models writable state tied to a changing source.

### Why no SSR or hydration?

> The approved deployment is a client-rendered hash-routed SPA. SSR guidance in the rulebook is conditional; enabling it would widen scope without a demonstrated assignment benefit.

## Data access and state

### Why is TypeScript not enough for JSON?

> Types disappear at runtime. Declaring `get<AppConfig>()` can make an invalid payload look safe. Fetching as `unknown` and validating the object protects the actual boundary.

### Why exact rather than permissive validation?

> Exact keys expose typos and drift instead of silently ignoring them. The trade-off is that schema evolution must be versioned and coordinated.

### Why not a validation dependency?

> The schema is finite and manageable without one. If it grows, a shared schema source that generates runtime and TypeScript contracts may be more maintainable and reduce hand-written drift.

### How are ingredients mapped?

> A mapper iterates all fifteen `strIngredientN`/`strMeasureN` slots, trims text, skips empty ingredient names, preserves missing measures, and emits application-owned objects.

### What does `drinks: null` mean?

> Empty list for list responses; not-found for detail. Malformed envelopes raise a payload-free boundary error and become fixed user-facing failure messages.

### How are languages selected?

> The mapper extracts meaningful supported translations. The page intersects those with brand allowances and prefers configured default, then English, then first available. Disabling the selector never removes instructions.

### How are stale detail requests handled?

> The validated ID drives a component-scoped resource. Changing it replaces the request; destroying the page/drawer destroys the resource. Tests cover changing IDs.

## White-label architecture

### Why runtime JSON rather than environment files?

> Environment files are build-time. Runtime JSON allows one tested build to serve many brand deployments within a validated contract.

### Can a query fetch arbitrary config?

> No. It selects only a manifest ID; unknown explicit values become controlled errors and never become paths.

### How would schema version 2 work?

> Define compatibility, migrate every delivered brand in CI, report unsupported versions, document rollout/rollback, and never silently coerce an unknown future schema.

### Are client feature flags security controls?

> No. Protected capability still needs server-side authorization.

### Biggest delivered white-label gap?

> The final examples do not demonstrate different typography or list-layout values. The schema and UI support them, but examples do not visibly prove every option. I would add divergent values and browser assertions.

## Testing and evidence

### Why not real API calls in E2E?

> Product tests should fail from our regression, not a free external API outage. Fixtures and interception are deterministic; the live command separately checks current compatibility.

### What do four browser scenarios cover?

> Both brands cover list, Saved, detail, ingredients, language behavior, reload, Back, query persistence, and exact requests. Desktop covers brand switching and keyboard drawer focus lifecycle.

### Why only four?

> They are high-value critical journeys, not a duplicate of 276 focused tests. Focused specs cover combinations cheaply; browser tests cover router, DOM, focus, and interception integration.

### What makes a regression test meaningful?

> It fails for the expected behavioral reason before correction and passes after. A setup failure is not product evidence.

### Why is `npm run quality` not sufficient alone?

> It covers formatting, lint, Vitest, and production compilation—not browser focus, deployment, remote images, or live API compatibility.

## Accessibility and security

### Which accessibility behaviors were intentional?

> Semantic controls, skip link, visible focus, touch targets, image alternatives, `aria-pressed`, loading/error labels, reduced motion, modal focus containment, Escape close, and focus restoration after drawer or Saved changes.

### What remains?

> A real screen-reader session and broader manual WCAG audit. Automated and keyboard evidence is not a full accessibility certification.

### How is XSS constrained?

> API text uses Angular text/property binding. There is no unsafe HTML or sanitizer bypass. Asset and theme inputs are validated, and CSS property names are fixed in code.

### Main remaining security hardening?

> Drink images may use arbitrary HTTP/HTTPS hosts. Production should allow-list expected hosts and deploy a suitable CSP. This is documented, not presented as solved.

### Can brand JSON contain secrets?

> No. It is public browser-delivered product configuration and must never contain credentials.

## Deployment and Capacitor

### Why hash routing?

> Static hosts do not rewrite every route automatically. Hash routing keeps client navigation after the fragment so detail reload requests the root. The brand remains in the outer query and therefore needs explicit persistence.

### Why Vercel rather than GitHub Pages?

> The Pages workflow passed local checks, but the private fork/account plan could not enable Pages. I documented the limitation and deployed through Vercel instead of claiming a deployment that did not exist.

### What does Capacitor-ready mean?

> Core and CLI point to Angular browser output with separate mobile scripts. No iOS/Android platform was generated, signed, or tested, so this is packaging readiness rather than a native app claim.

## Orchestration and leadership

### Why multiple agents if one writes?

> Specialization and independent challenge matter as much as parallel typing. Research and review can run against stable inputs. Serial writers protected shared contracts in this repository.

### How did you prevent context loss?

> Handoffs contained source precedence, starting state, exact scope, acceptance criteria, constraints, commands, and structured evidence. Shared contracts were frozen before dependent work, and the lead integrated shared files.

### What if reviewer and implementer disagree?

> Neither wins by title. I inspect source and turn a finding into a reproducible test or source-backed argument. Confirmed blockers return to the owner; speculative or out-of-scope findings are rejected with reasons.

### Why bound correction rounds?

> Repeated loops can hide design disagreement and expand scope. A cap forces human escalation rather than endless autonomous redesign.

### What would you change in the agent setup?

> Mechanically enforce task-specific file/command allow-lists, validate handoff schemas, lock against concurrent writers, and generate tool adapters from one canonical definition.

### Where did Superpowers help?

> It made planning, red-green-refactor, debugging, review, and verification explicit workflows. Repository instructions and real command output still remained higher authority than a generic template.

### Did the process fit four to six hours?

> The original plan was ambitious, and review correctly identified hidden coordination/deployment time. I prioritized mandatory boundaries and moved Capacitor after delivery. In hindsight I would stop optional polish earlier and add the documentation entry point sooner.

---

# Requirement-to-evidence map

| Assignment expectation | Implementation evidence | Verification evidence | Honest boundary |
|---|---|---|---|
| Alcoholic drinks list | Root list data service, store, page, cards | Focused tests, both-brand mobile E2E, live list contract | Availability remains external. |
| Recipe details | Component-scoped detail service/store/page/view | Detail tests, E2E recipe, live fixed-ID lookup | Remote images remain external. |
| Ingredients and measures | Fifteen-slot mapper | Mapper edge specs and browser assertion | Missing measures use presentation fallback. |
| Instruction languages | Finite mapping and config fallback | Mapper/detail tests; Citrine Spanish browser path | Only meaningful API translations are offered. |
| Runtime white-label config | Manifest, exact JSON, config/theme services | Validator/theme tests, both-brand E2E, deployed manifest | Final fonts/layout values do not differ. |
| Feature flags | Typed exact feature section | Config/component tests and browser difference | Not authorization. |
| Different branding | IDs, wordmarks, palettes, radii | Delivered-config tests and browser journeys | Typography is shared. |
| Separation of concerns | Config, data access, services, stores, containers, views | Contract tests, source review, strict build | List page still coordinates several UI concerns. |
| Edge states | Loading, empty, error, retry, not-found unions | Store/component tests and invalid fixtures | Deployment cannot reproduce every outage. |
| Current Angular | Angular 22, standalone, lazy, strict, signals, zoneless, Vitest | Lint, 276 tests, build | SSR/hydration intentionally absent. |
| Mobile UX | Responsive navigation, cards, detail, touch/focus foundations | 390×844 browser journeys | Native devices not tested. |
| AI orchestration | Constitution, role/workflow files, session ledger | Prompts, corrections, command exit codes | Some rules are prompt-enforced. |
| Deployment | Vercel and prepared Pages workflow | Build, HTTP checks, recorded browser smoke | Pages unavailable under current plan/repo setup. |
| Live feature reserved | No submitted operation, flag, state, control, fixture, test, or dormant abstraction | Source inspection and review gates | Must be built transparently live. |

---

# Honest limitations and improvement order

## Verified limitations

1. Delivered brand examples do not demonstrate different typography or list-layout/density values.
2. No complete manual screen-reader audit was performed.
3. No native iOS/Android project was generated or tested.
4. External drink-image hosts lack an allow-list/final CSP hardening.
5. Prep time, calories, occasion, tint, and fallback glass are demo presentation data.
6. Favorites and theme mode are browser-local, not synchronized.
7. Runtime drink availability depends on CocktailDB.
8. GitHub Pages could not be enabled for the private fork/account-plan combination.
9. Agent permission envelopes and tool-specific workflow duplication could be stricter.
10. The chronological AI record needs a shorter navigation layer.

## Production-value improvement order

1. Image-host policy and CSP.
2. Real screen-reader testing and fixes.
3. Executive orchestration summary and traceability index.
4. Full configuration schema/versioning/onboarding runbook.
5. Mechanical agent permission and scope enforcement.
6. Stronger delivered brand variation with browser tests.
7. Visual regression coverage.
8. Native projects only when mobile delivery is prioritized.

---

# Live-session anti-patterns

- Giving the agent only “Implement ingredient filtering.”
- Coding before clarifying control and reset behavior.
- Assuming the new response matches an old one.
- Adding a generic framework for one operation.
- Allowing two writers on shared contracts.
- Reviewing while the diff changes.
- Calling a setup failure a valid TDD red.
- Testing private methods instead of public behavior.
- Weakening E2E interception to make it pass.
- Letting a disabled brand issue hidden requests.
- Ignoring request replacement.
- Running only build and saying everything passes.
- Presenting focused evidence as full-app proof.
- Committing before diff review and required gates.
- Hiding failure or silently changing criteria.
- Spending final minutes on refactors instead of a verified vertical slice.

## Useful phrases while driving the agent

- “That is an inference; I want to verify the contract before encoding it.”
- “The test is red, but I am checking the behavioral reason.”
- “This abstraction has one consumer and is outside the acceptance criteria.”
- “This is focused evidence; the repository gate remains pending.”
- “The reviewer finding is plausible; I will reproduce it before correction.”
- “The curveball changes this criterion, not the whole architecture.”
- “The application owns the defect; the test must not repair it.”
- “I have not verified that, so I will not call it fact.”
- “If the full gate does not fit, I will state exactly what remains.”

---

# One-page final cheat sheet

## Opening

> I built a runtime-configured Angular 22 drinks product with clear config, data, state, and UI boundaries. I used specialized AI roles, one writer, independent review, behavior tests, and command-backed gates. Ingredient filtering is deliberately absent and will be added transparently today.

## Five architecture facts

1. Brand JSON and CocktailDB payloads enter as `unknown` and are validated.
2. CocktailDB DTO names never leave data access.
3. Root list state survives navigation; component detail state isolates requests.
4. Configuration publishes typed state before the shell and applies fixed CSS tokens.
5. Hash routing supports static hosting; a focused service preserves the outer brand query.

## Five orchestration facts

1. Approval checkpoint before non-trivial writes.
2. One writer at a time with exact ownership.
3. Implementer-owned tests begin red.
4. Read-only review runs on a stable diff.
5. The lead reruns gates and curates evidence.

## Evidence numbers

- 31/31 Vitest files
- 276/276 Vitest tests
- 4/4 Playwright Chromium scenarios
- Production build exit `0`
- Live CocktailDB: 100 list entries and detail `15423`
- Vercel root and manifest: HTTP `200`

## Feedback answer

> Workflow: strong human guardrails, but permissions, changed-path scope, writer locking, handoff validation, and duplicated adapters should become machine-enforced.

> Documentation: strong raw audit trail, but it needs an executive summary, requirement-to-evidence index, config schema/versioning guide, and brand runbook.

## Live-build sequence

```text
clarify behavior
-> inspect contracts
-> verify endpoint
-> approval checkpoint
-> config red/green
-> data red/green
-> state red/green
-> UI red/green
-> stable-diff review
-> focused checks
-> full gate if time
-> truthful risk handoff
```

## Primary reflection

> Plausible configuration code passed happy paths but had a concurrency race. Review supplied the hypothesis; a public race test proved it; coalescing corrected it; the same regression and full gate proved the fix.

## Limitations to volunteer when relevant

- Delivered typography/layout values do not differ.
- No complete screen-reader or native-platform test.
- External image/CSP hardening remains.
- Demo metadata is not CocktailDB data.
- Vercel is active, not GitHub Pages.

## Closing

> The result I want to demonstrate is not merely a working feature. It is a controlled path from ambiguous requirement to explicit behavior, bounded AI ownership, verified external contracts, reproducible tests, independent review, and an honest release decision. AI accelerates the work; accountability remains with the engineer.
