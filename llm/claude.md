# Claude code
## Prompt:

Hey! 

In readme.md and why-macros.md, I drafted a possible evolution of the angular syntax in templates and proposed a new authoring format.

Considering such files and the current angular best practices, could you please create an example of components composition using directives, declarations and fragments?  

Thanks

## Result:

```ts
/**
 * Comprehensive Component Composition Example
 *
 * This example demonstrates:
 * - Fragments for content projection and customization
 * - Directives for behavior enhancement
 * - Declarations for reusable template logic with DI
 * - Component composition with spread syntax
 * - Best practices for Angular signal components
 */

import {
  signal,
  computed,
  linkedSignal,
  input,
  output,
  model,
  fragment,
  inject,
  provide,
  injectionToken,
  ref,
  afterRenderEffect,
  Renderer2,
  Signal,
} from '@angular/core';
import { Render } from '@angular/common';

// ============================================================================
// DATA MODELS
// ============================================================================

export interface Product {
  id: string;
  name: string;
  description: string;
  price: number;
  inStock: boolean;
  imageUrl: string;
}

export interface CartItem {
  product: Product;
  quantity: number;
}

// ============================================================================
// DIRECTIVES
// ============================================================================

/**
 * Highlight directive - changes background on hover
 * Demonstrates: directive with inputs and host manipulation
 */
export #directive highlight({
  color = input<string>('yellow'),
  intensity = input<number>(0.3),
}) {
  script: ({ host }) => {
    const renderer = inject(Renderer2);
    let originalBackground = '';

    afterRenderEffect(() => {
      const element = host();
      if (!element) return;

      originalBackground = element.style.backgroundColor;

      const enter = () => {
        const rgb = colorToRgb(color());
        renderer.setStyle(
          element,
          'background-color',
          `rgba(${rgb.r}, ${rgb.g}, ${rgb.b}, ${intensity()})`
        );
      };

      const leave = () => {
        renderer.setStyle(element, 'background-color', originalBackground);
      };

      element.addEventListener('mouseenter', enter);
      element.addEventListener('mouseleave', leave);

      return () => {
        element.removeEventListener('mouseenter', enter);
        element.removeEventListener('mouseleave', leave);
      };
    });
  },
}

/**
 * Click outside directive - triggers callback when clicking outside element
 * Demonstrates: directive with outputs
 */
export #directive clickOutside({
  trigger = output<void>(),
}) {
  script: ({ host }) => {
    afterRenderEffect(() => {
      const element = host();
      if (!element) return;

      const handler = (event: MouseEvent) => {
        if (!element.contains(event.target as Node)) {
          trigger.emit();
        }
      };

      document.addEventListener('click', handler);
      return () => document.removeEventListener('click', handler);
    });
  },
}

/**
 * Animate in directive - adds entrance animation
 * Demonstrates: directive with lifecycle and ref exports
 */
export #directive animateIn({
  duration = input<number>(300),
  delay = input<number>(0),
}) {
  script: ({ host }) => {
    const renderer = inject(Renderer2);
    const isVisible = signal(false);

    afterRenderEffect(() => {
      const element = host();
      if (!element) return;

      renderer.setStyle(element, 'opacity', '0');
      renderer.setStyle(element, 'transform', 'translateY(20px)');
      renderer.setStyle(element, 'transition', `all ${duration()}ms ease-out ${delay()}ms`);

      setTimeout(() => {
        renderer.setStyle(element, 'opacity', '1');
        renderer.setStyle(element, 'transform', 'translateY(0)');
        isVisible.set(true);
      }, 50);
    });

    return {
      exports: {
        isVisible: isVisible.asReadonly(),
        reset: () => isVisible.set(false),
      },
    };
  },
}

// ============================================================================
// DECLARATIONS
// ============================================================================

/**
 * Currency formatter declaration
 * Demonstrates: declaration with DI for reusable template logic
 */
#declaration formatCurrency() {
  script: () => {
    const localeId = inject(LOCALE_ID);

    return (
      value: () => number | undefined,
      currencyCode = 'USD',
    ) => computed(() => {
      const v = value();
      if (v === undefined) return 'N/A';

      return new Intl.NumberFormat(localeId, {
        style: 'currency',
        currency: currencyCode,
      }).format(v);
    });
  },
}

/**
 * Date formatter declaration
 * Demonstrates: declaration for complex formatting logic
 */
#declaration formatDate() {
  script: () => {
    const localeId = inject(LOCALE_ID);

    return (
      value: () => Date | undefined,
      format: 'short' | 'medium' | 'long' = 'medium',
    ) => computed(() => {
      const v = value();
      if (!v) return 'N/A';

      const options: Intl.DateTimeFormatOptions = format === 'short'
        ? { month: 'short', day: 'numeric' }
        : format === 'medium'
        ? { month: 'short', day: 'numeric', year: 'numeric' }
        : { weekday: 'long', month: 'long', day: 'numeric', year: 'numeric' };

      return new Intl.DateTimeFormat(localeId, options).format(v);
    });
  },
}

/**
 * Debounced value declaration
 * Demonstrates: declaration with reactive state management
 */
#declaration debounced() {
  script: () => {
    return <T>(
      value: () => T,
      delay = 300,
    ) => {
      const debouncedValue = signal<T>(value());
      let timeoutId: ReturnType<typeof setTimeout> | null = null;

      afterRenderEffect(() => {
        if (timeoutId) clearTimeout(timeoutId);
        timeoutId = setTimeout(() => {
          debouncedValue.set(value());
        }, delay);
      });

      return debouncedValue.asReadonly();
    };
  },
}

// ============================================================================
// REUSABLE UI COMPONENTS
// ============================================================================

/**
 * Card component - generic container with optional header/footer
 * Demonstrates: fragments for flexible content projection
 */
export #component Card({
  elevated = input<boolean>(false),
  padding = input<number>(16),
  header = fragment<void>(),
  footer = fragment<void>(),
}) {
  script: ({ children }) => {
    const classes = computed(() => ({
      'card': true,
      'card-elevated': elevated(),
    }));

    return {
      template: `
        <div class={classNames(classes())} style:padding={padding() + 'px'}>
          @if (header()) {
            <div class="card-header">
              <Render fragment={header()} />
            </div>
          }

          <div class="card-body">
            <Render fragment={children()} />
          </div>

          @if (footer()) {
            <div class="card-footer">
              <Render fragment={footer()} />
            </div>
          }
        </div>
      `,
      style: `
        .card {
          border: 1px solid #e0e0e0;
          border-radius: 8px;
          background: white;
        }

        .card-elevated {
          box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
        }

        .card-header {
          padding-bottom: 12px;
          border-bottom: 1px solid #e0e0e0;
          margin-bottom: 12px;
        }

        .card-footer {
          padding-top: 12px;
          border-top: 1px solid #e0e0e0;
          margin-top: 12px;
        }
      `,
    };
  },
}

/**
 * Button component - wraps native button with directives support
 * Demonstrates: spread syntax and directive forwarding
 */
export #component Button({
  variant = input<'primary' | 'secondary' | 'danger'>('primary'),
  size = input<'small' | 'medium' | 'large'>('medium'),
  disabled = input<boolean>(false),
  loading = input<boolean>(false),
  click = output<void>(),
  ...rest,
}) {
  script: ({ children }) => {
    const dirs = retrieveDirectives(rest);

    const classes = computed(() => ({
      'btn': true,
      [`btn-${variant()}`]: true,
      [`btn-${size()}`]: true,
      'btn-loading': loading(),
    }));

    return {
      template: `
        <button
          @**={dirs()}
          class={classNames(classes())}
          disabled={disabled() || loading()}
          on:click={() => click.emit()}>

          @if (loading()) {
            <span class="spinner"></span>
          }

          <Render fragment={children()} />
        </button>
      `,
      style: `
        .btn {
          padding: 8px 16px;
          border: none;
          border-radius: 4px;
          cursor: pointer;
          font-weight: 500;
          transition: all 200ms;
        }

        .btn:hover:not(:disabled) {
          transform: translateY(-1px);
          box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
        }

        .btn:disabled {
          opacity: 0.6;
          cursor: not-allowed;
        }

        .btn-primary { background: #007bff; color: white; }
        .btn-secondary { background: #6c757d; color: white; }
        .btn-danger { background: #dc3545; color: white; }

        .btn-small { font-size: 12px; padding: 4px 8px; }
        .btn-medium { font-size: 14px; padding: 8px 16px; }
        .btn-large { font-size: 16px; padding: 12px 24px; }

        .spinner {
          display: inline-block;
          width: 12px;
          height: 12px;
          border: 2px solid rgba(255, 255, 255, 0.3);
          border-top-color: white;
          border-radius: 50%;
          animation: spin 0.6s linear infinite;
          margin-right: 8px;
        }

        @keyframes spin {
          to { transform: rotate(360deg); }
        }
      `,
    };
  },
}

/**
 * List component - generic list with customizable item rendering
 * Demonstrates: fragments for item customization + empty state
 */
export #component List<T extends { id: string }>({
  items = input.required<T[]>(),
  item = fragment<[T, number]>(),
  emptyState = fragment<void>(),
  gap = input<number>(8),
}) {
  script: () => ({
    template: `
      @if (items().length > 0) {
        <div class="list" style:gap={gap() + 'px'}>
          @for (i of items(); track i.id; let idx = $index) {
            <Render fragment={item()} params={[i, idx]} />
          }
        </div>
      } @else {
        @if (emptyState()) {
          <Render fragment={emptyState()} />
        } @else {
          <div class="empty-state">No items to display</div>
        }
      }
    `,
    style: `
      .list {
        display: flex;
        flex-direction: column;
      }

      .empty-state {
        padding: 32px;
        text-align: center;
        color: #6c757d;
      }
    `,
  }),
}

/**
 * Modal component - overlay with click-outside support
 * Demonstrates: directives on components + portal-like behavior
 */
export #component Modal({
  open = model.required<boolean>(),
  title = input<string>(''),
  actions = fragment<void>(),
}) {
  script: ({ children }) => {
    function close() {
      open.set(false);
    }

    return {
      template: `
        @if (open()) {
          <div class="modal-backdrop" @animateIn(duration={200})>
            <div
              class="modal-content"
              @clickOutside(on:trigger={close})
              @animateIn(duration={300} delay={100})>

              <div class="modal-header">
                <h2>{title()}</h2>
                <button class="close-btn" on:click={close}>×</button>
              </div>

              <div class="modal-body">
                <Render fragment={children()} />
              </div>

              @if (actions()) {
                <div class="modal-actions">
                  <Render fragment={actions()} />
                </div>
              }
            </div>
          </div>
        }
      `,
      style: `
        .modal-backdrop {
          position: fixed;
          top: 0;
          left: 0;
          right: 0;
          bottom: 0;
          background: rgba(0, 0, 0, 0.5);
          display: flex;
          align-items: center;
          justify-content: center;
          z-index: 1000;
        }

        .modal-content {
          background: white;
          border-radius: 8px;
          max-width: 600px;
          width: 90%;
          max-height: 80vh;
          overflow: auto;
        }

        .modal-header {
          display: flex;
          justify-content: space-between;
          align-items: center;
          padding: 16px 24px;
          border-bottom: 1px solid #e0e0e0;
        }

        .modal-header h2 {
          margin: 0;
          font-size: 20px;
        }

        .close-btn {
          background: none;
          border: none;
          font-size: 28px;
          cursor: pointer;
          color: #6c757d;
          line-height: 1;
        }

        .close-btn:hover {
          color: #000;
        }

        .modal-body {
          padding: 24px;
        }

        .modal-actions {
          padding: 16px 24px;
          border-top: 1px solid #e0e0e0;
          display: flex;
          justify-content: flex-end;
          gap: 8px;
        }
      `,
    };
  },
}

// ============================================================================
// BUSINESS LOGIC - CART STORE
// ============================================================================

const CartStoreToken = injectionToken('CartStore', {
  factory: () => new CartStore(),
});

class CartStore {
  private readonly items = signal<CartItem[]>([]);
  readonly cartItems = this.items.asReadonly();

  readonly total = computed(() => {
    return this.items().reduce(
      (sum, item) => sum + item.product.price * item.quantity,
      0
    );
  });

  readonly itemCount = computed(() => {
    return this.items().reduce((sum, item) => sum + item.quantity, 0);
  });

  addItem(product: Product) {
    const existing = this.items().find(i => i.product.id === product.id);

    if (existing) {
      this.updateQuantity(product.id, existing.quantity + 1);
    } else {
      this.items.update(items => [...items, { product, quantity: 1 }]);
    }
  }

  removeItem(productId: string) {
    this.items.update(items => items.filter(i => i.product.id !== productId));
  }

  updateQuantity(productId: string, quantity: number) {
    if (quantity <= 0) {
      this.removeItem(productId);
      return;
    }

    this.items.update(items =>
      items.map(item =>
        item.product.id === productId
          ? { ...item, quantity }
          : item
      )
    );
  }

  clear() {
    this.items.set([]);
  }
}

// ============================================================================
// DOMAIN COMPONENTS - PRODUCT DISPLAY
// ============================================================================

/**
 * Product card component
 * Demonstrates: composition with directives and declarations
 */
export #component ProductCard({
  product = input.required<Product>(),
  addToCart = output<Product>(),
  showDetails = output<Product>(),
}) {
  script: () => {
    const isAdding = signal(false);

    async function handleAddToCart() {
      isAdding.set(true);
      // Simulate async operation
      await new Promise(resolve => setTimeout(resolve, 500));
      addToCart.emit(product());
      isAdding.set(false);
    }

    return {
      template: `
        <Card elevated={true}>
          @fragment header() {
            <div
              class="product-image"
              @highlight(color={'blue'} intensity={0.1})
              @animateIn(duration={400})
              style:background-image={'url(' + product().imageUrl + ')'}>
            </div>
          }

          <div class="product-info">
            <h3>{product().name}</h3>
            <p class="product-description">{product().description}</p>

            @const formattedPrice = @formatCurrency(() => product().price, 'USD');
            <p class="product-price">{formattedPrice()}</p>

            @if (!product().inStock) {
              <p class="out-of-stock">Out of Stock</p>
            }
          </div>

          @fragment footer() {
            <div class="product-actions">
              <Button
                variant={'secondary'}
                size={'small'}
                on:click={() => showDetails.emit(product())}>
                View Details
              </Button>

              <Button
                variant={'primary'}
                size={'small'}
                disabled={!product().inStock}
                loading={isAdding()}
                on:click={handleAddToCart}>
                Add to Cart
              </Button>
            </div>
          }
        </Card>
      `,
      style: `
        .product-image {
          width: 100%;
          height: 200px;
          background-size: cover;
          background-position: center;
          border-radius: 4px;
        }

        .product-info h3 {
          margin: 0 0 8px 0;
          font-size: 18px;
        }

        .product-description {
          color: #6c757d;
          font-size: 14px;
          margin: 0 0 12px 0;
        }

        .product-price {
          font-size: 20px;
          font-weight: bold;
          color: #007bff;
          margin: 0;
        }

        .out-of-stock {
          color: #dc3545;
          font-weight: 500;
          margin: 8px 0 0 0;
        }

        .product-actions {
          display: flex;
          gap: 8px;
          justify-content: space-between;
        }
      `,
    };
  },
}

/**
 * Product list with search
 * Demonstrates: complex composition with multiple features
 */
export #component ProductList({
  products = input.required<Product[]>(),
  productDetails = output<Product>(),
}) {
  script: () => {
    const cart = inject(CartStoreToken);
    const searchTerm = signal('');

    // Use declaration for debouncing
    const debouncedSearch = computed(() => searchTerm());

    const filteredProducts = computed(() => {
      const term = debouncedSearch().toLowerCase();
      if (!term) return products();

      return products().filter(p =>
        p.name.toLowerCase().includes(term) ||
        p.description.toLowerCase().includes(term)
      );
    });

    function handleAddToCart(product: Product) {
      cart.addItem(product);
    }

    return {
      template: `
        <div class="product-list-container">
          <div class="search-bar">
            <input
              type="text"
              placeholder="Search products..."
              model:value={searchTerm}
              class="search-input" />

            @if (searchTerm()) {
              <button
                class="clear-search"
                on:click={() => searchTerm.set('')}>
                Clear
              </button>
            }
          </div>

          <p class="results-count">
            Showing {filteredProducts().length} of {products().length} products
          </p>

          <List items={filteredProducts()} gap={16}>
            @fragment item(product: Product, index: number) {
              <div @animateIn(delay={index * 50}>
                <ProductCard
                  product={product}
                  on:addToCart={(p) => handleAddToCart(p)}
                  on:showDetails={(p) => productDetails.emit(p)} />
              </div>
            }

            @fragment emptyState() {
              <div class="no-results">
                <p>No products found matching "{searchTerm()}"</p>
                <Button
                  variant={'secondary'}
                  on:click={() => searchTerm.set('')}>
                  Clear Search
                </Button>
              </div>
            }
          </List>
        </div>
      `,
      style: `
        .product-list-container {
          max-width: 1200px;
          margin: 0 auto;
        }

        .search-bar {
          display: flex;
          gap: 8px;
          margin-bottom: 16px;
        }

        .search-input {
          flex: 1;
          padding: 12px 16px;
          border: 2px solid #e0e0e0;
          border-radius: 4px;
          font-size: 16px;
        }

        .search-input:focus {
          outline: none;
          border-color: #007bff;
        }

        .clear-search {
          padding: 12px 24px;
          background: #dc3545;
          color: white;
          border: none;
          border-radius: 4px;
          cursor: pointer;
        }

        .results-count {
          color: #6c757d;
          margin-bottom: 16px;
        }

        .no-results {
          text-align: center;
          padding: 48px;
        }

        .no-results p {
          margin-bottom: 16px;
          font-size: 18px;
          color: #6c757d;
        }
      `,
    };
  },
}

// ============================================================================
// DOMAIN COMPONENTS - CART
// ============================================================================

/**
 * Cart item component
 * Demonstrates: model binding and event handling
 */
export #component CartItemRow({
  item = input.required<CartItem>(),
  quantityChange = output<number>(),
  remove = output<void>(),
}) {
  script: () => {
    const quantity = linkedSignal(() => item().quantity);

    function handleQuantityChange() {
      quantityChange.emit(quantity());
    }

    return {
      template: `
        <div class="cart-item" @highlight(color={'green'} intensity={0.05})>
          <div class="item-info">
            <img
              src={item().product.imageUrl}
              alt={item().product.name}
              class="item-image" />

            <div>
              <h4>{item().product.name}</h4>

              @const price = @formatCurrency(() => item().product.price, 'USD');
              <p class="item-price">{price()}</p>
            </div>
          </div>

          <div class="item-controls">
            <div class="quantity-control">
              <button
                class="qty-btn"
                on:click={() => {quantity.update(q => Math.max(1, q - 1)); handleQuantityChange();}}>
                −
              </button>

              <input
                type="number"
                model:value={quantity}
                on:input={handleQuantityChange}
                min={1}
                class="qty-input" />

              <button
                class="qty-btn"
                on:click={() => {quantity.update(q => q + 1); handleQuantityChange();}}>
                +
              </button>
            </div>

            @const total = @formatCurrency(() => item().product.price * quantity(), 'USD');
            <p class="item-total">{total()}</p>

            <button
              class="remove-btn"
              on:click={() => remove.emit()}>
              Remove
            </button>
          </div>
        </div>
      `,
      style: `
        .cart-item {
          display: flex;
          justify-content: space-between;
          align-items: center;
          padding: 16px;
          border: 1px solid #e0e0e0;
          border-radius: 4px;
        }

        .item-info {
          display: flex;
          gap: 16px;
          align-items: center;
        }

        .item-image {
          width: 80px;
          height: 80px;
          object-fit: cover;
          border-radius: 4px;
        }

        .item-info h4 {
          margin: 0 0 8px 0;
        }

        .item-price {
          color: #6c757d;
          margin: 0;
        }

        .item-controls {
          display: flex;
          align-items: center;
          gap: 16px;
        }

        .quantity-control {
          display: flex;
          align-items: center;
          gap: 8px;
        }

        .qty-btn {
          width: 32px;
          height: 32px;
          border: 1px solid #e0e0e0;
          background: white;
          border-radius: 4px;
          cursor: pointer;
          font-size: 18px;
        }

        .qty-btn:hover {
          background: #f8f9fa;
        }

        .qty-input {
          width: 60px;
          padding: 8px;
          text-align: center;
          border: 1px solid #e0e0e0;
          border-radius: 4px;
        }

        .item-total {
          font-weight: bold;
          min-width: 80px;
          text-align: right;
          margin: 0;
        }

        .remove-btn {
          padding: 8px 16px;
          background: #dc3545;
          color: white;
          border: none;
          border-radius: 4px;
          cursor: pointer;
        }

        .remove-btn:hover {
          background: #c82333;
        }
      `,
    };
  },
}

/**
 * Shopping cart component
 * Demonstrates: state management with store injection
 */
export #component ShoppingCart() {
  script: () => {
    const cart = inject(CartStoreToken);
    const modalOpen = signal(false);

    function updateQuantity(productId: string, quantity: number) {
      cart.updateQuantity(productId, quantity);
    }

    function removeItem(productId: string) {
      cart.removeItem(productId);
    }

    function handleCheckout() {
      modalOpen.set(true);
    }

    function confirmCheckout() {
      // Simulate checkout
      cart.clear();
      modalOpen.set(false);
    }

    return {
      template: `
        <div class="shopping-cart">
          <div class="cart-header">
            <h2>Shopping Cart</h2>
            <span class="item-count">
              {cart.itemCount()} {cart.itemCount() === 1 ? 'item' : 'items'}
            </span>
          </div>

          <List items={cart.cartItems()} gap={12}>
            @fragment item(cartItem: CartItem) {
              <CartItemRow
                item={cartItem}
                on:quantityChange={(qty) => updateQuantity(cartItem.product.id, qty)}
                on:remove={() => removeItem(cartItem.product.id)} />
            }

            @fragment emptyState() {
              <div class="empty-cart">
                <p>Your cart is empty</p>
              </div>
            }
          </List>

          @if (cart.cartItems().length > 0) {
            <div class="cart-footer">
              @const totalFormatted = @formatCurrency(() => cart.total(), 'USD');

              <div class="total-row">
                <span class="total-label">Total:</span>
                <span class="total-amount">{totalFormatted()}</span>
              </div>

              <Button
                variant={'primary'}
                size={'large'}
                on:click={handleCheckout}>
                Proceed to Checkout
              </Button>
            </div>
          }

          <Modal model:open={modalOpen} title={'Checkout Confirmation'}>
            <p>Are you sure you want to complete this purchase?</p>

            @const total = @formatCurrency(() => cart.total(), 'USD');
            <p class="checkout-total">Total: {total()}</p>

            @fragment actions() {
              <Button
                variant={'secondary'}
                on:click={() => modalOpen.set(false)}>
                Cancel
              </Button>

              <Button
                variant={'primary'}
                on:click={confirmCheckout}>
                Confirm Purchase
              </Button>
            }
          </Modal>
        </div>
      `,
      style: `
        .shopping-cart {
          max-width: 800px;
          margin: 0 auto;
        }

        .cart-header {
          display: flex;
          justify-content: space-between;
          align-items: center;
          margin-bottom: 24px;
        }

        .cart-header h2 {
          margin: 0;
        }

        .item-count {
          color: #6c757d;
          font-size: 14px;
        }

        .empty-cart {
          text-align: center;
          padding: 48px;
          color: #6c757d;
        }

        .cart-footer {
          margin-top: 24px;
          padding: 24px;
          background: #f8f9fa;
          border-radius: 4px;
        }

        .total-row {
          display: flex;
          justify-content: space-between;
          align-items: center;
          margin-bottom: 16px;
          font-size: 20px;
        }

        .total-label {
          font-weight: 500;
        }

        .total-amount {
          font-weight: bold;
          color: #007bff;
        }

        .checkout-total {
          font-size: 18px;
          font-weight: bold;
          color: #007bff;
        }
      `,
    };
  },
}

// ============================================================================
// MAIN APP COMPONENT
// ============================================================================

/**
 * Main application component
 * Demonstrates: full composition with all features
 */
export #component ShoppingApp() {
  script: () => {
    const selectedProduct = signal<Product | null>(null);

    // Mock product data
    const products = signal<Product[]>([
      {
        id: '1',
        name: 'Wireless Headphones',
        description: 'Premium noise-cancelling headphones',
        price: 299.99,
        inStock: true,
        imageUrl: 'https://via.placeholder.com/300x200?text=Headphones',
      },
      {
        id: '2',
        name: 'Smart Watch',
        description: 'Fitness tracker with heart rate monitor',
        price: 399.99,
        inStock: true,
        imageUrl: 'https://via.placeholder.com/300x200?text=Watch',
      },
      {
        id: '3',
        name: 'Laptop Stand',
        description: 'Ergonomic aluminum laptop stand',
        price: 59.99,
        inStock: false,
        imageUrl: 'https://via.placeholder.com/300x200?text=Stand',
      },
    ]);

    return {
      template: `
        <div class="app-container">
          <header class="app-header">
            <h1>Shopping Demo</h1>
            <p>Component Composition Example</p>
          </header>

          <main class="app-main">
            <section class="products-section">
              <ProductList
                products={products()}
                on:productDetails={(p) => selectedProduct.set(p)} />
            </section>

            <aside class="cart-section">
              <ShoppingCart />
            </aside>
          </main>

          <Modal
            model:open={signal(selectedProduct() !== null)}
            title={selectedProduct()?.name ?? ''}>

            @if (selectedProduct()) {
              <div class="product-details">
                <img
                  src={selectedProduct()!.imageUrl}
                  alt={selectedProduct()!.name}
                  class="detail-image" />

                <p>{selectedProduct()!.description}</p>

                @const price = @formatCurrency(() => selectedProduct()!.price, 'USD');
                <p class="detail-price">{price()}</p>

                <p class="stock-status">
                  {selectedProduct()!.inStock ? 'In Stock' : 'Out of Stock'}
                </p>
              </div>
            }

            @fragment actions() {
              <Button
                variant={'primary'}
                on:click={() => selectedProduct.set(null)}>
                Close
              </Button>
            }
          </Modal>
        </div>
      `,
      style: `
        .app-container {
          min-height: 100vh;
          background: #f8f9fa;
        }

        .app-header {
          background: white;
          padding: 24px;
          border-bottom: 1px solid #e0e0e0;
          text-align: center;
        }

        .app-header h1 {
          margin: 0 0 8px 0;
          color: #007bff;
        }

        .app-header p {
          margin: 0;
          color: #6c757d;
        }

        .app-main {
          display: grid;
          grid-template-columns: 2fr 1fr;
          gap: 24px;
          padding: 24px;
          max-width: 1400px;
          margin: 0 auto;
        }

        @media (max-width: 1024px) {
          .app-main {
            grid-template-columns: 1fr;
          }
        }

        .product-details {
          text-align: center;
        }

        .detail-image {
          width: 100%;
          max-width: 400px;
          border-radius: 8px;
          margin-bottom: 16px;
        }

        .detail-price {
          font-size: 24px;
          font-weight: bold;
          color: #007bff;
        }

        .stock-status {
          font-weight: 500;
          color: #28a745;
        }
      `,
    };
  },
  providers: [
    provide({ token: CartStoreToken, useFactory: () => new CartStore() }),
  ],
}

// ============================================================================
// UTILITY FUNCTIONS
// ============================================================================

function classNames(classes: Record<string, boolean>): string {
  return Object.entries(classes)
    .filter(([_, value]) => value)
    .map(([key]) => key)
    .join(' ');
}

function colorToRgb(color: string): { r: number; g: number; b: number } {
  // Simplified color conversion - in real app use proper color library
  const colors: Record<string, { r: number; g: number; b: number }> = {
    yellow: { r: 255, g: 255, b: 0 },
    blue: { r: 0, g: 123, b: 255 },
    green: { r: 40, g: 167, b: 69 },
  };

  return colors[color] || { r: 255, g: 255, b: 0 };
}
```
