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
