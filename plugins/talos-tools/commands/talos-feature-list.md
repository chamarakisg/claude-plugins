---
description: Scaffold an Angular feature that renders a Talos entity list with the ready-made `<talos-list>` component from @saicongames/talos-integration-angular. Takes a required entity argument and an optional mockup argument — `/talos-feature-list <entity> [mockup]`. Resolves (or generates) a `ListBasePresenter` for the entity, then generates a host component + module that drops in `<talos-list>` with a `#defaultItem` card template — loading, empty state, paging, and load-more are handled by the component. Use when the user asks to "list/show <entity> with talos-list", "make a paginated/infinite-scroll list of <entity>", "render a talos-list of <entity>", or "add a grid/slider of <entity>".
argument-hint: <entity> [mockup-path-or-description]
---

# Talos list-feature scaffolder

Generate a host component + module that renders a Talos entity list using the library's `<talos-list>` component. Unlike `/talos-feature` (which wraps `TalosService` in a service and hand-rolls the list in the template), this command leans on `<talos-list>`: you supply a **presenter** and a **card template**, and the component handles loading, empty state, pagination, load-more, grid, and slider for you.

Run this from an Angular/Ionic app that depends on `@saicongames/talos-integration-angular` and `@saicongames/talos-api`.

Locked design decisions (do not re-ask):
- **Rendering:** use the exported `<talos-list>` component (from `TalosFeatureModule`). Do **not** hand-roll `*ngFor`/loading/paging — that is the whole point of `<talos-list>`.
- **Data source:** a `ListBasePresenter`. Prefer a **built-in** presenter exported from the package when one matches the entity; otherwise generate a thin presenter subclass.
- **Card template:** each item is rendered via a projected `<ng-template #defaultItem let-item let-i="index">`. Keep it simple unless a mockup says otherwise.
- **Output layout:** one feature folder `src/app/<entity>/` — `<entity>.component.ts` / `.html` / `.scss`, `<entity>.module.ts`, and `<entity>-list.presenter.ts` **only if** a built-in presenter does not already exist.
- **Module style:** NgModule, `standalone: false`, imports `TalosFeatureModule` (plus `CommonModule` and, if the host app is Ionic, `IonicModule`).
- **Routing:** generate only. Do **not** edit routing files — print the snippet to add instead.

This command is **generic** — nothing below is hardcoded to a specific entity. `quiz`/`Quiz`/`QuizDTO`/`TalosQuizListPresenter` appears only as an illustrative fill-in of the placeholders. Always resolve the real entity, presenter, and DTO from the installed packages.

## Inputs

Two arguments (from `$ARGUMENTS`):

1. **entity** (`$1`) — *required*. The Talos entity to list (e.g. `quiz`, `tournament`, `achievements`, `favorites`).
2. **mockup** (`$2`, may be a quoted string) — *optional*. How each card / the layout should look. Either a path to a file (image, `.html`, `.md`) or an inline description. When present, it drives Step 4 (the card template) and Step 5 (grid vs slider vs stacked). When absent, Step 4 falls back to a minimal card.

Usage: `/talos-feature-list <entity> [mockup]` — e.g. `/talos-feature-list quiz` or `/talos-feature-list tournament ./designs/tournament-grid.png`.

## Preflight — verify the Talos packages are installed (do this first)

**Before anything else** — before resolving the entity, reading any `.d.ts`, or generating files — confirm the Talos integration is present. Check that `@saicongames/talos-integration-angular` is **declared** in `package.json` and that both packages are **installed** under `node_modules`. Only `@saicongames/talos-integration-angular` is a direct dependency — `@saicongames/talos-api` is its **peer dependency**, so it resolves under `node_modules` but is **not** expected in the app's `package.json`.

1. Read the project root `package.json` and look under `dependencies` (and `devDependencies`) for **`@saicongames/talos-integration-angular`** only.
2. Confirm both packages resolve on disk:
   - `node_modules/@saicongames/talos-integration-angular`
   - `node_modules/@saicongames/talos-api`

