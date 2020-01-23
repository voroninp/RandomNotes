# How to dispatch actions which are instances of classes

I won't discuss how "smart" was the decision to prohibit the dispatch of class instances,
but guys defintely do not understand the difference between immutability/purity and object oriented way of doing things.
OOP and FP are not mutually exlusive, they are orthogonal.

So, how to dispatch something more advanced than plain js objects?

First we need redux middlewhare:

```js
// Action is just a base class for all actions.
import { Action } from "../actions/Actions";

// eslint-disable-next-line no-unused-vars
export default store => next => action => {

  let plainAction = action;
  
  if (action instanceof Action){
    plainAction = {...action};
    // That is the trick.
    plainAction.__classAction = action;
  }

  next(plainAction);
};
```

That's fine, but if we use Redux-Saga, it's better to get action back as a class instance.

For that we need to create a wrapper for saga middleware:

(the example was found [here](https://gist.github.com/keeth/96b97c6cf890187d0b17d98ce9d4fbd4))

```js
import createSagaMiddleware from "redux-saga";
import { Action } from "../actions/Actions";
const noop = () => {};

function createSagaMiddlewareWithActionPreprocessor(actionPreprocesor) {

  const sagaMiddleware = createSagaMiddleware();

  const extendedSagaMiddleware = store => next => {

    const actionHandler = sagaMiddleware(store)(noop);

    return action => {
      // send to reducers, keep the result
      const result = next(action);

      action = actionPreprocesor(action);

      if (action){
        actionHandler(action);
      }

      // pass new state back up the middleware chain
      return result;
    };
  };

  extendedSagaMiddleware.run = function(...args){
    sagaMiddleware.run(...args);
  }

  return extendedSagaMiddleware;
}

function createSagaMiddlewareWithClassActions(){
  return createSagaMiddlewareWithActionPreprocessor(action => {
    if (action.__classAction instanceof Action){
      action = action.__classAction;
    }

    return action;
  });
}

export {
  createSagaMiddlewareWithActionPreprocessor,
  createSagaMiddlewareWithClassActions
};
```

And here is `Action` class:

```js
/**
 * Base class for all actions.
 *
 * @class Action
 */
class Action {
  /**
   * Creates an instance of Action.
   * @memberof Action
   */
  constructor(payload) {
    this.type = this.constructor.name;
    this.payload = payload;
  }

  /**
   * Base method with state reduction logic. Override it to change sate as required for the given action.
   *
   * The con of this approach is that it couples action and logic of the state change, 
   * but it is relevant only for the systems having multiple processors of the same action/message.
   * 
   * @param {any} state
   * @returns update state.
   * @memberof Action
   */
  getUpdatedState(state) {
    return state;
  }

  static tryReduceState(state, action){
    if (action instanceof Action){
      return [true, action.getUpdatedState(state)];
    }
    else if (action.__classAction instanceof Action){
      return [true, action.__classAction.getUpdatedState(state)];
    }

    return [false, state];
  }

  /**
   * Convenience method to generate a predicate which surves as a saga pattern.
   * 
   * For better testability define static property `sagaPattern` on the action whcih should be processed:
   * ```js
   * static sagaPattern = Action.sagaPatternFor(ThisActionType);
   * ``` 
   *
   * @static
   * @param {*} ActionType
   * @returns {(action:any) => boolean} - predicate checking whether action should be processed.
   * @memberof Action
   */
  static sagaPatternFor(ActionType) {
    return function(action) {
      return action instanceof ActionType ||
      // This check is added because redux does'not allow dispatching class objects,
      // so with custom middleware we have a hackish solution.
      action.__classAction instanceof ActionType;
    };
  }
}
```

static method `tryReduceState` may be used in reducer:

```js
...
const [isReduced, reducedState] = Action.tryReduceState(state, action);
if (isReduced){
    return reducedState;
}
...
```