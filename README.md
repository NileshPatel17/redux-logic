# redux-logic

Redux middleware for organizing business logic and action side effects.

> "I wrote the rxjs code so you didn't have to."

You declare some behavior that wraps your code providing things like filtering, cancelation, limiting, etc., then write just the simple business logic code that runs in the center.

## tl;dr

One place to keep all of your business logic and side effects with redux

With simple code you can:

 - validate, verify, auth check actions and allow/reject or modify actions
 - tranform - augment/enhance/modify actions
 - process - async processing and dispatching, orchestration, I/O (ajax, REST, web sockets, ...)

Built-in declarative functionality

 - filtering, cancellation, takeLatest, throttling, debouncing

## Quick Example

```js
const fetchPollsLogic = createLogic({

  // declarative built-in functionality wraps your code
  type: FETCH_POLLS, // only apply this logic to this type
  cancelType: CANCEL_FETCH_POLLS, // cancel on this type
  latest: true, // only take latest

  // your code here, hook into one or more of these execution
  // phases: validate, transform, and/or process
  process({ getState, action }, dispatch) {
    axios.get('https://survey.codewinds.com/polls')
      .then(resp => resp.data.polls)
      .then(polls => dispatch({ type: FETCH_POLLS_SUCESS,
                                payload: polls }))
      .catch(({ statusText }) =>
             dispatch({ type: FETCH_POLLS_FAILED, payload: statusText,
                        error: true })

  }
});
```

## Goals

 - organize business logic keeping action creators and reducers clean
   - action creators are light and just post action objects
   - reducers just focus on updating state
   - intercept and perform validations, verifications, authentication
   - intercept and transform actions
   - perform async processing, orchestration, dispatch actions
 - wrap your core business logic code with declarative behavior
   - filtered - apply to one or many action types or even all actions
   - cancellable - async work can be cancelled
   - limiting (like taking only the latest, throttling, and debouncing)
 - features to support business logic and large apps
   - have access to full state to make decisions
   - easily composable to support large applications
   - inject dependencies into your logic, so you have everything needed in your logic code
   - dynamic loading of logic for splitting bundles in your app
   - your core logic code stays focussed and simple, don't use generators or observables unless you want to.
   - create subscriptions - streaming updates
   - easy testing - since your code is just a function it's easy to isolate and test

## Compared to fat action creators

 - no easy way to cancel or do limiting like take latest
 - action creators would not have access to the full global state so you might have to pass down lots of extra data that isn't needed for rendering. Every time business logic changes might require new data to be made available
 - no global interception - applying logic or transformations across all or many actions
 -  Testing components and fat action creators may require running the code (possibly mocked).

## Compared to redux-thunk

 - With thunks business logic is spread over action creators
 - With thunks there is not an easy way to cancel async work nor to perform limiting (take latest, throttling, debouncing)
 - no global interception - applying logic or transformations across all or many actions
 - Testing components and thunked action creators may require running the code (possibly mocked).


## Compared to redux-observable

 - redux-logic doesn't require the developer to use rxjs observables. It uses observables under the covers to provide cancellation, throttling, etc. You simply configure these parameters to get this functionality. You can still use rxjs in your code if you want, but not a requirement.
 - redux-logic hooks in before the reducer stack like middleware allowing validation, verification, auth, tranformations. Allow, reject, tranform actions before they hit your reducers to update your state as well as accessing state after reducers have run. redux-observable hooks in after the reducers have updated state so they have no opportuntity to prevent the updates.

## Compared to redux-saga

 - redux-logic doesn't require you to code with generators
 - redux-saga relies on pulling data (usually in a never ending loop) while redux-logic and logic are reactive, responding to data as it is available
 - redux-saga runs after reducers have been run, redux-logic can intercept and allow/reject/modify before reducers run also as well as after
 - redux-saga doesn't support dynamic loading of code


