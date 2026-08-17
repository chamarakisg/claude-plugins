---
description: Bootstrap a Talos integration in an Angular/Ionic app. Reads the npm auth token from the `NPM_TOKEN` environment variable and takes a block of environment values. Writes `.npmrc` for the `@saicongames` private registry, installs `@saicongames/talos-integration-angular`, writes `src/environments/environment.ts` with the Talos config keys, and wires `TalosModule.forRoot(...)` into `AppModule`. Use when the user asks to "init/bootstrap/set up talos", "add the talos integration", or "wire up TalosModule" in a fresh app.
argument-hint: <env-values-block>
---

# Talos init

One-shot bootstrap that gets a Talos integration running in an Angular/Ionic app: private-registry auth, the integration package, the `environment.ts` config, and `TalosModule.forRoot(...)` in `AppModule`. This is the setup step you run **once** before `/talos-feature`.

Run this from the root of the Angular/Ionic app (where `package.json` and `src/environments/` live), not from the plugin repo.

Locked decisions (do not re-ask):
- **Registry:** `@saicongames` scope → `https://registry.npmjs.org`. `.npmrc` references the token via `${NPM_TOKEN}`; the literal secret is **never** written to the file.
- **Install:** only `@saicongames/talos-integration-angular`. `@saicongames/talos-api` is a peer dep and comes in automatically — never install it separately.
- **Config shape:** `environment.ts` exports a plain `environment` object with the exact keys in Step 3.
- **Wiring:** `TalosModule.forRoot({...})` added to the `imports` array of `AppModule`, mapping each option from `environment`.

## Inputs

The npm auth token is **not** a command input — `.npmrc` references it via the `NPM_TOKEN` environment variable, which npm reads at install time. The only argument is:

1. **environment values** (`$ARGUMENTS`) — *required*. The config values, accepted as a pasted object/`key: value` block. The required keys are:

   ```
   BASE_URL, GAME_TYPE_ID, USERGROUP_ID,
   APPLICATION_ID_PORTAL, APPLICATION_ID_ANDROID, APPLICATION_ID_IOS,
   MAGIC_KEY_PORTAL, MAGIC_KEY_ANDROID, MAGIC_KEY_IOS,
   VERSION_ID
   ```

   If no values were passed, or any key is missing, **ask for the whole block and stop** until you have all of them. Do not invent, guess, or use placeholder values — every key must come from the user.

Usage: `/talos-init <env-values-block>`, or run `/talos-init` and paste the values when asked. The `NPM_TOKEN` environment variable must be set for the install in Step 2 to authenticate; if it isn't, the install will fail and Step 2 explains how to fix it.

## Step 1 — Configure the private registry (`.npmrc`)

The `@saicongames` packages are `access: restricted`, so npm needs the token to read them (a missing/wrong token fails the install with `401`/`403`).

Write (create or update) `.npmrc` in the project root with both lines **exactly as shown** — keep `${NPM_TOKEN}` as a literal reference; do not expand or inline the token value. npm substitutes the environment variable at read time:

```ini
@saicongames:registry=https://registry.npmjs.org
//registry.npmjs.org/:_authToken=${NPM_TOKEN}
```

If `.npmrc` already has these lines, leave the `${NPM_TOKEN}` reference in place (do not replace it with a literal token). Because no secret is written to the file, `.gitignore` is not strictly required for the token — but still confirm `.npmrc` is git-ignored if the project treats it as local config.

## Step 2 — Install the integration package

Do **not** check whether `NPM_TOKEN` is set beforehand — don't run any command to read or validate it. Once `.npmrc` exists (Step 1), just run the install and let npm succeed or fail:

```bash
npm install @saicongames/talos-integration-angular
```

`@saicongames/talos-api` (peer dep) is pulled in automatically — do **not** install it separately.

**On success**, verify both resolve under `node_modules`, then continue to Step 3:

- `node_modules/@saicongames/talos-integration-angular`
- `node_modules/@saicongames/talos-api`

**If the install fails**, stop and don't proceed to Step 3. When the failure looks auth-related — `401`/`403`, `ENEEDAUTH`, "unable to authenticate", or "404 Not Found" on a `@saicongames` package — the most likely cause is that `.npmrc` resolved `${NPM_TOKEN}` to an empty value. Tell the user:

> The install failed authenticating against the `@saicongames` registry. This is most likely because the `NPM_TOKEN` environment variable is empty or unset — `.npmrc` reads the token from it. Check that it has a value (Windows: `$env:NPM_TOKEN` · macOS/Linux: `echo $NPM_TOKEN`). If it's empty, set it and **restart the terminal/editor** so npm can see it:
>
> - **Windows (PowerShell):** `[Environment]::SetEnvironmentVariable("NPM_TOKEN","<token>","User")`
> - **macOS / Linux:** add `export NPM_TOKEN="<token>"` to `~/.zshrc` or `~/.bashrc`
>
> Then re-run `/talos-init`. If `NPM_TOKEN` is already set, the token itself may be wrong or lack read access to the `@saicongames` scope.

For a non-auth failure (network, disk, unrelated dependency error), report the actual npm error instead — don't assume it's the token.

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

- Confirm the four artifacts: `.npmrc` (referencing `${NPM_TOKEN}`), the installed package under `node_modules`, `environment.ts` with the ten keys, and `TalosModule.forRoot(...)` in `AppModule`.
- Remind the user that `.npmrc` holds no secret (only `${NPM_TOKEN}`), so it's safe to commit — but the same `NPM_TOKEN` environment variable must be set wherever the install runs, including CI.
- Suggest `npm run build` (or `npm start`) to verify the app compiles with `TalosModule` wired.
- Point at `/talos-feature <entity>` as the next step to scaffold a feature against the now-injectable `TalosService`.
