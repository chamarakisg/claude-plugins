# Talos Commands Plugin

Slash commands for building Talos-backed Angular features. Ships `/talos-init`, which bootstraps the Talos integration into an app; `/talos-feature`, which scaffolds a typed service, a component, and a view for any Talos entity wrapping `TalosService`; and `/talos-feature-list`, which scaffolds a component + module that renders an entity list using the ready-made `<talos-list>` component from `@saicongames/talos-integration-angular`.

## Overview

Instead of hand-writing the boilerplate to consume a Talos API in an Angular/Ionic app, this plugin drives the whole setup and scaffold from single commands. `/talos-init` gets the integration wired once; `/talos-feature` then inspects the installed Talos type definitions to find an entity's real API and generates code that matches — no invented methods. `/talos-feature-list` covers the common "just render a list" case by generating a component around the library's `<talos-list>` component, so loading, paging, and empty state come for free.

## Commands

### `/talos-init <env-values-block>`

One-shot bootstrap that gets a Talos integration running in a fresh app. Run it **once** before `/talos-feature`.

The npm auth token is read from the **`NPM_TOKEN` environment variable**, not passed as an argument. It must be set (as a system/user env var with read access to the `@saicongames` scope) before running. The command doesn't pre-check it — if `NPM_TOKEN` is empty, the `npm install` in step 2 fails to authenticate and the command tells you to set it and retry.

- **`<env-values-block>`** — *required*. The environment config values (`BASE_URL`, `GAME_TYPE_ID`, `USERGROUP_ID`, the three `APPLICATION_ID_*`, the three `MAGIC_KEY_*`, `VERSION_ID`). If omitted or incomplete, the command asks for the whole block and stops.

**What it does:**

1. Writes `.npmrc` for the `@saicongames` private registry referencing `${NPM_TOKEN}` (no secret in the file).
2. Installs `@saicongames/talos-integration-angular` (the `@saicongames/talos-api` peer dep comes in automatically). If the install fails to authenticate, it points you at `NPM_TOKEN` as the likely cause.
3. Writes `src/environments/environment.ts` with the ten Talos config keys.
4. Wires `TalosModule.forRoot(...)` into `AppModule`, mapping each option from `environment`.

**Usage:**

First make sure the `NPM_TOKEN` environment variable is set (with read access to the `@saicongames` scope) and that you've restarted the terminal/editor so the value is inherited. Set it however your OS expects — a persistent user/system variable, a shell profile `export`, or your CI's secret store.

Then run with just the environment values as an object of key/value pairs:

```bash
/talos-init {
  BASE_URL: 'https://...',
  GAME_TYPE_ID: '...',
  USERGROUP_ID: '...',
  APPLICATION_ID_PORTAL: '...',
  APPLICATION_ID_ANDROID: '...',
  APPLICATION_ID_IOS: '...',
  MAGIC_KEY_PORTAL: '...',
  MAGIC_KEY_ANDROID: '...',
  MAGIC_KEY_IOS: '...',
  VERSION_ID: '...'
}
```

Or run `/talos-init` and paste the environment values object when the command asks for them. The command needs a complete values object to proceed; if `NPM_TOKEN` isn't set, it still runs but the install step fails to authenticate and tells you to set the token and retry.

### `/talos-feature <entity> [mockup]`

- **`<entity>`** — *required*. The Talos entity to scaffold (e.g. `quiz`, `tournament`, `favorites`). If omitted, the command asks for it and stops until answered.
- **`[mockup]`** — *optional*. A file path (image, `.html`, `.md`) or an inline description of how the view should look. When given, the generated view matches it; otherwise the command mirrors an existing view or falls back to a simple Ionic list.

**What it does:**