Interpret the result:

- **`@saicongames/talos-integration-angular` declared in `package.json` but either package missing from `node_modules`** — dependencies aren't installed. Tell the user to run `npm install`, then re-run `/talos-feature-list`.
- **`@saicongames/talos-integration-angular` missing from `package.json`** — the integration was never bootstrapped. **Stop** and point the user at **`/talos-init`** (it writes `.npmrc`, installs the package, writes `environment.ts`, and wires `TalosModule.forRoot(...)`). Do not resolve the entity or write any files.

Only proceed once the package is declared in `package.json` **and** both packages are present in `node_modules`.

## Step 1 — Resolve the entity (required input)

Read the **entity** from `$1`. If no arguments were given, **ask and stop until answered**: *"Which Talos entity do you want to render as a list? (e.g. quiz, tournament, achievements)"*.

Also capture the optional **mockup** (`$2` onward) if one was passed; hold it for Steps 4–5. Do not ask for it — its absence is valid.

Normalize the entity and derive the casings used as placeholders throughout:
- `<entity>` — folder / selector / file base (kebab, lowercase), e.g. `quiz`
- `<Entity>` — class base (PascalCase), e.g. `Quiz`
- `<Entity>DTO` — the item type — **resolved in Step 3, never guessed.**
- `<Presenter>` — the presenter class — **resolved in Step 2, never guessed.**

## Step 2 — Resolve the presenter

`<talos-list>` needs a `ListBasePresenter` bound via `[presenter]`. Find the best source, in this order:

1. **Built-in exported presenter (preferred).** Read `node_modules/@saicongames/talos-integration-angular/dist/public-api.d.ts` (or grep the `dist` `.d.ts` files) for exported classes ending in `Presenter` whose name matches the entity — e.g. `TalosQuizListPresenter`, `TalosAchievementsPresenter`, `TalosTournamentsPresenter`, `TalosFavoritesPresenter`. If one clearly matches, use it: it is `@Injectable({ providedIn: 'root' })`, so just inject it in the component constructor and bind it. **No presenter file is generated in this case.**
   - Read that presenter's `.d.ts` to learn the shape of `getListItems(queryParams, body, pathParams, extraParams)` — specifically which of `queryParams` / `pathParams` it actually reads (e.g. many user-scoped presenters require `pathParams['userId']`). That drives the bindings in Step 4.
2. **No matching built-in presenter.** Generate a thin one at `src/app/<entity>/<entity>-list.presenter.ts` that extends `ListBasePresenter`, calls the right `TalosService` API for the entity (resolve it exactly as `/talos-feature` Step 2 does — find the `readonly <talosProp>: <ApiClass>` on `talos.service.d.ts`, read the `<ApiClass>.d.ts` for the real list method), and pushes the result through `this.itemObservable`:

   ```ts
   import { Injectable } from '@angular/core';
   import { ListBasePresenter, TalosService } from '@saicongames/talos-integration-angular';
   import type { <Entity>DTO } from '@saicongames/talos-api';
   // import the exact query-param interface the list method uses

   @Injectable({ providedIn: 'root' })
   export class <Entity>ListPresenter extends ListBasePresenter {
     constructor(private talos: TalosService) {
       super();
     }

     // Signature matches ListBasePresenter.getListItems. <talos-list> injects
     // rangeFrom/rangeTo into queryParams automatically before this is called.
     override async getListItems(
       queryParams: any,
       body: any,
       pathParams?: { [id: string]: string },
     ): Promise<void> {
       const result = await this.talos.<talosProp>
         .list<Entity>(queryParams)          // ← the real list method from the .d.ts
         .catch(() => undefined);
       this.itemObservable.next(result ?? []);   // push [] on failure so the empty state shows
     }
   }
   ```

   - If the API list method is **user-scoped** (takes a `userId`), read it from `pathParams['userId']` and bail early if absent — mirror the built-in `TalosQuizListPresenter` pattern:
     ```ts
     const userId = pathParams?.['userId'];
     if (!userId) return;
     const result = await this.talos.<talosProp>.list<Entity>WithUserProgress(userId, queryParams).catch(() => undefined);
     ```
   - **Paging & total count:** to enable load-more / the pager, the response must carry the item-count header so `ListBaseComponent` can compute `totalCount`. If the API returns it, pass the raw array through unchanged (the header rides along on the array object). Do not strip it.

