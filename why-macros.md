## Why macros
Angular components has already some special hoisting rules: class fields are visible in templates (string literals) despite they are not in the same scope. Assuming such principles are applied to `script` and `template` (where template is resolved similarly to right now), one might argue that macros are not really necessary cause a component could be written as a "sort of valid function". On the other hand: 
```ts
import { component ... } from '@angular/core';

let Comp = component(({
  /** ... **/
}) => {
  const unwanted = 'unwanted';
  return {
    providers: [...],
    script: () => {...},
    template: (<>...</>),
    style: (<>...</>),
  };
});
```

So with macros and `**.ng` files
- you avoid unwanted flexility (definitions: let / var),
- you avoid strange scope behaviours (`unwanted` above),
- you have clear markers for tools,
- you keep DI separated from script / template and at the same time enable the definition of providers depending on inputs, but not on variables defined inside script. 

Note that an alternative approach would be something like below: it's more verbose, but it doesn't have the special `script` / `template` hoisting rules. It'd likely imply having 2 ways of defining components. 
```ts
import { linkedSignal, input, WritableSignal, provide, inject } from '@angular/core';

class CounterStore {
  private readonly counter: WritableSignal<number>;
  readonly value = this.counter.asReadonly();

  constructor(c = () => 0) {
    this.counter = linkedSignal(() => c());
  }

  decrease() {/** ... **/}
  increase() {/** ... **/}
}

export #component Counter({
  c = input.required<number>(),
}) {
  providers: [
    provide({ token: CounterStore, useFactory: () => new CounterStore(c) }),
  ],
  script: () => {
    const store = inject(CounterStore);
    
    return {
      template: (
        <>
          <h1>Counter</h1>
          <div>Value: {store.value()}</div>
          <button on:click={() => store.decrease()}>-</button>
          <button on:click={() => store.increase()}>+</button>
        </>
      ),
      style: (<>...</>),
      exports: {/** public interface **/},
    };
  },
}

export #component CounterWithoutDefiningProviders() {
  const store = inject(CounterStore);
  
  return {
    template: (
      <>
        <h1>Counter</h1>
        <div>Value: {store.value()}</div>
        <button on:click={() => store.decrease()}>-</button>
        <button on:click={() => store.increase()}>+</button>
      </>
    ),
    style: (<>...</>),
    exports: {/** public interface **/},
  };
}
```

### Another example
```ts
import { ... } from '@angular/core';
import { Card, HStack, Img, VStack, Title, Description } from '@lib/card';

export interface Item {
  id: string;
  imgUrl: string;
  title: string;
  description: string;
  price: number;
}

#directive tooltip({
  message = input.required<string>(),
  elRef = ref<HTMLElement>(),
}) {
  script: () => {
    const renderer = inject(Renderer2);

    afterRenderEffect(() => {
      /** ... **/
    });
  },
}

#declaration currency() {
  script: () => {
    const localeId = inject(LOCALE_ID);
    
    return (
      value: () => (number | undefined),
      currencyCode: string | undefined,
    ) => computed(/** ... **/);
  },
}

#component List({
  items = input.required<Item[]>(),
  item = fragment<[Item]>(),
}) {
  template: (
    <>
      @for (i of items(); track i.id) {
        <Render fragment={item()} params={[i]} />
      }
    </>
  ),
}

class ItemsStore {
  /** ... **/
}

export #component ItemsPage() {
  providers: [
    provide({ token: ItemsStore, useFactory: () => new ItemsStore() }),
  ],
  script: () => {
    const store = inject(ItemsStore);
  
    function goTo(item: Item) {
      // ..
    }
  },
  template: (
    <>
      <List items={store.items()}>
        @fragment item(i: Item) {
          <Card on:click={(i: Item) => goTo(i)}>
            <HStack width={100}>
              <Img url={i.imgUrl} />
              <VStack>
                <Title title={i.title} />
                <Description @tooltip(message={i.title}) description={i.description} />
                
                <hr />                
                
                @const price = @currency(() => i.price, 'EUR');
                <p>Price: {price}</p>
              </VStack>
            </HStack>
          </Card>
        }
      </List>
    </>
  ),
  styleUrl: './items-page.css',
}
```