1. Resolves the entity from the argument (or asks).
2. Finds the matching API property on `TalosService` (`node_modules/@saicongames/talos-integration-angular`) and reads its real method signatures from `@saicongames/talos-api`.
3. Generates `src/app/<entity>/<entity>.service.ts` — a thin, typed wrapper over the relevant API calls.
4. Generates `<entity>.component.ts` + `<entity>.module.ts` (NgModule, `standalone: false`, Ionic) that use the service.
5. Generates the view from the mockup, a reference view, or a simple list.
6. Prints the routing snippet to add (never edits routing files).

**Usage:**

```bash
/talos-feature quiz
/talos-feature quiz ./designs/quiz.png
/talos-feature tournament "a card grid with a leaderboard header"
```

### `/talos-feature-list <entity> [mockup]`

Scaffolds a component + module that renders a Talos entity list with the library's ready-made **`<talos-list>`** component. Where `/talos-feature` hand-rolls the list in the template, this command leans on `<talos-list>`: you supply a **presenter** and a **`#defaultItem` card template**, and the component handles loading, empty state, pagination, load-more, grid, and slider for you.

- **`<entity>`** — *required*. The Talos entity to list (e.g. `quiz`, `tournament`, `achievements`, `favorites`). If omitted, the command asks for it and stops until answered.
- **`[mockup]`** — *optional*. A file path (image, `.html`, `.md`) or an inline description. It drives the card template and the layout (stacked / `[columns]` grid / `slider` / `[pagerEnabled]` pager).

**What it does:**

1. Resolves the entity from the argument (or asks).
2. Resolves the data source — prefers a **built-in** presenter exported from the package (e.g. `TalosQuizListPresenter`, `TalosAchievementsPresenter`); if none matches, generates a thin `ListBasePresenter` subclass wrapping the real `TalosService` API call.
3. Resolves the item DTO from the installed `.d.ts` files so the card binds to real fields, and picks the identity field for `trackByKey`.
4. Generates `<entity>.component.ts` / `.html` / `.scss` + `<entity>.module.ts` (NgModule, `standalone: false`, importing `TalosFeatureModule`) that drop in `<talos-list>` with a projected `#defaultItem` card.
5. Optionally wires imperative create/edit/delete via `@ViewChild(TalosListComponent)` (`prependItem` / `updateItem` / `removeItem` / `reload`).
6. Prints the routing snippet to add (never edits routing files).

**Usage:**

```bash
/talos-feature-list quiz
/talos-feature-list tournament ./designs/tournament-grid.png
/talos-feature-list achievements "a horizontal slider of badge cards"
```

## Requirements

- An Angular + Ionic app (NgModule style) that depends on `@saicongames/talos-integration-angular`. `@saicongames/talos-api` is a peer dependency and is installed automatically — do **not** install it separately.
- `TalosModule.forRoot(...)` wired in `AppModule` so `TalosService` is injectable.

If the packages aren't installed, the command stops and walks you through it.

**1. Configure the private registry.** These are `access: restricted` packages, so npm needs an auth token with read access to the `@saicongames` scope. Set the token as the `NPM_TOKEN` environment variable, then create/update `.npmrc` in the project root referencing it:

```ini
@saicongames:registry=https://registry.npmjs.org
//registry.npmjs.org/:_authToken=${NPM_TOKEN}
```

npm expands `${NPM_TOKEN}` at read time — keep it as a literal reference (no secret in the file). If `NPM_TOKEN` is unset/empty, the install fails with `401`/`403`.

**2. Install the integration package only** (the peer dep comes in automatically):

```bash
npm install @saicongames/talos-integration-angular
```

## Design decisions

- Service wraps `TalosService` directly (thin typed methods).
- One feature folder per entity under `src/app/<entity>/`.
- Generate-only for routing — the snippet is printed, not applied.
- `/talos-feature-list` renders via the exported `<talos-list>` component (never hand-rolled `*ngFor`/paging) and prefers a built-in `ListBasePresenter` over a generated one.

## Author

George Chamarakis (hamarakisg@iconplatforms.com)

## Version

1.0.0