Do not invent presenter classes or API methods — only use what the `.d.ts` files actually declare.

## Step 3 — Resolve the item DTO

Find the item type the list renders so the card template binds to real fields:

1. From the presenter's list method return type (Step 2), note the DTO, e.g. `Promise<<Entity>DTO[]>`.
2. Read the DTO under `node_modules/@saicongames/talos-api/dist/types/models/<Entity>DTO.d.ts` (or wherever the API `.d.ts` imports it from) and note a couple of obvious display fields (name/title/id) and the natural identity field (`id`, `code`, `tournamentId`, …). That identity field becomes `trackByKey` in Step 4.

If the DTO isn't exported from the top-level `@saicongames/talos-api`, fall back to the deep import path shown in the API `.d.ts`.

## Step 4 — Generate the host component + module

`src/app/<entity>/<entity>.component.ts` — inject the presenter (built-in or generated) and expose any params `<talos-list>` needs. **Do not** fetch data here — `<talos-list>` does that via the presenter.

```ts
import { Component } from '@angular/core';
import { TAValues } from '@saicongames/talos-api';
import { <Presenter> } from '@saicongames/talos-integration-angular'; // built-in
// — or, for a generated presenter:
// import { <Entity>ListPresenter } from './<entity>-list.presenter';

@Component({
  selector: 'app-<entity>',
  templateUrl: './<entity>.component.html',
  styleUrls: ['./<entity>.component.scss'],
  standalone: false,
})
export class <Entity>Component {
  // Query params forwarded to the presenter. rangeFrom/rangeTo are injected by
  // <talos-list> automatically — only add real API query params here.
  queryParams: any = {};

  // Only when the presenter is user-scoped (Step 2):
  pathParams = { userId: TAValues.UserId! };

  constructor(public <entity>Presenter: <Presenter>) {}
}
```

`src/app/<entity>/<entity>.component.html` — the payoff. Bind the presenter and provide the `#defaultItem` card template:

```html
<talos-list
  [presenter]="<entity>Presenter"
  [queryParams]="queryParams"
  [paging]="10"
  trackByKey="<idField>">
  <!-- add [pathParams]="pathParams" only for a user-scoped presenter -->

  <ng-template #defaultItem let-item let-i="index">
    <!-- Step 5 fills this in from the mockup or the default card -->
    <div class="<entity>-card">
      <strong>{{ item.name || item.title || item.id }}</strong>
    </div>
  </ng-template>
</talos-list>
```

`src/app/<entity>/<entity>.module.ts` — must import `TalosFeatureModule` (that is what declares `<talos-list>`). Declare the generated presenter here only if you generated one; built-in presenters are `providedIn: 'root'` and need no declaration.

```ts
import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FormsModule } from '@angular/forms';
import { IonicModule } from '@ionic/angular'; // omit if the host app is not Ionic
import { TalosFeatureModule } from '@saicongames/talos-integration-angular';
import { <Entity>Component } from './<entity>.component';

@NgModule({
  imports: [CommonModule, FormsModule, IonicModule, TalosFeatureModule],
  declarations: [<Entity>Component],
  exports: [<Entity>Component],
})
export class <Entity>Module {}
```

## Step 5 — The card template + layout (`#defaultItem` + `.scss`)

Pick the first that applies for the **card** (`#defaultItem` body):

