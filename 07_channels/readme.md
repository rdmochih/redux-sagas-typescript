# 07 Channels

# Summary

Channels is a power mechanism, makes a lot of sense when combined with sockets (real time information) on
the server side.

In this sample: 
  - We are going to expose a fake socket server (it will just emulate sending patients heart bits real time).

  - We are going list through that socket using sagas + channels.

You will need to run both server and client side.

# Steps

- We will take as starting point _01_hello_saga_ let's copy the content of that project 
and execute from bash / cmd the following command:

Change directory to _./frontend_ and execute:

```bash
npm install
```

Change directory to _./backend_ and execute:

```bash
npm install
```


- Let's run the server side project.

Change directory to _./backend_ and execute:

```
npm start
```

- Now we will change directory to front end and start working on that.

- Let's start by working on the saga that will read from the socket on por 1337 (to avoid
adding extra steps to this sample we won't bother about configuring enviroment variables
for this project).

Steps:

0. Let's install _socket.io_ library to read fro the websocket.

1. We will define two tasks: 
 
 - START_SOCKET_SUBSCRIPTION: Will be called on the _componentWillMount_ event of the table that will display that data in real time.
 
 - STOP_SOCKET_SUBSCRIPTION: Will be called on the _componentWillUnmount_ event of of the table that will display that data in real time.

2. We will c

Le'ts get started

- First let's install _socket.io_ library (under frontend folder) and it's type definition:

```bash
npm install socket.io-client --save
```

```bash
npm install @types/socket.io-client --save-dev
```

- Let's create the action Id's:

_./common/index.ts_

```diff
export const actionIds = {
  GET_NUMBER_REQUEST_START: '[0] Request a new number to the NumberGenerator async service.',
  GET_NUMBER_REQUEST_COMPLETED: '[1] NumberGenerator async service returned a new number.',
+  START_SOCKET_SUBSCRIPTION: '[2] Start listening to the web socket',
+  STOP_SOCKET_SUBSCRIPTION: '[3] Close socket connection',
}

export interface BaseAction {
  type : string;
  payload: any;
}
```

- Now let's define the action creators (real life project would pass extra params in this 
action creators). Add this content to the bottom of the file.

_./actions/index.ts_

```typescript
export const startSocketSubscriptionAction : () => BaseAction = () => ({
 type: actionIds.START_SOCKET_SUBSCRIPTION,
 payload: null,
});

export const stopSocketSubscriptionAction : () => BaseAction = () => ({
 type: actionIds.STOP_SOCKET_SUBSCRIPTION,
 payload: null,
});
```

- Let's start implementing the _socket_ sagas.

First we will add a helper method that will allow us connect to a server and return
a promise wrapping the result of this connection in a promise (socket.io returns
just a callback), we could wrap this into a separate file.

_./src/sagas/socket.ts_

```typescript
import * as ioClient from 'socket.io-client';
import { all, fork, take, call } from 'redux-saga/effects';
import { actionIds } from '../common';

function connect() {
  // Real life project extract this into an API module
  const socket = ioClient.connect('http://localhost:1337/', null);

  // We need to wrap the socket connection into a promise (socket returs callback)
  return new Promise((resolve, reject) => {
    socket.on('connect', () => {
      socket.emit('messages');
      resolve({ socket });
    });

    socket.on('connect_error', err => {
      console.log('connect failed :-(');
      reject(new Error('ws:connect_failed '));
    });
  }).catch(error => ({ socket, error }));
}
```

> We have harcoded connection url and settings, in a real project we would parametrize this info.

- Let's register the _flow_ saga in the _index_ saga file. This will listen for the *START_LISTENING_SOCKET*
action, once is fired it will try to establish a connection with the socket (we will extend that saga 
later on). Append this content to the bottom of the file.

_./src/sagas/socket.ts_

```typescript
function* flow() {
	while(true) {
		yield take(actionIds.START_SOCKET_SUBSCRIPTION);
		const {socket, error} = yield call(connect);
		if(socket) {
			console.log('connection to socket succeeded');
		} else {
			console.log('error connecting');
		}
	}
}
```

- Let's wrap all this in a _socketRootSaga_. Append this content to the bottom of the file.

_./src/sagas/socket.ts_

