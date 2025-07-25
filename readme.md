# Anatomy of signal components
**Note: just personal thoughts from a DEV perspective on [the future of angular](https://myconf.dev/videos/2024-keynote-session-the-future-of-angular) (template level).**

Points:
1. building blocks as functions:
    - `component`: a treble `script` / `template` / `style`,
    - `fragment`: a duo `template` / `style` that captures some markup for reusability,
    - `directive`: a `script` that can change the appearance or behavior of DOM elements,
    - `derivation`: a `script` for creating computeds in templates,
2. ts expressions with `{}`: bindings + text interpolation,
3. extra bindings for DOM elements: `class:`, `style:`, `attr:`, `bind:`, `on:`, `animate:`,
4. hostless components + ts lexical scoping for templates,
5. component inputs: lifted up + immediately available in the script,
6. composition with fragments and directives,
7. template ref,
8. DI enhancements,
9. Concepts affected by these changes.

## Components
Component structure and element bindings:
```ts
import { component, signal, linkedSignal, input, output } from '@angular/core';

/**
 * value / valueChange are always created
 * (no real "props decostruction")
 *
 * by the time script is called,
 * inputs are populated with parent data
 */
export const TextSearch = component(({
  value = input.required<string>(), // definitions + types
  valueChange = output<string>(),
}) => ({
  script: () => {
    const text = linkedSignal(() => value());
    const isDanger = signal(false);

    function textChange() {
      valueChange.emit(text());
    }

    // exposed as public interface
    return {
      text: text.asReadonly(),
    };
  },
  template: `
    <!-- 2way binding for input / select / textarea: bind:property={var} -->

    <label class:danger={isDanger()}> Text: </label>
    <input type="text" bind:value={text} on:input={textChange} />

    <button disabled={text().length === 0} on:click={() => text.set('')}>
      {`Reset ${text()}`}
    </button>`,
  style: `
    .danger {
      color: red;
    }`,
}));
```

External files:
```ts
import { component, input, output, booleanAttribute } from '@angular/core';

export const Checkbox = component(({
  value = input.required<any>({ transform: booleanAttribute }),
  valueChange = output<boolean>(),
}) => ({
  script: () => { /** ... **/ },
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
    const email = signal<string>(...);

    function processEmail() { /** ... **/ }
    function makeAdmin() { /** ... **/ }
  },
  template: `
    <!-- 2way binding for comp: bind:model={var} on:modelChange={func} -->

    <UserDetail
      user={user()}
      bind:email={email}
      on:emailChange={() => processEmail()}
      on:makeAdmin={makeAdmin} />`,
}));

// -- UserDetail -----------------------------------
import { component, input, model, output } from '@angular/core';

export interface User { /** ... **/ }

export const UserDetail = component(({
  user = input<User>(),
  email = model<string>(),
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
    const message = signal('Message');

    function valueChange() { /** ... **/ }
    function doSomething() { /** ... **/ }
  },
  template: `
    <!-- ... -->

    <!-- encapsulation of directive data: @directive(...) -->

    <input
      type="text"
      bind:value={text} on:input={valueChange}
      @tooltip(message={message()} on:dismiss={doSomething}) />

    <p>Value: {text()}</p>`,
}));

// -- tooltip in @mylib/tooltip --------------------
import { directive, input, output, inject, ElementRef, Renderer2 } from '@angular/core';

export const tooltip = directive(({
  message = input.required<string>(),
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

## Derivations
Declaratively transforms data within templates:
```ts
import { component, input, signal } from '@angular/core';
import { bestSellers, memo, currency, half } from '@mylib/derivations';

interface Item { /** ... **/ }

export const Items = component(({
  items = input.required<Item[]>(),
}) => ({
  script: () => { /** ... **/ },
  template: `
    <!-- ... -->

    @let filteredItems = @bestSellers(items);

    @for (item of filteredItems(); track item.id) {
      @let memoItem = @memo(() => item);

      @if (memoItem().discount) {
        @let price = @currency(@half(() => memoItem().price), () => 'EUR');
        <div>Price: {price()}</div>
      } @else {
        @let price = @currency(() => memoItem().price, () => 'EUR');
        <div>Price: {price()}</div>
      }
    }`,
}));

// -- currency in @mylib/derivations --------------------
import { derivation, computed, inject, LOCALE_ID } from '@angular/core';

export const currency = derivation(() => ({
  script: () => {
    const localeId = inject(LOCALE_ID);

    return (
      value: () => (number | string | undefined),
      currencyCode: (() => string) | undefined,
    ) => {
      // ...

      return computed( /** ... **/ );
    }
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

  constructor(c = () => 0) {
    this.counter = linkedSignal(() => c());
  }

  decrease() { /** ... **/ }
  increase() { /** ... **/ }
}

export const Counter = component(({
  c = input.required<number>(),
}) => ({
  providers: [
    provide({ token: CounterStore, useFactory: () => new CounterStore(c) }),
  ],
  script: () => {
    const store = inject(CounterStore);
  },
  template: `
    <h1>Counter</h1>
    <div>Value: {store.value()}</div>
    <button on:click={() => store.decrease()}> - </button>
    <button on:click={() => store.increase()}> + </button>`,
}));
```

## Composition with fragments and directives
Fragments are very similar to [`svelte snippets`](https://svelte.dev/docs/svelte/snippet): functions returning html markup. Returned markup is opaque: cannot manipulate it similarly to [`react Children (legacy)`](https://react.dev/reference/react/Children) or [`solid children`](https://www.solidjs.com/tutorial/props_children).

Implicit children fragment (where + when) and binding context:
```ts
import { component, signal } from '@angular/core';
import { Menu, MenuItem } from '@mylib/menu';

export const MenuConsumer = component(() => ({
  script: () => {
    const first = signal('First');
    const second = signal('Second');
  },
  template: `
    <Menu>
      <MenuItem>{first()}</MenuItem>
      <MenuItem>{second()}</MenuItem>
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
      <Render fragment={children()} />
    } @else {
       <!-- ... -->
    }`,
}));

export const MenuItem = component(({
  children = input.required<Fragment<void>>(),
}) => ({
  script: () => { /** ... **/ },
  template: `
    <Render fragment={children()} />`,
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
    @fragment menuItem(item: {id: string, desc: string}) {
      <MyMenuItem>{item.desc}</MyMenuItem>
    }

    <Menu items={items()} menuItem={menuItem} />`,
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
    <h1> Total items: {items().length} </h1>

    @for (item of items(); track item.id) {
      <Render fragment={menuItem()} params={[item]} />
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
      <span class:red={node.desc === 'unknown'}>{node.desc}</span>
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
    <Tree nodes={nodes()} custumNode={myNode} />`,
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
        <Render fragment={custumNode()} params={[node]} />
      } @else {
        <!-- ... -->
      }
    }`,
}));
```

Directives passed as inputs and bound to an element:
```ts
import { component, signal } from '@angular/core';

import { Button } from '@mylib/button';
import { ripple } from '@mylib/ripple';
import { tooltip } from '@mylib/tooltip';

export const ButtonConsumer = component(() => ({
  script: () => {
    const tooltipMsg = signal('');
    const valid = signal(false);

    function doSomething() { /** ... **/ }
  },
  template: `
    <Button
      @ripple
      @tooltip(message={tooltipMsg()})
      disabled={!valid()}
      on:click={doSomething}>
        Click / Hover me
    </Button>`,
}));

// -- button in @mylib/button --------------------
import { component, input, output, toBindings } from '@angular/core';
import { Render } from '@angular/common';

export const Button = component(({
  children = input.required<Fragment<void>>(),
  disabled = input<boolean>(false),
  click = output<void>(),
}) => ({
  script: () => { /** ... **/ },
  template: `
    <!-- fallthrough directives (ripple / tooltip) from the consumer -->

    <button @** disabled={disabled()} on:click={() => click.emit()}>
      <Render fragment={children()} />
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

    <Dynamic component={comp()} inputs={inputs()} />`,
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
    const aComp = ref<{ text: Signal<string> }>('aComp');
    const manyComp = ref<{ text: Signal<string> }[]>(Child, { many: true });

    const tlp2 = ref<{ toggle: () => void }>('tlp');
    const tlp3 = ref(tooltip);
  },
  template: `
    <div
      #el
      @ripple=#rpl
      @tooltip(message={'something'})=#tlp>
        Something
    </div>

    <Child #aComp />
    <Child />

    <button on:click={() => tlp.toggle()}> Toggle tlp </button>`,
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
      @MatTooltip(message={tooltipMessage()} disabled={hasPermissions()})>
        Admin
    </MatButton:a>`,
}));
```

## Concepts affected by these changes
- `ng-content`: replaced by `fragments`,
- `ng-template` (`let-*` shorthands + `ngTemplateGuard_*`): likely replaced by `fragments`,
- structural directives: likely replaced by `fragments`,
- `Ng**Outlet` + `ng-container`: likely replaced by the new things,
- `queries`: if `ref` makes sense, likely not needed anymore; if they stay, it would be nice to improve the retrieval of data: no way to `read` providers from `injector` tree,
- multiple `directives` applied to the same element: as for the previous point, it would be nice to avoid directives injeciton when applied to the same element (see [`ngModel hijacking`](https://stackblitz.com/edit/stackblitz-starters-ezryrmmy)); instead, it should be an explicit operation with a `#ref` passed as an `input`,
- `directives` attached to the host (components): not possible anymore, but you can pass directives as inputs and use `@**` (or equivalent syntax).

Unresolved points:
- `spread props`: this (together with fragments and directives passed as inputs) is an important point in the context of "wrapper components" (`<Button ... />`). At the moment, inputs are created (then syncronised) any time a component / directive is created rather than derived from already existing signals (solid / svelte).
This is great for interoperability, but it implies there isn't any props object to spread. An alternative solution coulbe be something like vue [`fallthrough`](https://vuejs.org/guide/components/attrs.html) where props are aggregated using DI;
```ts
export const Button = component(({
  children = input.required<Fragment<void>>(),
  disabled = input<boolean>(false),
  type = input<'button' | 'reset' | 'submit'>('button'),
  click = output<void>(),
}) => ({
  script: () => {
    const fallthrough = inject(FALLTROUGH);
    // ...
  },
  template: `
    <button
      @**
      **={fallthrough.inputs}
      bind:**={fallthrough.twoWay}
      on:**={fallthrough.outputs}>
        <Render fragment={fallthrough.children()} />
    </button>`,
}));
```
- there isn't any obvious `short notation` for passing props (like svelte / vue);
```ts
<User user={user()} bind:address={address} on:userChange={userChange} />

// maybe something like "matching the name"?
// error in case of string interpolation or similar

<User {user()} bind:{address} on:{userChange} />
```
- there isn't any obvious way to conditionally apply directives;
```ts
// maybe using another ()?

<Button (@tooltip(message={tooltipMsg()}) && {enabled()})>
  Click / Hover me
</Button>
```
- can reassign inputs / outputs inside script:
  - `https://github.com/microsoft/TypeScript/issues/18497`,
  - [`no-param-reassign`](https://eslint.org/docs/latest/rules/no-param-reassign).
- parent component styling children (difficult point): maybe something based on css-variables similarly to [`svelte`](https://svelte.dev/docs/svelte/custom-properties)? Alternatevely, directives could support style `https://github.com/angular/angular/issues/17766`.
