# Gemini 2.5 pro
## Prompt:

Hey! 

In readme.md and why-macros.md, I drafted a possible evolution of the angular syntax in templates and proposed a new authoring format.

Considering such files and the current angular best practices, could you please create an example of components composition using directives and declarations?  

Thanks

## Result:

Based on the new syntax and concepts you've outlined in your readme.md and why-macros.md files, here is an example of component composition that uses both a declaration and a directive.

The scenario is a ProductCard component that:

- Uses a #declaration (@formatPrice) to format a product's price, which requires LOCALE_ID from the dependency injection (DI) system.

- Uses a #directive (@trackView) to notify a parent component or service when the product card's image is viewed (e.g., via on:click).

- Composes these pieces together in its template.

### The Declaration (@formatPrice)
First, we define a #declaration. According to your notes, this is a way to declare a const variable in the template that can use DI. This example creates a price-formatting function that injects the LOCALE_ID.

```ts
// --- format-price.ng.ts ---
import { computed, inject, LOCALE_ID } from '@angular/core';

/**
 * A declaration that creates a computed signal
 * for a formatted currency string, using DI to get the locale.
 *
 */
#declaration formatPrice = () => {
  script: () => {
    // 1. Uses the injection context
    const locale = inject(LOCALE_ID);
    
    // 2. Returns a function to be used in the template
    return (
      price: () => (number | undefined),
      currencyCode: string = 'USD'
    ) => computed(() => {
      const p = price();
      if (p === undefined) {
        return 'Price unavailable';
      }
      return new Intl.NumberFormat(locale, {
        style: 'currency',
        currency: currencyCode,
      }).format(p);
    });
  },
};
```

### The Directive (@trackView)
Next, we define a #directive. This directive will be applied to an element and will emit an output event when clicked. It also gets a reference to its host element (elRef).

```ts
// --- track-view.ng.ts ---
import { input, output, ref, inject, Renderer2, afterRenderEffect } from '@angular/core';

/**
 * A directive that changes the behavior of a DOM element.
 * It attaches a click listener and emits an event.
 */
export #directive trackView = ({
  /**
   * An input to identify what is being viewed.
   */
  itemId = input.required<string>(),
  
  /**
   * An output to notify when the element is clicked.
   */
  viewed = output<string>(),
  
  /**
   * The reference to the host element, managed by Angular.
   */
  elRef = ref<HTMLElement>(),
}) => {
  script: () => {
    const renderer = inject(Renderer2);

    // Use afterRenderEffect to interact with the DOM element
    afterRenderEffect((onCleanup) => {
      const element = elRef(); // Get the native element
      if (!element) return;

      const onClick = ()R => {
        // Emit the itemId when clicked
        viewed.emit(itemId());
      };

      const unlisten = renderer.listen(element, 'click', onClick);
      
      // Clean up the listener when the directive is destroyed
      onCleanup(() => unlisten());
    });
  },
};
```

### The Component (ProductCard)
Finally, the ProductCard component composes these pieces. It imports the declaration and directive, then uses them in its template:

- @const price = @formatPrice(...) to use the declaration.
- @trackView(...) to apply the directive to the image.
```ts
// --- product-card.ng.ts ---
import { signal, input } from '@angular/core';
import { formatPrice } from './format-price.ng';
import { trackView } from './track-view.ng';

// Interface for the product
export interface Product {
  id: string;
  name: string;
  price: number;
  imageUrl: string;
}

export #component ProductCard = ({
  product = input.required<Product>(),
}) => {
  script: () => {
    // This function will be called by the @trackView directive's output
    function onProductViewed(id: string) {
      console.log(`Product ${id} was viewed!`);
      // --> Here you could call an injected AnalyticsService
    }
    
    // Expose the handler to the template
    return { onProductViewed };
  },
  template: (
    <>
      @const price = @formatPrice(
        () => product().price, 
        'EUR'
      );

      <div class="card">
        <img 
          src={product().imageUrl} 
          alt={product().name}
          @trackView(
            itemId={product().id} 
            on:viewed={onProductViewed}
          ) 
        />
        
        <h3>{product().name}</h3>
        
        <p>{price()}</p>
      </div>
    </>
  ),
  style: (
    <>
      .card {
        border: 1px solid #eee;
        border-radius: 8px;
        padding: 1rem;
        box-shadow: 0 2px 4px rgba(0,0,0,0.1);
      }
      img {
        width: 100%;
        aspect-ratio: 16 / 9;
        object-fit: cover;
        cursor: pointer;
      }
    </>
  ),
};
```