```typescript
export function *socketRootSaga() {
	yield all([
		fork(flow),
	])
}
```

- Let's register this _socketRootSaga_ in the _saga/index_ main saga.

_./src/sagas/index.ts_

```diff
import { actionIds } from '../common'
+ import { socketRootSaga } from './socket'

// Register all your watchers
export const rootSaga = function* root() {
  yield all([
    fork(watchNewGeneratedNumberRequestStart),
+   fork(socketRootSaga),    
  ])
}
```

- Right now let's make a quick stop and check that we are able to establish connection with the socket.

- Time to jump into the ui side.

- Let's create a component we will call it _currency-table.component.tsx_, it will just 
fire the _Start Socket Subscription_ task.

_./src/components/currency-table/currency-table.component.tsx_

```typescript
import * as React from 'react';

interface Props {
  connectCurrencyUpdateSockets : () => void;
  disconnectCurrencyUpdateSockets : () => void;
}

export class CurrencyTableComponent extends React.PureComponent<Props> {
  componentWillMount() {
    this.props.connectCurrencyUpdateSockets();  
  }

  componentWillUnmount() {
    this.props.disconnectCurrencyUpdateSockets();  
  }

  render() {
    return (
      <h3>Currency Table component</h3>
    )
  }
}
```

- Let's create a container we will call it _bids-table.container.tsx_,it will connect
the _START_SOCKET_SUBSCRIPTION_ action creator with the _startSocketConnection_ 
component prop callback.

_./src/components/currency-table/currency-table.container.tsx_

```typescript
import {connect} from 'react-redux';
import {State} from '../../reducers';
import {CurrencyTableComponent} from './currency-table.component';
import {startSocketSubscriptionAction, stopSocketSubscriptionAction} from '../../actions';

const mapStateToProps = (state : State) => ({
})

const mapDispatchToProps = (dispatch) => ({
  connectCurrencyUpdateSockets: () => dispatch(startSocketSubscriptionAction()),
  disconnectCurrencyUpdateSockets: () => dispatch(stopSocketSubscriptionAction()),
})

export const BidsTableContainer = connect(
  mapStateToProps,
  mapDispatchToProps
)(CurrencyTableComponent);
```

- Let's add the _BidsTableContainer_ to the components barrel.

_./src/components/index.ts_

```diff
export {MyNumberBrowserContainer} from './my-number/browser/my-number-container';
export {MyNumberSetterContainer} from './my-number/setter/my-number-setter.container';
+ export {CurrencyTableContainer} from './currency-table/currency-table.container';
```

- Let's add this container to the _main.tsx_ main component:

_./src/main.tsx_

```diff
- import { MyNumberBrowserContainer, MyNumberSetterContainer } from './components';
+ import { MyNumberBrowserContainer, MyNumberSetterContainer, BidsTableContainer } from './components';
//(...)

ReactDOM.render(
  <Provider store={store}>
    <>
+     <CurrencyTableContainer/>
+     <br/>    
      <MyNumberSetterContainer />
      <MyNumberBrowserContainer />    
    </>
  </Provider>,
  document.getElementById('root'));
```

- Let's launch the project and include breakpoints in the _socket_ saga to check that
connection is succesfully established.

First we will launch our socket backend.

```bash
cd backend
npm start
```

Now let's launch the front end:

```bash
cd frontend
npm start
```

> We can check as well the browser console and check the connection traces we have added.

- Time to move forward, now we want to read from the socket incoming data.


- On the sagas side: Let's start reading data from the socket, in order to do this:  
  - On the sagas side, we will start by creating a channel:
    - We will wrap it in a _subscribe_ function, passing as parameter the _socket_ 
    connection.
    - This channel exposes an _emit_ param that will let us communicate between the 
    channel itself and subscribers of this channel.
    - We will read for the socket _messages_ that we will recieve and let other
    generic / useful events (disconnection, error...)

The main message we are going to liste is "patients" , this will call an action creator
_onSocketMessageReceived_ that will dispatch the message.

Let's add this function right after the _connect_ function

_./src/sagas/socket.ts_

