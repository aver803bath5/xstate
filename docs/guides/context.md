# Context

[:rocket: Quick Reference](#quick-reference)

While _finite_ states are well-defined in finite state machines and statecharts, state that represents _quantitative data_ (e.g., arbitrary strings, numbers, objects, etc.) that can be potentially infinite is represented as [extended state](https://en.wikipedia.org/wiki/UML_state_machine#Extended_states) instead. This makes statecharts much more useful for real-life applications.

In XState, extended state is known as **context**. Below is an example of how `context` is used to simulate filling a glass of water:

```js
import { Machine, assign } from 'xstate';

// Action to increment the context amount
const addWater = assign({
  amount: (context, event) => context.amount + 1
});

// Guard to check if the glass is full
function glassIsFull(context, event) {
  return context.amount >= 10;
}

const glassMachine = Machine(
  {
    id: 'glass',
    // the initial context (extended state) of the statechart
    context: {
      amount: 0
    },
    initial: 'empty',
    states: {
      empty: {
        on: {
          FILL: {
            target: 'filling',
            actions: 'addWater'
          }
        }
      },
      filling: {
        on: {
          // Transient transition
          '': {
            target: 'full',
            cond: 'glassIsFull'
          },
          FILL: {
            target: 'filling',
            actions: 'addWater'
          }
        }
      },
      full: {}
    }
  },
  {
    actions: { addWater },
    guards: { glassIsFull }
  }
);
```

The current context is referenced on the `State` as `state.context`:

```js
const nextState = glassMachine.transition(glassMachine.initialState, 'FILL');

nextState.context;
// => { count: 1 }
```

## Initial context

The initial context is specified on the `context` property of the `Machine`:

```js
const counterMachine = Machine({
  id: 'counter',
  // initial context
  context: {
    count: 0,
    message: 'Currently empty',
    user: {
      name: 'David'
    },
    allowedToIncrement: true
    // ... etc.
  },
  states: {
    // ...
  }
});
```

For dynamic `context` (that is, `context` whose initial value is retrieved or provided externally), you can use a machine factory function that creates the machine with the provided context values (implementation may vary):

```js
const createCounterMachine = (count, time) => {
  return Machine({
    id: 'counter',
    // values provided from function arguments
    context: {
      count,
      time
    }
    // ...
  });
};

const counterMachine = createCounterMachine(42, Date.now());
```

Or for existing machines, `machine.withContext(...)` should be used:

```js
const counterMachine = Machine({
  /* ... */
});

// retrieved dynamically
const someContext = { count: 42, time: Date.now() };

const dynamicCounterMachine = counterMachine.withContext(someContext);
```

## Initial Context

The initial context of a machine can be retrieved from its initial state:

```js
dynamicCounterMachine.initialState.context;
// => { count: 42, time: 1543687816981 }
```

This is preferred to accessing `machine.context` directly, since the initial state is computed with initial `assign(...)` actions and transient transitions, if any.

## Assigning To Context With `assign`

The `assign()` action is used to update the machine's `context`. It takes the context "assigner", which represents how values in the current context should be assigned.

The "assigner" can be an object (recommended):

```js
import { Machine, assign } from 'xstate';
// example: property assigner

// ...
  actions: assign({
    // increment the current count by the event value
    count: (context, event) => context.count + event.value,

    // assign static value to the message (no function needed)
    message: 'Count changed'
  }),
// ...
```

Or it can be a function that returns the updated state:

```js
// example: context assigner

// ...

  // return a partial (or full) updated context
  actions: assign((context, event) => {
    return {
      count: context.count + event.value,
      message: 'Count changed'
    }
  }),
// ...
```

Both the property assigner and context assigner function signatures above are given 3 arguments:

- `context` (TContext): the current context (extended state) of the machine
- `event` (EventObject): the event that caused the `assign` action
- `meta` <Badge text="4.7+" /> (AssignMeta): an object with meta data containing:
  - `state` - the current state in a normal transition (`undefined` for the initial state transition)
  - `action` - the assign action

::: warning
The `assign(...)` function is an **action creator**; it is a pure function that only returns an action object and does _not_ imperatively make assignments to the context.
:::

## Action order

Custom actions are always executed with regard to the _next state_ in the transition. When a state transition has `assign(...)` actions, those actions are always batched and computed _first_, to determine the next state. This is because a state is a combination of the finite state and the extended state (context).

For example, in this counter machine, the custom actions will not work as expected:

```js
const counterMachine = Machine({
  id: 'counter',
  context: { count: 0 },
  initial: 'active',
  states: {
    active: {
      on: {
        INC_TWICE: {
          actions: [
            context => console.log(`Before: ${context.count}`),
            assign({ count: context => context.count + 1 }), // count === 1
            assign({ count: context => context.count + 1 }), // count === 2
            context => console.log(`After: ${context.count}`)
          ]
        }
      }
    }
  }
});

interpret(counterMachine).start().send('INC_TWICE');
// => "Before: 2"
// => "After: 2"
```

This is because both `assign(...)` actions are batched in order and executed first (in the microstep), so the next state `context` is `{ count: 2 }`, which is passed to both custom actions. Another way of thinking about this transition is reading it like:

> When in the `active` state and the `INC_TWICE` event occurs, the next state is the `active` state with `context.count` updated, and _then_ these custom actions are executed on that state.

A good way to refactor this to get the desired result is modeling the `context` with explicit _previous_ values, if those are needed:

```js
const counterMachine = Machine({
  id: 'counter',
  context: { count: 0, prevCount: undefined },
  initial: 'active',
  states: {
    active: {
      on: {
        INC_TWICE: {
          actions: [
            context => console.log(`Before: ${context.prevCount}`),
            assign({
              count: context => context.count + 1,
              prevCount: context => context.count
            }), // count === 1, prevCount === 0
            assign({ count: context => context + 1 }), // count === 2
            context => console.log(`After: ${context.count}`)
          ]
        }
      }
    }
  }
});

interpret(counterMachine).send('INC_TWICE');
// => "Before: 0"
// => "After: 2"
```

The benefits of this are:

1. The extended state (context) is modeled more explicitly
2. There are no implicit intermediate states, preventing hard-to-catch bugs
3. The action order is more independent (the "Before" log can even go after the "After" log!)
4. Facilitates testing and examining the state

## Notes

- 🚫 Never mutate the machine's `context` externally. Everything happens for a reason, and every context change should happen explicitly due to an event.
- Prefer the object syntax of `assign({ ... })`. This makes it possible for future analysis tools to predict _how_ certain properties can change declaratively.
- Assignments can be stacked, and will run sequentially:

```js
// ...
  actions: [
    assign({ count: 3 }), // context.count === 3
    assign({ count: context => context.count * 2 }) // context.count === 6
  ],
// ...
```

- Just like with `actions`, it's best to represent `assign()` actions as strings or functions, and then reference them in the machine options:

```js {5}
const countMachine = Machine({
  initial: 'start',
  context: { count: 0 }
  states: {
    start: {
      entry: 'increment'
    }
  }
}, {
  actions: {
    increment: assign({ count: context => context.count + 1 }),
    decrement: assign({ count: context => context.count - 1 })
  }
});
```

Or as named functions (same result as above):

```js {9}
const increment = assign({ count: context => context.count + 1 });
const decrement = assign({ count: context => context.count - 1 });

const countMachine = Machine({
  initial: 'start',
  context: { count: 0 }
  states: {
    start: {
      // Named function
      entry: increment
    }
  }
});
```

- Ideally, the `context` should be representable as a plain JavaScript object; i.e., it should be serializable as JSON.
- Since `assign()` actions are _raised_, the context is updated before other actions are executed. This means that other actions within the same step will get the _updated_ `context` rather than what it was before the `assign()` action was executed. You shouldn't rely on action order for your states, but keep this in mind. See [action order](#action-order) for more details.

## TypeScript

For proper type inference, add the context type as the first type parameter to `Machine<TContext, ...>`:

```ts
interface CounterContext {
  count: number;
  user?: {
    name: string;
  };
}

const machine = Machine<CounterContext>({
  // ...
  context: {
    count: 0,
    user: undefined
  }
  // ...
});
```

When applicable, you can also use `typeof ...` as a shorthand:

```ts
const context = {
  count: 0,
  user: { name: '' }
};

const machine = Machine<typeof context>({
  // ...
  context
  // ...
});
```

<Badge text="4.7+"> The types for `context` and `event` in `assign(...)` actions will be automatically inferred from the type parameters passed into `Machine<TContext, TEvent>`:

```ts
interface CounterContext {
  count: number;
}

const machine = Machine<CounterContext>({
  // ...
  context: {
    count: 0
  },
  // ...
  {
    on: {
      INCREMENT: {
        actions: assign({
          count: (context) => {
            // context: { count: number }
            return context.count + 1;
          }
        })
      }
    }
  }
});
```

If this inference fails (and in XState versions below 4.7), pass in the `context` type (and `event` type, if applicable) to the `assign(...)` action creator:

```ts {3}
// ...
on: {
  INCREMENT: {
    actions: assign<CounterContext, CounterEvent>({
      count: context => {
        // context: { count: number }
        return context.count + 1;
      }
    });
  }
}
// ...
```

## Quick Reference

**Set initial context**

```js
const machine = Machine({
  // ...
  context: {
    count: 0,
    user: undefined
    // ...
  }
});
```

**Set dynamic initial context**

```js
const createMachine = (count, user) => {
  return Machine({
    // ...
    // Provided from arguments; your implementation may vary
    context: {
      count,
      user
      // ...
    }
  });
};
```

**Set custom initial context**

```js
const machine = Machine({
  // ...
  // Provided from arguments; your implementation may vary
  context: {
    count: 0,
    user: undefined
    // ...
  }
});

const myMachine = machine.withContext({
  count: 10,
  user: {
    name: 'David'
  }
});
```

**Assign to context**

```js
const machine = Machine({
  // ...
  context: {
    count: 0,
    user: undefined
    // ...
  },
  // ...
  on: {
    INCREMENT: {
      actions: assign({
        count: (context, event) => context.count + 1
      })
    }
  }
});
```

**Assignment (static)**

```js
// ...
actions: assign({
  counter: 42
}),
// ...
```

**Assignment (property)**

```js
// ...
actions: assign({
  counter: (context, event) => {
    return context.count + event.value;
  }
}),
// ...
```

**Assignment (context)**

```js
// ...
actions: assign((context, event) => {
  return {
    counter: context.count + event.value,
    time: event.time,
    // ...
  }
}),
// ...
```

**Assignment (multiple)**

```js
// ...
// assume context.count === 1
actions: [
  // assigns context.count to 1 + 1 = 2
  assign({ count: (context) => context.count + 1 }),
  // assigns context.count to 2 * 3 = 6
  assign({ count: (context) => context.count * 3 })
],
// ...
```
