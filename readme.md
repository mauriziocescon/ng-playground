# Anatomy of signal components
**⚠️ Note ⚠️: just personal thoughts from a DEV perspective on [the future of angular](https://myconf.dev/videos/2024-keynote-session-the-future-of-angular) (template level).**

Points:
1. building blocks as functions:
    - `component`: a quad `providers` / `script` / `template` / `style`,
    - `declaration`: a way to declare `const` variables in templates that depend on DI,
    - `directive`: a `script` that can change the appearance or behaviour of DOM elements,
    - `fragment`: a way to capture some markup in the form of a function,
2. hostless components + ts lexical scoping for templates,
3. ts expressions with `{}`: bindings + text interpolation,
4. extra bindings for DOM elements: `bind:`, `on:`, `model:`, `class:`, `style:`, `attr:`, `animate:`,
5. component inputs: lifted up + immediately available in the script,
6. composition with fragments, directives and fallthrough attributes,
7. template ref,
8. DI enhancements,
9. Final considerations (`!important`).

## Components
Component structure and element bindings:
```ts
import { component, signal, linkedSignal, input, output } from '@angular/core';

export const TextSearch = component(({
  /**
   * value / valueChange are always created
   * for interoperability
   * (no real "props deconstruction")
   *
   * by the time script is called,
   * inputs are populated with parent data
   */
  value = input.required<string>(), // definition + type
  valueChange = output<string>(),
}) => ({
  script: () => {
    const text = linkedSignal(() => value());
    const isDanger = signal(false);

    function textChange() {
      valueChange.emit(text());
    }
  },
  template: `
    <!-- bind: can be omitted (default) while on: is mandatory -->
    <!-- 2way binding for input / select / textarea: model:property={var} -->

    <label class:danger={isDanger()}> Text: </label>
    <input type="text" model:value={text} on:input={textChange} />

    <button disabled={text().length === 0} on:click={() => text.set('')}>
      {'Reset ' + text()}
    </button>`,
  style: `
    .danger {
      color: red;
    }`,
}));
```

External template / style files:
```ts
import { component, input, output } from '@angular/core';

/**
 * Have to import what's used in **.ng.html:
 * @import { Comp } from '...';
 */
export const Checkbox = component(({
  value = input.required<boolean>(),
  valueChange = output<boolean>(),
}) => ({
  script: () => { /** ... **/ },
  templateUrl: `./checkbox.ng.html`,
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
    <!-- any component can be used directly in the template -->
    <!-- 2way binding for comp: model:name={var} on:nameChange={func} -->

    <UserDetail
      user={user()}
      model:email={email}
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

## Declarations
Lexical scoping: template => `script` => func / const / enum / interface imported in the file => global.
```ts
import { component } from '@angular/core';

enum Type {
  Counter = 'counter',
  Other = 'other',
}

const type = Type.Counter;

const counter = (value: number) => `Let's count till ${value}`;

/**
 * short version in case of
 * missing providers / script / style
 */
export const Counter = component(() => ({
  template: `
    @if (type === Type.Counter) {
      <p>{counter(5)}</p>
    } @else {
      <!-- ... -->
    }`,
}));
```

Definition of `@const` variables in the template (creation happens once) that can run in an injection context.
```ts
import { component, declaration, signal, computed, inject, LOCALE_ID } from '@angular/core';

const counter = (value?: number) => {
  const count = signal(value ?? 0);
  const price = computed(() => 10 * count());

  return {
    value: count.asReadonly(),
    price,
    decrease: () => count.update(c => c - 1),
    increment: () => count. update(c => c + 1),
  };
};

const currency = declaration((
  value: () => (number | undefined),
  currencyCode: string | undefined,
) => {
  // injection context like component.script
  const localeId = inject(LOCALE_ID);
  return computed(/** ... **/);
});

export const Counter = component(() => ({
  template: `
    @const count = counter(0);
  
    <!-- requires @ -->
    @const price = @currency(count.value, 'EUR');
  
    <h1>Counter</h1>
    <div>Value: {count.value()}</div>
    <div>Price: {price()}</div>
    <button on:click={() => count.decrease()}>-</button>
    <button on:click={() => count.increase()}>+</button>`,
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
    <!-- encapsulation of directive data: @directive(...) -->
    <!-- any directive can be used directly in the template -->

    <input
      type="text"
      model:value={text}
      on:input={valueChange}
      @tooltip(message={message()} on:dismiss={doSomething}) />

    <p>Value: {text()}</p>`,
}));

// -- tooltip in @mylib/tooltip --------------------
import { directive, input, output, inject, Renderer2, ref, afterNextRender } from '@angular/core';

export const tooltip = directive(({
  message = input.required<string>(),
  dismiss = output<void>(),
  /**
   * Readonly signal managed by ng.
   *
   * elRef: name reserved to the framework
   */
  elRef = ref<HTMLElement>(),
}) => ({
  script: () => {
    const renderer = inject(Renderer2);

    afterNextRender(() => {
      // something with elRef
    });
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
    <button on:click={() => store.decrease()}>-</button>
    <button on:click={() => store.increase()}>+</button>`,
}));
```

## Composition with fragments, directives and fallthrough attributes
Fragments are very similar to [`svelte snippets`](https://svelte.dev/docs/svelte/snippet): functions returning html markup. Returned markup is opaque: cannot manipulate it similarly to [`react Children (legacy)`](https://react.dev/reference/react/Children) or [`solid children`](https://www.solidjs.com/tutorial/props_children).

Directives follows very similar rules as [`svelte attachments`](https://svelte.dev/docs/svelte/@attach).

Fallthrough attributes are inspired by the same concept in [`vue`](https://vuejs.org/guide/components/attrs.html) and covers the usual react `spread props` need. At the moment, inputs are created (then syncronized) any time a component / directive is created rather than derived from already existing signals (solid / svelte). This is great for interoperability, but it implies there isn't any props object to spread. Note the examples below are simplified.

Implicit children fragment (where + when) and binding context:
```ts
import { component, signal } from '@angular/core';
import { Menu, MenuItem } from '@mylib/menu';

export const MenuConsumer = component(() => ({
  script: () => ({
    const first = signal('First');
    const second = signal('Second');
  },
  template: `
    <!-- markup inside comp tag => implicitly become an input called children -->

    <Menu>
      <MenuItem>{first()}</MenuItem>
      <MenuItem>{second()}</MenuItem>
    </Menu>`,
}));

// -- Menu in @mylib/menu --------------------------
import { component, input, Fragment } from '@angular/core';
import { Render } from '@angular/common';

export const Menu = component(({
  /**
   * children: name reserved to the framework
   */
  children = input<Fragment<void>>(),
}) => ({
  script: () => { /** ... **/ },
  template: `
    <!-- no need to have an explicit anchor point like ng-container -->

    @if (children()) {
      <Render fragment={children()} />
    } @else {
       <!-- ... -->
    }`,
}));

export const MenuItem = component(({
  children = input.required<Fragment<void>>(),
}) => ({
  template: `
    <Render fragment={children()} />`,
}));
```

Customising components:
```ts
import { component, signal } from '@angular/core';
import { Menu } from '@mylib/menu';
import { MyMenuItem } from './my-menu-item';

export interface Item {
  id: string;
  desc: string;
}

export const MenuConsumer = component(() => ({
  script: () => {
    const items = signal<Item[]>(/** ... **/);
  },
  template: `
    <!-- menuItem inside <Menu></Menu> automatically becomes an input -->

    @fragment menuItem(item: Item) {
      <div class="my-menu-item">
        <MyMenuItem>{item.desc}</MyMenuItem>
      </div>
    }
    <Menu items={items()} menuItem={menuItem} />`,
  styleUrl: './menu-consumer.css',
}));

// -- Menu in @mylib/menu --------------------------
import { component, input, Fragment } from '@angular/core';
import { Render } from '@angular/common';

export const Menu = component(({
  items = input.required<{ id: string, desc: string }[]>(),
  menuItem = input.required<Fragment<[{ id: string, desc: string }]>>(),
}) => {
  template: `
    <h1> Total items: {items().length} </h1>
  
    @for (item of items(); track item.id) {
      <Render fragment={menuItem()} params={[item]} />
    }`,
}));
```

Directives passed as inputs and bound to an element at runtime:
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
import { component, input, output } from '@angular/core';
import { Render } from '@angular/common';

export const Button = component(({
  children = input.required<Fragment<void>>(),
  disabled = input<boolean>(false),
  click = output<void>(),
}) => ({
  template: `
    <!-- @** => fallthrough directives (ripple / tooltip) from the consumer -->
  
    <button @** disabled={disabled()} on:click={() => click.emit()}>
      <Render fragment={children()} />
    </button>`,
}));
```

Wrapping components and passing inputs / outputs:
```ts
import { component, input, computed, fallthroughAttrs } from '@angular/core';
import { UserDetail, User, UserDetailProps } from './user-detail';

export const UserDetailConsumer = component(() => ({
  script: () => {
    const user = signal<User>(...);
    const email = signal<string>(...);

    function processEmail() { /** ... **/ }
    function makeAdmin() { /** ... **/ }

    const outputs = {
      makeAdmin,
      emailChange: () => processEmail()
    };
  },
  template: `
    <!-- bind:**={object} bind all entries of object; same for model / on -->

    <MyUserDetail
      bind:**={{user}}
      model:**={{email}}
      on:**={outputs} />`,
}));

export const MyUserDetail = component(({
  user = input<User>(),
  /**
   * whatever is not matching inputs / outputs
   * defined explicitly (like user).
   *
   * attrs entries:
   * - in: inputs, attributes, ..
   * - on: events,
   * - mod: 2way.
   *
   * attrs: name reserved to the framework
   */
  attrs = fallthroughAttrs<Omit<UserDetailProps, 'user'>>(),
}) => ({
  script: () => {
    const other = computed(() => /** something depending on user or a default value **/);
  },
  template: `
    <UserDetail
      user={other()}
      bind:**={attrs.in}
      model:**={attrs.mod}
      on:**={attrs.on} />`,
}));

// -- UserDetail -----------------------------------
import { component, input, model, output, Props } from '@angular/core';

export interface User { /** ... **/ }

export type UserDetailProps = Props<UserDetail>;

export const UserDetail = component(({
  user = input<User>(),
  email = model<string>(),
  makeAdmin = output<void>(),
}) => ({
  // ...
}));
```

Wrapping native elements and passing attributes / properties / event listeners:
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
    <!-- can pass down attributes, properties, event listeners either static or bound -->

    <Button
      type="button"
      style="background-color: cyan"
      class={valid() ? 'global-css-valid' : ''}
      @ripple
      @tooltip(message={tooltipMsg()})
      disabled={!valid()}
      on:click={doSomething}>
        Click / Hover me
    </Button>`,
}));

// -- button in @mylib/button --------------------
import { component, input, fallthroughAttrs } from '@angular/core';
import { HTMLButtonAttributes } from '@angular/core/elements';

export const Button = component(({
  children = input.required<Fragment<void>>(),
  attrs = fallthroughAttrs<HTMLButtonAttributes>(),
}) => ({
  template: `
    <button @** bind:**={attrs.in} on:**={attrs.on}>
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
import { component, ref, Signal, signal, afterNextRender } from '@angular/core';
import { tooltip } from '@mylib/tooltip';

const Child = component(() => ({
  script: () => {
    const text = signal('');
    // ...

    // can return an object that
    // any ref can use to interact
    // with the component
    // (public interface)
    return {
      text: text.asReadonly(),
    };
  },
  template: `<!-- ... -->`,
}));

export const Parent = component(() => ({
  script: () => {
    // readonly signal
    const el = ref<HTMLDivElement>('el');

    // 1. can only use what's returned by Child.script
    // 2. templates only lookup: cannot retrieve providers
    //    defined in the Child comp tree
    const child = ref('child');

    // using what's returned by tooltip.script
    const tlp = ref<{ toggle: () => void }>('tlp');
    const many = signal<{ text: Signal<string> }[]>([]);

    afterNextRender(() => {
      // something with refs
    });
  },
  template: `
    <div
      #el
      @ripple=#rpl
      @tooltip(message={'something'})=#tlp>
        Something
    </div>

    <Child #child />

    <!-- binding to a function -->

    <Child ref={(c) => many.update(v => [...v, c])} />
    <Child ref={(c) => many.update(v => [...v, c])} />

    <button on:click={() => tlp().toggle()}> Toggle tlp </button>`,
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

## Final considerations

### Concepts affected by these changes
- `ng-content`: replaced by `fragments`,
- `ng-template` (`let-*` shorthands + `ngTemplateGuard_*`): likely replaced by `fragments`,
- structural directives: likely replaced by `fragments`,
- `Ng**Outlet` + `ng-container`: likely replaced by the new things,
- `pipes`: replaced by declarations,
- `event delegation`: not consider `https://github.com/angular/angular/issues/15905`,
- `@let`: likely obsolete and not needed anymore,
- `directives` attached to the host (components): not possible anymore, but you can pass directives as inputs and use `@**` (or equivalent syntax),
- `queries`: if `ref` makes sense, likely not needed anymore; if they stay, it would be nice to improve the retrieval of data: no way to `read` providers from `injector` tree,
- multiple `directives` applied to the same element: as for the previous point, it would be nice to avoid directives injection when applied to the same element (see [`ngModel hijacking`](https://stackblitz.com/edit/stackblitz-starters-ezryrmmy)); instead, it should be an explicit operation with a `ref` passed as an `input`,
- in general, the concept of injecting components / directives inside each others should be restricted cause it generates lots of indirection / complexity; the downside is that some ng-reserved names are necessary.

### Unresolved points
- there isn't any obvious `short notation` for passing props (like svelte / vue);
```ts
<User user={user()} age={age()} gender={gender()} model:address={address} on:userChange={userChange} />

// maybe something like "matching the name only for signals"?
// error in case of string interpolation or similar

<User {user} {age} {gender} model:{address} on:{userChange} />
```
- there isn't any obvious way to conditionally apply directives at runtime;
```ts
// maybe using another ()?

<Button ( @tooltip(message={tooltipMsg()}) && {enabled()} )>
  Click / Hover me
</Button>
```
- can reassign inputs / outputs inside script:
  - `https://github.com/microsoft/TypeScript/issues/18497`,
  - [`no-param-reassign`](https://eslint.org/docs/latest/rules/no-param-reassign).

### Pros and cons and evolution
Pros: 
- familiar, 
- relatively easy to migrate (just a move + reshuffle of things),

Cons:
- the general shape of defining components like above has a major problem: 
```ts
export const Name = component(({
  i = input.required<string>(),
}) => {
  const unwanted = 'unwanted';
  return {
    providers: [...],
    script: () => {...},
    template: `...`,
    style: `...`,
  };
});
```
But since angular is a compiled framework, the problem can be fixed by introducing `**.ng` files (typescript superset), `component` / `directive` / `declaration` keywords (see Ripple) and by turning the syntax above into the following: 
```ts
export component Name = ({
  i = input.required<string>(),
}) => {
  providers: [...],
  script: () => {...},
  template: `...`,
  style: `...`,
};
```
See [`Alternatives`](https://github.com/mauriziocescon/ng-playground/blob/main/alternatives.md) for a full example.
