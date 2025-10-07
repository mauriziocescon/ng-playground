## Returns an object
Easy to migrate from the current decorators based setup. Clear distinction between script and providers. On the other hand, you can still define variables before returning the object... which makes no sense.  
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
}) => {
  template: `
    @for (i of items(); track item.id) {
      <Render fragment={item()} inputs={[i]} />
    }`,
});
```

## More abstract approach
Clear distinction between all the blocks: easy to set the lexical scope for each one. Not really typescript anymore. 
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

export component ItemsPage() {
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
}

export component List({
  items = input.required<Item[]>(),
  item = input.required<Fragment<[Item]>>(),
}) {
  template: `
    @for (i of items(); track item.id) {
      <Render fragment={item()} inputs={[i]} />
    }`,
}
```
