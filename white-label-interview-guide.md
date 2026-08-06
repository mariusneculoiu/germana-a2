# Eliq Drinks App — White-Label Architecture and Interview Guide

This document explains how the submitted Eliq Drinks App implements runtime white-labeling, how to present it during the technical interview, what evidence supports it, and which limitations should be stated honestly.

The technical baseline is submitted commit `f22f9be`.

## Executive summary

The application uses one Angular build for multiple brands. A small runtime manifest identifies the default brand and the allowed configuration files. The selected brand JSON is loaded as untrusted data, validated against an exact schema, converted into a typed `AppConfig`, and then consumed by theming, layout, localization, and feature behavior.

White-labeling therefore covers more than colors:

- product identity and local wordmark;
- semantic light/dark design tokens;
- typography and corner radius;
- list layout and content density;
- allowed/default instruction languages;
- explicit product feature flags.

### Requirement verdict

The implementation substantially satisfies the core white-label requirement:

- one build supports multiple runtime-selected brands;
- configuration is external to the Angular bundle;
- two brand files demonstrate different identity, palettes, radius, localization, and feature sets;
- configuration is validated before normal UI rendering;
- invalid or unknown brands produce controlled error state;
- the schema can be extended deliberately for the live feature.

The delivered examples do not demonstrate every supported dimension:

- both use the same typography stacks;
- both use `grid` and `comfortable` layout values;
- the desktop switch is hardcoded for the two demonstration brands rather than derived from the manifest;
- schema version `1` is validated, but no migration framework exists yet.

The accurate interview claim is:

> The runtime white-label architecture is complete and extensible. The delivered brands prove different identity, colors, radius, localization, and feature sets. Typography and layout variation are supported by the schema and UI but are not demonstrated by the final two configuration files.

---

# 1. Architectural overview

## System context

```text
Deployment / static assets
│
├── config/brand-manifest.json
│     ├── default brand ID
│     └── allow-listed brand ID -> config path entries
│
├── config/brands/citrus.json
├── config/brands/midnight.json
└── branding/*.svg
         │
         ▼
Angular application bootstrap
         │
         ├── read outer ?brand= query
         ├── load manifest as unknown
         ├── validate manifest exactly
         ├── select default or allow-listed override
         ├── load selected brand JSON as unknown
         ├── validate exact AppConfig schema
         └── publish loading | ready | error state
                         │
                         ▼
Typed AppConfig
├── identity -> shell/header/brand switch
├── theme -> fixed CSS custom properties
├── layout -> list layout and density
├── localization -> detail language options/fallback
└── features -> metadata and language-selector visibility
```

## Runtime data flow

```text
outer URL query
  -> BRAND_QUERY_VALUE token
  -> brand manifest resource
  -> validateBrandManifest(unknown)
  -> selectBrand(manifest, override)
  -> selected config resource
  -> validateAppConfig(unknown, selectedBrandId)
  -> ConfigLoadState
       ├── loading
       ├── error { code, safe message }
       └── ready { config: AppConfig }
              ├── ThemeService.apply(config, activeMode)
              └── components derive typed behavior
```

## Layer responsibilities

| Layer/file area                     | Responsibility                                                 | Does not own                      |
| ----------------------------------- | -------------------------------------------------------------- | --------------------------------- |
| `public/config/brand-manifest.json` | Default brand and allowed config paths                         | UI rendering or arbitrary URLs    |
| `public/config/brands/*.json`       | Public brand identity and behavior settings                    | Secrets or authorization          |
| `core/config/interfaces`            | Compile-time application contracts                             | Runtime trust                     |
| `runtime-config.validator.ts`       | Runtime validation and reconstruction of typed values          | DOM or HTTP presentation          |
| `brand-selection.ts`                | Default/override selection rules                               | Fetching arbitrary query paths    |
| `ConfigurationService`              | Resource loading, lifecycle, typed state, retry, theme trigger | Feature-specific UI rendering     |
| `ThemeService`                      | Fixed mapping from typed tokens to CSS properties              | Arbitrary CSS from JSON           |
| `BrandQueryPersistenceService`      | Preserve outer brand query during hash navigation              | Brand authorization or validation |
| Feature components                  | Consume typed config to render known behavior                  | Parsing raw JSON                  |

