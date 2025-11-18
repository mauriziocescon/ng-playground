# Gemini 2.5 pro
## Prompt:

Hey! 

In readme.md and why-macros.md, I drafted a possible evolution of the angular syntax in templates and proposed a new authoring format.

Considering such files and the current angular best practices, could you please create an example of components composition using directives, declarations and fragments?  

Thanks

## Result:

Based on the speculative syntax outlined in your uploaded files (readme.md and why-macros.md), here is a comprehensive example of a Product Showcase feature.

This example demonstrates:

- Directives: To handle DOM manipulations (animations).
- Declarations: To handle reusable template logic (price calculation).
- Fragments: To allow the parent component to compose the content of the child component (inversion of control).

The Scenario: E-Commerce Product List

We are building a list of products where the parent component decides exactly how the "Action Area" (e.g., Add to Cart button) looks for each card using fragments, while the logic for calculating discounts is handled by a declaration, and the entry animation is handled by a directive.

### The Directive (fadeIn) and Declaration (discount)
First, we define the reusable building blocks.

```ts
import { 
  input, 
  ref, 
  inject, 
  computed, 
  ElementRef, 
  afterNextRender 
} from '@angular/core';

// --- DIRECTIVE: Fade In Animation ---
// Manipulates the DOM element to add a class when mounted
export #directive fadeIn = ({
  // duration input with a default value
  duration = input<number>(500),
  elRef = ref<HTMLElement>(),
}) => {
  script: () => {
    afterNextRender(() => {
      const el = elRef();
      if (el) {
        el.style.transition = `opacity ${duration()}ms ease-in`;
        el.style.opacity = '0';
        
        // Trigger reflow/wait a tick
        requestAnimationFrame(() => {
           el.style.opacity = '1';
        });
      }
    });
  },
};

// --- DECLARATION: Discount Calculator ---
// Reusable logic instantiated in the template injection context
export #declaration discount = () => {
  script: () => {
    // Logic that might rely on DI (e.g., current currency settings)
    // const config = inject(AppConfig); 

    // Returns a function acceptable by the template
    return (originalPrice: () => number, discountPct: number) => 
      computed(() => {
        const price = originalPrice();
        return price - (price * (discountPct / 100));
      });
  },
};
```

### The Child Component (ProductCard)
This component handles the layout but delegates the rendering of the specific action button to the parent via a fragment.

```ts
import { input, fragment } from '@angular/core';
import { Render } from '@angular/common';

export interface Product {
  id: string;
  name: string;
  price: number;
}

export #component ProductCard = ({
  data = input.required<Product>(),
  // The parent will provide this HTML fragment
  actionArea = fragment<[Product]>(), 
}) => {
  template: (
    <>
      <div class="card">
        <h3>{data().name}</h3>
        
        <div class="actions">
           <Render fragment={actionArea()} params={[data()]} />
        </div>
      </div>
    </>
  ),
  style: (
    <>
      .card {
        border: 1px solid #ccc;
        padding: 1rem;
        border-radius: 8px;
      }
      .actions {
        margin-top: 10px;
        padding-top: 10px;
        border-top: 1px dashed #eee;
      }
    </>
  )
};
```

### The Parent Component (ProductPage)
This component composes everything together. It uses the fadeIn directive on the list, uses the discount declaration to calculate prices inside the fragment, and passes that fragment down to the ProductCard.

```ts
import { signal, computed } from '@angular/core';
import { fadeIn, discount } from './utils.ng'; // Assuming previous block
import { ProductCard, Product } from './product-card.ng';

export #component ProductPage = () => {
  script: () => {
    const products = signal<Product[]>([
      { id: '1', name: 'Super Gadget', price: 100 },
      { id: '2', name: 'Mega Widget', price: 200 },
    ]);

    const cartCount = signal(0);

    function addToCart(p: Product) {
      console.log(`Added ${p.name}`);
      cartCount.update(c => c + 1);
    }
  },
  template: (
    <>
      <h1>Shop (Items: {cartCount()})</h1>

      <div class="grid">
        @for (p of products(); track p.id) {
          
          <div @fadeIn(duration={800})>
            
            <ProductCard data={p}>
              
              @fragment actionArea(item: Product) {
                
                @const finalPrice = @discount(() => item.price, 20); 
                
                <div class="price-tag">
                  <span class="strikethrough">${item.price}</span>
                  <strong> ${finalPrice()} </strong>
                </div>

                <button 
                  class="btn-primary"
                  on:click={() => addToCart(item)}>
                    Buy {item.name}
                </button>
              }

            </ProductCard>
          </div>
        }
      </div>
    </>
  ),
  style: (
    <>
      .grid { display: grid; gap: 1rem; grid-template-columns: 1fr 1fr; }
      .strikethrough { text-decoration: line-through; color: #999; margin-right: 8px; }
      .btn-primary { background: blue; color: white; }
    </>
  )
};
```

### Analysis of the Syntax Used
- Macro Definitions: We used #component, #directive, and #declaration to define the blocks, which prevents the "unwanted flexibility" of returning objects directly and enforces the structure providers / script / template / style.

- Script & Lexical Scope: Inputs (like data in ProductCard) are immediately available in the script block as signals.

- Fragments:
    1. Defined in the parent using @fragment name(params) { ... }.
    2. Passed to the child implicitly (or explicitly as an input if named).
    3. Rendered in the child using <Render fragment={...} />. This effectively replaces ng-content and ng-template.

- Directives: The @fadeIn directive is applied using the @ syntax (@fadeIn(duration={800})). It uses elRef to access the native element safely.

- Declarations: The @discount declaration creates a reactive value (finalPrice) within the template scope. This acts as a replacement for Pipes, allowing logic that depends on DI (though not used in this specific mock, it runs in an injection context).
