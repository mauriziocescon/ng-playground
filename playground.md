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
<input [(ngModel)]="text" />
<input @NgModel([(ngModel)]="text") />
<input @bind(value={text}) />
<input bind:value={text} />

<input [(ngModel)]="text" (ngModelChange)="textChanges()"/>
<input @NgModel([(ngModel)]="text" (ngModelChange)="textChanges()") />
<input bind:value={text} on:valueChange={textChanges()} />

<div [tooltip] [message]="shortText()">{{ text() }}</div>
<div @tooltip([message]="shortText()")>{{ text() }}</div>
<div use:tooltip(message={shortText()})>{ text() }</div>

<span (click)="callMethod()">{{ text() }}</span>
<span on:click={callMethod()}>{ text() }</span>

<input type="text" on:keyup.enter={callMethod($event)} />

<ul [attr.role]="listRole()"></ul>
<ul attr:role={listRole()}></ul>

<ul [class.expanded]="isExpanded()"></ul>
<ul class:expanded={isExpanded()}></ul>

<section [style.height.px]="sectionHeightInPixels()"></section>
<section style:height.px={sectionHeightInPixels()}></section>

<User user={user()} bind:someModel={x} />
<User user={user()} bind:someModel={x} on:someModelChange={method()} />
<User user={user()} use:tooltip(message={msg()} on:dismiss={dismiss()}) />
<User user={user()} use:tooltip(bind:message={msg} on:messageChange={doSomething()}) />

@if (valid()) {
    <MatButton:a
       id="fff"
       href="/admin"
       use:hasRipple
       use:tooltip(message="cannot navigate" disabled={hasPermissions()})>
          Admin
    </MatButton:a>
}
```
