# Anatomy of signal components
**Note: just personal thoughts from a DEV perspective on [the future of angular](https://myconf.dev/videos/2024-keynote-session-the-future-of-angular) (template level).**

Points:
1. building blocks as functions:
    - `**.ng` files,
    - `component`: a quad `script` / `template` / `style` / `providers`,
    - `declaration`: a way to declare vars in templates that depends on DI,
    - `directive`: a `script` that can change the appearance or behavior of DOM elements,
    - `fragment`: a way to capture some markup in the form of a function,
2. hostless components + ts lexical scoping for templates,
3. ts expressions with `{}`: bindings + text interpolation,
4. extra bindings for DOM elements: `bind:`, `on:`, `model:`, `class:`, `style:`, `attr:`, `animate:`,
5. component inputs: lifted up + immediately available in the script,
6. composition with fragments, directives and fallthrough attributes,
7. template ref,
8. DI enhancements,
9. Concepts affected by these changes.

## Components
Component structure and element bindings:
```ts
import { signal, linkedSignal, input, output } from '@angular/core';

#Component
export const TextSearch = ({
  /**
   * value / valueChange are always created
   * for interoperability
   * (no real "props decostruction")
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

    // exposed as public interface
    return {
      text: text.asReadonly(),
    };
  },
  template: `
    <!-- bind: can be omitted (default) while on: is mandatory -->
    <!-- 2way binding for input / select / textarea: model:property={var} -->

    <label class:danger={isDanger()}> Text: </label>
    <input type="text" model:value={text} on:input={textChange} />

    <button disabled={text().length === 0} on:click={() => text.set('')}>
      {`Reset ${text()}`}
    </button>`,
  style: `
    .danger {
      color: red;
    }`,
});
```

External template / style files:
```ts
import { input, output } from '@angular/core';

/**
 * Have to import what's used in **.ng.html:
 * @import { Comp } from '...';
 */
#Component
export const Checkbox = ({
  value = input.required<boolean>(),
  valueChange = output<boolean>(),
}) => ({
  script: () => { /** ... **/ },
  templateUrl: `./checkbox.ng.html`,
  styleUrl: `./checkbox.css`,
});
```

Component bindings:
```ts
import { signal } from '@angular/core';
import { UserDetail, User } from './user-detail.ng';

#Component
export const UserDetailConsumer = () => ({
  script: () => {
    const user = signal<User>(...);
    const email = signal<string>(...);

    function processEmail() { /** ... **/ }
    function makeAdmin() { /** ... **/ }
  },
  template: `
    <!-- any #Component can be used directly in the template -->
    <!-- 2way binding for comp: model:name={var} on:nameChange={func} -->

    <UserDetail
      user={user()}
      model:email={email}
      on:emailChange={() => processEmail()}
      on:makeAdmin={makeAdmin} />`,
});

// -- UserDetail -----------------------------------
import { input, model, output } from '@angular/core';

export interface User { /** ... **/ }

#Component
export const UserDetail = ({
  user = input<User>(),
  email = model<string>(),
  makeAdmin = output<void>(),
}) => ({
  // ...
});
```

Lexical scoping: template > `script` > any imported / defined const, func, enum, interface with `#Template` > global scope.
```ts
/**
 * #Template has no effect on Type. It just
 * makes Type available in the template of any
 * component of the **.ng file.
 */
#Template
enum Type {
  Counter = 'counter',
  Other = 'other',
}

#Template
const type = Type.Counter;

#Template
function counter(value: number) {
  return `Let's count till ` + value;
}

/**
 * short version in case of
 * missing script / style / providers
 */
#Component
export const Counter = () => `
  @if (type === Type.Counter) {
    <p>{counter(5)}</p>
  } @else {
    <!-- ... -->
  }`;
```

## Declarations
Definition of `@const` variables in the template that depends on DI (creation happens once).
```ts
import { signal, computed, inject, LOCALE_ID } from '@angular/core';

#Template
function counter(value?: number) {
  const count = signal(value ?? 0);
  const price = computed(() => 10 * count());

  return {
    value: count.asReadonly(),
    price,
    decrease: () => count.update(c => c - 1),
    increment: () => count. update(c => c + 1),
  };
}

#Declaration
function currency(
  value: () => (number | undefined),
  currencyCode: string | undefined,
) {
  const localeId = inject(LOCALE_ID);
  return computed(/** ... **/);
}

#Component
export const Counter = () => `
  @const count = counter(0);

  <!-- requires @ -->
  @const price = @currency(count.value, 'EUR');

  <h1>Counter</h1>
  <div>Value: {count.value()}</div>
  <div>Price: {price()}</div>
  <button on:click={() => count.decrease()}> - </button>
  <button on:click={() => count.increase()}> + </button>`;
