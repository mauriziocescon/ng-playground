# Anatomy of signal components
**Note: just personal thoughts from a DEV perspective on [the future of angular](https://myconf.dev/videos/2024-keynote-session-the-future-of-angular) (template level). Strongly inspired by svelte / solid.**

Points:
1. building blocks as functions:
    - `component`: a treble `script` / `template` / `style`,
    - `fragment`: a duo `template` / `style` that captures some markup for reusability,
    - `directive`: a `script` that can change the appearance or behavior of DOM elements.
2. ts expressions with `{}`: bindings + text interpolation,
3. extra bindings for DOM elements: `class:`, `style:`, `attr:`, `bind:`, `use:`, `on:`,
4. hostless components + ts lexical scoping for templates,
5. component inputs: lifted up + immediately available + spread,
6. composition with fragments and directives,
7. template ref, 
8. lifecycle hooks: after** + onDestroy,
9. DI enhancements. 

## Components
Component structure and element bindings:
```ts
import { component, signal, linkedSignal, input, output } from '@angular/core';

export const TextSearch = component(({
  value = input.required<string>(),
  valueChange = output<string>(),
}) => ({
  script: () => {
    const text = linkedSignal(() => value());
    const isDanger = signal(false);

    function textChange() {
      valueChange.emit(text());
    }

    // exposed as public: inject(TextSearch), ...
    // the rest is private
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

export const TextSearch = component(({
  value = input.required<string>(),
  valueChange = output<string>(),
}) => ({
  script: () => { /** ... **/ },
  templateUrl: `./text-search.html`,
  styleUrl: `./text-search.css`,
}));
```

Component bindings:
```ts
import { component, signal, computed } from '@angular/core';

import { UserDetail, User } from './user-detail';

export const UserDetailConsumer = component(() => ({
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

export const UserDetail = component(({
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

export const TextSearch = component(() => ({
  script: () => {
    const text = signal('');
    const valid = signal(false);

    function doSomething() {
      // ...
    }
  },
  template: `
    <!-- ... -->

    <!-- grouping of directive data (if any):
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

export const tooltip = directive(({
  message = input.required<string>(),
  valid = model<boolean>(),
  dismiss = output<void>(),
}) => ({
  script: () => {
    const elRef = inject(ElementRef);
    // ...

    // exposed as public
    return {
      // ...
    };
  },
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

export const Counter = component(({ 
  c = input<number>(), 
}) => ({
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

## Composition with fragments and directives
Fragments are very similar to svelte [`snippets`](https://svelte.dev/docs/svelte/snippet).

Implicit children fragment (where + when):
```ts
import { component, inject, provide } from '@angular/core';

import { Card, Header, Content, Footer } from '@mylib/card';

import { Item } from '.../model/item';

class ItemStore { /** ... **/ }

export const Item = component(({
  item = input.required<Item>(), 
}) => ({
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
    <!-- ... -->

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

Customising components and binding context:
```ts
import { component, inject, provide, input } from '@angular/core';

import { Card } from '@mylib/card';

import { MyHeader, MyContent, MyFooter } from './my-comp';
import { Item } from '../model/item';

class ItemStore { /** ... **/ }

export const Item = component(({ 
  item = input.required<Item>(), 
}) => ({
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

export const TreeConsumer = component(({
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

export const Tree = component(({
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

export const TextSearchConsumer = component(() => ({
  script: () => {
    const tooltipMsg = signal('');
  },
  template: `
    <!-- ... -->

    <TextSearch withTooltip={ tooltip } tooltipMsg={ tooltipMsg() } />`,
}));

// -- TextSearch -----------------------------------
import { component, signal, DirProps } from '@angular/core';

export const TextSearch = component(({
  withTooltip = input<DirProps<{ message: string }>>(),
  tooltipMsg = input<string>(),
}) => ({
  script: () => {
    const tooltip = withTooltip();
  },
  template: `
    <!-- ... -->

    <!-- tooltip = undefined => not applied -->
    <div use:tooltip(message={ tooltipMsg() })> Something </div>`,
}));
```

Dynamic components:
```ts
import { component, signal, computed } from '@angular/core';
import { Dynamic } from '@angular/common';

import { AComp } from './a-comp';
import { BComp } from './b-comp';

export const Something = component(() => ({
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

## Template ref (difficult point)
Retrieving references of elements / components / directives:
```ts
import { component, ref, signal, Signal} from '@angular/core';

import { tooltip } from '@mylib/tooltip';

const Child = component(() => ({
  script: () => {
    const text = signal('');
    // ...

    // exposed as public
    return {
      text: text.asReadonly(),
    };
  },
  template: `
    <!-- ... -->`,
}));

export const Parent = component(() => ({
  script: () => {
    const value = signal('');

    // signal
    const elRef = ref<ElementRef<HTMLDivElement>>('el');

    // can only use what's returned by Child.script
    const childRef1 = ref<{ text: Signal<string> }>('c');
    const childRef2 = ref(Child);
    const childRef3 = ref<{ text: Signal<string> }[]>(Child, { any: true });
  },
  template: `
    <div 
      ref:this="el" 
      use:tooltip(message={ value() } ref:this="tlp")>
        Something
    </div>

    <Child ref:this="c" />

    <Child />`,
}));
```
Retrieving content dynamically:
```ts
import { component } from '@angular/core';

import { Menu, MenuItem } from '@mylib/menu';

export const MenuConsumer = component(() => ({
  script: () => { /** ... **/ },
  template: `
    <!-- ... -->
    <Menu>
      <MenuItem>First</MenuItem>
      <MenuItem>Second</MenuItem>
    </Menu>`,
}));

// -- Menu in @mylib/menu --------------------------
import { component, ref, input, Fragment, computed } from '@angular/core';

export const Menu = component(({
  children = input.required<Fragment<void>>(), 
}) => ({
  script: () => {
    const menuItems = ref(MenuItem, { any: true });
    const numOfItems = computed(() => menuItems()?.length ?? 0);
    // ...
  },
  template: `
    <!-- ... -->
    <Render fragment={ children() } />
    <!-- ... ->`,
}));

export const MenuItem = component(({
  children = input.required<Fragment<void>>(), 
}) => ({
  script: () => { /** ... **/ },
  template: `
    <!-- ... -->
    <Render fragment={ children() } />
    <!-- ... -->`,
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

export const Counter = component(({ 
  initialValue = input<number>(), 
}) => ({
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

## Backward compatibility
Still can use the legacy concepts for composition: 
```ts
import { component, input } from '@angular/core';

import { MatButton } from '@angular/material/button';
import { MatTooltip } from '@angular/material/tooltip';

export const AdminLinkWithTooltip = component(({
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
- `queries`: might not be needed anymore, but it's difficult to say because `ref` could be completely wrong. In general, it would be nice to better encapsulate the retrieval of data (`read` anything from `injector` tree),
- directives attached to the host: not possible anymore; there might be some cases where
the concept of the host makes sense (debatable).
