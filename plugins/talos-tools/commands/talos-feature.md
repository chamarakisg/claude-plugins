---
description: Scaffold an Angular feature (service + component + view) for a Talos entity. Takes a required entity argument and an optional mockup argument — `/talos-feature <entity> [mockup]`. Finds the entity's API on TalosService in @saicongames/talos-integration-angular, generates a typed service wrapping the relevant API calls, and a component + module that renders a view (matching the mockup if given). Use when the user asks to "generate/scaffold a talos feature/service/component/view", "add a page for <entity>", or wire up a Talos API entity in this app.
argument-hint: <entity> [mockup-path-or-description]
---

# Talos feature scaffolder

Generate a service + component + view for one Talos entity, wrapping `TalosService`. Run this from an Angular/Ionic app that depends on `@saicongames/talos-integration-angular` and `@saicongames/talos-api`.

Locked design decisions (do not re-ask):
- **Service pattern:** wrap `TalosService` directly with thin typed methods.
- **Output layout:** one feature folder `src/app/<entity>/` — `<entity>.service.ts`, `<entity>.component.ts` / `.html` / `.scss`, `<entity>.module.ts`.
- **Module style:** NgModule, `standalone: false`, Ionic components (match the host app's existing pages, e.g. the tabs starter).
- **Routing:** generate only. Do **not** edit routing files — print the snippet to add instead.

This command is **generic** — it works for any Talos entity. Nothing below is hardcoded to a specific API; `quiz`/`Quiz`/`QuizApi` appears only as an illustrative fill-in of the placeholders. Always resolve the real entity from the user and the real API from `TalosService`.

## Inputs

The command takes two arguments (from `$ARGUMENTS`):

1. **entity** (`$1`) — *required*. The Talos entity to scaffold (e.g. `quiz`, `tournament`, `favorites`).
2. **mockup** (`$2`, may be a quoted string) — *optional*. How the view should look. Either a path to a file (image, `.html`, `.md`) or an inline description. When present, it drives Step 5; when absent, Step 5 falls back to a reference view or a simple list.

Usage: `/talos-feature <entity> [mockup]` — e.g. `/talos-feature quiz` or `/talos-feature quiz ./designs/quiz.png`.

## Step 1 — Resolve the entity (required input)

Read the **entity** from `$1`. If no arguments were given, **ask and stop until answered** — it is required: *"Which Talos entity do you want to scaffold a feature for? (e.g. quiz, tournament, favorites)"*.

Also capture the optional **mockup** (`$2` onward) if one was passed (a path or a description); hold it for Step 5. Do not ask for it — its absence is valid.

Normalize the entity and derive the casings used throughout as placeholders:
- `<entity>` — folder / selector / file base (kebab, lowercase), e.g. `quiz`
- `<Entity>` — class base (PascalCase), e.g. `Quiz`
- `<talosProp>` / `<ApiClass>` — TalosService property and its API type: **resolved in Step 2, never guessed.**

## Preflight — verify the Talos packages are installed

Before Step 2, check that both packages resolve under `node_modules`:

- `node_modules/@saicongames/talos-integration-angular`
- `node_modules/@saicongames/talos-api`

If **either** is missing, **stop** — do not attempt Step 2 against absent files, and do not run `npm install` or write any token yourself. Instead, tell the user the Talos packages aren't installed and point them at the sibling command that sets everything up:

> The Talos packages aren't installed yet. Run **`/talos-init <npm-auth-token>`** to bootstrap the integration — it writes the `.npmrc` for the `@saicongames` private registry with your token, installs `@saicongames/talos-integration-angular` (the `@saicongames/talos-api` peer dep comes in automatically), writes `environment.ts`, and wires `TalosModule.forRoot(...)` into `AppModule`. Then re-run `/talos-feature`.

`/talos-init` needs an npm auth token with read access to the `@saicongames` scope (the packages are `access: restricted`, so without it the install fails with `401`/`403`).

Only continue to Step 2 once both packages are present in `node_modules`.

## Step 2 — Find the API on TalosService

1. Read `node_modules/@saicongames/talos-integration-angular/dist/lib/talos.service.d.ts`. It exposes every API as a `readonly` property (e.g. `readonly quiz: QuizApi;`). Find the property whose name matches the entity — that name is `<talosProp>` (the accessor `this.talos.<talosProp>`) and its type is `<ApiClass>`.
2. If no property matches, **stop** and show the user the list of available property names (grep the `readonly` lines) so they can pick the right one.
3. Read the API class to get exact signatures: `node_modules/@saicongames/talos-api/dist/types/api/<ApiClass>.d.ts`. Note for each method: name, param interfaces, and the return DTO type. The same file declares the query-param / input interfaces it references.

Do not invent methods — only wrap methods that actually exist in that `.d.ts`.

## Step 3 — Generate the service

`src/app/<entity>/<entity>.service.ts`. Inject `TalosService`; add one thin method per **relevant** API method (favor the list/read methods for the default view; include write/action methods the view needs). Import DTO and param types from `@saicongames/talos-api`.

Conventions:
- Methods return the API's `Promise<...>` directly (don't wrap in RxJS unless a reference view does).
- Where an API method takes a `userId`, default it to `TAValues.UserId!` unless the caller passes one.

