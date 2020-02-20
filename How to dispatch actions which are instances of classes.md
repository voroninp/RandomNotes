# How to dispatch actions which are instances of classes

I won't discuss how "smart" was the decision to prohibit the dispatch of class instances,
but guys definitely do not understand the difference between immutability/purity and object oriented way of doing things.
OOP and FP are not mutually exclusive, they are orthogonal.

So, how to dispatch something more advanced than plain js objects?

First we need redux middleware:

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

function createSagaMiddlewareWithActionPreprocessor(actionPreprocessor) {

  const sagaMiddleware = createSagaMiddleware();

  const extendedSagaMiddleware = store => next => {

    const actionHandler = sagaMiddleware(store)(noop);

    return action => {
      // send to reducers, keep the result
      const result = next(action);

      action = actionPreprocessor(action);

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
import { shallowEqual } from "react-redux";

/**
 * Base class for all actions.
 * 
 * State reduction must be performed in root reducer!
 *
 * @class Action
 */
class Action {
  /**
   * Creates an instance of Action.
   * @memberof Action
   * 
   * @param {any} [payload]
   * @param {string[]} [substatePath] - The path to the substate which should be updated.
   * This property is overridden by `getSubstatePath` method. 
   */
  constructor(payload, substatePath) {
    this.type = this.constructor.name;
    this.substatePath = substatePath;
    this.payload = payload;
  }

  /**
   * Base method with state reduction logic. Override it to change sate as required for the given action.
   * As state reduction for the action must be done in root reducer `state` is the global state of the store.
   * If you are not willing to process global state, override `getSubstatePath` and `getUpdatedSubstate`
   * methods accordingly.
   *
   * The con of this approach is that it couples action and logic of the state change, 
   * but it is relevant only for the systems having multiple processors of the same action/message.
   * 
   * If `substatePath` contains elements, then the whole object graph is recreated to guarantee that
   * references changed along the whole path to the substate. 
   * Then `getUpdateSubstate` is called on the deepest value of the path.
   * 
   * This default implementation allows to scope states in a similar way how combining reducers work.
   * 
   * @param {any} state
   * @returns update state.
   * @memberof Action
   */
  getUpdatedState(state) {
    const substatePath = (this.getSubstatePath && this.getSubstatePath()) || this.substatePath;
    if (Array.isArray(substatePath) && substatePath.length > 0) {
      //  follow  the path and refresh the  instances
      const spLength = substatePath.length;
      const newState = {...state};
      let currentSubstate = newState;
      // last property is not touched in a loop, it is processed later, that's why spLengh-1
      for (var pi=0; pi < spLength-1;pi++) {
        const property = substatePath[pi];
        if (property == null) {
          throw new Error("Substate path properties cannot be null.");
        }
        const newSubstate = {...currentSubstate[property]};
        currentSubstate[property] = newSubstate;
        currentSubstate = newSubstate;
      }

      const deepestPropertyIndex = spLength-1;
      const deepestProperty = substatePath[deepestPropertyIndex];
      const deepestSubstate = currentSubstate[deepestProperty];
      currentSubstate[deepestProperty] = this.getUpdatedSubstate(deepestSubstate);

      return newState;
    }

    // DECISION: I deliberately avoid returning it like `return getUpdatedSubstate(state)`
    // because this is very important to provide both `getUpdateSubstate` and `substatePath`.
    // So if developer forgets to do it, and overrides only `getUpdatedSubstate`, method won't be called,
    // and global state won't be updated and potentially corrupted, as it could happen otherwise,
    // if was called as `return getUpdateSubstate(state)` where `state` is a global state. 
    // This  helps to isolate the error just to the code which depends on the particular action.
    return state;
  }

  getSubstatePath() { return this.substatePath; }

  getUpdatedSubstate(substate){
    return substate;
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

  as(ActionType, ...args) {
    const newAction = new ActionType(...args);
    for (const name in this) {
      if (newAction[name] === undefined) {
        newAction[name] = this[name];
      }
    }

    return newAction;
  }

  /**
   * Convenience method to generate a predicate which serves as a saga pattern.
   * 
   * For better testability define static property `sagaPattern` on the action which should be processed:
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
      // This check is added because redux does not allow dispatching class objects,
      // so with custom middleware we have a hackish solution.
      action.__classAction instanceof ActionType;
    };
  }

  static fromPlain(action){
    if (action instanceof Action){
      return action;
    }

    if (!action.__classAction){
      throw Error("Expected special plain object action containing class instance action. Is 'supportClassActionsMiddleware' enabled?");
    }

    return action.__classAction;
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