## Why these boundaries matter

The external JSON never directly controls the DOM. It must pass through a narrow contract first:

```text
untrusted input -> validation -> typed application state -> known consumers
```

This prevents three common white-label failures:

1. A misspelled setting is silently ignored.
2. A query or JSON path becomes an arbitrary browser request.
3. Configuration injects uncontrolled CSS property names or markup.

---

# 2. Runtime lifecycle, step by step

## Step 1 — Angular starts configuration initialization

`app.config.ts` registers an application initializer that creates the brand-query persistence service and calls `ConfigurationService.initialize()`.

The normal routed shell is not published until configuration reaches either a usable ready state or a controlled error state.

Why:

- avoid an unbranded flash;
- avoid components reading partial configuration;
- still allow a load/validation error screen to render.

## Step 2 — Read the outer brand override

The public URL uses the query before the hash:

```text
/?brand=midnight#/drinks
```

The value is input to selection, not a file path. The application does not build a URL such as `config/brands/${query}.json`.

## Step 3 — Load and validate the manifest

The application loads:

```text
config/brand-manifest.json
```

The response enters as `unknown`. `validateBrandManifest()` requires exactly:

```json
{
  "defaultBrandId": "citrus",
  "brands": [
    { "id": "midnight", "configPath": "config/brands/midnight.json" },
    { "id": "citrus", "configPath": "config/brands/citrus.json" }
  ]
}
```

Validation checks:

- exact top-level and entry keys;
- at least one brand;
- safe lowercase/hyphenated brand IDs;
- unique brand IDs;
- exactly one matching default;
- safe local paths under `config/brands/`;
- no query, fragment, percent escape, backslash, or unsafe path segment.

## Step 4 — Select a brand

Selection rules:

```text
no override
  -> select manifest.defaultBrandId

known override
  -> select matching manifest entry

unknown explicit override
  -> controlled unknown-brand error
```

An unknown override is not silently converted into a fetch and is not accepted as a best-effort brand.

## Step 5 — Load and validate selected brand JSON

Only the config path already validated from the manifest is requested. The payload again enters as `unknown`.

`validateAppConfig(value, selectedBrandId)` checks the complete schema and also verifies that `brand.id` matches the selected manifest ID.

## Step 6 — Publish typed application state

The service exposes:

```text
loading
ready { config }
error { code, safe message }
```

Expected failures do not expose raw payloads and do not abort the entire Angular bootstrap. The root component can show a controlled diagnostic and Retry action.

Concurrent calls to `initialize()` share one in-flight promise. This prevents duplicate initialization work and stale state publication while still allowing retry after settlement.

## Step 7 — Apply the active theme

When configuration is ready or theme mode changes, `ThemeService` selects the validated light/dark color set and writes only known CSS custom properties.

```text
AppConfig.theme.light/dark
  -> THEME_TOKEN_PROPERTIES fixed map
  -> document.documentElement.style.setProperty(knownName, validatedValue)
```

JSON controls the value but never the CSS property name.

## Step 8 — Components derive behavior

Components consume typed sections:

- shell and header use `brand.name` and logo metadata;
- list page uses `layout.drinkList` and `contentDensity`;
- detail page uses `features.drinkMetadata`;
- detail page uses `features.instructionLanguageSelector`;
- language fallback uses `localization`;
- all visual components inherit semantic CSS tokens.

## Step 9 — Preserve brand selection across navigation

Hash routing keeps Angular routes after `#`, while the brand query is outside it. Standard Angular query preservation does not automatically preserve that outer query in every navigation flow.

`BrandQueryPersistenceService` keeps the validated outer brand value through list/detail navigation and reload. This behavior was added after a browser test exposed that a test workaround had hidden real brand loss.

---

# 3. Configuration schema

## Manifest schema

