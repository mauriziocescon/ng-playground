# LLM Results
Prompt (gemini 2.5 pro):

>Hey! 

>In readme.md and why-keywords.md, I drafted a possible evolution of the angular syntax in templates and proposed a new authoring format.

>Considering such files and the current angular best practices, could you please create an example of components composition using directives and declarations?  

>Thanks

Result:
>Of course! I've reviewed your documents and the proposed syntax. It's an interesting take on evolving Angular's authoring format, emphasizing explicitness and reducing magic.

>Based on your ideas and current Angular best practices, here is an example of component composition that uses your proposed #component, #directive, and #declaration keywords.

>This example showcases:
>- A currency declaration to format prices.
>- A tooltip directive to show hints.
>- A Card component for UI structure.
>- A main ProductPage component that composes everything.

>I've combined them into a single file for clarity.
```ts
import {
  signal,
  computed,
  input,
  output,
  ref,
  inject,
  afterRenderEffect,
  Renderer2,
  LOCALE_ID,
  Fragment,
} from '@angular/core';
import { Render } from '@angular/common';

// 1. DECLARATION
// A declaration for creating a computed currency value in a template.
// It uses dependency injection to get the current locale.

#declaration currency = () => {
  script: () => {
    const localeId = inject(LOCALE_ID);

    return (
      value: () => (number | undefined),
      currencyCode: string = 'USD',
    ) => computed(() => {
      const val = value();
      if (val === undefined) {
        return '';
      }
      return new Intl.NumberFormat(localeId, {
        style: 'currency',
        currency: currencyCode,
      }).format(val);
    });
  },
};

// 2. DIRECTIVE
// A directive that adds a simple browser tooltip to an element.
// It receives the message via an input.

export #directive tooltip = ({
  message = input.required<string>(),
  elRef = ref<HTMLElement>(),
}) => {
  script: () => {
    const renderer = inject(Renderer2);

    afterRenderEffect(() => {
      // In a real scenario, you might create a custom tooltip element,
      // but for this example, we'll just use the native `title` attribute.
      renderer.setAttribute(elRef(), 'title', message());
    });
  },
};

// 3. CHILD COMPONENT
// A simple, reusable Card component with a title input
// and a slot for content projection (`children`).

export #component Card = ({
  title = input.required<string>(),
  children = input<Fragment<void>>(),
}) => {
  template: (
    <>
      <div class="card">
        <h2>{title()}</h2>
        @if (children()) {
          <div class="card-content">
            <Render fragment={children()} />
          </div>
        }
      </div>
    </>
  ),
  style: (
    <>
      .card {
        border: 1px solid #ccc;
        border-radius: 8px;
        padding: 16px;
        margin: 8px;
        box-shadow: 0 2px 4px rgba(0,0,0,0.1);
        max-width: 300px;
      }
      .card-content {
        margin-top: 12px;
        font-style: italic;
      }
      h2 {
        margin-top: 0;
      }
    </>
  )
};


// 4. PARENT COMPONENT (COMPOSITION ROOT)
// The main component that composes the Card, tooltip, and currency declaration.

export #component ProductPage = () => {
  script: () => {
    const product = signal({
      name: 'Awesome Gadget',
      description: 'A truly awesome gadget that will solve all your problems. Hover for details!',
      price: 99.99,
    });
  },
  template: (
    <>
      <h1>Product Details</h1>

      <Card title={product().name}>
        {/* The tooltip directive is applied to this paragraph */}
        <p @tooltip(message={product().description})>
          {product().description}
        </p>

        {/* The currency declaration is used to create a formatted price constant */}
        @const price = @currency(() => product().price, 'EUR');
        <p><strong>Price:</strong> {price()}</p>
      </Card>
    </>
  ),
};
```

>This example aligns with modern Angular ideas like signals, computed values, and clear dependency injection, while fitting into the new authoring format you've proposed. Let me know if you have any other questions
