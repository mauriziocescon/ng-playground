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
export Generic = component(({
  items = input.required<Item[]>(),
}) => ({
  script: () => {
    function goTo(item: Item) {
      // ..
    }
  },
  template: `
    @fragment card(item: Item) {
      <Card on:click={goTo(item)}>
        <HStack width={100}>
          <Img url={item.imgUrl} />
          <VStack>
            <Title title={item.title} />
            <Description description={item.description} />
          </VStack>
        </HStack>
      </Card>
    }
    <List items={items()} item={card} />

    <List items={items()}>
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
}));
```