| Field                 | Meaning                            | Rules                              |
| --------------------- | ---------------------------------- | ---------------------------------- |
| `defaultBrandId`      | Brand used when no override exists | Must match exactly one entry       |
| `brands`              | Deployment allow-list              | Non-empty, unique IDs              |
| `brands[].id`         | Public brand identifier            | Safe lowercase/hyphen format       |
| `brands[].configPath` | Local JSON asset                   | Must remain below `config/brands/` |

## Brand configuration schema

### `schemaVersion`

```json
"schemaVersion": 1
```

Purpose:

- make the contract version explicit;
- reject incompatible future configurations;
- provide a future migration/compatibility boundary.

Current state: only version `1` is accepted; migrations are not implemented.

### `brand`

```json
"brand": {
  "id": "citrus",
  "name": "Citrine Sips",
  "logoUrl": "branding/citrus-wordmark.svg",
  "logoAlt": "Citrine Sips"
}
```

Controls identity. Validation requires matching selected ID, bounded non-empty display strings, and a local path under `branding/`.

### `theme`

Contains:

- twelve semantic light colors;
- twelve semantic dark colors;
- brand/body/mono font stacks;
- corner radius.

Semantic tokens such as `surface`, `primary`, and `onSurface` are preferable to values such as `cardHeaderBlue`, because they can be reused consistently across components and brands.

Validation requires:

- exact token names;
- six-digit hex colors;
- allow-listed font stacks;
- radius in bounded `px` or `rem` format.

### `layout`

```json
"layout": {
  "drinkList": "grid",
  "contentDensity": "comfortable"
}
```

Supported values:

- `drinkList`: `grid | list`;
- `contentDensity`: `comfortable | compact`.

The final delivered brands both use `grid/comfortable`, so support exists but variation is not demonstrated.

### `localization`

```json
"localization": {
  "defaultInstructionLanguage": "en",
  "allowedInstructionLanguages": ["en", "es", "de", "fr", "it"]
}
```

Validation requires:

- finite application-supported language codes;
- non-empty allowed list;
- no duplicates;
- default language included in the allowed set.

The detail page still intersects these allowed languages with translations actually present for the selected drink.

### `features`

```json
"features": {
  "instructionLanguageSelector": true,
  "drinkMetadata": true
}
```

Feature flags control known product presentation behavior. Exact keys prevent arbitrary flags from silently accumulating.

They are public browser configuration, not security authorization.

---

# 4. Delivered brand comparison

| Dimension           | Citrine Sips                    | Midnight Mix              | Demonstrated difference? |
| ------------------- | ------------------------------- | ------------------------- | ------------------------ |
| ID/name             | `citrus` / Citrine Sips         | `midnight` / Midnight Mix | Yes                      |
| Wordmark            | `citrus-wordmark.svg`           | `midnight-wordmark.svg`   | Yes                      |
| Light/dark palettes | green/citrus                    | cyan/blue                 | Yes                      |
| Corner radius       | `20px`                          | `12px`                    | Yes                      |
| Allowed languages   | `en, es, de, fr, it`            | `en`                      | Yes                      |
| Language selector   | enabled                         | disabled                  | Yes                      |
| Drink metadata      | enabled                         | disabled                  | Yes                      |
| Typography          | Sora / DM Sans / JetBrains Mono | same values               | No                       |
| List layout         | grid                            | grid                      | No                       |
| Density             | comfortable                     | comfortable               | No                       |

What to say:

> The two examples demonstrate identity, palettes, shape, localization, and feature variation. Typography, list layout, and density are implemented configuration dimensions, but the final JSON files use the same values, so I present those as supported rather than demonstrated.

---

# 5. Runtime validation and security boundaries

## Why JSON enters as `unknown`

TypeScript interfaces disappear at runtime. This is unsafe by itself:

```text
HTTP response -> asserted AppConfig type -> application
```

The application instead uses:

```text
HTTP response: unknown
  -> structural and value validation
  -> newly constructed AppConfig
  -> application
```

This is especially important for white-label deployments because a configuration typo can otherwise break only one customer deployment while compiling successfully in the shared codebase.

## Exact-key validation

`hasExactKeys()` verifies both required and unexpected fields. Benefits:

- catches misspellings;
- catches partially deployed future schemas;
- prevents ignored configuration drift;
- keeps the public configuration surface intentional.

