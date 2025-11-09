## Returns an object
You can define variables before returning the object which makes no sense.  
```ts
import { component, directive, declaration, ... } from '@angular/core';
import { Card, HStack, Img, VStack, Title, Description } from '@lib/card';

export interface Item {
  id: string;
  imgUrl: string;
  title: string;
  description: string;
  price: number;
}

const tooltip = directive(({
  message = input.required<string>(),
  elRef = ref<HTMLElement>(),
}) => {
  const unwanted = 'unwanted';
  
  return {
    script: () => {
      const renderer = inject(Renderer2);
  
      afterRenderEffect(() => {
        /** ... **/
      });
    },
  };
});

const currency = declaration(() => {
  const unwanted = 'unwanted';
  
  return {
    script: () => {
      const localeId = inject(LOCALE_ID);
      
      return (
        value: () => (number | undefined),
        currencyCode: string | undefined,
      ) => computed(/** ... **/);
    },    
  };
});

const List = component(({
  items = input.required<Item[]>(),
  item = input.required<Fragment<[Item]>>(),
}) => {
  const unwanted = 'unwanted';
  
  return {
    template: (
      <>
        @for (i of items(); track i.id) {
          <Render fragment={item()} inputs={[i]} />
        }
      </>
    ),
  };
});

class ItemsStore {
  /** ... **/
}

export const ItemsPage = component(() => {
  const unwanted = 'unwanted';
  
  return {
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
            <Card on:click={goTo(i)}>
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
  };
});
```

## More abstract approach
`**.ng` files (typescript superset) with `component` / `directive` / `declaration` keywords (see RippleJS). 
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

directive tooltip = ({
  message = input.required<string>(),
  elRef = ref<HTMLElement>(),
}) => {
  script: () => {
    const renderer = inject(Renderer2);

    afterRenderEffect(() => {
      /** ... **/
    });
  },
};

declaration currency = () => {
  script: () => {
    const localeId = inject(LOCALE_ID);
    
    return (
      value: () => (number | undefined),
      currencyCode: string | undefined,
    ) => computed(/** ... **/);
  },
};

component List = ({
  items = input.required<Item[]>(),
  item = input.required<Fragment<[Item]>>(),
}) => {
  template: (
    <>
      @for (i of items(); track i.id) {
        <Render fragment={item()} inputs={[i]} />
      }
    </>
  ),
};

class ItemsStore {
  /** ... **/
}

export component ItemsPage = () => {
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
          <Card on:click={goTo(i)}>
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
};
```