## Compared to custom redux middleware

 - Both are fully featured to do any type of business logic (validations, tranformations, processing)
 - redux-logic already has built-in capabilities for some of the hard stuff like cancellation, limiting, dynamic loading of code. With custom middleware you have to implement all functionality.
 - No safety net, if things break it could stop all of your future actions
 - Testing requires some mocking or setup

## Compared to SAM or PAL Pattern

 - With redux-logic you can implement the SAM / PAL pattern without giving up React and Redux. Namely you can separate out your business logic from your action creators and reducers keeping them thin. redux-logic provides a nice place to accept, reject, and transform actions before your reducers are run. You have access to the full state to make decisions and you can trigger actions based on the updated state as well.
 - If you implement SAM / PAL without React, Redux, redux-logic you will have lots of boilerplate code to write.

## Usage

```js
// in configureStore.js
import { createLogicMiddleware } from './redux-logic';
import rootReducer from './rootReducer';
import epics from './logic';

const deps = { // optional injected dependencies for logic
  // anything you need to have available in your logic
  A_SECRET_KEY: 'dsfjsdkfjsdlfjls',
  firebase: firebaseInstance
};

const logicMiddleware = createLogicMiddleware(logic, deps);

const middleware = applyMiddleware(
  logicMiddleware
);

const enhancer = middleware; // could compose in dev tools too

export default function configureStore() {
  const store = createStore(rootReducer, enhancer);
  return store;
}


// in logic.js - combines logic from across many files, just
// a simple array of logic to be used for this app
export default [
 ...todoLogic,
 ...pollsLogic
];


// in polls/logic.js

const validationLogic = createLogic({
  type: ADD_USER,
  validate({ getState, action }, allow, reject) {
    const user = action.payload;
    if (!getState().users[user.id]) { // can also hit server to check
      allow(action);
    } else {
      reject({ type: USER_EXISTS_ERROR, payload: user, error: true })
    }
  }
});

const addUniqueId = createLogic({
  transform({ getState, action }, next) {
    // add unique tid to action.meta of every action
    const existingMeta = action.meta || {};
    const meta = {
      ...existingMeta,
      tid: shortid.generate()
    },
    next({
      ...action,
      meta
    });
  }
});

const fetchPollsLogic = createLogic({
  type: FETCH_POLLS, // only apply this logic to this type
  cancelType: CANCEL_FETCH_POLLS, // cancel on this type
  latest: true, // only take latest
  process({ getState, action }, dispatch) {
    axios.get('https://survey.codewinds.com/polls')
      .then(resp => resp.data.polls)
      .then(polls => dispatch({ type: FETCH_POLLS_SUCESS,
                                payload: polls }))
      .catch(({ statusText }) =>
             dispatch({ type: FETCH_POLLS_FAILED, payload: statusText,
                        error: true })

  }
});

// pollsLogic
export default [
  validationLogic,
  addUniqueId,
  fetchPollsLogic
];

```

## Full API

```js
const fooLogic = createLogic({
  // filtering/cancelling
  type, // required string, regex, array of str/regex, use '*' for all
  cancelType, // string, regex, array of strings or regexes

  // limiting - optionally define one of these
  latest, // only take latest, default false
  debounce, // debounce for N ms, default 0
  throttle, // throttle for N ms, default 0

  // Put your business logic into one or more of these
  // execution phase hooks.
  //
  // Note: If you provided any optional dependencies in your
  // createLogicMiddleware call, then these will be provided to
  // your code in the first argument along with getState and action
  validate({ getState, action }, allow, reject) {
    // run your verification logic and then call allow or reject
    // with the action to pass along. You may pass the original action
    // or a modified/different action. Use undefined to prevent any
    // action from being propagated like allow() or reject()
    allow(action); // OR reject(action)
  }),

  transform({ getState, action }, next) {
    // perform any transformation and provide the new action to next
    next(action);
  }),

  process({ getState, action, cancelled$ }, dispatch) {
    // Perform your processing then call dispatch with an action
    // or use dispatch() to complete without dispatching anything.
    // Advanced use: dispatch an observable to perform multiple
    // dispatches or to use a long running subscription
    dispatch(myNewAction);
  })
});

const logicMiddleware = createLogicMiddleware(
  arrLogic, // array of logic items
  deps   // optional injected deps/config, supplied to logic
);

// dynamically add logic later at runtime, keeping logic state
logicMiddleware.addLogic(arrNewLogic);

// replacing logic, logic state is reset but in-flight logic
// should still complete properly
logicMiddleware.replaceLogic(arrReplacementLogic);
```

