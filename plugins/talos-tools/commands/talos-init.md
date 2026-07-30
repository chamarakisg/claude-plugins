---
description: Bootstrap a Talos integration in an Angular/Ionic app. Takes a required npm auth token and a block of environment values. Writes `.npmrc` for the `@saicongames` private registry, installs `@saicongames/talos-integration-angular`, writes `src/environments/environment.ts` with the Talos config keys, and wires `TalosModule.forRoot(...)` into `AppModule`. Use when the user asks to "init/bootstrap/set up talos", "add the talos integration", or "wire up TalosModule" in a fresh app.
argument-hint: <npm-auth-token> [env-values-block]
---

# Talos init

One-shot bootstrap that gets a Talos integration running in an Angular/Ionic app: private-registry auth, the integration package, the `environment.ts` config, and `TalosModule.forRoot(...)` in `AppModule`. This is the setup step you run **once** before `/talos-feature`.

Run this from the root of the Angular/Ionic app (where `package.json` and `src/environments/` live), not from the plugin repo.

Locked decisions (do not re-ask):
- **Registry:** `@saicongames` scope → `https://registry.npmjs.org`, token written to `.npmrc` in the project root.
- **Install:** only `@saicongames/talos-integration-angular`. `@saicongames/talos-api` is a peer dep and comes in automatically — never install it separately.
- **Config shape:** `environment.ts` exports a plain `environment` object with the exact keys in Step 3.
- **Wiring:** `TalosModule.forRoot({...})` added to the `imports` array of `AppModule`, mapping each option from `environment`.

## Inputs

Two inputs (from `$ARGUMENTS`):

1. **npm auth token** (`$1`) — *required*. The token with read access to the `@saicongames` scope. It is a secret — write it only to `.npmrc` (which must be git-ignored). If not provided, **ask and stop until answered**: *"Paste the npm auth token with read access to the `@saicongames` scope."*
2. **environment values** (`$2` onward) — the config values. Accept them as a pasted object/`key: value` block. The required keys are:

   ```
   BASE_URL, GAME_TYPE_ID, USERGROUP_ID,
   APPLICATION_ID_PORTAL, APPLICATION_ID_ANDROID, APPLICATION_ID_IOS,
   MAGIC_KEY_PORTAL, MAGIC_KEY_ANDROID, MAGIC_KEY_IOS,
   VERSION_ID
   ```

   If no values were passed, or any key is missing, **ask for the whole block and stop** until you have all of them. Do not invent, guess, or use placeholder values — every key must come from the user.

Usage: `/talos-init <npm-auth-token>` then paste the values when asked, or pass both up front.

## Step 1 — Configure the private registry (`.npmrc`)

The `@saicongames` packages are `access: restricted`, so npm needs the token to read them (a missing/wrong token fails the install with `401`/`403`).

Write (create or update) `.npmrc` in the project root with both lines, substituting the token from `$1`:

```ini
@saicongames:registry=https://registry.npmjs.org
//registry.npmjs.org/:_authToken=<NPM_AUTH_TOKEN>
```

If `.npmrc` already has these lines, update the token in place rather than duplicating. Then make sure the token can't be committed: confirm `.npmrc` is listed in `.gitignore`, and add it if it isn't.

## Step 2 — Install the integration package

Run from the project root:

```bash
npm install @saicongames/talos-integration-angular
```

`@saicongames/talos-api` (peer dep) is pulled in automatically — do **not** install it separately. After it finishes, verify both resolve under `node_modules`:

- `node_modules/@saicongames/talos-integration-angular`
- `node_modules/@saicongames/talos-api`

If the install fails with `401`/`403`, the token is wrong or lacks scope access — stop and tell the user; don't proceed to Step 3.

## Step 3 — Write `src/environments/environment.ts`

Write the environment file with exactly these keys, filled from the values input. Keep the key order below:

```ts
export const environment = {
  BASE_URL: '<value>',
  GAME_TYPE_ID: '<value>',
  USERGROUP_ID: '<value>',
  APPLICATION_ID_PORTAL: '<value>',
  APPLICATION_ID_ANDROID: '<value>',
  APPLICATION_ID_IOS: '<value>',
  MAGIC_KEY_PORTAL: '<value>',
  MAGIC_KEY_ANDROID: '<value>',
  MAGIC_KEY_IOS: '<value>',
  VERSION_ID: '<value>',
};
```

If `environment.ts` already exists and holds other keys, **merge** — set/overwrite the ten Talos keys above and leave any unrelated keys intact. If a sibling `environment.prod.ts` (or other env file) exists, mention it so the user can mirror the values; do not edit it unless asked.

## Step 4 — Wire `TalosModule.forRoot(...)` into `AppModule`

Edit `src/app/app.module.ts`:

1. Add the imports if missing:
   ```ts
   import { TalosModule } from '@saicongames/talos-integration-angular';
   import { environment } from 'src/environments/environment';
   ```
2. Add `TalosModule.forRoot({...})` to the `@NgModule` `imports` array (typically after `AppRoutingModule`):
   ```ts
   TalosModule.forRoot({
     server: environment.BASE_URL,
     usergroupId: environment.USERGROUP_ID,
     versionId: environment.VERSION_ID,
     applicationIdIos: environment.APPLICATION_ID_IOS,
     applicationIdAndroid: environment.APPLICATION_ID_ANDROID,
     applicationIdPortal: environment.APPLICATION_ID_PORTAL,
     magicKeyIos: environment.MAGIC_KEY_IOS,
     magicKeyAndroid: environment.MAGIC_KEY_ANDROID,
     magicKeyPortal: environment.MAGIC_KEY_PORTAL,
   }),
   ```

If `TalosModule.forRoot(...)` is already in `imports`, update its option mappings to match the block above instead of adding a second one. Preserve the existing formatting and any other modules in the array.

## Finish

- Confirm the four artifacts: `.npmrc` (git-ignored), the installed package under `node_modules`, `environment.ts` with the ten keys, and `TalosModule.forRoot(...)` in `AppModule`.
- Remind the user **not** to commit the token, and that the same token/registry setup is needed in CI.
- Suggest `npm run build` (or `npm start`) to verify the app compiles with `TalosModule` wired.
- Point at `/talos-feature <entity>` as the next step to scaffold a feature against the now-injectable `TalosService`.
