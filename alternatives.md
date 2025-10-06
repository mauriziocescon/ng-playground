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

```ts
import { input, provide, inject, Fragment, Render } from '@angular/core';
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

export #comp ItemsPage = () => {
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

  <style>
    @import url('./items-page.css');
  </style>
};

export #comp List = ({
  items = input.required<Item[]>(),
  item = input.required<Fragment<[Item]>>(),
}) => {
  @for (i of items(); track item.id) {
    <Render fragment={item()} inputs={[i]} />
  }
};
```

```ts
import { input, provide, inject, Comp, Render } from '@angular/core';
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

export #comp ItemsPage = () => {
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

  <List items={store.items()}>
    #comp Item = ({
      i = input.required<Item>(),
    }) => {
      <Card on:click={goTo(i)}>
        <HStack width={100}>
          <Img url={i.imgUrl} />
          <VStack>
            <Title title={i.title} />
            <Description description={i.description} />
          </VStack>
        </HStack>
      </Card>
    };
  </List>

  <style>
    @import url('./items-page.css');
  </style>
};

export #comp List = ({
  items = input.required<Item[]>(),
  Item = input.required<Comp<{i: InputSignal<Item>}>>(),
}) => {
  @for (i of items(); track item.id) {
    <Render component={Item()} inputs={{i}} />
  }
};
```
