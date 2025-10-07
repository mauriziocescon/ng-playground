## Returns an object
Easy to migrate from the current decorators based setup. Clear distinction between script and providers. On the other hand, you can still define variables before returning the object. 
```ts
import { component, input, provide, inject, Fragment, Render } from '@angular/core';
import { Card, HStack, Img, VStack, Title, Description } from '@lib/card';

export interface Item {
  id: string;
  imgUrl: string;
  title: string;
  description: string;
}

class ItemsStore {
  /** ... **/
}

export const ItemsPage = component(() => ({
  providers: [
    provide({ token: ItemsStore, useFactory: () => new ItemsStore() }),
  ],
  script: () => {
    const store = inject(ItemsStore);

    function goTo(item: Item) {
      // ..
    }

    return {
      goTo,
    };
  },
  template: `
    <List items={store.items()}>
      @fragment item(i: Item) {
        <Card on:click={goTo(i)}>
          <HStack width={100}>
            <Img url={i.imgUrl} />
            <VStack>
              <Title title={i.title} />
              <Description description={i.description} />
            </VStack>
          </HStack>
        </Card>
      }
    </List>`,
  styleUrl: './items-page.css',
}));

export const List = component(({
  items = input.required<Item[]>(),
  item = input.required<Fragment<[Item]>>(),
}) => `
  @for (i of items(); track item.id) {
    <Render fragment={item()} inputs={[i]} />
  }`,
);
```

## merge script and providers
Reduced boilerplate, but there is confusion at the level of `provide` and `inject`: the order matters. 
```ts
import { component, input, provide, inject, Fragment, Render } from '@angular/core';
import { Card, HStack, Img, VStack, Title, Description } from '@lib/card';

export interface Item {
  id: string;
  imgUrl: string;
  title: string;
  description: string;
}

class ItemsStore {
  /** ... **/
}

export const ItemsPage = component(() => {
  provide({ token: ItemsStore, useFactory: () => new ItemsStore() });
  
  const store = inject(ItemsStore);

  function goTo(item: Item) {
    // ..
  }
  
  return {
    goTo,
  };
  
  <!ng>
    <List items={store.items()}>
      @fragment item(i: Item) {
        <Card on:click={goTo(i)}>
          <HStack width={100}>
            <Img url={i.imgUrl} />
            <VStack>
              <Title title={i.title} />
              <Description description={i.description} />
            </VStack>
          </HStack>
        </Card>
      }
    </List>
  </ng>
  
  <style>
    @import './items-page.css';
  </style>
});

export const List = component(({
  items = input.required<Item[]>(),
  item = input.required<Fragment<[Item]>>(),
}) => {
  <!ng>
    @for (i of items(); track item.id) {
      <Render fragment={item()} inputs={[i]} />
    }
  </ng>
});
```

## two separte scripts 
Clear distinction between all the blocks; less familiar for angular devs and more abstract. 
```ts
import { component, input, provide, inject, Fragment, Render } from '@angular/core';
import { Card, HStack, Img, VStack, Title, Description } from '@lib/card';

export interface Item {
  id: string;
  imgUrl: string;
  title: string;
  description: string;
}

class ItemsStore {
  /** ... **/
}

export const ItemsPage = component(() => {
  <script providers>
    provide({ token: ItemsStore, useFactory: () => new ItemsStore() });
  </script>

  <script>
    const store = inject(ItemsStore);

    function goTo(item: Item) {
      // ..
    }

    export {
      goTo,
    };
  </script>

  <!ng>
    <List items={store.items()}>
      @fragment item(i: Item) {
        <Card on:click={goTo(i)}>
          <HStack width={100}>
            <Img url={i.imgUrl} />
            <VStack>
              <Title title={i.title} />
              <Description description={i.description} />
            </VStack>
          </HStack>
        </Card>
      }
    </List>
  </ng>

  <style>
    @import './items-page.css';
  </style>
});

export const List = component({
  items = input.required<Item[]>(),
  item = input.required<Fragment<[Item]>>(),
}) => {
  <!ng>
    @for (i of items(); track item.id) {
      <Render fragment={item()} inputs={[i]} />
    }
  </ng>
});
```
