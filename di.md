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

## DI enhancements
Better ergonomics around types / tokens:
```ts
import { component, inject, provide, injectionToken, input } from '@angular/core';

/**
 * not provided in root by default: the token
 * must be provided somewhere
 * 
 * a factory defines a default implementation and type
 */
const compToken = injectionToken('desc', {
  factory: () => {
    const counter = signal(0);

    return {
      value: counter.asReadonly(),
      decrease: () => {
        counter.update(v => v - 1);
      },
      increase: () => {
        counter.update(v => v + 1);
      },
    };
  },
});

/**
 * root provider: no need to provide it
 */
const rootToken = injectionToken('desc', {
  level: 'root',
  factory: () => {
    const counter = signal(0);

    return {
      value: counter.asReadonly(),
      decrease: () => {
        counter.update(v => v - 1);
      },
      increase: () => {
        counter.update(v => v + 1);
      },
    };
  },
});

/**
 * multi defined at token creation
 */
const multiToken = injectionToken('desc', {
  multi: true,
  factory: () => Math.random(),
});

/**
 * provide compToken at Counter level using the default factory
 */
export const Counter = component(({
  initialValue = input<number>(),
}) => ({
  providers: [
    provide(compToken),
    provide(multiToken),
    provide(multiToken),
    provide({ token: multiToken, useFactory: () => 10 }),
    provide({ token: multiToken, useFactory: () => initialValue() }),
  ],
  script: () => {
    const rootCounter = inject(rootToken);
    const compCounter = inject(compToken);
    const multi = inject(multiToken); // array of numbers
    // ...
  },
  // ...
}));
```
