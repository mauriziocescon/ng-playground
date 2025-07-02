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

<MyComponent compInput={compValue()} on:compOutput={compFunc()} />

<MyComponent
  compInput={compValue()}
  on:compOutput={compFunc()}
  bind:compTwoWay={compTwoWay}
  on:compTwoWayChange={compTwoWayChange()}
  @myDir(dirInput={dirValue()} on:dirOutput={output()} bind:twoWay={someValue}) />

<MyComponent
  compInput={compValue()}
  on:compOutput={compFunc()}
  bind:compTwoWay={compTwoWay}
  on:compTwoWayChange={compTwoWayChange()}
  use:myDir(dirInput={dirValue()} on:dirOutput={output()} bind:twoWay={someValue}) />
```

```html
<input bind:value={text} />
<span on:click={() => callMethod()}>{text()}</span>
<input type="text" on:keyup.enter={($event) => callMethod($event)} />
<ul attr:role={listRole()}></ul>
<ul class:expanded={isExpanded()}></ul>
<section style:height.px={sectionHeightInPixels()}></section>

<example-cmp animate:out="fancy-animation-class" />
<example-cmp animate:out={myDynamicCSSClasses()}" />
<other-example-cmp animate:out={() => animateFn($event)} />

<div @tooltip(message={shortText()})>{text()}</div>

<User user={user()} bind:someModel={x} />
<User user={user()} bind:someModel={x} on:someModelChange={method} />
<User user={user()} @tooltip(message={msg()} on:dismiss={() => dismiss()}) />
<User user={user()} @tooltip(bind:message={msg} on:messageChange={() => doSomething()}) />

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
