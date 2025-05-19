# Anatomy of signal components
**Note: just personal thoughts from a DEV perspective on [the future of angular](https://myconf.dev/videos/2024-keynote-session-the-future-of-angular) (template level). Strongly inspired by svelte / solid.**

Points:
1. building blocks as functions:
    - `component`: a treble `script` / `template` / `style`,
    - `fragment`: a duo `template` / `style` that captures some markup for reusability,
    - `directive`: a `script` that can change the appearance or behavior of DOM elements.
2. ts expressions with `{}`: bindings + text interpolation,
3. extra bindings for DOM elements: `class:`, `style:`, `attr:`, `bind:`, `on:`,
4. hostless components + ts lexical scoping for templates,
5. component inputs: lifted up + immediately available,
6. composition with fragments and directives,
7. template ref,
8. lifecycle hooks: after** + onDestroy,
9. DI enhancements.

## Components
Component structure and element bindings:
```ts
import { component, signal, linkedSignal, input, output } from '@angular/core';

export const TextSearch = component(({
  text: value = input.required<string>(), // types + default + rename (alias)
  valueChange = output<string>(),
}) => ({
  script: () => {
    const text = linkedSignal(() => value());
    const isDanger = signal(false);

    function textChange() {
      valueChange.emit(text());
    }

    // exposed as public: inject(TextSearch), ...
    return {
      text: text.asReadonly(),
    };
  },
  template: `
    <!-- 2way binding for input:
         bind:property={ var } on:propertyChange={ varChange() } -->

    <label class:danger={ isDanger() }>Text:</label>
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

Props and external files:
```ts
import { component, InputSignal, OutputRef, propsMap, booleanAttribute } from '@angular/core';

interface CheckboxProps {
  value: InputSignal<any>;
  valueChange?: OutputRef<boolean>;
}

export const Checkbox = component((props: CheckboxProps) => ({
  script: () => {
    const { value, valueChange } = propsMap(props, {
      value: { transform: booleanAttribute },
    });
  },
  templateUrl: `./checkbox.html`,
  styleUrl: `./checkbox.css`,
}));
```

Component bindings:
```ts
import { component, signal } from '@angular/core';
import { UserDetail, User } from './user-detail';

export const UserDetailConsumer = component(() => ({
  script: () => {
    const user = signal<User>(...);
    const isValid = signal<boolean>(...);

    function isValidChange() { /** ... **/ }
    function makeAdmin() { /** ... **/ }
  },
  template: `
    <UserDetail
      user={ user() }
      bind:valid={ isValid }
      on:validChange={ isValidChange() }
      on:makeAdmin={ makeAdmin() } />`,
}));

// -- UserDetail -----------------------------------
import { component, input, model, output } from '@angular/core';

export interface User { /** ... **/ }

export const UserDetail = component(({
  user = input<User>(),
  valid = model<boolean>(),
  makeAdmin = output<void>(),
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

    function doSomething() { /** ... **/ }
  },
  template: `
    <!-- ... -->

    <!-- encapsulation of directive data: @directive( ... ) -->

    <div @tooltip(
      message={ text() }
      bind:valid={ valid }
      on:dismiss={ doSomething() }
    )>
      Value: { text() }
    </div>`,
}));

// -- tooltip in @mylib/tooltip --------------------
import { directive, input, model, output, inject, ElementRef, Renderer2 } from '@angular/core';

export const tooltip = directive(({
  message = input.required<string>(),
  valid = model<boolean>(),
  dismiss = output<void>(),
}) => ({
  script: () => {
    const elRef = inject(ElementRef);
    const renderer = inject(Renderer2);
    // ...

    // exposed as public
    return { /** ... **/ };
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

  decrease() { /** ... **/ }
  increase() { /** ... **/ }
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
Fragments are very similar to [`svelte snippets`](https://svelte.dev/docs/svelte/snippet): functions returning html markup. Returned markup is opaque: cannot manipulate it similarly to [`react Children (legacy)`](https://react.dev/reference/react/Children) and [`solid children`](https://www.solidjs.com/tutorial/props_children).

Implicit children fragment (where + when) and binding context:
```ts
import { component } from '@angular/core';
import { Menu, MenuItem } from '@mylib/menu';

export const MenuConsumer = component(() => ({
  script: () => { /** ... **/ },
  template: `
    <Menu>
      <MenuItem> First </MenuItem>
      <MenuItem> Second </MenuItem>
    </Menu>`,
}));

// -- Menu in @mylib/menu --------------------------
import { component, input, Fragment } from '@angular/core';
import { Render } from '@angular/common';

export const Menu = component(({
  children = input<Fragment<void>>(),
}) => ({
  script: () => { /** ... **/ },
  template: `
    <!--  no need to have an explicit anchor point like ng-container -->

    @if (children()) {
      <Render fragment={ children() } />
    } @else {
       <!-- ... -->
    }`,
}));

export const MenuItem = component(({
  children = input.required<Fragment<void>>(),
}) => ({
  script: () => { /** ... **/ },
  template: `
    <Render fragment={ children() } />`,
}));
```

Customising components:
```ts
import { component, computed } from '@angular/core';
import { Menu } from '@mylib/menu';
import { MyMenuItem } from './my-menu-item';

export const MenuConsumer = component(() => ({
  script: () => {
    const items = computed(() => [{ id: '1', desc: 'First' }, { id: '2', desc: 'Second' }]);
  },
  template: `
    @fragment menuItem(item: { id: string, desc: string }) {
      <MyMenuItem>{ item.desc }</MyHeader>
    }

    <Menu items={ items() } menuItem={ menuItem } />`,
}));

// -- Menu in @mylib/menu --------------------------
import { component, input, Fragment } from '@angular/core';
import { Render } from '@angular/common';

export const Menu = component(({
  items = input.required<{ id: string, desc: string }[]>(),
  menuItem = input.required<Fragment<[{ id: string, desc: string }]>>(),
}) => ({
  script: () => { /** ... **/ },
  template: `
    <h1> Total items: { items().length } </h1>

    @for (item of items(); track item.id) {
      <Render fragment={ children() } params={ [item] } />
    }`,
}));
```

Reusable fragments:
```ts
import { component, input, fragment } from '@angular/core';
import { Tree, Node } from '@mylib/tree';

interface CustomNode extends Node { /** ... **/ }

/**
 * Reusable fragment: can read state in the template,
 * but cannot set it! Importable from another file.
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
}) => ({
  template: `
    <Tree nodes={ nodes() } custumNode={ myNode } />`,
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

Directives passed as inputs and bound to an element using `bind:this={ props }` (react props spread equivalent):
```ts
import { component, signal } from '@angular/core';

import { Button } from '@mylib/button';
import { tooltip } from '@mylib/tooltip';

export const ButtonConsumer = component(() => ({
  script: () => {
    const tooltipMsg = signal('');
    const valid = signal(false);

    function doSomething() { /** ... **/ }
  },
  template: `
    <Button
      @tooltip(message={ tooltipMsg() })
      disabled={ !valid() }
      on:click={ doSomething() }>
        Click / Hover me
    </Button>`,
}));

