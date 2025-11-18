# Gemini 2.5 pro
## Prompt:

Hey! 

In readme.md and why-macros.md, I drafted a possible evolution of the angular syntax in templates and proposed a new authoring format.

Considering such files and the current angular best practices, could you please create an example of components composition using directives, declarations and fragments?  

Thanks

## Result:

Based on your request and the provided documents (why-macros.md and readme.md), I have designed a composition example demonstrating a Data Grid scenario.

This example highlights how the proposed "Signal Components" replace traditional patterns:

- Declarations replace Pipes.
- Fragments replace ng-template/ng-content.
- Directives are function-based and attach to elements via elRef.

We will build a grid that accepts a custom row layout (Fragment), formats dates (Declaration), and highlights rows on hover (Directive).

### The Declaration: Date Formatter
Instead of a Pipe, we use a #declaration. It runs in an injection context and returns a transformation function.

```ts
import { inject, LOCALE_ID, formatDate, computed } from '@angular/core';

// Replaces the DatePipe
export #declaration dateFmt = () => {
  script: () => {
    const locale = inject(LOCALE_ID);
    
    // Returns a function that accepts a signal (or getter) and returns a computed signal
    return (value: () => string | Date, format: string = 'medium') => 
      computed(() => formatDate(value(), format, locale));
  },
};
```

### The Directive: Hover Highlighter
This directive encapsulates DOM behavior. Note that elRef is a reserved name injected by the framework.

```ts
import { input, ref, inject, Renderer2, afterNextRender } from '@angular/core';

export #directive highlight = ({
  color = input<string>('#f0f0f0'),
  elRef = ref<HTMLElement>(), 
}) => {
  script: () => {
    const renderer = inject(Renderer2);

    afterNextRender(() => {
      const el = elRef();
      // Simple event binding via renderer (or native addEventListener)
      el.addEventListener('mouseenter', () => {
        el.style.backgroundColor = color();
      });
      el.addEventListener('mouseleave', () => {
        el.style.backgroundColor = '';
      });
    });
  },
};
```

### The Container Component
This component accepts a Fragment (template snippet) to define how rows are rendered. This replaces ng-content and ng-template.

```ts
import { input, fragment } from '@angular/core';
import { Render } from '@angular/common';

interface GridProps<T> {
  data: T[];
}

export #component DataGrid = ({
  items = input.required<any[]>(),
  // Define a fragment input that accepts a single parameter of type any
  rowTemplate = fragment<[any]>(),
}) => {
  template: (
    <>
      <table class="grid-table">
        <thead>
          <tr>
            <th>ID</th>
            <th>Content</th>
          </tr>
        </thead>
        <tbody>
          @for (item of items(); track item.id) {
             <Render fragment={rowTemplate()} params={[item]} />
          }
        </tbody>
      </table>
    </>
  ),
  style: (
    <>
      .grid-table { width: 100%; border-collapse: collapse; }
      th { text-align: left; border-bottom: 1px solid #ccc; }
    </>
  )
};
```

### The Consumer (Composition)
Here we bring it all together. We use lexical scoping to access the imports directly in the template.

```ts
import { signal } from '@angular/core';
import { DataGrid } from './data-grid.ng'; // The component
import { highlight } from './highlight.ng'; // The directive
import { dateFmt } from './date-fmt.ng'; // The declaration

interface User {
  id: number;
  name: string;
  role: string;
  lastLogin: string;
}

export #component UserPage = () => {
  script: () => {
    const users = signal<User[]>([
      { id: 1, name: 'Alice', role: 'Admin', lastLogin: '2023-11-01T10:00:00' },
      { id: 2, name: 'Bob', role: 'User', lastLogin: '2023-11-02T14:30:00' },
    ]);
    
    const activeColor = signal('cyan');
  },
  template: (
    <>
      <h1>User Management</h1>
      
      <DataGrid items={users()}>
      
        @fragment row(u: User) {
          
          <tr @highlight(color={activeColor()})>
            
            <td>{u.id}</td>
            <td>
              <strong>{u.name}</strong> ({u.role})
              <br/>
              
              @const lastSeen = @dateFmt(() => u.lastLogin, 'shortDate');
              
              <small>Last seen: {lastSeen()}</small>
            </td>
            
          </tr>
        }
      </DataGrid>
    </>
  ),
};
```