Trade-off: schema evolution must be explicit. That is desirable for deployments but requires version/migration discipline.

## Safe local paths

Manifest and branding paths are restricted to known prefixes and safe segments. Rejected inputs include query strings, fragments, percent escapes, backslashes, empty segments, and paths outside the intended public asset directories.

This ensures that a brand query selects a manifest entry rather than constructing an arbitrary outbound path.

## Bounded visual values

- Colors must be six-digit hex values.
- Fonts come from a finite allow-list.
- Radius uses a bounded numeric `px` or `rem` token.
- Layout and density use finite unions.
- Languages use a finite application-owned union.

The contract intentionally does not accept arbitrary CSS or language codes simply because they are strings.

## Fixed theme output surface

`ThemeService` contains the CSS property names in application code. It does not iterate arbitrary JSON keys into `style.setProperty()`.

This preserves runtime flexibility while keeping the set of writable design tokens reviewable.

## Cross-field validation

Important rules depend on more than one field:

- selected manifest ID must match `brand.id`;
- default brand must exist exactly once in the manifest;
- default instruction language must be present in the allowed language array;
- brand IDs must be unique;
- allowed languages must be unique.

## Feature flags are not access control

The configuration is downloaded by the browser and can be inspected by users. Its feature flags may hide or show product behavior, but cannot protect sensitive operations.

If a future feature requires permission, the server must authorize the operation independently.

## Remaining hardening

The white-label configuration paths and CSS tokens are constrained. CocktailDB image URLs are validated as HTTP/HTTPS, but hosts are not allow-listed. A production deployment should add an expected image-host policy and Content Security Policy.

---

# 6. How to add another brand

## Development/onboarding procedure

### Step 1 — Choose a stable brand ID

Use a lowercase, hyphen-separated ID, for example:

```text
ocean-bar
```

The ID is a deployment/configuration identifier, not display copy.

### Step 2 — Add owned local assets

Add the approved wordmark under:

```text
public/branding/
```

Provide useful alternative text. Review asset origin, licensing, dimensions, light/dark visibility, and file size.

### Step 3 — Create the brand JSON

Copy one version-1 configuration and update every section deliberately:

- identity;
- light/dark semantic tokens;
- typography and radius;
- layout/density;
- allowed/default languages;
- feature flags.

Do not leave values copied accidentally merely because validation accepts them.

### Step 4 — Add a manifest entry

Add the unique ID and safe local config path to `brand-manifest.json`. Change `defaultBrandId` only if the deployment should default to the new brand.

### Step 5 — Validate all delivered configuration

Run the repository quality gate. The delivered-config specs should parse every shipped JSON file through the same runtime validator rather than only checking TypeScript compilation.

### Step 6 — Preview both modes and target viewports

Inspect:

- logo and alternative text;
- semantic contrast in light and dark modes;
- list and detail at phone/tablet/desktop widths;
- compact/comfortable density if changed;
- grid/list layout if changed;
- long brand names and localized text;
- loading, empty, error, and not-found states;
- keyboard focus and touch targets.

### Step 7 — Add behavior evidence for real differences

If a new brand changes feature flags, layout, density, or localization, add a parameterized test case that proves the resulting public behavior. Do not rely only on a snapshot of its JSON.

### Step 8 — Deploy and smoke-test

Verify:

- root configuration resources return successfully;
- the selected query resolves to the expected brand;
- list and detail routes preserve the brand;
- direct detail reload does not revert to default;
- remote drink data and images behave as expected;
- configuration errors remain safe and actionable.

### Step 9 — Define rollback

Keep the prior config/assets deployable. If the new configuration fails, roll back the static assets or remove the manifest entry without requiring a new product-code design.

## Production onboarding ownership

A mature process should identify:

- product owner for feature decisions;
- brand/design owner for visual tokens and assets;
- engineering owner for schema and migrations;
- accessibility approval;
- deployment approver;
- rollback decision owner.

---

# 7. Scaling to 30+ brands

The current architecture scales better than build-time forks because the product code remains shared. Scaling still requires process and schema discipline.

## What already scales