```typescript
function subscribe(socket) {
  return eventChannel(emit => {
    socket.on('patients', (message) => {
      console.log(message);      
    });
    socket.on('disconnect', e => {
      // TODO: handle
    });
    socket.on('error', error => {
      // TODO: handle
      console.log('Error while trying to connect, TODO: proper handle of this event');
    });

    return () => { };
  });
}
```

> About the _return () => {}_ statement is a function that is called on channel closed / cleanup
(free resources).

- Let's create a _read_ saga, that will subscribe to the _eventChannel_ we have created before.

_./src/sagas/socket.ts_

```typescript
function* read(socket) {
  const channel = yield call(subscribe, socket);
  while (true) {
    let action = yield take(channel);
    yield put(action);
  }
}
```

- Altough in this task we are going to read and not write, we will create an
  IO saga that would be able to launch read and write tasks.

_./src/sagas/socket.ts_

```typescript
function* handleIO(socket) {
  yield fork(read, socket);
  // TODO in the future we could add here a write fork
}
```

- We will call this _handleIO_ saga from the _flow_ saga (remember to add the redux-saga
needed imports, e.g. _cancel_.

```diff
function* flow() {
	while(true) {
		yield take(actionIds.START_SOCKET_SUBSCRIPTION);
		const {socket, error} = yield call(connect);
		if(socket) {
+			console.log('connection to socket succeeded');
+     const ioTask = yield fork(handleIO, socket);
+     yield take(actionIds.STOP_SOCKET_SUBSCRIPTION);
+     yield cancel(ioTask);
		} else {
			console.log('error connecting');
		}
+   socket.disconnect();    
	}
}
```

- Let's give a quick try (debugging again...). Remember to fire as well the server socket.

```
npm start
```

- It's time to add some ui.

- On the redux side.
   - Let's add a model.
   - Let's add an action creator _onSocketMessageReceived_
   - Let's create a _currencies.reducers.ts_
   - Let's register it.

- We are going to add a simple typescript entity that will hold the currency deltas.

_./src/model/index.ts_

```typescript
export interface CurrencyUpdate {
  id: string;
  currency : string;
  change: string;
}
```

- Let's add a new action id (currency udpate):

_./src/common/index_

```diff
export const actionIds = {
  GET_NUMBER_REQUEST_START: '[0] Request a new number to the NumberGenerator async service.',
  GET_NUMBER_REQUEST_COMPLETED: '[1] NumberGenerator async service returned a new number.',
  START_SOCKET_SUBSCRIPTION: '[2] Start listening to the web socket',
  STOP_SOCKET_SUBSCRIPTION: '[3] Close socket connection',
+  CURRENCY_UPDATE_RECEIVED: '[5] Got a currency update from the server',
}
```

- Let's add an action creator Let's add an action creator _onSocketMessageReceived_ (append
this code at the end of the file)

_./src/actions/index.ts_

```typescript
import { CurrencyUpdate } from '../model';
// (...)

export const currencyUpdateReceivedAction : (update : CurrencyUpdate) => BaseAction = (update) => ({
  type: actionIds.CURRENCY_UPDATE_RECEIVED,
  payload: update,
 });
```

- Let's create a _currencies.reducer.ts_

_./src/reducers/currencies.reducer.ts_

```typescript
import { BaseAction, actionIds } from '../common';
import { CurrencyUpdate } from '../model';

export type CurrenciesState = CurrencyUpdate[];

export const currenciesReducer = (state: CurrenciesState = [], action: BaseAction) => {
  switch (action.type) {
    case actionIds.CURRENCY_UPDATE_RECEIVED:
      return handleCurrencyUpdateCompleted(state, action.payload); 
  }

  return state;
}

const handleCurrencyUpdateCompleted = (state : CurrenciesState, currencyUpdate : CurrencyUpdate) : CurrenciesState => {
  const notUpdated = state.filter((currency) => currency.id != currencyUpdate.id);

  return [currencyUpdate, ...notUpdated];
```

- Let's register it.

__./src/reducers/index.ts_

```diff
import { combineReducers} from 'redux';
import { myNumberCollectionReducer, MyNumberCollectionState } from './my-number.reducer';
+ import { currenciesReducer, CurrenciesState} from './currencies.reducer';

export interface State {
  myNumberCollectionState : MyNumberCollectionState;
+  currenciesState : CurrenciesState;
};

export const reducers = combineReducers<State>({
  myNumberCollectionState: myNumberCollectionReducer,
+  currenciesState: currenciesReducer,
});
```


- Let's move back on Sagas and replace our _console.log_ call with the _onSocketMessageReceived_
actions call

_./src/sagas/socket.ts_

```diff
+ import {currencyUpdateReceivedAction} from '../actions'
// ...

function subscribe(socket) {
  return eventChannel(emit => {
    socket.on('currency', (message) => {
      console.log(message);
+     emit(currencyUpdateReceivedAction(message));
    });
``` 

- On the UI side:
   - Let's add a _currencies_ property to the currencies table component.
   - Let's display a table showing the current currencies values.      
   - Let's connect the bids container with the reducer property (currencies).

- Let's add a _currencies_ property to the currencies table component.

_./src/components/currency-table/currency-table.component.tsx_

```diff
import * as React from 'react';
+ import {CurrencyUpdate} from '../../model';

interface Props {
  connectCurrencyUpdateSockets : () => void;
  disconnectCurrencyUpdateSockets : () => void;
+ currencyCollection: CurrencyUpdate[];  
}
```  

- Let's display a table showing the current currencies values.      

_./src/components/currency-table/currency-table.component.tsx_

```diff
  render() {
    return (
-      <h3>Bids Table component</h3>
+      <table>
+        <tr>
+          <th>
+            Currency
+          </th>
+          <th>
+            Change
+          </th>
+        </tr>
+        {this.props.currencyCollection.map(
+          currency =>
+            <tr key={currency.id}>
+              <td>{currency.currency}</td>
+              <td>{currency.change}</td>
+            </tr>
+        )
+        }
+      </table>
    )
  }
```

- Let's connect the currency container with the reducer property (currencies).

_./src/components/currency-table/currency-table.container_

```diff
import {connect} from 'react-redux';
import {State} from '../../reducers';
import {CurrencyTableComponent} from './currency-table.component';
import {startSocketSubscriptionAction, stopSocketSubscriptionAction} from '../../actions';

const mapStateToProps = (state : State) => ({
+   currencyCollection: state.currenciesState,  
})
```

- Let's run the sample.


> Excercise A: implement a getAllCurrencies message.

Tips:

On backend:

## Excercise A:

Create a listen.js file

_listen.js_

```javascript
var currencyDb = require('./currencyDb');

module.exports = function (socket) {
  // Let's listen for the 'currencies' request then answer with a 'currencies'
  // including the whole list
  socket.on('currencies', () => {
    console.log('currencies request');
    currencyDb.find({}, (err, currenciesList) => {
      socket.emit('currencies', currenciesList);
    });
  });
}
```

On backend on main

_./socketIo.js_

```diff
+ var listen = require('./listen')

let clients = [];

const notifyClients = (msg, data) =>
    clients.forEach(socket => socket.emit(msg, data));

module.exports = (io) => {
    io.on('connection', function (socket) {
        clients.push(socket);

+        listen(socket);

        socket.on('disconnect', () => {
            clients = clients.filter((s) => s.id = socket.id);
        });
```
On the front end side:

- On the socket saga:

- On Connection emit _currencies_ message.

./src/sagas/socket.ts

```diff
  return new Promise((resolve, reject) => {
    socket.on('connect', () => {  
+      socket.emit('currencies');          
      resolve({ socket });
    });
```

- Listen to Messages socket message, we stop here on a console log.

./src/sagas/socket.ts

```diff
function subscribe(socket) {
  return eventChannel(emit => {
    socket.on('currency', (message) => {
      console.log(message);
      emit(currencyUpdateReceivedAction(message));
    });

+   socket.on('currencies', (message) => {
+      console.log(message);
+   });

    socket.on('disconnect', e => {
      // TODO: handle
    });
    socket.on('error', error => {
      // TODO: handle
      console.log('Error while trying to connect, TODO: proper handle of this event');
    });

    return () => { };
  });
}
```

- Nex Steps:
  - Create action that will replace all data.
  - Create reducer
  - Handle Action.
  - No need to update the UI 

## Excercise B:

> Excercise B: Instead of replace and place on top, keep the original current position and keep
the update immutable, plus implement a test on the reducers to ensure the update is immutable.




