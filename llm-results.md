# LLM Results
Prompt (gemini 2.5 pro):

>Hey! 

>In readme.md and alternatives.md, I drafted a possible evolution of the angular syntax in templates and proposed a new authoring format. 

>Considering such files and the current angular best practices, could you please create an example of components composition using directives and declarations?  

>Thanks

Result:
>Of course! I've reviewed the readme.md and alternatives.md files describing the new authoring format. It's an interesting evolution of Angular's syntax!

>To demonstrate component composition, I'll create a simple product page. This example will include:
>- A currency declaration to format product prices.
>- A highlight directive to visually mark products that are on sale.
>- A ProductCardComponent that displays a single product and is designed to be composed.
>- A ProductPageComponent that uses all of the above to display a list of products.

>Here are the files for the example:

>First, a simple data model for our products in product.model.ts:

```ts
export interface Product {
  id: number;
  name: string;
  price: number;
  onSale: boolean;
}
```
>Next, I'll create the `currency` declaration in `currency.declaration.ts`. This will be used to format the product price in the template. It uses dependency injection to get the current locale.

```ts
import { declaration, computed, inject, LOCALE_ID } from '@angular/core';
import { formatCurrency } from '@angular/common';

export declaration currency = () => {
  script: () => {
    const localeId = inject(LOCALE_ID);

    return (
      value: () => number | undefined,
      currencyCode: () => string,
      display: 'symbol' | 'narrowSymbol' | 'code' | 'symbol-narrow' | 'name' = 'symbol'
    ) => computed(() => {
      const v = value();
      if (v === undefined) {
        return '';
      }
      return formatCurrency(v, localeId, display, currencyCode());
    });
  },
};
```

>Here's the `highlight` directive in `highlight.directive.ts`. It will change the background color of the element it's applied to.

```ts
import { directive, input, ref, inject, Renderer2, effect } from '@angular/core';

export directive highlight = ({
  color = input('yellow'),
  elRef = ref<HTMLElement>(),
}) => {
  script: () => {
    const renderer = inject(Renderer2);

    effect(() => {
      renderer.setStyle(elRef(), 'backgroundColor', color());
      renderer.setStyle(elRef(), 'transition', 'background-color 0.5s');
    });
  },
};
```

>Now, the `ProductCardComponent` in `product-card.component.ng.ts`. This component is designed for composition. It accepts a `product` input, a `children` fragment for custom content, and uses `fallthroughAttrs` with `@**` to allow directives like `highlight` to be passed down from a parent to its own template elements.

```ts
import { component, input, fallthroughAttrs, Fragment } from '@angular/core';
import { HTMLAttributes } from '@angular/core/elements';
import { Render } from '@angular/common';
import { type Product } from './product.model';

export component ProductCardComponent = ({
  product = input.required<Product>(),
  children = input<Fragment<void>>(),
  attrs = fallthroughAttrs<HTMLAttributes>(),
}) => {
  template: (
    <>
      <div class="product-card" @** bind:**={attrs.in} on:**={attrs.on}>
        <h3>{product().name}</h3>
        @if (children()) {
          <div class="content">
            <Render fragment={children()} />
          </div>
        }
      </div>
    </>
  ),
  style: (
    <>
      .product-card {
        border: 1px solid #ccc;
        padding: 16px;
        margin: 8px;
        border-radius: 8px;
      }
      .content {
        margin-top: 1rem;
      }
    </>
  ),
};
```

>Finally, the `ProductPageComponent` in `product-page.component.ng.ts` brings everything together. It defines a list of products, uses the `@currency` declaration to format prices, and applies the `@highlight` directive to the `ProductCardComponent` when a product is on sale.

```ts
import { component, signal } from '@angular/core';
import { ProductCardComponent } from './product-card.component.ng';
import { highlight } from './highlight.directive';
import { currency } from './currency.declaration';
import { type Product } from './product.model';

export component ProductPageComponent = () => {
  script: () => {
    const products = signal<Product[]>([
      { id: 1, name: 'Fancy Keyboard', price: 99, onSale: true },
      { id: 2, name: 'Ergonomic Mouse', price: 75, onSale: false },
      { id: 3, name: '4K Monitor', price: 499, onSale: true },
    ]);

    function addToCart(product: Product) {
      console.log(`${product.name} added to cart!`);
    }
    
    function toggleSale(product: Product) {
      products.update(ps => ps.map(p => p.id === product.id ? {...p, onSale: !p.onSale} : p));
    }
  },
  template: (
    <>
      <h1>Products</h1>
      <div class="product-list">
        @for (product of products(); track product.id) {
          @const formattedPrice = @currency(() => product.price, () => 'USD');
          
          <ProductCardComponent 
            product={product}
            @highlight(color={product.onSale ? 'lightgreen' : 'transparent'})
            on:click={() => toggleSale(product)}>
            
            <p>Price: {formattedPrice()}</p>
            
            <button on:click={(e) => { e.stopPropagation(); addToCart(product)}}>Add to Cart</button>
          </ProductCardComponent>
        }
      </div>
    </>
  ),
  style: (
    <>
      .product-list {
        display: flex;
        flex-wrap: wrap;
      }
    </>
  ),
};
```

>This example illustrates how directives and declarations can be combined with component composition to create flexible and reusable UIs, following the elegant syntax you've proposed. Let me know if you have any other questions