- one product build;
- manifest-driven config paths;
- exact shared validator;
- semantic theme-token surface;
- explicit features/localization/layout;
- deterministic delivered-config checks;
- brand query independent of Angular route fragment.

## What should change as the schema grows

### Split validation by section and version

Keep an explicit root validator, but move brand, theme, layout, localization, and feature validation into focused versioned units. This reduces monolithic maintenance without accepting arbitrary keys.

### Maintain one schema source

If configuration becomes large, consider one authoritative schema capable of producing:

- runtime validation;
- TypeScript contracts;
- documentation/reference tables;
- editor autocomplete;
- test fixtures.

Adopt a library or generation workflow only when it reduces duplication more than it adds dependency and migration cost.

### Validate every brand in CI

For each manifest entry, CI should verify:

- path exists;
- config validates;
- selected ID matches;
- referenced asset exists;
- light/dark semantic contrast meets agreed thresholds;
- configured features/layout have required browser coverage.

### Avoid permanent flag accumulation

Feature flags should have owners and lifecycle. Remove obsolete flags after rollout rather than creating a permanent matrix of branches nobody can reason about.

### Derive brand tooling from the manifest

The current desktop switch is intentionally hardcoded for two demo brands. A production preview/switcher should derive available choices from the validated manifest or an administrative system.

The deployed consumer application may not need a brand switch at all: each deployment can choose a single default brand.

## Schema evolution strategy

For version `2`, decide before implementation:

1. Whether the app temporarily supports both versions.
2. Whether migrations occur at build/deployment time or in the client.
3. How every delivered brand is upgraded and verified.
4. How unsupported versions are reported.
5. How a partial rollout is prevented.
6. How rollback restores a known-compatible app/config pair.

Recommended default for this product:

```text
define v2
-> migrate all delivered configs in one reviewed change
-> validate both schema and UI behavior in CI
-> deploy app and configs atomically
-> retain previous deployment for rollback
```

Avoid silently applying guessed defaults to an unknown future schema.

---

# 8. Step-by-step interview demonstration

Target duration: three to four minutes.

## Prepare these tabs

1. Citrine Sips browser URL
2. Midnight Mix browser URL
3. `public/config/brand-manifest.json`
4. `public/config/brands/citrus.json`
5. `public/config/brands/midnight.json`
6. `src/app/core/config/validators/runtime-config.validator.ts`
7. `src/app/core/config/services/configuration.service.ts`
8. `src/app/core/config/services/theme.service.ts`
9. Optional: configuration tests

## Demo step 1 — Show the outcome

### Show

Citrine Sips list, then Midnight Mix.

Point out:

- brand name/wordmark;
- palette and radius;
- language/metadata behavior on detail.

### Say

> These are the same Angular build. The differences come from runtime JSON rather than brand-specific code forks.

### Why

Lead with visible product value before implementation details.

### Avoid

Do not claim the final brands use different typography or list layout.

## Demo step 2 — Show the manifest

### Show

`brand-manifest.json` in full; it is intentionally small.

### Say

> The manifest is the deployment-level entry point. It defines the default and allow-lists the IDs and config paths that can be selected. The query chooses an entry; it does not construct a path.

### Why

This answers both scalability and unsafe-path questions.

### Avoid

Do not spend time on Angular component code yet.

## Demo step 3 — Compare both JSON files

### Show

Place them side by side. Focus on:

1. `brand`;
2. a few semantic colors;
3. `cornerRadius`;
4. `localization`;
5. `features`.

### Say

> A brand config controls identity, design tokens, layout, localization, and known product features. Citrine enables five instruction languages, metadata, and the language selector. Midnight permits English only and disables those two UI features.

### Why

This proves behavior-driven white-labeling, not only recoloring.

### Avoid

Do not scroll slowly through every color token. Do not hide identical typography/layout values if asked.

## Demo step 4 — Show runtime validation

### Show

Move quickly between:

- `hasExactKeys`;
- `isSafeLocalPath`;
- font/color/radius checks;
- localization cross-field validation;
- `validateFeatures`;
- root `validateAppConfig`.

### Say

