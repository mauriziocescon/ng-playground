```html
<MatButton:a
  href="/admin"
  @HasRipple
  @Tooltip(message="Cannot navigate" [disabled]="hasPermissions")>
  Admin
</MatButton:a>

<MyComponent [compInput]="compValue()" (compOutput)="compFunc()" />
<MyComponent
  [compInput]="compValue()"
  (compOutput)="compFunc()"
  [(compTwoWay)]="compTwoWay"
  (compTwoWayChange)="compTwoWayChange()"
  @myDir([dirInput]="dirValue()" (dirOutput)="dirFunc()" [(dirTwoWay)]="dirValue") />

<MyComponent compInput={compValue()} on:compOutput={() => compFunc()} />
<MyComponent
  compInput={compValue()}
  on:compOutput={compFunc()}
  model:compTwoWay={compTwoWay}
  on:compTwoWayChange={compTwoWayChange()}
  @myDir(dirInput={dirValue()} on:dirOutput={() => output()} model:dirTwoWay={someValue}) />
```

```html
<input model:value={text} />
<span on:click={() => callMethod()}>{text()}</span>
<input type="text" on:keyup.enter={($event) => callMethod($event)} />
<ul attr:role={listRole()}></ul>
<ul class:expanded={isExpanded()}></ul>
<section style:height.px={sectionHeightInPixels()}></section>

<example-cmp animate:leave="fancy-animation-class" />
<example-cmp animate:leave={myDynamicCSSClasses()}" />
<other-example-cmp animate:leave={() => animateFn($event)} />

<div @tooltip(message={shortText()})>{text()}</div>

<User user={user()} model:someModel={x} />
<User user={user()} model:someModel={x} on:someModelChange={method} />
<User user={user()} @tooltip(message={msg()} on:dismiss={() => dismiss()}) />
<User user={user()} @tooltip(model:message={msg} on:messageChange={() => doSomething()}) />

@if (valid()) {
  <MatButton:a
    id="fff"
    href="/admin"
    @hasRipple
    @tooltip(message="cannot navigate" disabled={hasPermissions()})>
      Admin
  </MatButton:a>
}
```

```ts
import { input, provide, inject, Fragment } from '@angular/core';

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
  <providers>
    provide({ token: ItemsStore, useFactory: () => new ItemsStore() });
  </providers>

  <script>
    const store = inject(ItemsStore);

    function goTo(item: Item) {
      // ..
    }

    export {
      goTo,
    };
  </script>

  <>
    <List items={store.items()}>
      #comp Item = ({
        i = input.required<Item>(),
      }) => {
        <>
          <Card on:click={goTo(i)}>
            <HStack width={100}>
              <Img url={i.imgUrl} />
              <VStack>
                <Title title={i.title} />
                <Description description={i.description} />
              </VStack>
            </HStack>
          </Card>
        </>
      };
    </List>
  </>

  <style>
    /** **/
  </style>
};

export #comp List = ({
  items = input.required<Item[]>(),
  item = input.required<Fragment<[Item]>>(),
}) => {
  <>
    @for (i of items(); track item.id) {
      <Render fragment={item()} inputs={[i]} />
    }
  </>
};
```