Template — replace `<Entity>` / `<entity>` / `<talosProp>` and the method set with what you found in the actual `.d.ts` (the two methods here are just placeholders):

```ts
import { Injectable } from '@angular/core';
import { TalosService } from '@saicongames/talos-integration-angular';
import { TAValues } from '@saicongames/talos-api';
import type {
  <Entity>DTO,
  // ...import the exact param/input interfaces the methods use
} from '@saicongames/talos-api';

@Injectable({ providedIn: 'root' })
export class <Entity>Service {
  constructor(private talos: TalosService) {}

  // one thin method per relevant API method found on this.talos.<talosProp>
  list<Entity>(queryParams?: List<Entity>QueryParams): Promise<<Entity>DTO[]> {
    return this.talos.<talosProp>.list<Entity>(queryParams);
  }

  // where the API method takes a userId, default it to the current user:
  someUserScopedCall(
    queryParams: SomeQueryParams,
    userId: string = TAValues.UserId!,
  ): Promise<<Entity>DTO[]> {
    return this.talos.<talosProp>.someUserScopedCall(userId, queryParams);
  }
}
```

If a DTO/param type isn't exported from the top-level `@saicongames/talos-api`, fall back to the deep path shown in the API `.d.ts` import (e.g. `@saicongames/talos-api/dist/types/...`), or use the method's declared return type via inference and type the field as that.

## Step 4 — Generate the component + module

`src/app/<entity>/<entity>.component.ts` — inject the generated service, load the primary list/read method in `ngOnInit`, expose data + a `loading` flag.

```ts
import { Component, OnInit, signal } from '@angular/core';
import { <Entity>Service } from './<entity>.service';
import type { <Entity>DTO } from '@saicongames/talos-api';

@Component({
  selector: 'app-<entity>',
  templateUrl: '<entity>.component.html',
  styleUrls: ['<entity>.component.scss'],
  standalone: false,
})
export class <Entity>Component implements OnInit {
  items = signal<<Entity>DTO[]>([]);
  loading = signal(false);

  constructor(private <entity>Service: <Entity>Service) {}

  async ngOnInit(): Promise<void> {
    this.loading.set(true);
    try {
      this.items.set(await this.<entity>Service.list<Entity>());
    } catch (err) {
      console.error('Failed to load <entity>', err);
    } finally {
      this.loading.set(false);
    }
  }
}
```

`src/app/<entity>/<entity>.module.ts` — mirror the host app's existing page modules (e.g. `src/app/tab1/tab1.module.ts`):

```ts
import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FormsModule } from '@angular/forms';
import { IonicModule } from '@ionic/angular';
import { <Entity>Component } from './<entity>.component';

@NgModule({
  imports: [CommonModule, FormsModule, IonicModule],
  declarations: [<Entity>Component],
  exports: [<Entity>Component],
})
export class <Entity>Module {}
```

## Step 5 — The view (`<entity>.component.html` + `.scss`)

Pick the first that applies:

1. **Mockup input was provided** (Step 1): build the template to match it, binding to the service data. If it's a file path, read it first (use the Read tool — it renders images); if it's an inline description, follow it directly.
2. **No mockup, but a reference view exists** (user names a file, or a similar `*.component.html` / `*.page.html` already exists in `src/app`): read it and mirror its structure, Ionic components, and SCSS conventions for the new entity.
3. **Neither:** generate a simple Ionic list view over the primary DTO. Use `@if (loading())` for a spinner and `@for` over the items, rendering a couple of obvious fields (name/title/id). Keep `.scss` minimal.

Default simple view:

```html
<ion-header>
  <ion-toolbar>
    <ion-title><Entity></ion-title>
  </ion-toolbar>
</ion-header>

<ion-content class="ion-padding">
  @if (loading()) {
    <ion-spinner></ion-spinner>
  } @else {
    <ion-list>
      @for (item of items(); track item.id) {
        <ion-item>
          <ion-label>{{ item.name || item.id }}</ion-label>
        </ion-item>
      }
    </ion-list>
  }
</ion-content>
```

Adjust the bound fields to real properties of the DTO (check the DTO in `node_modules/@saicongames/talos-api/dist/types/models`).

## Finish

- Confirm `TalosModule.forRoot(...)` is wired in the host app's `src/app/app.module.ts` (so `TalosService` is injectable). If it isn't, tell the user it's required and point at the package README.
- Print (do not apply) the routing snippet, e.g.:
  ```ts
  // src/app/tabs/tabs-routing.module.ts — add a child route:
  { path: '<entity>', loadChildren: () => import('../<entity>/<entity>.module').then(m => m.<Entity>Module) }
  ```
  Note that lazy-loading a plain component module needs a routing module or a `RouterModule.forChild` route to a page; if the user wants it reachable, offer to convert the component to a page with its own `*-routing.module.ts` like the tabs.
- Optionally suggest `npm run lint` to verify the generated files.

Summarize what was created (paths) and any DTO fields you guessed so the user can correct them.