> The JSON enters as `unknown`. TypeScript alone cannot validate deployment files. The validator checks exact structure, finite values, safe paths, and relationships such as the default language being allowed. Only then does it construct `AppConfig`.

### Why

This is the strongest correctness and anti-hallucination part of the white-label design.

### Avoid

Do not read the validator line by line.

## Demo step 5 — Show lifecycle and output boundary

### Show

`ConfigurationService`, then `ThemeService`.

### Say

> The configuration service turns manifest and brand resources into typed loading, ready, or error state. It resolves expected failures into a screen the user can retry rather than rejecting bootstrap. Once ready, the theme service maps validated values only to a fixed CSS-token list.

### Why

This connects input validation to visible UI and safe failure behavior.

### Avoid

Do not say the theme service accepts arbitrary CSS from JSON.

## Demo step 6 — Close with scale and honesty

### Say

> Adding a brand means adding validated JSON and assets, registering it in the manifest, and running shared configuration and browser checks. It does not require another Angular product fork. For dozens of brands I would version the schema, validate every config in CI, derive tooling from the manifest, and define migrations and rollback.

> One limitation is that the final two examples share typography and grid density. Those dimensions are supported but not demonstrated. I would strengthen the examples and add parameterized browser assertions.

---

# 9. Spoken presentations

## Concise 90-second version

> I designed white-labeling as runtime product configuration, not a set of compile-time themes. The application loads a small manifest that defines the default brand and an allow-list of brand IDs and configuration paths. A known `brand` query selects an entry; an unknown query cannot become an arbitrary fetch path.
>
> Each brand JSON controls identity, light and dark semantic tokens, typography, corner radius, layout, localization, and explicit features. Because this is runtime JSON, it enters as `unknown`. An exact validator rejects unexpected keys, mismatched IDs, unsafe asset paths, malformed design tokens, duplicate languages, and invalid defaults.
>
> After validation, the result becomes a typed `AppConfig`. `ConfigurationService` publishes loading, ready, or controlled error state, while `ThemeService` applies values only through fixed CSS custom-property names. Components consume typed layout, localization, and feature settings.
>
> This allows one tested Angular build to support many brand deployments without code forks. The delivered examples prove identity, palette, radius, language, and feature differences. Typography and layout variation are supported but not demonstrated by the final files.

## Extended three-to-four-minute version

> From a product perspective, white-labeling means more than changing a primary color. The same drinks product must be deployable for another brand with different identity, design language, localization, layout choices, and enabled functionality, without creating a separate Angular fork.
>
> The entry point is `brand-manifest.json`. It defines a default brand and allow-lists the IDs and local configuration paths the deployment can load. The optional outer `brand` query selects only one of those entries. It is not interpolated into a file path, so an unknown value produces a controlled error instead of arbitrary fetching.
>
> The selected configuration contains a schema version, brand identity, light and dark semantic tokens, typography, corner radius, list layout and density, localization, and explicit feature flags. Citrine and Midnight use the same compiled application but different identity, palettes, radius, language allowance, and recipe features.
>
> The important trust boundary is that neither manifest nor brand JSON is trusted because TypeScript has an interface. Both enter as `unknown`. The runtime validator requires exact keys and verifies safe IDs and paths, bounded design values, finite fonts and languages, unique values, and cross-field rules. It also confirms that the loaded brand ID matches the manifest selection.
>
> Configuration loading finishes before the normal routed shell is published. Successful validation produces a typed `AppConfig`; expected failures become typed error state that the root can display and retry. When ready, the theme service writes only a fixed list of CSS property names. The JSON controls validated values but cannot invent the output surface.
>
> Components then derive known behavior: list layout, instruction language fallback, metadata visibility, and language-selector visibility. Brand selection is also preserved outside the hash route through navigation and reload.
>
> For a third brand, I would add owned assets, create a complete version-1 JSON file, register its safe path in the manifest, validate every delivered config, run the relevant browser behavior, and smoke-test the deployment. At 30 or more brands I would split validation by versioned section, generate documentation from a canonical schema, validate every brand in CI, and define migration and rollback policy.
>
> The honest limitation is that the final two files use identical typography and grid/comfortable layout values. Those configuration dimensions exist and are consumed, but the examples do not visually demonstrate them. The desktop switch is also a two-brand demo affordance; production tooling should derive choices from the manifest, or each deployment should expose only its configured brand.

