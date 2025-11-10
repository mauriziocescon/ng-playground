# GPT-5
Prompt:

>Hey!

>In readme.md and why-macros.md, I drafted a possible evolution of the angular syntax in templates and proposed a new authoring format.

>Considering such files and the current angular best practices, could you please create an example of components composition using directives and declarations?

>Thanks

Result:
>Below is a concise, self-contained example showing component composition with:

>- a reusable child component,
>- a directive attached to a native element,
>- a declaration used to create a computed value in the template,
>- a parent that composes everything together via inputs, projected children, and modern Angular features (signals, fragments, control flow).

>It follows current Angular best practices:

>- Standalone, signal-based APIs
>- Typed inputs/outputs/models
>- Modern template control flow (@if, @for)
>- Explicit DI and encapsulated directives
>Based on your ideas and current Angular best practices, here is an example of component composition that uses your proposed #component, #directive, and #declaration macros.

```ts
// File: ui.ng
import {
  signal,
  computed,
  input,
  output,
  model,
  inject,
  ref,
  Renderer2,
  LOCALE_ID,
  Fragment,
} from '@angular/core';
import { Render } from '@angular/common';

/* -------------------------
   Declaration: currency()
   ------------------------- */
#declaration currency = () => {
  script: () => {
    const localeId = inject(LOCALE_ID);

    return (
      value: () => number | undefined,
      code: string = 'USD',
    ) =>
      computed(() => {
        const v = value();
        if (v == null) return '';
        return new Intl.NumberFormat(localeId, {
          style: 'currency',
          currency: code,
        }).format(v);
      });
  },
};

/* -------------------------
   Directive: tooltip
   ------------------------- */
export #directive tooltip = ({
  message = input.required<string>(),
  elRef = ref<HTMLElement>(),
}) => {
  script: () => {
    const renderer = inject(Renderer2);

    // Keep it simple: native title attribute
    renderer.setAttribute(elRef(), 'title', message());
  },
};

/* -------------------------
   Child component: PriceTag
   ------------------------- */
export #component PriceTag = ({
  amount = input.required<number>(),
  code = input<string>('USD'),
  children = input<Fragment<void>>(),
}) => {
  template: (
    <>
      @const price = @currency(() => amount(), code());
      <span class="price">
        {price()}
        @if (children()) {
          <span class="suffix"> <Render fragment={children()} /> </span>
        }
      </span>
    </>
  ),
  style: (
    <>
      .price { font-weight: 600; }
      .suffix { color: #666; margin-left: 4px; font-weight: 400; }
    </>
  ),
};

/* -------------------------
   Child component: Card
   ------------------------- */
export #component Card = ({
  title = input.required<string>(),
  children = input<Fragment<void>>(),
}) => {
  template: (
    <>
      <article class="card">
        <header class="card__header">{title()}</header>
        @if (children()) {
          <section class="card__content">
            <Render fragment={children()} />
          </section>
        }
      </article>
    </>
  ),
  style: (
    <>
      .card { border: 1px solid #ddd; border-radius: 8px; padding: 12px; }
      .card__header { font-size: 1.1rem; margin-bottom: 8px; }
      .card__content { color: #333; }
    </>
  ),
};

/* -------------------------
   Parent component: ProductPage
   ------------------------- */
export #component ProductPage = () => {
  script: () => {
    const name = signal('Angular Hoodie');
    const desc = signal('A comfy hoodie for Angular enthusiasts.');
    const price = signal(49.9);
    const qty = model<number>(1);

    const total = computed(() => price() * qty());
  },
  template: (
    <>
      <h1>Product</h1>

      <Card title={name()}>
        <p @tooltip(message={desc()})>
          {desc()}
        </p>

        <div class="row">
          <label>Qty:</label>
          <input type="number" min="1" model:value={qty} />
        </div>

        <div class="row">
          <span>Unit:</span>
          <PriceTag amount={price()}>
            each
          </PriceTag>
        </div>

        <div class="row total">
          <span>Total:</span>
          <PriceTag amount={total()} code="EUR" />
        </div>

        <button class="buy" disabled={total() <= 0}>
          Buy now
        </button>
      </Card>
    </>
  ),
  style: (
    <>
      .row { display: flex; gap: 8px; align-items: center; margin: 8px 0; }
      .total { font-size: 1.05rem; }
      .buy { margin-top: 12px; }
    </>
  ),
};
```

>This example aligns with modern Angular ideas like signals, computed values, and clear dependency injection, while fitting into the new authoring format you've proposed. Let me know if you have any other questions
