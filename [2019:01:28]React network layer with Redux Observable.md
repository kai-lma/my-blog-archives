# React network layer with Redux Observable
## Introduction
* Redux is good for state management.
* Redux midleware is for handling side effect (network request, storage...).
* You can create your own midleware but there's alot of well maintained midlewares available out there.
* Popular midleware libraries: `thunk`  `promise`  `saga` `observable`...
* As the title, I will pick Redux Observable to implement network layer in React app.

> This is my ever fist blog about React, It definitely have lots of flaws.Though, It's open for critism and your kind feedback.

## Reasons for picking Redux Observable:
* It's more declative with functional programming style.
* Function reusability are high
* Easy to test
* Can transfer skill between `Rx` family (RxSwift, RxJava, RxJS...).
* Fancy things like throttle, debounce, retry...  works right out of the box.
* It's DOPE

## What we will make:
* RESTful api request layer
* On success and error callback
* Pre error handling (Refresh token, Server healthcheck...)
* Debounce to reduce ajax request pressure
* Discard request when logout or when the stream is idle

## Naivety approaches
* Install each stream for each endpoint.
	* It's simple 👍
	* But more boilderplates and code duplication 👎
	* Somewhat problem when too much active streams always listen for its not-always coming actions. 👎

* One "All Request Actions" listening stream
	* This helps reduce code 👍
	* Lost `debounce` `retry` functionalies out of the box 🤔 👎

* Better approach:
	* One "Api Request Actions" listening stream -> reduce code 👍
	* Then spawning new stream listen for that request action -> keep `debounce` and friends work right out of the box 👍
	* Dispose stream when it become idle -> performance improve 👍

## Let's do it.

### First create Request Action builder:
```js
export const REQUEST = 'REQUEST';

export const createApiRequestAction = ({
  type,
  method = 'GET',
  endpoint,
  queryParams,
  pathParams,
  bodyParams,
  timeout = 5000,
  onSuccess = () => {},
  onError = () => {},
  showLoading = true,
  debounceTime = 200,
}) => ({
  metaType: REQUEST,
  type,
  method,
  endpoint,
  queryParams,
  pathParams,
  bodyParams,
  timeout,
  onSuccess,
  onError,
  showLoading,
  debounceTime,
});

export const succeedApiRequest = (data, requestAction) => ({
  type: `${requestAction.type}_DONE`,
  payload: data,
  requestAction,
});

export const failedApiRequest = (error, requestAction) => ({
  type: `${requestAction.type}_FAIL`,
  payload: error,
  requestAction,
});
```

### Make our api epic stream

#### Create one stream listen for all actions that have metaType is `REQUEST`

```js
const apiEpic = (action$, store$) => {
  return action$.pipe(
    // Stream of all request actions
    filter(action => action.metaType === REQUEST),
    )
  );
};
```

#### Then open new stream for that type

```js
const apiEpic = (action$, store$) => {
  const openingApiActionStreams = {};
  return action$.pipe(
    // Stream of request actions
    filter(
      action => action.metaType === REQUEST &&
      !openingApiActionStreams[action.type],
    ),

    // Tracking stream opening states
    tap(action => {
      console.log(`${action.type} stream created`);
      openingApiActionStreams[action.type] = true;
    }),

    // Open new stream of this action type
    flatMap(action =>
      action$.ofType(action.type).pipe(
        // Begin new stream with this trigger action
        startWith(action),

        // ...

        // Update stream opening states when stream is closed
        finalize(() => {
          console.log(`${action.type} stream closed`);
          openingApiActionStreams[action.type] = false;
        }),
      ),
    ),
  );
};
```

#### Add debounce time to reduce ajax request pressure

* You can find more about debounce time [here](https://rxjs.dev/api/operators/debounceTime).
* Simply, It's useful when user countinously hit the like button multiple time which fire like 20 unneccesary requests, then the `debounceTime` operator help us to take only the last event and save your api server. 
* With RxJS we'll just call `debounceTime` operator that does it all for us.

```js
flatMap(action =>
  action$.ofType(action.type).pipe(
    // snip...

    debounceTime(action.debounceTime),

    // snip...
  ),
),
```

#### Add stream terminator

* As mention above, when we open too many stream that listening for one time dispatched action but keep it forever would be a bad idea, we will terminate it when it is unused.
* Just like `debounceTime`, we can use `takeUntil` operator to terminate the stream like this:

```js
flatMap(action =>
  action$.ofType(action.type).pipe(
    // snip...

    takeUntil(terminator$(action, action$)),

    // snip...
  ),
),
```

* We will close stream when `SIGN_OUT` or idle. So our terminator stream will be like:

```js
const terminator$ = (action, action$) =>
  merge(
    // Dispose stream when signed out
    action$.pipe(ofType(SIGNOUT)),

    // Dispose stream when it's idle 10 seconds
    action$.pipe(
      ofType(action.type, `${action.type}_DONE`, `${action.type}_FAIL`),
      debounceTime(10000),
    ),
  );
```

#### Finally the ajax request stream

```js
flatMap(action =>
  action$.ofType(action.type).pipe(
    // snip...

    // Start async request flow
    switchMap(action => request$(action, store$)),

    // snip...
  ),
),
```

```js
const request$ = (action, store$) =>
  from(ajax(action, getAccessToken(store$))).pipe(
    switchMap(response => {
      // Callback & dispatch result
      action.onSuccess(response.data);
      return of(succeedApiRequest(response.data, action));
    }),

    // Handle errors
    catchError(error => {
      const apiError = parseApiError(error);

      // Pre-handles
      switch (apiError.errorCode) {
        case ApiErrorCode.TokenExpired:
          return of(refreshToken(action));
        case ApiErrorCode.InvalidToken:
          return of(signout());
        default:
          break;
      }

      // Callback & dispatch Error
      action.onError(apiError);
      return of(failedApiRequest(apiError, action));
    }),
  );
```

* That's it. We made it.

#### Completed api epic stream

```js
const apiEpic = (action$, store$) => {
  const openingApiActionStreams = {};
  return action$.pipe(
    // Stream of request actions
    filter(
      action => action.metaType === REQUEST &&
      !openingApiActionStreams[action.type],
    ),

    // Tracking stream opening states
    tap(action => {
      console.log(`${action.type} stream created`);
      openingApiActionStreams[action.type] = true;
    }),

    // Open new stream of this action type
    flatMap(action =>
      action$.ofType(action.type).pipe(
        // Begin new stream with this trigger action
        startWith(action),

        // Lossy back-pressure
        debounceTime(action.debounceTime),

        // Start async request flow
        switchMap(action => request$(action, store$)),

        // Stream of this action type's terminator
        takeUntil(terminator$(action, action$)),

        // Tracking stream opening states
        finalize(() => {
          console.log(`${action.type} stream closed`);
          openingApiActionStreams[action.type] = false;
        }),
      ),
    ),
  );
};
```

## References
* [Introduction · learn-rxjs](https://www.learnrxjs.io/)
* [Introduction · redux-observable](https://redux-observable.js.org/)
* [Redux-Saga V.S. Redux-Observable - HackMD](https://hackmd.io/@2qVnJRlJRHCk20dvVxsySA/H1xLHUQ8e)