---

# 10. Testing and evidence

Different checks prove different claims.

| Evidence layer                | What it covers                                                                                              | What it does not prove                             |
| ----------------------------- | ----------------------------------------------------------------------------------------------------------- | -------------------------------------------------- |
| Manifest/validator unit tests | Exact keys, IDs, paths, colors, radius, typography, layout, languages, features, cross-field rules          | Browser appearance                                 |
| Delivered-config tests        | Every shipped JSON file passes the same runtime validator and references expected local assets              | That every configured visual difference looks good |
| Configuration service tests   | Loading, selection, unknown brand, load/validation errors, retry, concurrency                               | Full routed application behavior                   |
| Theme service tests           | Correct fixed CSS-token mapping for typed config and mode                                                   | Visual contrast in all components                  |
| Component tests               | Feature flags, layout classes, language behavior, error UI                                                  | Real navigation/reload integration                 |
| Playwright                    | Both brands, exact config requests, brand persistence, visible feature differences, mobile/desktop behavior | Permanent deployment/API availability              |
| Vercel smoke                  | Current deployed root/config/brand behavior at check time                                                   | Future availability or all edge cases              |

## Evidence to mention in the interview

- Brand JSON is not considered correct merely because it matches a TypeScript interface.
- Both delivered brand files are validated by tests.
- Unknown brand IDs and malformed config have controlled states.
- Theme values are asserted without depending only on screenshots.
- Playwright intercepts configuration and external data and rejects unexpected requests.
- Browser tests caught outer-query persistence behavior that unit tests missed.

## Strong evidence phrasing

> Validator tests prove the runtime contract. Delivered-config tests prove the checked-in files satisfy it. Component tests prove feature/layout consumption. Browser tests prove selection, navigation, and visible brand behavior work together. No one layer proves all of those claims.

---

# 11. Gaps and improvements

## Gap 1 — Typography variation is not demonstrated

Both brands configure:

```text
Sora / DM Sans / JetBrains Mono
```

Improvement:

- select an already allow-listed alternative for one brand;
- verify readable fallback behavior;
- add a browser/computed-style assertion only if it provides stable value.

## Gap 2 — Layout variation is not demonstrated

Both brands configure:

```text
grid / comfortable
```

Improvement:

- deliver one `list/compact` example;
- inspect phone/desktop behavior;
- parameterize an E2E assertion for layout class and usable card interaction.

## Gap 3 — Demo switch is hardcoded

The list page knows the two delivered IDs and target labels.

Improvement options:

1. Remove the switch in single-brand production deployments.
2. Derive a preview switcher from the validated manifest.
3. Place administrative brand selection outside the consumer app.

Do not make application components responsible for reading raw manifest JSON.

## Gap 4 — No schema migration mechanism

Version `1` is explicit and rejected when different, which is a good safety boundary. The project does not yet include multi-version parsing or migrations.

Improvement: define version policy only when a real incompatible change exists. Avoid speculative migration infrastructure for one version.

## Gap 5 — Documentation navigation

The README explains the schema, but a larger deployment program needs:

- generated field reference;
- examples and failure codes;
- brand onboarding checklist;
- version compatibility table;
- owner/approval/rollback runbook.

---

# 12. Likely interview questions

## Why runtime JSON instead of Angular environment files?

> Environment files are compiled into a build. Runtime JSON lets the same tested artifact serve multiple brand deployments and lets deployment configuration change within a validated contract without rebuilding product code.

## Why a manifest plus separate brand files?

> The manifest gives the deployment one controlled default and an allow-list. Separate files keep each brand independently owned and deployable without one large configuration object.

## Why not construct the config path from `?brand=`?

> It would turn user-controlled input into a fetch path. Selection is restricted to paths already validated from the manifest.

## Why exact validation?

> Silent unknown keys are dangerous in deployment configuration because a typo can look like a valid rollout. Exact validation turns drift into a controlled failure.

## Why not use only TypeScript interfaces?