```

## Element directives
Change the appearance or behavior of DOM elements:
```ts
import { signal } from '@angular/core';
import { tooltip } from '@mylib/tooltip';

#Component
export const TextSearch = () => ({
  script: () => {
    const text = signal('');
    const message = signal('Message');

    function valueChange() { /** ... **/ }
    function doSomething() { /** ... **/ }
  },
  template: `
    <!-- encapsulation of directive data: @directive(...) -->

    <input
      type="text"
      model:value={text}
      on:input={valueChange}
      @tooltip(message={message()} on:dismiss={doSomething}) />

    <p>Value: {text()}</p>`,
});

// -- tooltip in @mylib/tooltip --------------------
import { input, output, inject, Renderer2, ref, afterNextRender } from '@angular/core';

#Directive
export const tooltip = ({
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

    // exposed as public
    return { /** ... **/ };
  },
});
```

## Inputs
Inputs lifted up for providers init:
```ts
import { linkedSignal, input, WritableSignal, provide, inject } from '@angular/core';

class CounterStore {
  private readonly counter: WritableSignal<number>;
  readonly value = this.counter.asReadonly();

  constructor(c = () => 0) {
    this.counter = linkedSignal(() => c());
  }

  decrease() { /** ... **/ }
  increase() { /** ... **/ }
}

#Component
export const Counter = ({
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
});
```

## Composition with fragments, directives and fallthrough attributes
Fragments are very similar to [`svelte snippets`](https://svelte.dev/docs/svelte/snippet): functions returning html markup. Returned markup is opaque: cannot manipulate it similarly to [`react Children (legacy)`](https://react.dev/reference/react/Children) or [`solid children`](https://www.solidjs.com/tutorial/props_children).

Directives follows very similar rules as [`svelte attachments`](https://svelte.dev/docs/svelte/@attach).

Fallthrough attributes are inspired by the same concept in [`vue`](https://vuejs.org/guide/components/attrs.html) and covers the usual react `spread props` need. At the moment, inputs are created (then syncronised) any time a component / directive is created rather than derived from already existing signals (solid / svelte). This is great for interoperability, but it implies there isn't any props object to spread. Note the examples below are simplified.

Implicit children fragment (where + when) and binding context:
```ts
import { signal } from '@angular/core';
import { Menu, MenuItem } from '@mylib/menu';

#Component
export const MenuConsumer = () => ({
  script: () => {
    const first = signal('First');
    const second = signal('Second');
  },
  template: `
    <!-- markup inside comp tag => implicitly become an input called children -->

    <Menu>
      <MenuItem>{first()}</MenuItem>
      <MenuItem>{second()}</MenuItem>
    </Menu>`,
});

// -- Menu in @mylib/menu --------------------------
import { input, Fragment } from '@angular/core';
import { Render } from '@angular/common';

#Component
export const Menu = ({
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
});

#Component
export const MenuItem = ({
  children = input.required<Fragment<void>>(),
}) => ({
  template: `
    <Render fragment={children()} />`,
});
```

Customising components:
```ts
import { computed } from '@angular/core';
import { Menu } from '@mylib/menu';
import { MyMenuItem } from './my-menu-item.ng';

#Component
export const MenuConsumer = () => ({
  script: () => {
    const items = computed(() => [{ id: '1', desc: 'First' }, { id: '2', desc: 'Second' }]);
  },
  template: `
    <!-- menuItem inside <Menu></Menu> automatically becomes an input -->

    @fragment menuItem(item: {id: string, desc: string}) {
      <MyMenuItem>{item.desc}</MyMenuItem>
    }
    <Menu items={items()} menuItem={menuItem} />`,
});

// -- Menu in @mylib/menu --------------------------
import { input, Fragment } from '@angular/core';
import { Render } from '@angular/common';

#Component
export const Menu = ({
  items = input.required<{ id: string, desc: string }[]>(),
  menuItem = input.required<Fragment<[{ id: string, desc: string }]>>(),
}) => ({
  template: `
    <h1> Total items: {items().length} </h1>

    @for (item of items(); track item.id) {
      <Render fragment={menuItem()} params={[item]} />
    }`,
});
```

Directives passed as inputs and bound to an element at runtime:
```ts
import { signal } from '@angular/core';
import { Button } from '@mylib/button';
import { ripple } from '@mylib/ripple';
import { tooltip } from '@mylib/tooltip';

#Component
export const ButtonConsumer = () => ({
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
});

// -- button in @mylib/button --------------------
import { input, output } from '@angular/core';
import { Render } from '@angular/common';

