---
title: A Brief Introduce to Redux Saga
date: 2019-04-28 14:35:50
tags: [Redux, Saga, React]
photos: ["../images/redux-saga.JPG"]
---
<div style="text-align: justify">
If you are quite experienced with redux, which is a predictable state container for JavaScript applications (**Note:** even thouth React and Redux is a popular combination to build fast and powerful apps, Redux is not necessarily combined with React), you are definitely feeling comfortable with its powerful store which manages all the global states and provides much cleaner logic flows to change them. If you are new to redux, [here](https://redux.js.org/introduction/getting-started) is the guide to dive before we start our topic.</div>

<div style="text-align: justify">
In a complex javascript application, asynchronous function is always one of the most annoying part where encounters tons of bugs. If not handle them properly, the app usually ends up with ***call back hell***.
</div>

## **Haven't heard of *CallBack Hell*?**
<div style="text-align: justify">Well, in javascript, the only way you can suspend a computation and have the rest operations doing later is to put the rest operations into a callback function. This callback function usually returns a ***Promise*** (And has a type of ***Promise<\any>***). In order to easily mark those async functions, after ***ES6*** javascript provides extra modifiers ***async*** and ***await***, which actually wraps up the original utilities of promise and makes it more readable to programmers. Hummm, sounds like things are going better... ~~**NO!! It doesn't resolve anything!**~~ The core problem leads to a callback hell is the hierarchical async calls, for example</div>

you have some simple synchronous functions which are in a chain to accomplish some logics:
```javascript
a = getSomething( );
b = getMore(a);
c = getMoreAndMore(b);
...
```
<div style="text-align: justify">It looks fine for now, but what if they all turn out to be async? Then you have to follow the callback style to make them operate one right after another is done:</div>```javascript
getSomthing( function( a ) {
    getMore( a, function( b ) {
        getMoreAndMore( b, function( c ) {
            //keep going...
        });
    });
});
```
Or you prefer ***ES6***:
```javascript
async function getSomething( a ) {
    await b = ToDo( a );
    return await getMore( ( b ) => {
        return await ToDo( b );
    }).then( ( c ) => {
        return await ToDo( c );
    }).then(...);
}
```
Looks really confused? This will getting even uglier if we are using callbacks in loops. 
</br>
## Redux Thunks
<div style="text-align: justify">Back to our redux app, we usually want to update some states after an async call to inform the UI that the data is ready to be fetched. That is always achieved by dispatching an action from the component to the reducer:</div>```javascript
async const callAPI = ( ) => {
    ...
    return response;
}
...
async const updateUI = ( ...params ) => {
    const res = await callAPI( );
    if (res.status === 200)
        dispatch( {type: "UPDATE", isSuccess: true} );
}
...
render ( ) {
    ...
    this.props.isSuccess?
        showData( ) : showError( )
}
```
<div style="text-align: justify">This isn't bad, but we are always looking for something better. An advanced way to rewrite it is using redux middleware. ***Middleware*** is somewhere you can put the code after the framework receives a request and before it generates a response. For example, we want to add a logger in the redux store so that when the store catches an action, before it returns the new state, the logger can log the previous state and the new generated state. This is what can be added as a middleware:</div>```javascript
function logger( store ) {
    return function wrapDispatch ( next ) {
        return function dispatchAndLog ( action ) {
            console.log( "dispatching.. ", action );
            let result = next( action );
            console.log( "new state", store.getState( ) );
            return result;
        }
    }
}
```
<div style="text-align: justify">There are more advanced ways to add a logger. If you are interested, please refer to the [offical documentation](https://redux.js.org/advanced/middleware). With our middleware, the previous example can be written in a cleaner way:</div>```javascript
const callAPI = ( ) => {
    return( ( dispatch ) => {
        dispatch( startCallingApiAction );
        actualCallApi( ).then( data => {
            dispatch(successAction( data ));
        }).fail( err => {
            dispatch( failedAction(err) );
        });
    });
}
```
<div style="text-align: justify">The successful response data is wrapped in the payload of the action, sent to the reducer. Once the store updates the data, it will be mapped as a prop back to the component and request for a rerender. This middleware is also called ***thunk***. By applying thunk to decouple the presentation layer, we can get rid of most of the side effects in components, instead, managing and orchestrating side effects in thunks.</div>
<div style="text-align: justify">This is great, so why are we even considering ***saga***? Well, one of the advantages of middleware is that it can be chained. Every middleware mounted in redux store starts an individual thread (or something really looks like a thread in ***NodeJS***). When a middleware captures an action and handles its side effect, it can dispatch a new action to another middleware to do nested logics. This behavior of middleware indicates that thunks can be chained as well, for example thunkA forwards its return payload to thunkB and thunkB forwards its return payload to... **Wait! That sounds quite familiar!! Is that the case of callback hell??** Unfortunately, a good thing plus another good feature doesn't always end up with something better. ~~It could be some shit as well (笑)~~ In this case, true, this is exactly the callback hell.</div></br>
## Redux Saga
<div style="text-align: justify">To handle the possible endless callback functions and also to make it more easily to test in a component which has complicated logics, we need to change our previous thoughts. Just like shifting from Process Oriented Programming to Object Oriented Programming, instead of telling the application how to handle the side effects, suppose it already knows how to call a function and how to dispatch an action, all we need to do is to **give instructions about what to do next** and we don't care about how those instructions will be executed (Saga handles the executions).</div>
Then the thunks example can be changed as following:
```javascript
export function* apiSideEffect( action ) {
    try{
        const data = yield call( actualCallApi );
        yield put({ type: "SUCCESS", payload: data });
    } catch ( err ) {
        yield put({ type: "FAILED", payload: err });
    }
}

export function* apiSaga( ) {
    yield takeEvery( "CLICK_TO_CALL_API", apiSideEffect );
}
```
There are serval fucntions already being integrated in Saga:
>***Call:*** the method call will return only a plain object describing the operation so redux-saga can take care of the invocation and returns the result to the generator. The first parameter is the generator function ready to be called and the rest params are all the arguments in the generator.

>***Put:*** Instead of dispatching an action inside the generator (Don't ever ever do that), ***put*** Returns an object with instructions for the middleware to dispatch the action.

>***Select:*** Returns value from the selector function, similar with **getState( )**. ***Note:*** It is not recommended to use this function because it returns the value corresponding to the contents of the store state tree, which is most likely a plain Javascript object and is **mutable** (Redux wants you to handle state immutably, which means return a new state instead of changing the old one).

>***Take:*** It creates a command object that tells the middleware to wait for a specific action. The resulting behavior of the call Effect is the same as when the middleware suspends the generator until a ***promise*** resolves. In the take case, it'll suspend the generator until a matching action is dispatched

By working with Saga, we make the side effects to be ***declarative*** rather than ***imperative***.
>***Declarative:*** describing what the program must accomplish, rather than describe how to accomplish it

>***Imperative:*** consists of commands for the computer to perform, focuses on describing how a program operates

<div style="text-align: justify">In the case of take, the control is inverted. Instead of the actions being pushed to the handler tasks, the **Saga is pulling the action by itself**. An additional generator, known as ***watcher*** which contains ***take*** has to be created to watch a specific action and being triggered once the following action is dispatched in the application. There are two ways to create a watcher, one is using the buid-in functions (***Saga Helper***):</div>```javascript
function* watchFetchData( ) {
    yield takeEvery( "FETCH_REQUEST", callFetchDataApi );
}
```
<div style="text-align: justify">***takeEvery*** allows multiple request to be proceeding at the same time. Or if you just want the latest request to be fired (the older one will be overrided during each time the watcher is triggered):</div>```javascript
function* watchFetchData( ) {
    yield takeLatest( "FETCH_REQUEST", callFetchDataApi );
}
```
<div style="text-align: justify">However by using ***take***, it is possible to fully control an action observation process to build complex control flow:</div>```javascript
function* watchFetchData( ) {
    while(true) {
        const action = yield take( "FETCH_REQUEST" );
        console.log( action );
        yield call( callFetchDataApi, action.payload );
    }
}
```
<div style="text-align: justify">All right, now you have been exposed to everything you need to know before start trying redux saga on your own. Here is a short overall example that may also help:</div>
Store:
```javascript
const sagaMiddleware = createSagaMiddleware( );
const store = createStore( rootReducer, appluMiddleware(sagaMiddleware) );
sagaMiddleware.run( watchFetch );
```
Sagas:
```javascript
function* watchFetch( ): Generator<*, *, *> {
    yield takeEvery( "FETCH_ACTION", callFetchAPI );
}

function* callFetchAPI( ): Generator<*, *, *> {
    try {
        yield put({ type: "FETCHING", payload: ... });
        const data = yield call( actualCallApi );
        yield put({ type: "FETCH_SUCCESS", payload: data });
    } catch ( err ) {
        yield put({ type: "FETCH_FAILED", payload: err });
    }
}
```
Reducer:
```javascript
const reducer = ( state = initState, action ) => {
    switch( action ) {
        case "FETCHING":
            return { loading: true, ...state };
        case "FETCH_SUCCESS":
            return { loading: false, success: true, data: action.payload, ...state };
        case "FETCH_FAILED":
            return { loading: false, success: false, error: true, ...state };
        default:
            return { ...state };
    }
}
```
Component:
```javascript
class myComponent extends React.Component {
    const mapStateToProps = ...
    const mapDispatchToProps = ...
    render( ) {
        return (
            <button onClick = { ( ) => this.props.dispatch({ type: "FETCH_ACTION" }) }/>
            {
                this.props.loading?
                    <p>Loading..</p> : this.props.error?
                        <p>Error!</p> : <p>{this.props.data}</p>
            }
        );
    }
}
export default connect( mapStateToProps, mapDispatchToProps )( myComponent );

```
<div style="text-align: justify">For more advanced concepts, there is a well-organized [Saga offical documentation](https://redux-saga.js.org/docs/advanced/) you can refer to if you want to dive deeper.</div></br>
## How to test Saga?
<div style="text-align: justify">A function that returns a simple object is easier to test than a function that directly makes an asynchronous call. For redux saga, each time you yield a function call will return a plain javascript object which makes the workflow much easier to test. You don’t need to use the real API, fake it, or mock it, instead just iterating over the generator function, asserting for equality on the values yielded.</div>```javascript
describe( "fetch work flow", ( ) => {
    const generator = cloneableGenerator( callFetchAPI )({ type: "FETCH_ACTION" });
    expect( generator.next( ).value ).toEqual( put({ type: "FETCHING", payload: ... }) );

    test( "fetch success", ( ) => {
        const clone = generator.clone( );
        expect( clone.next( ).value ).toEqual( put({ type: "FETCH_SUCCESS" }) );
        expect( generator.next( ).done ).toEqual( true );
    });
});
```
<div style="text-align: justify">In the above example, we use **clone( )** to test different control flows and **next( )** to iterate to the next function ready be yielded. The mock return value can also be injected as an argument of **next( )**:</div>```javascript
expect( clone.next( false ).value ).toEqual( put( fetchFailedAction( ) ) );
```
</br>
## Saga vs Observables
<div style="text-align: justify">Redux saga is not the only solution to our apps which may have complex control flows, they are other helpful tools providing different trade-offs which can also resolve the async problems. Here are some good [code snippets](https://hackmd.io/s/H1xLHUQ8e) of saga vs observables that can open your mind :D</div>
</br>
</br>
## References:
https://redux-saga.js.org/
https://stackoverflow.com/questions/25098066/what-is-callback-hell-and-how-and-why-rx-solves-it
https://redux.js.org/advanced/middleware
https://pub.dartlang.org/packages/redux_thunk
https://codeburst.io/how-i-test-redux-saga-fcc425cda018
https://engineering.universe.com/what-is-redux-saga-c1252fc2f4d1
https://www.sitepoint.com/redux-without-react-state-management-vanilla-javascript/
https://redux.js.org/introduction/getting-started
https://blog.logrocket.com/understanding-redux-saga-from-action-creators-to-sagas-2587298b5e71
