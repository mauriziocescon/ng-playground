# LLM Results
Prompt (gemini 2.5 pro):

>Hey!
In readme.md and di.md, I drafted a possible evolution of the angular syntax in templates and proposed a new authoring format.
Considering such file and the current angular best practices, could you please create a todo-list component following the style?
Thanks

Result:
>Of course! Based on the new authoring format you've drafted in and Angular's best practices, I've created a `TodoList` component. `readme.md`
Here are the files for the new component:
I'll create a `todo.model.ts` file for the data model:

```ts
export interface Todo {
  id: number;
  text: string;
  completed: boolean;
}
```
>Next, here is the `TodoListComponent` in `todo-list.component.ng.ts`. It's a presentational component that receives the list of todos and emits events when the list is modified.

```ts
import { component, input, output, signal, computed } from '@angular/core';
import { type Todo } from './todo.model';

export const TodoListComponent = component(({
  todos = input.required<Todo[]>(),
  todosChange = output<Todo[]>(),
}) => ({
  script: () => {
    const newTodoText = signal('');
    const remaining = computed(() => todos().filter(t => !t.completed).length);

    function addTodo() {
      if (newTodoText().trim() === '') return;
      const newTodo: Todo = {
        id: Date.now(),
        text: newTodoText(),
        completed: false,
      };
      todosChange.emit([...todos(), newTodo]);
      newTodoText.set('');
    }

    function removeTodo(todoId: number) {
      todosChange.emit(todos().filter(t => t.id !== todoId));
    }

    function toggleCompletion(todoId: number) {
      todosChange.emit(
        todos().map(t => (t.id === todoId ? { ...t, completed: !t.completed } : t))
      );
    }
  },
  template: (
    <>
      <h2>Todo List ({remaining()} remaining)</h2>
      
      <div class="add-todo">
        <input type="text" placeholder="What needs to be done?" model:value={newTodoText} />
        <button on:click={addTodo} disabled={newTodoText().trim().length === 0}>Add</button>
      </div>

      <ul>
        @for (todo of todos(); track todo.id) {
          <li>
            <input
              type="checkbox"
              checked={todo.completed}
              on:change={() => toggleCompletion(todo.id)}
            />
            <span class:completed={todo.completed}>
              {todo.text}
            </span>
            <button class="remove" on:click={() => removeTodo(todo.id)}>❌</button>
          </li>
        }
      </ul>
    </>
  ),
  style: (
    <>
      :host {
        display: block;
        max-width: 500px;
        margin: 2rem auto;
      }
      .add-todo {
        display: flex;
        gap: 0.5rem;
        margin-bottom: 1rem;
      }
      input[type="text"] {
        flex-grow: 1;
      }
      ul {
        list-style: none;
        padding: 0;
      }
      li {
        display: flex;
        align-items: center;
        padding: 0.5rem 0;
        gap: 0.5rem;
      }
      li span {
        flex-grow: 1;
      }
      .completed {
        text-decoration: line-through;
        color: #888;
      }
      .remove {
        background: none;
        border: none;
        cursor: pointer;
        padding: 0;
        font-size: 1rem;
      }
    </>
  ),
}));
```

>Finally, here’s a `TodoAppComponent` that consumes the `TodoListComponent` and manages the state.

```ts
import { component, signal } from '@angular/core';
import { TodoListComponent } from './todo-list.component.ng';
import { type Todo } from './todo.model';

export const TodoAppComponent = component(() => ({
  script: () => {
    const todos = signal<Todo[]>([
      { id: 1, text: 'Learn new Angular syntax', completed: true },
      { id: 2, text: 'Create a todo app', completed: false },
    ]);
  },
  template: (
    <>
      <h1>My Todo App</h1>
      <TodoListComponent model:todos={todos} />
    </>
  ),
  style: (
    <>
      :host {
        font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif, "Apple Color Emoji", "Segoe UI Emoji", "Segoe UI Symbol";
        font-size: 14px;
        color: #333;
        box-sizing: border-box;
      }
      h1 {
        text-align: center;
      }
    </>
  )
}));
```
>These components follow the proposed syntax and structure, using signals for state management and the new template features like `model:` binding and `@for` block. Let me know if you have any other questions 
