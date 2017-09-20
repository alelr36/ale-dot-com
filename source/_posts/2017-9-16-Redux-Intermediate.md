---
title: Redux Selectors, Sagas, Middlewares and more...
author: Ale Arce
summary: >
          This article is aimed to show some middle level angles when using
          Redux in the front end. Here we will cover some <b><i>non-typical</i></b>
          actors / tools, identifying their roles and limitations. By doing
          this, I intend to explain how important is to decouple responsibilities.
date: 2017-09-16 11:25:58
tags: ['redux', 'frontend', 'js', 'javascript', 'architecture', 'selectors', 'sagas', 'middlewares']
share_link: http://www.alearce.io/2017/09/16/Redux-Intermediate/
---
## Intro

In my first post I wanted to share some appreciations I have regarding specific practices in the frontend development when implementing Redux. I will try to explain how you can<b> shape your assets in the frontend</b> so you can keep an organized structure. I will try to improve code examples gradually, from simple to more complex but robust code...
If I am lucky, I will receive some feedback on this and then I can perform updates on this post.
In several parts of the article, I point to the [Reducers Recommended Structure](#Reducers-Recommended-Structure) section, you may want to read it first and then re-read it when necessary, due this reducers structure is one of the core concepts of the article.

I assume that you have basic knowledge of Redux and strong knowledge of ES6. Also I will be using React to build example components.
## What we will cover
<i>[Folder Structure](#Folder-Structure)</i>
<i>[Why Smart and Dumb Components](#Why-Smart-and-Dumb-Components)</i>
<i>[Standard Actions](#Standard-Actions)</i>
<i>[Middlewares](#Middlewares)</i>
<i>[Redux Selectors](#Redux-Selectors)</i>
<i>[Reducers Recommended Structure](#Reducers-Recommended-Structure)</i>
<i>[Store Config](#Store-Config)</i>

## Folder Structure
I found this structure very effective. By no means I intend to say it will be useful for you, but in order to understand some of the examples I provide on this post, it's necessary to see the files and folders structure, plus a brief summary of each one of them. I took this example form a previous project I was working on, I will focus on the redux-related parts:
<p align="center">![](/images/folder-structure.png)</p>
### /actions
Contains business-entity-grouped actions. We will go deeper on this.
### /api
This folder contains the API communication layer of the project.
### /components
Dumb components, no business logic, no redux needed.
### /constants
I recommend to place one single file containing all the string constants representing action names. I will provide more detail for this file, and of course you may decide to have several files instead of a single one.
### /containers
Smart components, specific business logic, redux needed.
### /reducers
Pretty self explanatory, reducers go here.
### /sagas
Here we will place the needed sagas. I will provide an explanation of redux-sagas, however, it's a quite advanced concept and I'm pretty much a beginner on this.
### /stores
Here we create the store, one of the key parts of Redux. We apply the middlewares to the store as well.

## Why Smart and Dumb Components?

One of the most important things when using redux, is to keep in mind that you should follow a <i>"reusable components"</i> philosophy. More precisely, inside the Redux world most of the developers have taken an smart-components and dumb-components approach:

### Smart Components
> Those that have specific business logic and probably are specific to your app. These components <strong>use Redux</strong>, because they attach (connect) themselves to the parts of the app-state they need in order to work. This is what Redux is meant for. An example of a smart component would be a UsersListComponent, or may be a BirthdaySelectComponent.

### Dumb Components
> They are context / business agnostic, and this makes them fully reusable. So, an example of a dumb-component would be a ListComponent. This is not a UsersListComponent, or a ProductsListComponent. Dumb components don't have business knowledge, they are fully reusable as long as you provide "the firm" they need to render, and most importantly, <strong>they don't need Redux</strong>, because all its data is provided by some other smarter component.

Redux is a framework to maintain the app's state, <strong><i>why am I talking about smart and dumb components</i></strong>?
Smart components will use redux, dumb components won't. This is important for this article given that I'm trying to explain a way to keep your code clean. <strong><i>If all your components access the app state, your code can get messy very quickly</i></strong>.

## Standard Actions
I would describe actions as public notices, distributed within your app. This means that someone will trigger an action and some other subscribers may do something regarding that action. Pretty much like an observer pattern, but cooler. Actions must first be defined, and then they can be triggered, intercepted and analyzed to do something about them. These are examples of <strong><i>action definitions</i></strong>:
```javascript
// src/actions/products.js
import { createAction } from 'redux-actions'

export const selectProduct = createAction('PRODUCTS_SELECT', product => product.id)
export const fetchProducts = createAction('PRODUCTS_FETCH', () => [{name: 'P1', id: 1}, {name: 'P2', id: 2}])
/* We will define fetchProducts soon, per now it just hardcodes an array of products as payload */
/* This file uses some ES6 features. It may be confusing if you are not used to them */
```

<details>
  <summary><i>A bit more detail regarding createAction</i></summary>`createAction` is a function that receives 3 parameters: `actionName`, `payloadCreator` and `metadataCreator`. I won't deepen that much on this topic, but here's a basic explanation:
  \- `actionName`: a string representing the identifier of the action.
  \- `payloadCreator`: a function definition, that will receive the arguments provided by the action invoker, and returns the payload accessible in reducers watching the action (action.payload).
  \- `metadataCreator`: a function definition, that will receive the arguments provided by the action invoker, and returns the metadata accessible in reducers watching the action (action.metadata).
  Deciding if something is payload or metadata, is up to you.
</details>

Note here that actions are function definitions that expect to be invoked with some data. In this example, `selectProduct` is a function that expects to be invoked with an object, <strong><i>locally called</i></strong> product. The action definition trusts that the product will have an `id` property. As this action definition is returning `product.id`, that `id` will be the payload in the reducer subscribed to the action `PRODUCTS_SELECT`. If we want we can send the entire product to the reducer, by simply doing `product => product)`.
Im my experience, it's very common to see `lodash` usages (`find`, `get`, `filter`, `reduce`, `first`), in these action definitions. I believe there's no problem with that. Remember, what `payloadCreator` returns, will be `action.payload` in the reducers watching for the action, and that's what you will be able to include in the app's state. We want to keep the state as dumb as possible, very simple. If we need to transform / decorate the simple data we stored in the app's state, we can do it by using [selectors](#Redux-Selectors).

Given those two defined actions, some smart component (consisting of two files), may do something like this:

```javascript
// src/containers/products/index.js
import { connect } from 'react-redux'
import { bindActionCreators } from 'redux'
import { fetchProducts, selectProduct } from 'actions/products'
import ProductsListComponent from './ProductsListComponent'

let mapStateToProps = state => ({
  products: state.products.productsList,
  selectedProduct: state.products.selectedProduct
})

let mapDispatchToProps = dispatch => bindActionCreators({
  selectProduct,
  fetchProducts
}, dispatch)

export default connect(mapStateToProps, mapDispatchToProps)(ProductsListComponent)
```
At this point, in the component definition has access to four props:
* `this.props.products`: <strong>connected</strong> to `state.products.productsList`.

* `this.props.selectedProduct`: <strong>connected</strong> to `state.products.selectedProduct`.

* `this.props.selectProduct`: a function I can call, which will result in an action (`PRODUCTS_SELECT`)

* `this.props.fetchProducts`: a function I can call, which will result in an action (`PRODUCTS_FETCH`)

```javascript
// src/containers/products/products.jsx
import React from 'react'

class ProductsListComponent extends React.Component {
  componentDidMount() {
    this.props.fetchProducts() // once mounted, we will fetch. Fetch is returning hardcoded data for now
  }

  render() {
    return (
      <div>
        <ul>
          {
            this.props.products.map((product, key) => (
              <li key={key} onClick={() => this.props.selectProduct(product)}>{product.name}</li>
            ))
          }
        </ul>
      </div>
    )
  }
}

export default ProductsListComponent
```

> Note that this smart component is triggering actions differently:
* `this.props.selectProduct(product)` is triggered and it includes data for the `payloadCreator`.
* `this.props.fetchProducts()` is triggered but it doesn't include a payload, because it's not needed so far.

> <strong><i>If you want to send a payload when executing an action</i></strong>, you need to provide arguments to the function call, i.e. `this.props.myActionName({some: 'payload'}, {some: 'metadata'})`. These parameters are received by the action creators, like `product` is being received by selectProduct, which takes it and returns `product.id`. All reducers expecting for a `PRODUCTS_SELECT` action, will receive the new `id`.

Until now, we have have defined some actions and some component connections to those actions and to some parts of the state as well. The next step is <strong><i>understand the needed reducer</i></strong>, which will receive the payload and update the corresponding part of the state. By doing this, <strong><i>all the components connected to those parts of the state</i></strong>, will receive the new version of the state.
You may find this reducer example a bit strange, I haven't completely described my approach to reducers yet. You can [take a look at that section](#Reducers-Recommended-Structure) now if you want.
```javascript
// src/reducers/products.js

import { handleActions } from 'redux-actions'
import * as actionTypes from 'constants/action-types'

const { PRODUCTS } = actionTypes

const initialState = {
  productsList: [],
  selectedProduct: ''
}

export default handleActions({
  [`${PRODUCTS.SELECT}`]: (state, action) => ({
    ...state,
    selectedProduct: action.payload // action.payload will be the product id provided by the action as I described earlier
  }),
  [`${PRODUCTS.FETCH}`]: (state, action) => ({
    ...state,
    productsList: action.payload // action.payload will be the hardcoded array I provided earlier
  })
}, initialState)
```
Now, the Redux cycle on this practical example:
* A smart component is mounted and it triggers, `this.props.fetchProducts()`. No payload needed
* The action definition for `fetchProducts` is per now returning a hardcoded list of products, that will end up in the reducer watching for the action `PRODUCTS_FETCH`.
* The reducer updates the state with the list of products. All components connected to `state.products.productsList` will receive the update.
  * Particularly, `products.jsx` will now iterate through the provided array instead of an empty one (initial state).

## Middlewares
Middlewares are fragments of code you integrate to your app. <i>They work in the middle</i> of the actions pipeline, analyzing every action and deciding if they should do something about it or not, and then passing the action to the next middleware or actor in the pipeline. For this article I want to introduce the following middlewares:

[Redux Thunk](https://github.com/gaearon/redux-thunk)
[Redux Promise Middleware](https://github.com/pburtchaell/redux-promise-middleware)
[Redux Saga](https://github.com/redux-saga/redux-saga)
[Redux Logger](https://github.com/evgenyrodionov/redux-logger)

I have found these middlewares really useful. These tools provide a mechanism to improve the actions workflow in the web app and keep it clean and logic. You can find how to include middlewares in the [store config section](#Store-Config). Let's talk about each one of them.
### Redux Thunk
So far, I've been talking about [reducers returning new versions of the state](#Reducers-Recommended-Structure), i.e. returning an object. The redux-thunk middleware <strong><i>checks if the </i></strong>`payloadCreator`<strong><i> returns a function instead of a plain object</i></strong>. If the `payloadCreator` is returning a function, then the middleware invokes that function with two parameters: `dispatch` and `getState`, both functions.
I wrote this based on the [official examples](https://github.com/gaearon/redux-thunk#motivation), the code can be improved but I wanted to leave it as explicit as possible.

```javascript
// src/actions/products.js
import actionTypes from 'constants/action-types'
import { createAction } from 'redux-actions'

const { PRODUCTS } = actionTypes

const selectProductAction = createAction('PRODUCTS_SELECT', productId => productId)

export let selectProduct = (product) => {
  return (dispatch, getState) => { // function returned, which redux-thunk will invoke with the well-know parameters
    const { products } = getState()

    if (products.selectedProduct === product.id) {
      return
    }

    dispatch(selectProductAction(product.id));
  }
}
```
My goal with this example is to show the following:
* An action to select a product is created.
* A thunk to select a product is created as well.
* The thunk checks if the payload provided is the same as the existing in the state.
  * If the data is the same, it returns null (single return statement).
  * If the data is different, then it dispatches the proper action.

Instead of directly returning a payload and which will update all the related reducers, you check something to decide between executing the action or not.

Why would you select a product that is already selected? <strong><i>Redux Thunk enables you to evaluate some criteria before dispatching an action</i></strong>.

### Redux Promise Middleware
I find this middleware a bit complicated to explain, so I'll do my best.

Remember that middlewares are (in part) action analyzers. In this case, to use `redux-promise-middleware`, the function `payloadCreator` needs to return an object whose only key is `promise`, and it's value is a promise instance, i.e. `{promise: promiseInstance}`.

```javascript
// src/api/products.js
import axios from 'axios'

export let fetchProducts = () => axios.get('https://some-api-url.com/products')
// axios' simple way to perform a get method
```

```javascript
// src/actions/products.js
import { createAction } from 'redux-actions'
import * as productsApi from 'api/products'
import actionTypes from 'constants/action-types'

const { PRODUCTS } = actionTypes

export const fetchProducts = createAction(PRODUCTS.FETCH, () => {
  const promise = productsApi.fetchProducts()

  return { promise }
})
// Also using some ES6 features here
```
`Redux-promise-middleware` detects this and automatically modifies the default action pipeline, avoiding the dispatch of `PRODUCTS_FETCH`, and producing two possible results:
* `PRODUCTS_FETCH_PENDING`
* `PRODUCTS_FETCH_FULFILLED`

Or...

* `PRODUCTS_FETCH_PENDING`
* `PRODUCTS_FETCH_REJECTED`

This two flows represent the possible states of a promise. With this, your reducers watch for actions `_PENDING`, `_FULFILLED` and `_REJECTED`.

Why is this useful? Let's see:
* By watching `_PENDING` you can set some loading value, in order to activate spinners or loading components in your front-end.
* By watching `_FULFILLED` you will receive in `action.payload`, the data provided by the back end response.
* By watching `_REJECTED` you can specify error messages based on the back end response.

```javascript
// src/reducers/products.js
import { handleActions } from 'redux-actions'
import * as actionTypes from 'constants/action-types'

const { PRODUCTS } = actionTypes

const initialState = {
  productsList: [],
  selectedProduct: '',
  loading: false,
  error: null
}

export default handleActions({
  [`${PRODUCTS.FETCH}_PENDING`]: (state, action) => ({
    ...state,
    loading: true,
    productsList: [],
    error: null
  }),
  [`${PRODUCTS.FETCH}_FULFILLED`]: (state, action) => ({
    ...state,
    loading: false,
    productsList: action.payload.data,
    error: null
    // use lodash to get the data _.get(action, 'payload.data', [])
    // remember action.payload in this case is a back-end response
  }),
  [`${PRODUCTS.FETCH}_REJECTED`]: (state, action) => ({
    ...state,
    loading: false
    productsList: [],
    error: `Something went wrong, ${action.payload.error.message}`,
    // use lodash to get the data _.get(action, 'payload.error.message', null)
    // remember action.payload in this case is a back-end response
  })
}, initialState)
```

With this, your components connected to these parts of the state, can logically change their content when these actions produce a change in the state. The action `PRODUCTS_FETCH` will never be dispatched. Instead, the middleware will ensure that `PRODUCTS_FETCH_PENDING` and the corresponding `_FULFILLED` or `_REJECTED` are thrown.
Let's see a possible component's code.

```javascript
// src/containers/products/index.js
import { connect } from 'react-redux'
import { bindActionCreators } from 'redux'
import { fetchProducts, selectProduct } from 'actions/products'
import ProductsListComponent from './ProductsListComponent'

let mapStateToProps = state => ({
  products: state.products.productsList,
  loading: state.products.loading, //new prop connected to the state
  error: state.products.error, //new prop connected to the state
  selectedProduct: state.products.selectedProduct
})

let mapDispatchToProps = dispatch => bindActionCreators({
  selectProduct,
  fetchProducts
}, dispatch)

export default connect(mapStateToProps, mapDispatchToProps)(ProductsListComponent)
```

```javascript
// src/containers/products/products.jsx
import React from 'react'

class ProductsListComponent extends React.Component {
  componentDidMount() {
    this.props.fetchProducts()
  }

  getProducts() {
    return (
      <ul>
        {
          this.props.products.map((product, key) => (
            <li key={key} onClick={() => this.props.selectProduct(product)}>{product.name}</li>
          ))
        }
      </ul>
    )
  }

  render() {
    return (
      <div>
        {this.props.loading && 'loading component'}
        {this.props.error && `Oops... ${error}`}
        {this.props.products.length > 0 && this.getProducts()}
      </div>
    )
  }
}

export default ProductsListComponent
```

This last example requires at least an intermediate knowledge of ES6.

### Redux Saga
I believe Sagas is a quite advanced concept, so I will be giving some very basic approach to understand what they do, and how to implement them.
Until now, we went through the main actors of Redux. If I was clear enough, you we can agree that so far, the only listeners / watchers of actions, are the reducers. The goal of a reducer is modify a specific part of the app's state. I would describe a Redux Saga as actions watchers as well, but in this case they do not modify the app's state. What a saga does, is to execute some code after an action is dispatched. This is useful in some specific cases:
* Given an action:
  * You want to dispatch another specific action, or several actions.
  * You want to change the url.
  * You want to store something in sessionStorage.
  * etc...

I will take the first case as an example to show you how to use Redux Sagas. Let's continue with the last example of `products/index.js` and `products/products.jsx`.
So far, when `state.products.productsList` is an array of products, `products.jsx` will render an `ul > li` of products. With the current code, clicking on any of those `li` will trigger a `PRODUCT_SELECT` action. Let's put a class on the selected `li`:

```javascript
getProducts() {
  return (
    <ul>
      {
        this.props.products.map((product, key) => (
          <li className={this.props.selectedProduct === product.id ? 'selected' : ''} key={key} onClick={() => this.props.selectProduct(product)}>
            {product.name}
          </li>
        ))
      }
    </ul>
  )
}
```
<small>You can use [classNames](https://github.com/JedWatson/classnames) to correctly implement a logical className approach. I just want to keep this as a stand alone example.</small>
Now, you can style that `.selected` class to see the result. But we have a problem, what happens on the first load, before any click? `state.products.selectedProduct` is an empty string, so at the beginning, no product will be selected. Let's suppose we want to select the first product by default, when the list loads. We have at least two ways to do it:

Modify the reducer, so with the `_FULFILLED` action not only the `productsList` will be returned, but also `selectedProduct`:
```javascript
[`${PRODUCTS.FETCH}_FULFILLED`]: (state, action) => ({
  ...state,
  loading: false,
  productsList: action.payload.data,
  error: null,
  selectedProduct: action.payload.data[0].id
  // if data is an empty array, this will cause problems
})
```

Or, in the other hand, we can take advantage of this case to implement Redux Sagas and properly dispatch a `PRODUCT_SELECT` action after a `PRODUCTS_FETCH_FULFILLED` action is completed:
```javascript
// src/sagas/index.js
import { fork } from 'redux-saga/effects'
import { watchProductsFetchFulfilled } from './products'

export default function* rootSaga() {
  yield fork(watchProductsFetchFulfilled)
}
// this code can grow very quickly so there may be a better organization than the present one...
```
```javascript
// src/sagas/products.js
import { takeLatest } from 'redux-saga'
import { select, put } from 'redux-saga/effects'
import { selectProduct } from 'actions/products'
import actionTypes from 'constants/action-types'

const { PRODUCTS } = actionTypes

export function* selectProductAfterProductsFulfilled() {
  const { products } = yield select() // select function is like getState()

  if (products.productsList.length > 0) {
    yield put(selectProduct(products.productsList[0].id)) // puts a new action on the flow
  }
}

export function* watchProductsFetchFulfilled() {
  yield* takeLatest(
    [`${PRODUCTS.FETCH}_FULFILLED`, /* Other actions you want to watch */],
    selectProductAfterProductsFulfilled
  )
}
```
Redux sagas use [generator functions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/function%2A), which is an entire concept by itself, so I won't go deeper on this.

### Redux Logger
This middleware is not precisely the most functional one, but in the development phase, I think it's a great resource. Basically what offers you is a console output, directly on the browser. There are several other tools that do the same and much more, [redux-devtools](https://github.com/gaearon/redux-devtools) for example, but I wanted to mention [Redux Logger](https://github.com/evgenyrodionov/redux-logger) given that it's quite comfortable for me in development.
The result you will accomplish is something like this:
<p align="center">![](/images/redux-logger.png)</p>
You can configure the action log to be collapsed / expanded by default. You can check the entire list of actions being dispatched, the app's state before and after, and also the payload of each action. I find this quite cool.

To implement this middleware, you just have to [include it in your store](#Store-Config).


## Redux Selectors
I wanted to introduce the concept of selectors because they are really helpful to keep our code decoupled and our components clean.
Let's suppose you receive a list of products like this one:
```javascript
[{
  name: 'apples',
  id: 1,
  harvested: '2017-08-12T20:17:46.384Z'
},
{
  name: 'pears',
  id: 2,
  harvested: '2017-07-11T20:15:03.204Z'
}]
```
This array would be in your `state.products.productsList` after a `_FULFILLED` action is dispatched. To display the dates, probably you will want to format their values using [moment](https://www.npmjs.com/package/moment), and you would be tempted to do so inside the component:

```javascript
let mapStateToProps = state => ({
  products: state.products.productsList.map(({name, id, harvested}) => ({
    name,
    id,
    harvested: moment('2017-07-11T20:15:03.204Z').format('YYYY-MM-DD')
  })),
  loading: state.products.loading, //new prop connected to the state
  error: state.products.error, //new prop connected to the state
  selectedProduct: state.products.selectedProduct
  // not the cool way
})
```
This is, creating a new array of all the products but with each date formatted, inside `mapStateToProps`. The method `mapStateToProps` should be, as it's name stands, a simple map between the state and the component's props. But here the component should not be <strong><i>decorating</i></strong> or <strong><i>calculating</i></strong> anything.
If you keep transforming / decorating inside your components, your code can quickly get messy. Keeping in mind that the app's state must remain as simple as we can, <strong><i>all the transformation, calculation and decoration of data will be done in selectors</i></strong>.

Selectors can be just functions that receive the state, perform all the needed work, and return the result to the components. That would be the same thing I just described but with the function definition somewhere else.
We will use a better approach, implementing `createSelector`, a method provided by the library [reselect](https://github.com/reactjs/reselect). By doing this, our selectors will be [memoized](https://github.com/reactjs/reselect#motivation-for-memoized-selectors), which basically means that our selector functions will be executed only when a part of the state they are watching changes (i.e. they will be executed only when needed).

```javascript
// src/selectors/products.js
import { createSelector } from 'reselect'
import moment from 'moment'

const selectProductsList = state => state.products.productsList
const selectSelectedProduct = state => state.products.selectedProduct

export const getProductsList = createSelector(
  [selectProductsList],
  productsList => productsList.map(({name, id, harvested}) => ({
    name,
    id,
    harvested: moment('2017-07-11T20:15:03.204Z').format('YYYY-MM-DD')
  }))
)

export const getSelectedProduct = createSelector(
  [selectProductsList, selectSelectedProduct],
  (productsList, selectedProduct) => productsList.find(product => product.id === selectedProduct)
)
```

And then, your smart component would pass the state to the selector methods, connecting to a <strong><i>decorated / calculated</i></strong> part of the state.

```javascript
// your mapStateToProps method
import { getSelectedProduct, getProductsList } from 'selectors/products'
// ...
let mapStateToProps = state => ({
  products: getProductsList(state),
  loading: state.products.loading, //new prop connected to the state
  error: state.products.error, //new prop connected to the state
  selectedProduct: getSelectedProduct(state)
})
```
With this, we have implemented a selector to transform our dates, but also we included `getSelectedProduct`, which is a selector that calculates the selected product object (with all it's data, not only the `id` as we had before). The state remains simple and it doesn't repeat data. The selector functions, based on that simple data, decorate and calculate new data, and then provides it to the components.
To conclude the example, our component render method would change from:
```javascript
<li className={this.props.selectedProduct === product.id ? 'selected' : ''} key={key} onClick={() =>
```
```javascript
<li className={this.props.selectedProduct.id === product.id ? 'selected' : ''} key={key} onClick={() =>
  // this.props.selectedProduct is now an object
```

## Reducers Recommended Structure
In first place, I've seen a lot code like this:
```javascript
// src/reducers/products.js
let productsReducer = (state = {productsList: [], selectedProduct: ''}, action) => {
  switch (action.type) {
    case 'PRODUCTS_RECEIVED':
      return {
        ...state,
        productsList: action.payload
      }
    case 'PRODUCTS_SELECT':
      return {
        ...state,
        selectedProduct: action.payload
      }
    default:
      return state
  }
}
```
First issue, your action types are completely hardcoded. Second, you are using a switch. Third, which is your initial state? It's the `{productsList: [], selectedProduct: ''}` fragment, really long line if your initial state grows, and not clear to see if some new dev enters to the team...
In addition, let's suppose your back end returns your products like this:
```javascript
{
  ...
  data: {
    main: {
      result: {
        array_data: {
          data: [{
            product: 'apples',
            harvested: '2017-08-12T20:17:46.384Z'
          },
          {
            test: 'pears',
            harvested: '2017-07-11T20:15:03.204Z'
          }]
        }
      }
    }
  }
  ...
}
```
You may find this rare or impossible. Trust me, API responses can be even worst than this. In this case, your state would receive a deeply nested object as data... Not cool at all.

I think that these reasons are enough to provide some advice in the Reducers structure.

In first place, define an action-names-constants-file at a project level, your actions names will be defined there. Here's a simple constants file defining actions:
```javascript
// src/constants/action-types.js
export const PRODUCTS_FETCH = 'PRODUCTS_FETCH'
export const PRODUCTS_SELECT = 'PRODUCTS_SELECT'
```

But I would advice to do something like this instead:
```javascript
// src/constants/action-types.js
import keyMirror from 'key-mirror-nested'

export default keyMirror({
  PRODUCTS: {
    FETCH: null,
    SELECT: null
  }
}, { connChar: '_' })
```
<details>
  <summary><i>Brief explanation</i></summary>[key-mirror-nested](https://github.com/apolkingg8/KeyMirrorNested) is a library I find very useful for this case. Here I export an object with one key, `PRODUCTS`, and if you try `PRODUCTS.FETCH`, that object will have the value `PRODUCTS_FETCH`. So that's the constant I need. Notice that I used underscore as `connChar`, you can use whatever you prefer.
</details>

And then your reducer would be something like:

```javascript
// src/reducers/products.js
import { handleActions } from 'redux-actions'
import * as actionTypes from 'constants/action-types'

const { PRODUCTS } = actionTypes

const initialState = {
  productsList: [],
  selectedProduct: ''
}

export default handleActions({
  [`${PRODUCTS.SELECT}`]: (state, action) => ({
    ...state,
    selectedProduct: action.payload
  }),
  [`${PRODUCTS.FETCH}`]: (state, action) => ({
    ...state,
    productsList: action.payload
  })
}, initialState)
```

By doing this
* The initial state is very clear
* Action types are not hardcoded
* We are using an object notation instead of a switch
* And we are clearly resolving the deeply nested problem the API may be sending us.

Finally, I recommend to have `src/reducers/index.js` to centralize all your reducers and keep a clear idea of your state:

```javascript
// src/reducers/index.js
import { combineReducers } from 'redux'
import products from './products'

export default combineReducers({
  products
  // other reducers
})
```
This file will then be consumed when we [configure our store](#Store-Config).

## Store Config
This is just an example of how to declare your store. You may find better / cleaner ways to do it. I believe this part of code was needed in the article, given that I mention so many things regarding middlewares. The configureStore is where you reference your `reducers/index.js` file, creating the store with the data you defined in your reducers, and attaching all the middlewares you want to the actions pipeline.

```javascript
// src/stores/configureStore.js
import 'babel-polyfill'

import { createStore, applyMiddleware } from 'redux'
import createLogger from 'redux-logger'
import { browserHistory } from 'react-router'
import { routerMiddleware } from 'react-router-redux'
import promiseMiddleware from 'redux-promise-middleware'
import createSagaMiddleware from 'redux-saga'
import thunkMiddleware from 'redux-thunk'

import indexReducer from 'reducers/index'
import rootSaga from 'sagas/index'

const logger = createLogger({ collapsed: true })
const router = routerMiddleware(browserHistory)
const promise = promiseMiddleware()
const saga = createSagaMiddleware()

const createStoreWithMiddleware = applyMiddleware(
    router,
    logger,
    promise,
    saga,
    thunkMiddleware)(createStore)

export let configureStore = () => {
  const store = createStoreWithMiddleware(rootReducer, {})
  saga.run(rootSaga)
  return store
}
```

Then you can have an `index.jsx` file, which will simply import the `configureSture` function exported by `configureStore.js`:

```javascript
import ReactDOM from 'react-dom'
import { configureStore } from './stores/configureStore'

const store = configureStore()

ReactDOM.render(
  <Provider store={store}>
  </Provider>,
  document.getElementById('yourRootElementId')
)
```