#Component
export const Button = ({
  children = input.required<Fragment<void>>(),
  disabled = input<boolean>(false),
  click = output<void>(),
}) => ({
  template: `
    <!-- @** => fallthrough directives (ripple / tooltip) from the consumer -->

    <button @** disabled={disabled()} on:click={() => click.emit()}>
      <Render fragment={children()} />
    </button>`,
});
```

Wrapping components and passing inputs / outputs:
```ts
import { input, computed, fallthroughAttrs } from '@angular/core';
import { UserDetail, User, UserDetailProps } from './user-detail.ng';

#Component
export const UserDetailConsumer = () => ({
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
});

#Component
export const MyUserDetail = ({
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
});

// -- UserDetail -----------------------------------
import { input, model, output, Props } from '@angular/core';

export interface User { /** ... **/ }

export type UserDetailProps = Props<UserDetail>;

#Component
export const UserDetail = ({
  user = input<User>(),
  email = model<string>(),
  makeAdmin = output<void>(),
}) => ({
  // ...
});
```

Wrapping native elements and passing attributes / properties / event listeners:
```ts
import { signal } from '@angular/core';
import { Button } from '@mylib/button';
import { ripple } from '@mylib/ripple';
import { tooltip } from '@mylib/tooltip';

#Component
export const ButtonConsumer = () => ({
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
});

// -- button in @mylib/button --------------------
import { input, fallthroughAttrs } from '@angular/core';
import { HTMLButtonAttributes } from '@angular/core/elements';

#Component
export const Button = ({
  children = input.required<Fragment<void>>(),
  attrs = fallthroughAttrs<HTMLButtonAttributes>(),
}) => ({
  template: `
    <button @** bind:**={attrs.in} on:**={attrs.on}>
      <Render fragment={children()} />
    </button>`,
});
```

Dynamic components:
```ts
import { signal, computed } from '@angular/core';
import { Dynamic } from '@angular/common';
import { AComp } from './a-comp.ng';
import { BComp } from './b-comp.ng';

#Component
export const Something = () => ({
  script: () => {
    const condition = signal<boolean>(/** ... **/);
    const comp = computed(() => condition() ? AComp : BComp);
    const inputs = computed(() => /** ... **/);
  },
  template: `
    <!-- ... -->

    <Dynamic component={comp()} inputs={inputs()} />`,
});
```

## Template ref
Retrieving references of elements / components / directives (runtime):
```ts
import { ref, Signal, signal, afterNextRender } from '@angular/core';
import { tooltip } from '@mylib/tooltip';

#Component
const Child = () => ({
  script: () => {
    const text = signal('');
    // ...

    // exposed as public
    return {
      text: text.asReadonly(),
    };
  },
  template: `<!-- ... -->`,
});

#Component
export const Parent = () => ({
  script: () => {
    // readonly signal
    const el = ref<HTMLDivElement>('el');

    // 1. can only use what's returned by Child.script
    // 2. templates only lookup: cannot retrieve providers
    //    defined in the Child comp tree
    const child = ref('child');
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
});
```

## DI enhancements
See [`DI enhancements`](https://github.com/mauriziocescon/ng-playground/blob/main/di.md)

## Backward compatibility
Still can use legacy concepts for composition:
```ts
import { input } from '@angular/core';
import { MatButton } from '@angular/material/button';
import { MatTooltip } from '@angular/material/tooltip';

#Component
export const AdminLinkWithTooltip = ({
  tooltipMessage = input.required<string>(),
  hasPermissions = input.required<boolean>(),
}) => ({
  template: `
    <MatButton:a
      href="/admin"
      @MatTooltip(message={tooltipMessage()} disabled={hasPermissions()})>
        Admin
    </MatButton:a>`,
});
```

## Concepts affected by these changes
- `ng-content`: replaced by `fragments`,
- `ng-template` (`let-*` shorthands + `ngTemplateGuard_*`): likely replaced by `fragments`,
- structural directives: likely replaced by `fragments`,
- `Ng**Outlet` + `ng-container`: likely replaced by the new things,
- `pipes`: replaced by derivations,
- `event delegation`: not consider `https://github.com/angular/angular/issues/15905`,
- `directives` attached to the host (components): not possible anymore, but you can pass directives as inputs and use `@**` (or equivalent syntax),
- `queries`: if `ref` makes sense, likely not needed anymore; if they stay, it would be nice to improve the retrieval of data: no way to `read` providers from `injector` tree,
- multiple `directives` applied to the same element: as for the previous point, it would be nice to avoid directives injection when applied to the same element (see [`ngModel hijacking`](https://stackblitz.com/edit/stackblitz-starters-ezryrmmy)); instead, it should be an explicit operation with a `ref` passed as an `input`,
- in general, the concept of injecting components / directives inside each others should be restricted cause it generates lots of indirection / complexity; the downside is that some ng-reserved names are necessary.

Unresolved points:
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
- parent component styling children (difficult point): maybe something based on css-variables similarly to [`svelte`](https://svelte.dev/docs/svelte/custom-properties)?
