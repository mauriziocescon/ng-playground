# Gemini 3 pro
## Prompt:

Hey! 

In readme.md and why-macros.md, I drafted a possible evolution of the angular syntax in templates and proposed a new authoring format.

Considering such files and the current angular best practices, could you please create an example of components composition using directives, declarations and fragments?  

Thanks

## Result:

Based on the hypothetical syntax outlined in your provided files (why-macros.md and readme.md), here is an example demonstrating component composition.

This example builds a ProductCatalog where:

1. Directives handle user interaction tracking (Analytics).
2. Declarations handle dependency-injected data formatting (Currency conversion).
3. Fragments allow the parent to define exactly how each product card looks, while the child handles the layout logic.

### The Example: E-Commerce Product Catalog
This example demonstrates how a generic Grid component receives a Fragment to render items. That fragment utilizes a Directive for behavior and a Declaration for pure data transformation.

```ts
import { input, output, inject, computed, fragment, LOCALE_ID } from '@angular/core';
import { AnalyticsService, ExchangeRateService } from './services';

// --------------------------------------------------------------------------
// 1. THE DIRECTIVE
// A reusable behavior that can be attached to any element
// --------------------------------------------------------------------------
#directive TrackClick({
  eventName = input.required<string>(),
  elRef = ref<HTMLElement>(), // "elRef" is reserved for the host element
}) {
  script: () => {
    const analytics = inject(AnalyticsService);

    // Logic to bind click listener to elRef and log to analytics
    function track() {
      analytics.logEvent(eventName());
    }
    
    // In a real scenario, we might use event delegation or direct binding here
    return {
      track
    };
  },
}

// --------------------------------------------------------------------------
// 2. THE DECLARATION
// A way to declare 'const' variables in templates that depend on DI
// Effectively replaces Pipes.
// --------------------------------------------------------------------------
#declaration LocalCurrency() {
  script: () => {
    const rates = inject(ExchangeRateService);
    const locale = inject(LOCALE_ID);

    // Returns a function that takes a signal/function and returns a computed signal
    return (
      amount: () => number, 
      currency: string
    ) => computed(() => {
      const converted = amount() * rates.getRate(currency);
      return new Intl.NumberFormat(locale, { style: 'currency', currency }).format(converted);
    });
  },
}

// --------------------------------------------------------------------------
// 3. THE GENERIC COMPONENT
// Uses fragments to delegate rendering logic to the parent
// --------------------------------------------------------------------------
export #component GridLayout({
  items = input.required<any[]>(),
  // "itemTemplate" is a fragment receiving a generic param
  itemTemplate = fragment<[any]>(), 
}) {
  template: `
    <div class="grid-container">
      @for (item of items(); track item.id) {
        <div class="grid-cell">
          <Render fragment={itemTemplate()} params={[item]} />
        </div>
      }
    </div>
  `,
  style: `
    .grid-container { display: grid; grid-template-columns: repeat(3, 1fr); gap: 1rem; }
    .grid-cell { border: 1px solid #ccc; padding: 1rem; }
  `
}

// --------------------------------------------------------------------------
// 4. THE COMPOSITION (Parent Component)
// Combines Component + Fragment + Directive + Declaration
// --------------------------------------------------------------------------
export interface Product {
  id: string;
  name: string;
  priceUSD: number;
}

export #component ProductPage() {
  script: () => {
    // Inputs lifted up immediately available in script
    // Mock data signal
    const products = signal<Product[]>([
      { id: '1', name: 'Laptop', priceUSD: 1000 },
      { id: '2', name: 'Mouse', priceUSD: 50 },
    ]);

    function addToCart(p: Product) {
      console.log('Added', p.name);
    }
  },
  template: `
    <h1>Summer Sale</h1>

    <GridLayout items={products()}>
      
      @fragment card(p: Product) {
        <div class="product-card">
          <h3>{p.name}</h3>
          
          @const localPrice = @LocalCurrency(() => p.priceUSD, 'EUR');
          
          <p>Price: {localPrice()}</p>

          <button 
            on:click={() => addToCart(p)}
            @TrackClick(eventName={'add_to_cart_' + p.id})>
            Add to Cart
          </button>
        </div>
      }

    </GridLayout>
  `,
}
```
Key Composition Features Highlighted: 

1. Scope Isolation: The TrackClick directive and LocalCurrency declaration logic reside in their own scripts with their own DI context. The ProductPage template simply consumes them without needing to instantiate services manually in its own script block. This keeps DI separated from the component script.

2. Fragment parameters: The GridLayout is agnostic of what it renders. It accepts a fragment<[any]>. The ProductPage defines the specific look (@fragment card(p: Product)), effectively passing a "render function" that retains access to the ProductPage's lexical scope (like addToCart).

3. Declarations as reactive variables: Instead of using a pipe inside an interpolation string, @LocalCurrency creates a computed signal (localPrice) via @const. This allows the result to be used multiple times or logic-checked before rendering, offering a cleaner alternative to pipes.

4. Implicit Directive Inputs: The TrackClick directive receives eventName as a reactive input. The syntax @TrackClick(eventName={...}) passes data directly to the directive's script input signal.