## Other possible feature additions

Some features under consideration.

### Timeout

 - timeout N ms, default 0 (disables)
 - timeoutType - action type to use in the event of a timeout

### Predetermined type or action creators for dispatch

 - successType - dispatch this action type using contents of dispatch as the payload (also would work with with promise or observable passed to dispatch)
 - failedType - dispatch this action type using contents of error as the payload, sets error: true (would also work for rejects of promises or error from observable)

The successType and failedType would enable this type of code, where you can simply dispatch a promise or observable that resolves to the payload and rejects on error.

```js
const fetchPollsLogic = createLogic({

  // declarative built-in functionality wraps your code
  type: FETCH_POLLS, // only apply this logic to this type
  cancelType: CANCEL_FETCH_POLLS, // cancel on this type
  latest: true, // only take latest
  successType: FETCH_POLLS_SUCCESS, // dispatch this success action type
  failedType: FETCH_POLLS_FAILED, // dispatch this failed action type

  process({ getState, action }, dispatch) {
    dispatch( // can just dispatch promise which resolves to payload
      axios.get('https://survey.codewinds.com/polls')
        .then(resp => resp.data.polls);
    )
  }
});
```

### Returning promise or observable with predetermined types/action creators

This allows us to get much of the action details out of the code leaving mostly just business logic. Instead of using dispatch return promise or observable.

Either you wouldn't provide the second argument or maybe it would be a different execution phase hook like `processReturn` indicating that we will be returning something rather than using dispatch.

 - successType - dispatch this action type/action creator using contents of dispatch as the payload (also would work with with promise or observable passed to dispatch)
 - failedType - dispatch this action type/action creator using contents of error as the payload, sets error: true (would also work for rejects of promises or error from observable)

The successType and failedType would enable this type of code, where you can simply return a promise or observable that resolves to the payload and rejects on error.

```js
const fetchPollsLogic = createLogic({

  // declarative built-in functionality wraps your code
  type: FETCH_POLLS, // only apply this logic to this type
  cancelType: CANCEL_FETCH_POLLS, // cancel on this type
  latest: true, // only take latest

  // provide action types or action creator functions to be used
  // with the resolved/rejected values from promise/observable returned
  successType: FETCH_POLLS_SUCCESS, // dispatch this success action type
  failedType: FETCH_POLLS_FAILED, // dispatch this failed action type

  // processReturn instead of process so we will use the resolved return
  processReturn({ getState, action }) {
    return axios.get('https://survey.codewinds.com/polls')
      .then(resp => resp.data.polls);
    )
  }
});
```

This is pretty nice leaving us with mainly our business logic code that could be easily extracted and called from here.


## Get involved

If you have input or ideas or would like to get involved, you may:

 - contact me via twitter @jeffbski  - <http://twitter.com/jeffbski>
 - open an issue on github to begin a discussion - <https://github.com/jeffbski/redux-logic/issues>
 - fork the repo and send a pull request (ideally with tests) - <https://github.com/jeffbski/redux-logic>
 - See the [contributing guide](http://github.com/jeffbski/redux-logic/raw/master/CONTRIBUTING.md)

<a name="license"/>
## License - MIT

 - [MIT license](http://github.com/jeffbski/redux-logic/raw/master/LICENSE.md)