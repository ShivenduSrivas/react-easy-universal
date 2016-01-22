# React Redux Universal

An opinionated, batteries-included universal routing &amp; rendering library for React+Redux apps.

## Status

Developer preview. Nothing to see, yet.


## What's included?

* Redux
* [React Router]()
* Automatic syncing between Redux and React Router via [redux-simple-router](https://github.com/rackt/redux-simple-router)
* History

## Why is this needed? (AKA, the old `n` busted way)

Universal routing & rendering with React and Redux is pretty great. For serious app projects, it can save you a ton of time, but if you have ever tried configuring it yourself, you'll know, it's a lot harder than it should be. Just to give you an idea of how complicated it can be, here's the example from `redux-simple-router`, which is AFAIK, the easiest way to configure this stuff right now:

```js
import React from 'react'
import ReactDOM from 'react-dom'
import { createStore, combineReducers, applyMiddleware } from 'redux'
import { Provider } from 'react-redux'
import { Router, Route, browserHistory } from 'react-router'
import { syncHistory, routeReducer } from 'redux-simple-router'
import reducers from '<project-path>/reducers'

const reducer = combineReducers(Object.assign({}, reducers, {
  routing: routeReducer
}))

// Sync dispatched route actions to the history
const reduxRouterMiddleware = syncHistory(browserHistory)
const createStoreWithMiddleware = applyMiddleware(reduxRouterMiddleware)(createStore)

const store = createStoreWithMiddleware(reducer)

// Required for replaying actions from devtools to work
reduxRouterMiddleware.listenForReplays(store)

ReactDOM.render(
  <Provider store={store}>
    <Router history={browserHistory}>
      <Route path="/" component={App}>
        <Route path="foo" component={Foo}/>
        <Route path="bar" component={Bar}/>
      </Route>
    </Router>
  </Provider>,
  document.getElementById('mount')
)
```

That's six dependencies that you have to manage in your own app, and good luck getting all the right versions that will play together nicely. And this is just the client side. The code above doesn't work on the server. To get that going, you'll have to create an Express route handler that manually calls `match()` from `react-router`, checks for errors & redirects, and maybe renders the document. It looks something like this:


```js
import React from 'react';
import { match } from 'react-router';

import renderLayout from 'server/render-layout';
import render from 'server/render';
import settings from 'server/settings';

import configureStore from 'shared/configure-store';
import createRoutes from 'shared/routes';

const store = configureStore();
const routes = createRoutes(React);
const initialState = store.getState();

export default (req, res) => {
  match({ routes, location: req.url }, (error, redirectLocation, renderProps) => {
    if (error) {
      res.status(500).send(error.message);
    } else if (redirectLocation) {
      res.redirect(302, redirectLocation.pathname + redirectLocation.search);
    } else if (renderProps) {
      const rootMarkup = render(React)(renderProps, store);
      res.status(200).send(renderLayout({ settings, rootMarkup, initialState }));
    } else {
      res.status(404).send('Not found');
    }
  });
};
```

It took me two days to get these examples working in one of my own projects. 2 days of fiddling with dependencies, copying the exact versions out of the example repositories and into my `package.json`. FYI, at the time of this writing, the dependency versions in the example repo are not compatible with the client snippet above.

So, you could track all these dependency versions yourself (and they're all being rapidly updated) -- or, you could use this library, plug in your routes & reducers, and get on with building an actual application instead of chasing all the moving parts around. About as fun as herding cats.

Now let's look at what this could look like:


## Getting Started

You'll need to create three files:

`create-app.js`:

```js
import universal from 'react-redux-universal';
import React from 'react';

import routes from './path/to/your/routes';
import reducers from './path/to/your/reducers';

const createApp = ({
  app,
  rootID, // default: 'root'
  rootRoute, // default: '/'
  renderLayout // Skeleton DOM render template for the server-side. Default: Barebones ES6 template
}) => universal({
  app, rootId, rootRoute, renderLayout,
  React, routes, reducers
});

export default createApp;
```


`client.js`:

```js
import React from 'react';
import createApp from './path/to/create-app.js';

const app = createApp({ React }); // use all the defaults

app(); // returns a function that must be invoked to trigger render
```


`server.js`:

```js
import express from 'express';

import renderLayout from './path/to/render-layout.js';
import createApp from './path/to/create-app.js';

const expressApp = express();

app.use('/static', express.static(staticDir));

// Passing in the express app lets it know you want the server
// version, and it wires up the route automatically
const app = createApp({ React, expressApp });

const port = process.env.APP_PORT || 3000;

app.listen(port, (err) => {
  if (err) {
    console.log(err);
    return;
  }

  console.log(`Listening at http://localhost:${ port }`);
});
```


### Defining Your Routes

Use this module instead of depending directly on React Router, and we'll worry about keeping all the version dependencies compatable and in-sync for you.

```js
import { Router, Route } from 'react-redux-universal';

import createHome from 'shared/components/home';
import createTestData from 'shared/components/test-data';

// It expects a factory function that it can inject dependencies into.
export default (React, browserHistory) => {

  return (
    <Router history={ browserHistory }>
      <Route path="/" component={ createHome(React) } />
      <Route path="/test-data" component={ createTestData(React) } />
    </Router>
  );
};
```