> Interfaces provide compile-time checking for code, not runtime checking for downloaded JSON. The payload must be validated before it is treated as `AppConfig`.

## Why custom validation instead of a library?

> The current schema is small and finite. Dependency-free validation is reviewable and avoids bundle/supply-chain cost. If the schema grows, a canonical schema with generated runtime/types/docs may reduce duplication.

## Why apply config before rendering?

> It prevents an unbranded flash and avoids feature components reading incomplete values. Expected failures still become a renderable typed error instead of a blank boot failure.

## What happens when configuration is invalid?

> The service exposes a fixed safe error code/message and Retry. Raw config or network payloads are not shown to the user.

## How do feature flags scale?

> Keep them explicit, typed, validated, owned, and removable. Avoid an arbitrary key/value dictionary and remove rollout flags when they are no longer needed.

## Are flags secure?

> No. They are public client configuration. Any sensitive operation needs server authorization.

## Can this support 30 brands?

> The runtime selection and shared build can. Operationally, I would add canonical schema documentation, automated validation for every manifest entry and asset, version migration rules, stronger visual checks, and clear brand ownership.

## Why semantic colors?

> Components depend on meaning such as surface/on-surface/primary rather than brand-specific names. A brand can replace values consistently without rewriting component CSS.

## How would you add a new feature flag live?

> Clarify behavior first, add a failing validator/delivered-config test, extend the typed contract and exact validator, update both brand files explicitly, consume the flag at the relevant feature boundary, add public behavior tests, and run the integration gate.

## How would you roll back a bad brand config?

> Restore the prior validated JSON/assets or manifest entry as one deployment unit. Keep safe defaults and prior deployment artifacts available, and do not rely on the client to guess how to repair an unknown schema.

## What is the biggest current weakness?

> The final examples do not demonstrate supported typography/layout variation, and the demo switch is hardcoded. Neither invalidates the runtime architecture, but both reduce how convincingly the delivered examples prove scale.

## Why is query persistence a separate service?

> The outer brand query has routing-specific lifecycle behavior under hash routing. Keeping it in focused configuration infrastructure prevents feature components from duplicating URL correction logic.

## Would you place API endpoints in brand config?

> Not automatically. Public endpoint selection changes the trust and operational boundary. If brands genuinely require endpoints, I would validate against an allow-list or deployment-controlled environment contract, not accept arbitrary URLs from public JSON.

---

# 13. One-page cheat sheet

## Central message

> White-labeling is runtime product configuration, not compile-time theme duplication. JSON is untrusted, exactly validated, converted to typed state, and consumed through narrow known interfaces.

## File order

1. Browser: Citrine
2. Browser: Midnight
3. `public/config/brand-manifest.json`
4. `public/config/brands/citrus.json`
5. `public/config/brands/midnight.json`
6. `runtime-config.validator.ts`
7. `configuration.service.ts`
8. `theme.service.ts`

## Architecture in one line

```text
manifest -> allow-listed selection -> unknown JSON -> exact validation -> typed AppConfig -> fixed theme/layout/localization/features
```

## Five strengths

1. One build, multiple runtime brands.
2. Manifest allow-list instead of query-built paths.
3. Exact runtime validation of all public config.
4. Controlled loading/error/retry lifecycle before normal shell.
5. Fixed CSS-token output plus typed feature consumption.

## Two limitations to volunteer

1. Delivered typography and layout values are identical.
2. The two-brand switch is a hardcoded demo affordance.

## Three scale improvements

1. Canonical versioned schema generating validation/types/docs.
2. Validate every config, asset, contrast rule, and behavior in CI.
3. Manifest-derived preview tooling plus migration/rollback policy.

## Do not claim

- that final typography/layout differs;
- that feature flags provide authorization;
- that TypeScript validates JSON;
- that an in-app switch is necessary for every deployment;
- that version migrations already exist.

## Closing statement

> The main architectural decision is that configuration cannot bypass application boundaries. It selects from known files, enters as `unknown`, passes an exact versioned validator, becomes typed application state, and affects only known theme and product surfaces. That gives the business runtime flexibility without turning every brand into a code fork or an unbounded input channel.
