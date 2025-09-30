## Inputs
Inputs lifted up for providers init:
```ts
import { linkedSignal, input, WritableSignal, provide, inject } from '@angular/core';

class CounterStore {
  private readonly counter: WritableSignal<number>;
  readonly value = this.counter.asReadonly();

  constructor(c: Signal<number>) {
    this.counter = linkedSignal(() => c());
  }

  decrease() { /** ... **/ }
  increase() { /** ... **/ }
}

/**
 * provide function for types safety
 */
#comp
export const Counter = ({
  c = input<number>() as const,
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

## DI enhancements
Better ergonomics around types / tokens:
```ts
import { inject, provide, provideForRoot, injectionToken, input } from '@angular/core';

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

/**
 * provide compToken at Counter level using the default factory
 */
#comp
export const Counter = ({
  initialValue = input<number>() as const,
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
});
```
