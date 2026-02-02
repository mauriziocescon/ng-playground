## Why macros
Angular components has already some special hoisting rules: class fields are visible in templates (string literals) despite they are not in the same ts scope. Assuming similar principles are applied to `script` and `template` (where template is resolved similarly to right now), one might argue that macros are not really necessary cause a component could be written as a "sort of valid function". On the other hand: 
```ts
import { component, ... } from '@angular/core';

let Comp = component(({
  /** ... **/
}) => {
  const unwanted = 'unwanted';
  return {
    script: () => {
      ...
      return {
        template: `...`,
        style: `...`,
      };
    },
    providers: [...],
  };
});
```

So with macros and `**.ng` files
- you avoid unwanted flexility (definitions: let / var),
- you avoid strange scope behaviours (`unwanted` above),
- you have clear markers for tools,
- you keep DI separated from script / template and at the same time enable the definition of providers depending on inputs, but not on variables defined inside script. 

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
}) {
  script: ({ host }) => {
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
  script: () => ({
    template: `
      @for (i of items(); track i.id) {
        <Render fragment={item()} params={[i]} />
      }
    `,
  }),
}

class ItemsStore {
  /** ... **/
}

export #component ItemsPage() {
  script: () => {
    const store = inject(ItemsStore);
  
    function goTo(item: Item) {
      // ..
    }
    
    return {
      template: `
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
      `,
      styleUrl: './items-page.css',
    };
  },
  providers: [
    provide({ token: ItemsStore, useFactory: () => new ItemsStore() }),
  ],
}
```