// -- button in @mylib/button --------------------
import { component, InputSignal, OutputRef, ModelSignal } from '@angular/core';
import { Render } from '@angular/common';

interface ButtonProps {
  children: InputSignal<Fragment<void>>;
  tooltip?: DirProps<{ message: string }>; // Note: DirProps is just an idea
  disabled?: ModelSignal<boolean>;
  click?: OutputRef<void>;
}

export const Button = component((props: ButtonProps) => ({
  script: () => {
    const { children, ...others } = propsMap(props, {
      children: { required: true },
    });
  },
  template: `
    <!-- bind others to button, directives included -->

    <button bind:this={ others }>
      <Render fragment={ children() } />
    </button>`,
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

## Template ref
Retrieving references of elements / components / directives (runtime):
```ts
import { component, ref, Signal } from '@angular/core';
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
  template: `<!-- ... -->`,
}));

export const Parent = component(() => ({
  script: () => {
    // readonly signal
    const el = ref<ElementRef<HTMLDivElement>>('el');

    // 1. can only use what's returned by Child.script
    // 2. templates only lookup: cannot retrieve providers
    //    defined in the Child comp tree
    const manyComp = ref<{ text: Signal<string> }[]>('manyComp', { any: true });
    const manyComp2 = ref(Child, { any: true });

    const tlp = ref<{ toggle: () => void }>('tlp');
    const tlp2 = ref(tooltip);
  },
  template: `
    <div
      ref:this="el"
      @tooltip(message={ 'something' } ref:this="tlp")>
        Something
    </div>

    <Child ref:many="manyComp" />
    <Child ref:many="manyComp" />

    <button on:click={ tlp().toggle() }> Toggle tlp </button>`,
}));
```

## DI enhancements
See [`DI enhancements`](https://github.com/mauriziocescon/ng-playground/blob/main/di.md)

## Backward compatibility
Still can use legacy concepts for composition:
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
      @MatTooltip(message={ tooltipMessage() } disabled={ hasPermissions() })>
        Admin
    </MatButton:a>`,
}));
```

## Concepts affected by these changes

- `ng-content`: replaced by `fragments`,
- `ng-template` (`let-*` shorthands + `ngTemplateGuard_*`): likely replaced by `fragments`,
- structural directives: likely replaced by `fragments`,
- `Ng**Outlet` + `ng-container`: likely replaced by the new things,
- `queries`: if `ref` makes sense, likely not needed anymore; if they stay, it would be nice to improve the retrieval of data: no way to `read` anything from `injector` tree,
- multiple `directives` applied to the same element: as for the previous point, no way for a directive to inject other ones applied to the same element (see [`ngModel hijacking`](https://stackblitz.com/edit/stackblitz-starters-ezryrmmy));
if needed, it should be an explicit operation with a `ref` passed as an `input`,
- `directives` attached to the host (components): not possible anymore, but you can pass directives as inputs and use `bind:this={ props }`,
- parent component styling children (difficult point): this should probably be based on css-variables similarly to [`svelte`](https://svelte.dev/docs/svelte/custom-properties).
