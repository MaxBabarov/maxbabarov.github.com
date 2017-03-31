# React Mutations Design Pattern

## The Problem
Redux makes it very easy to describe your application state as a pure function, with the signature `(previousState, action) => newState`.

React, in turn makes it very easy to describe your UI as a function of state. i.e. `state => view`. Or more accurately, since React handles
optimizations with how the view is updated, `(previousView, state) => newView`

However, while this model has the benefit of being simple and easy to reason about, there is inherent *information* loss.

For example, the logic responsible for the UI, has no way of knowing *how* the new state was derived. So if I'm rendering a list of a thousand card elements, and one has changed, the only way to figure out which one needs to be re-rendered, is to linearly walk through the entire list of elements and see which ones have changed in the store. So we go from a O(1) operation (if we made the change imperatively), to an O(N).

In the redux docs there is mention of a reason to use multiple stores:
 > Solving a performance issue caused by too frequent updates of some part of the state, when confirmed by profiling the app.

## Working towards a solution
Using multiple stores isolates the performance problem to a portion of the UI, and in some cases may be enough.

However the fundamental problem remains which is that the information loss prevents us from updating the UI efficiently.

So instead we need to move from `(previousView, state) => newView` to `(previousView, state, action) => newView`.

Unfortunately React doesn't give us a way natively to declaratively express our UI in terms of state *and* actions. It does, however, give us a [backdoor](https://facebook.github.io/react/docs/refs-and-the-dom.html).

The problem with Refs, however, is that they are an experimental API, and they force us to manually invoke the DOM API as side-effects in response to actions.

## Introducing Mutations
We need an abstraction to keep our application code from having to consume the DOM API directly. This is where mutations come in.

A mutation is a plain object which declaratively describes a change that should happen to a portion of the UI.

There are several types of mutations which could be useful to us in a React app.
  1. **DOM Mutation** An object which targets a specific DOM element (or elements) with one ore more DOM attributes that should be changed.

  i.e

  ```js
  {
    type: 'DOM',
    /** Supports targeting components using trailing wildcards (*) */
    target: ['scatter*'],
    exclude: [],
    eventKey: [payload.index],
    payload: {
      style: {
        strokeWidth: 3,
        fillOpacity: 1,
      },
    }
  }
  ```
  1. **Prop Mutation** An object which targets a specific React Component with new Props *while avoiding the React reconciliation process* of its ancestor components. These props will only be valid until new props are received. (Implementation is TODO).

  i.e.

  ```js
  {
    type: 'PROPS',
    target: ['tooltip', 'marker'],
    payload: {
      // ...
    }
  }
  ```
  1. **Sticky Props Mutation** An object which targets a specific React component with new props. Unlike `PROPS` mutations, the new props will persist across render cycles and can only be changed by targeting the component with a new `STICKY_PROPS` mutation.

  i.e.

  ```js
  {
    type: 'STICKY_PROPS',
    target: ['tooltip', 'marker'],
    payload: {
      // ...
    },
  }
```

This allows us us to model our UI updates as `(state, action) => mutation` and `(previousView, state, mutation) => newView`.

### Drawbacks
  1. We lose the clarity of expressing our view as a pure function of state.
  1. We need to do additional book keeping to make sure the mutations 'cleanup' after themselves.

### Benefits
  1. We get usable performance when rendering UIs that are driven by large data sets, while keeping the declarative and componentized structure of React apps.
  1. We are now expressing changes to the view once again as pure functions which can be very easily unit tested!

## Solution
### withMutation HOC
Because we are doing imperative side-effects, and using an experimental API, its important that we confine this logic to as small a surface area as possible.

That's where the `withMutationInterface` Higher Order Component (HOC) comes in. It accepts a React component, and returns one that can be modified according to the logic described above.

i.e.
```
export const InteractiveBar = withMutationInterface(VictoryBar);
```

This HOC exposes a `mutation$` prop which allows the component to accept a stream of mutation objects as an `Observable`.

It also exposes a `name` prop which will be matched against the array of targets on a mutation to
determine if that component is being targeted.

**Note**: By providing an `eventKey` prop in the mutation, children of the component may be targeted, while omitting this key will cause the component itself to be targeted.

**Note**: The current implementation is not fully generic and has logic specific to wrapping Victory components, though it will work with other React components as well.

### reactionBus
This is more of an implementation detail than a core part of the solution. It answers the question of what actions we are responding to.

For the frequency of interactions in our charting scenario, piggy-backing on the global redux actions, and causing our reducer-logic to continuously run is not the right approach.

So we use local state (similar to redux) where we can also provide a `deriveMutations` function, which is the pure function that produces `mutation` objects as a function of `state` and `action`. This state is encapsulated by the reactionBus, and is currently separate from component local state.

We use the `reactionBus` to dispatch actions upstream through the view tree and broadcast back downstream. They help to make sure that our application's dependency graph stays one-way (acyclic).

i.e.

```js
const deriveMutations = ({ action }) => {
  if (action.type === 'SOME_ACTION') {
    return {
      type: 'DOM',
      target: ['some component'],
      payload: {
        style: {
          color: 'blue',
        },
      },
    };
  }
}

// This tree-like pattern could be repeated indefinitely
const masterReactionBus = createReactionBus();
const slaveReactionBus1 = createReactionBus({
  master: masterReactionBus,
  deriveMutations: deriveMutations
});
const slaveReactionBus2 = createReactionBus({
  master: masterReactionBus,
  deriveMutations: () => {},
});

slaveReactionBus1.mutation$.subscribe(mutation => console.log(mutation));

slaveReactionBus2.dispatch({
  type: 'SOME_ACTION',
  payload: {
    // data describing action
  },
});

// console output:
// {
//   type: 'DOM',
//   target: ['some component'],
//   payload: {
//     style: {
//       color: 'blue',
//     },
//   },
// }


```

This approach  is appropriate to the charting scenario, but we could just as easily have implemented a one-way dependency graph in the way that Redux does with a globally available dispatcher.

In the reactionBus implementation, each reactionBus has it's own dispatcher, and routes actions upstream or downstream depending on whether it is at the "top" of the tree-like stack of reactionBus nodes.

In this way we guarantee that `deriveMutations` is only invoked once per action, and that we never get into loops where we continuously fire actions in response to other actions.

In current implementation, components using reactionBus accept an optional `reactionBus` prop which represents a master reactionBus, and then the component itself creates a reactionBus in its constructor, and recreates its reactionBus in `componentWillReceiveProps` if it receives a new master reactionBus.

**Note**: To facilitate generating mutations from multiple different actions, and cleaning up after actions, the actual function signature for `deriveMutations` is `({ action, previousAction, state }) => mutation`
where `state` is derived from a `reducer` that may be provided to the `reactionBus` on creation (similar to redux).

## Conclusion
This approach brings React closer to some Functional Reactive Programming models, bringing along the benefits of performant UI updates described in terms of pure functions.

However, it adds additional complexity when compared to the vanilla React + Redux approach, and should be used sparingly.

Future improvements to this pattern could involve making `withMutationInterface` fully generic (and fully implementing all the mutation types).

Also, it's not ideal how components with `reactionBus` have both state inside and outside the `reactionBus`. Ideally, we would implement this in a way where they shared the same component state, and we use the `shouldComponentUpdate` life-cycle hook to simply skip component updates for state changes that should only be reacted to with a mutation.