1. **Mockup was provided** (Step 1): build the card to match it, binding to the real DTO fields from Step 3. If it's a file path, read it first (the Read tool renders images); if inline, follow the description.
2. **No mockup, but a reference card/component exists** (user names one, or a similar `*-card.component.html` exists): mirror its structure and SCSS.
3. **Neither:** a minimal card rendering 1–2 obvious fields.

Then choose the **layout** on `<talos-list>` from the mockup (defaults to a stacked flex column):
- **Grid** — add `[columns]="N"` for an N-column card grid.
- **Slider** — add `slider` (optionally `[sliderDots]="true"`) for a horizontal carousel.
- **Pager instead of load-more** — add `[pagerEnabled]="true"`.

Available projected templates on `<talos-list>` (use only what the mockup calls for): `#defaultItem` (required), `#empty`, `#loader`, `#loadMoreButton`, `#pager`. Available inputs include `presenter`, `queryParams`, `pathParams`, `body`, `inputParams`, `paging`, `pager`, `pagerEnabled`, `loadMoreEnabled`, `hideLoadMore`, `forceRangeTo`, `columns`, `slider`, `sliderDots`, `emptyText`, `trackByKey`. Outputs: `loadMore`, `finished`, `itemUpdated`, `itemRemoved`.

Example — 3-column grid card, no mockup, quiz entity (illustrative):

```html
<talos-list [presenter]="quizPresenter" [queryParams]="queryParams" [columns]="3" [paging]="9" trackByKey="id">
  <ng-template #defaultItem let-item let-i="index">
    <div class="quiz-card">
      <strong>{{ item.title }}</strong>
      <span>{{ item.questionCount }} questions</span>
    </div>
  </ng-template>
  <ng-template #empty><p class="empty">No quizzes yet.</p></ng-template>
</talos-list>
```

## Step 6 (optional) — Imperative CRUD via `@ViewChild`

Offer this only if the user wants create/edit/delete on the list. `<talos-list>` exposes an imperative API so the parent can mutate the rendered list without a re-fetch:

```ts
import { Component, ViewChild } from '@angular/core';
import { TalosListComponent, TalosService } from '@saicongames/talos-integration-angular';

export class <Entity>Component {
  @ViewChild(TalosListComponent) list!: TalosListComponent;
  constructor(private talos: TalosService, public <entity>Presenter: <Presenter>) {}

  async onCreate(input: any): Promise<void> {
    const created = await this.talos.<talosProp>.create<Entity>(input);
    this.list.prependItem(created);   // insert at top
  }
  async onEdit(item: <Entity>DTO): Promise<void> {
    const updated = await this.talos.<talosProp>.update<Entity>(item.id!, item);
    this.list.updateItem(updated);    // replace by trackByKey
  }
  async onDelete(item: <Entity>DTO): Promise<void> {
    await this.talos.<talosProp>.delete<Entity>(item.id!);
    this.list.removeItem(item);       // or this.list.removeItem(item.id!)
  }
  refresh(): void { this.list.reload(); }   // re-fetch from page 1
}
```

`updateItem`/`removeItem` match by `trackByKey`, not object reference — set `trackByKey` to the DTO's real identity field (Step 3).

## Finish

- Confirm `TalosModule.forRoot(...)` is wired in the host app's `src/app/app.module.ts` (so `TalosService` and the presenters are injectable). If not, tell the user it's required and point at `/talos-init`.
- Confirm the feature module imports `TalosFeatureModule` — without it `<talos-list>` won't resolve (`'talos-list' is not a known element`).
- Print (do not apply) the routing snippet, e.g.:
  ```ts
  // add a child route:
  { path: '<entity>', loadChildren: () => import('../<entity>/<entity>.module').then(m => m.<Entity>Module) }
  ```
- Summarize what was created (paths), whether a built-in or generated presenter was used, and any DTO fields / identity key you guessed so the user can correct them.
- Suggest `npm run build` (or `npm run lint`) to verify the generated files compile.
