# Anatomy of signal components
**Note: just personal thoughts from a DEV perspective on [the future of angular](https://myconf.dev/videos/2024-keynote-session-the-future-of-angular). Strongly inspired by svelte / solid (without JSX 😅)**

Points:
1. components as functions:
    - `component`: a treble `script` / `template` / `style` where you can read / write state
    and capture user interaction,
    - `fragment`: a duo `template` / `style` that caputures some markup for reusability,
    - `directive`: an action that can change the appearance or behavior of DOM elements.
2. hostless components + ts lexical scoping for templates,
3. `{}` ts/ng expressions: bindings + text interpolation,
4. component inputs: lifted up + immediately available + spread,
5. new bindings for native elements: `class:`, `style:`, `attr:`, `bind:`, `use:`, `on:`,
6. composition with fragments and directives,
7. lifecycle hooks: after** + onDestroy,
8. DI enhancements,
9. concepts affected by these changes.

## Components
Component structure and element bindings:
```ts
import { component, signal, linkedSignal, input, output } from '@angular/core';

const TextSearch = component(({
  value = input.required<string>(),
  valueChange = output<string>(),
}) => ({
  script: () => {
    const text = linkedSignal(() => value());
    const isDanger = signal(false);

    function textChange() {
      valueChange.emit(text());
    }

    // exposed as public; the rest is private
    return {
      text: text.asReadonly(),
    };
  },
  template: `
    <label class:danger={ isDanger() }>Text:</label>

    <!-- 2way binding for input:
         bind:property={ var } on:propertyChange={ varChange() } -->

    <input type="text" bind:value={ text } on:valueChange={ textChange() } />

    <button disabled={ text().length === 0 } on:click={ text.set('') }>
      { `Reset ${text()}` }
    </button>`,
  style: `
    .danger {
      color: red;
    }`,
}));
```

External files:
```ts
import { component, input, output } from '@angular/core';

const TextSearch = component(({
  value = input.required<string>(),
  valueChange = output<string>(),
}) => ({
  script: () => { /** ... **/ },
  templateUrl: `./text-search.html`,
  styleUrl: `./text-search.css`,
}));
```

## Inputs
Inputs lifted up for providers init:
```ts
import { component, linkedSignal, input, WritableSignal, provide, inject } from '@angular/core';

class CounterStore {
  private readonly counter: WritableSignal<number>;
  readonly value = this.counter.asReadonly();

  constructor(c: Signal<number>) {
    this.counter = linkedSignal(() => c());
  }

  decrease() {
    this.counter.update(v => v - 1);
  }

  increase() {
    this.counter.update(v => v + 1);
  }
}

const Counter = component(({ c = input<number>() }) => ({
  providers: [
    provide({ token: CounterStore, useFactory: () => new CounterStore(c) }),
  ],
  script: () => {
    const store = inject(CounterStore);
  },
  template: `
    <h1>Counter</h1>
    <div>Value: { store.value() }</div>
    <button on:click={ store.decrease() }> - </button>
    <button on:click={ store.increase() }> + </button>`,
}));
```

Component bindings:
```ts
import { component, signal, computed } from '@angular/core';

import { UserDetail, User } from './user-detail';

const UserDetailConsumer = component(() => ({
  script: () => {
    const user = signal<User>(...);
    const another = signal<string>(...);
    const isValid = signal<boolean>(...);

    const inputs = computed(() => ({
      user: this.user(),
      another: this.another(),
    }));

    function isValidChange() { /** ... **/ }

    function makeAdmin() { /** ... **/ }
  },
  template: `
    <!-- spread inputs -->

    <UserDetail
      { ...inputs() }
      bind:valid={ isValid }
      on:validChange={ isValidChange() }
      on:makeAdmin={ makeAdmin() } />`,
}));

// -- UserDetail -----------------------------------
import { component, input, model, output } from '@angular/core';

export interface User { /** ... **/ }

export UserDetail = component(({
  user = input<User>(),
  another = input<string>(),
  valid = model<boolean>(),
  makeAdmin = output<boolean>(),
}) => ({
  // ...
}));
```

## Element directives

Change the appearance or behavior of DOM elements:
```ts
import { component, signal } from '@angular/core';

import { tooltip } from '@mylib/tooltip';

const TextSearch = component(() => ({
  script: () => {
    const text = signal('');
    const valid = signal(false);

    function doSomething() {
      // ...
    }
  },
  template: `
    <!-- ...-->

    <!-- simple grouping of props (if any):
         use:directive(
           input={ var }
           bind:model={ var }
           on:modelChange={ varChange() }
           on:output={ method() }) -->

    <div use:tooltip(
      message={ text() }
      bind:valid={ valid }
      on:dismiss={ doSomething() }
    )>
      Value: { text() }
    </div>`,
}));

// -- tooltip in @mylib/tooltip --------------------
import { directive, input, model, output, inject, ElementRef } from '@angular/core';

const tooltip = directive(({
  message = input.required<string>(),
  valid = model<boolean>(),
  dismiss = output<void>(),
}) => ({
  script: () => {
    const elRef = inject(ElementRef);

    // ...
  },
}));
```

## Composition with fragments and directives
Fragments are very similar to svelte [`snippets`](https://svelte.dev/docs/svelte/snippet) (or `ng-template`).

Default children fragment (where + when):
```ts
import { component, inject, provide } from '@angular/core';

import { Card, Header, Content, Footer } from '@mylib/card';

import { Item } from '.../model/item';

class ItemStore { /** ... **/ }

const Item = component(() => ({
  providers: [
    provide({ token: ItemStore, useFactory: () => new ItemStore(item) }),
  ],
  script: () => {
    const store = inject(ItemStore);
  },
  template: `
    <Card>
      <Header>{ store.title() }</Header>
      <Content>{ store.content() }</Content>
      <Footer>{ store.footer() }</Footer>
    </Card>`,
}));

// -- Card in @mylib/card --------------------------
import { component, input, Fragment } from '@angular/core';
import { Render } from '@angular/common';

export const Card = component(({
  children = input<Fragment<void>>(),
  header = input<Fragment<void>>(),
  content = input<Fragment<void>>(),
  footer = input<Fragment<void>>(),
}) => ({
  template: `
    <!-- ...-->

    @if (children()) {
      <div class="my-lib-card">

        <!--  similar to NgTemplateOutlet: no need
              to have an anchor point like ng-container -->

        <Render fragment={ children() } />

      </div>
    } @else {
       <div class="my-lib-card"> Default </div>
    }`,
}));
```

Customising components and capturing context:
```ts
import { component, inject, provide, input } from '@angular/core';

import { Card } from '@mylib/card';

import { MyHeader, MyContent, MyFooter } from './my-comp';
import { Item } from '../model/item';

class ItemStore { /** ... **/ }

const Item = component(({ item = input.required<Item>() }) => ({
  providers: [
    provide({ token: ItemStore, useFactory: () => new ItemStore(item) }),
  ],
  script: () => {
    const store = inject(ItemStore);
  },
  template: `
    <!-- fragments defined within component templates -->

    @fragment header() {
      <MyHeader>{ store.title() }</MyHeader>
    }

    @fragment content() {
      <MyContent>
        <span>{ store.content() }</span>
        <span>{ store.hasExtraContent() ? store.extraContent() : `` }</span>
      </MyContent>
    }

    @fragment footer() {
      <MyFooter>{ store.footer() }</MyFooter>
    }

    <!-- can omit input names if they match -->

    <Card { header } { content } { footer } />`,
}));

// -- Card in @mylib/card --------------------------
// see previous example
```

Reusable fragments:
```ts
import { component, input, fragment } from '@angular/core';

import { Tree, Node } from '@mylib/tree';

interface CustomNode extends Node {
    desc: string;
}

/**
* Reusable fragment: can read state in the template,
* but cannot set it!
*
* Note: no inputs / outputs / injection / ...
*/
const myNode = fragment((node: CustomNode) => ({
  template: `
    <div class="my-node">
      <span>Custom node: </span>
      <span class:red={ node.desc === 'unknown' }>{ node.desc }</span>
    </div>`,
  style: `
    .red {
      color: red;
    }`,
}));

const TreeConsumer = component(({
  nodes = input.required<CustomNode[]>(),
  custom = input.required<boolean>(),
}) => ({
  template: `
    @if (custom()) {
      <Tree nodes={ nodes() } custumNode={ myNode } />
    } @else {
      <Tree nodes={ nodes() } />
    }`,
}));

// -- tree in @mylib/tree --------------------------
import { component, input, Fragment } from '@angular/core';
import { Render } from '@angular/common';

export interface Node { /** ... **/ }

const Tree = component(({
  nodes = input.required<Node[]>(),
  custumNode = input<Fragment<[Node]>>(),
}) => ({
  template: `
    <!-- ... -->

    @for (node of nodes(); track node.id) {
      @if (custumNode()) {
        <Render fragment={ custumNode() } params={ [node] } />
      } @else {
        <!-- ... -->
      }
    }`,
}));
```

Directives passed as inputs and attached dynamically:
```ts
import { component, signal } from '@angular/core';

import { tooltip } from '@mylib/tooltip';

import { TextSearch } from './text-search';

const TextSearchConsumer = component(() => ({
  script: () => {
    const tooltipMsg = signal('');
  },
  template: `
    <!-- ... -->

    <TextSearch withTooltip={ tooltip } tooltipMsg={ tooltipMsg() } />`,
}));

// -- TextSearch -----------------------------------
import { component, signal, Dir } from '@angular/core';

const TextSearch = component(({
  withTooltip = input<Dir<{ message: string }>>(),
  tooltipMsg = input<string>(),
}) => ({
  script: () => {
    const tooltip = withTooltip();
  },
  template: `
    <!-- ...-->

    <div use:tooltip(message={ tooltipMsg() })> Something </div>`,
}));
```

Dynamic components:
```ts
import { component, signal, computed } from '@angular/core';
import { Dynamic } from '@angular/common';

import { AComp } from './a-comp';
import { BComp } from './b-comp';

const Something = component(() => ({
  script: () => {
    const condition = signal<boolean>(/** ... **/);
    const comp = computed(() => condition() ? AComp : BComp);
    const inputs = computed(() => /** ... **/);
  },
  template: `
    <!-- ... -->

    <Dynamic component={ comp() } inputs={ inputs() } />`,
}));
```

Template ref variables (difficult point):
```ts
import { component, ref, refs } from '@angular/core';

const AComp = component(() => ({
  script: () => {
    // ...
    // exposed as public; the rest is private
    return {
      // ...
    };
  },
  template: `
    <!-- ... -->`,
}));

const Something = component(() => ({
  script: () => {
    // signal
    const elRef = ref<ElementRef<HTMLDivElement>>('el');

    // can only call what's returned by AComp.script
    const aCompRef1 = ref<AComp>('aComp');
    const aCompRef2 = ref(AComp);

    const aCompRefs = refs<AComp[]>(AComp);
  },
  template: `
    <div ref:this="el"></div>

    <AComp ref:this="aComp" />

    <AComp />`,
}));
```

## DI enhancements
Better ergonomics around types / tokens:
```ts
import { component, inject, provide, provideForRoot, injectionToken, input } from '@angular/core';

/**
* define a default implementation (no need for an explicit interface)
* the token must be provided somewhere
*/
const compToken = injectionToken('desc', {
   factory: (initialValue?: Signal<number>) => {
     const counter = signal(initialValue ? initialValue() : 0);

     return {
       value: counter.asReadonly(),
       decrease: () => counter.update(v => v - 1),
       increase: () => counter.update(v => v + 1),
     };
   },
});

/**
* root provider (similar for platform)
*/
const rootToken = provideForRoot('desc', {
   factory: (initialValue?: Signal<number>) => {
     const counter = signal(initialValue ? initialValue() : 0);

     return {
       value: counter.asReadonly(),
       decrease: () => counter.update(v => v - 1),
       increase: () => counter.update(v => v + 1),
     };
   },
});

const Counter = component(({ initialValue = input<number>() }) => ({
  providers: [
    provide({ token: compToken, useFactory: () => compToken.factory(initialValue) }),
  ],
  script: () => {
    const rootCounter = inject(rootToken);
    const compCounter = inject(compToken);
    // ...
  },
  // ...
}));
```

## Backward compatible
Managing legacy implementations with `ng-content`, `ng-template`, ...and having a host:
```ts
import { component, input } from '@angular/core';

import { MatButton } from '@angular/material/button';
import { MatTooltip } from '@angular/material/tooltip';

const AdminLinkWithTooltip = component(({
  tooltipMessage = input.required<string>(),
  hasPermissions = input.required<boolean>(),
}) => ({
  template: `
    <MatButton:a
      href="/admin"
      use:MatTooltip(message={ tooltipMessage() } disabled={ hasPermissions() })>
        Admin
    </MatButton:a>`,
}));
```

## Concepts affected by these changes

- `ng-content`: replaced by `fragments`,
- `ng-template` (`let-*` shorthands + `ngTemplateGuard_*`): likely replaced by `fragments`,
- structural directives: likely replaced by `fragments`,
- `Ng**Outlet` + `ng-container`: likely not needed anymore cause components are hostless,
- `queries`: might not be needed anymore, but it's difficult to say because `ref` might be completely wrong. In general, it would be nice to better encapsulate the retrieval of data (`read` any provider),
- directives attached to the host: not possible anymore; there might be some cases where
the concept of the host makes sense (debatable).
