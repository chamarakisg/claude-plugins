# Talos Commands Plugin

Slash commands for building Talos-backed Angular features. Ships `/talos-init`, which bootstraps the Talos integration into an app, and `/talos-feature`, which scaffolds a typed service, a component, and a view for any Talos entity, wrapping `TalosService` from `@saicongames/talos-integration-angular`.

## Overview

Instead of hand-writing the boilerplate to consume a Talos API in an Angular/Ionic app, this plugin drives the whole setup and scaffold from single commands. `/talos-init` gets the integration wired once; `/talos-feature` then inspects the installed Talos type definitions to find an entity's real API and generates code that matches — no invented methods.

## Commands

### `/talos-init <npm-auth-token> <env-values-block>`

One-shot bootstrap that gets a Talos integration running in a fresh app. Run it **once** before `/talos-feature`. Both inputs are required.

- **`<npm-auth-token>`** — *required*. Token with read access to the `@saicongames` scope. Written only to `.npmrc` (kept out of git). If omitted, the command asks and stops.
- **`<env-values-block>`** — *required*. The environment config values (`BASE_URL`, `GAME_TYPE_ID`, `USERGROUP_ID`, the three `APPLICATION_ID_*`, the three `MAGIC_KEY_*`, `VERSION_ID`). If omitted or incomplete, the command asks for the whole block and stops.

**What it does:**

1. Writes `.npmrc` for the `@saicongames` private registry with the token (and ensures it's git-ignored).
2. Installs `@saicongames/talos-integration-angular` (the `@saicongames/talos-api` peer dep comes in automatically).
3. Writes `src/environments/environment.ts` with the ten Talos config keys.
4. Wires `TalosModule.forRoot(...)` into `AppModule`, mapping each option from `environment`.

**Usage:**

Pass both inputs up front — the token, then the environment values as an object of key/value pairs:

```bash
/talos-init npm_xxxxxxxxxxxxxxxxxxxx {
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

Or pass just the token and paste the environment values object when the command asks for them. Both are required — the command won't proceed until it has the token **and** a complete values object.

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

## Requirements

- An Angular + Ionic app (NgModule style) that depends on `@saicongames/talos-integration-angular`. `@saicongames/talos-api` is a peer dependency and is installed automatically — do **not** install it separately.
- `TalosModule.forRoot(...)` wired in `AppModule` so `TalosService` is injectable.

If the packages aren't installed, the command stops and walks you through it.

**1. Configure the private registry.** These are `access: restricted` packages, so npm needs an auth token with read access to the `@saicongames` scope. Create/update `.npmrc` in the project root:

```ini
@saicongames:registry=https://registry.npmjs.org
//registry.npmjs.org/:_authToken=YOUR_NPM_TOKEN
```

Replace `YOUR_NPM_TOKEN` with a real token, or the install fails with `401`/`403`.

**2. Install the integration package only** (the peer dep comes in automatically):

```bash
npm install @saicongames/talos-integration-angular
```

## Design decisions

- Service wraps `TalosService` directly (thin typed methods).
- One feature folder per entity under `src/app/<entity>/`.
- Generate-only for routing — the snippet is printed, not applied.

## Author

George Chamarakis (hamarakisg@iconplatforms.com)

## Version

1.0.0
