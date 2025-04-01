---
layout: single
title:  "Passing Data Between Components using props, useReducer, and useContext"
date:   2021-05-12 10:00:00 +0100
categories: dev
tagline: "We will explain it with an example (which is NOT a to-do app)"
header:
  overlay_image: /assets/images/mahdi-bafande-OSONMQPZz08-unsplash.jpg
  overlay_filter: 0.7
  caption: "Photo by [Mahdi Bafande](https://unsplash.com/@mahdibafande) on [Unsplash](https://unsplash.com/photos/a-blurry-photo-of-a-street-light-at-night-OSONMQPZz08)"
---

The good thing about React (or any other good SDKs, or toolkits, or programming languages for that matter) is that you can get the same result using different methods. In this post, I would like to demonstrate how you can pass data between different components using different methods. We will introduce the concept of *reducer*, *context*, and how to use them through the `useReducer` and `useContext` hooks.

<!--more-->
(This post was originally posted in my [hashnode](https://notes.yhg.io/passing-data-between-components-using-props-usereducer-and-usecontext) blog.)

Imagine a typical app which has a top toolbar and a bottom view, which shows a list of items:

```react
export default App = () => {
   return (
      <Main>
         <TopBar />
         <ItemListView />
      </Main>
   );
}
```

It is very common that the `ItemListView` has to communication with the `TopBar`. For example, when you click and select an item, the `TopBar` shows some action buttons for you to do with it. How can you implement this?

### Method 1: Through props

The most straightforward way is to use *props*. We can write a handler function and pass it to `ItemListView`, which, when called, will change a local state variable, which is passed to `TopBar`. Like this:

```react
export default App = () => {
   const [ selection, setSelection ] = useState(0);
   return (
      <Main>
         <TopBar selection={selection} />
         <ItemListView onSelectionChanged={setSelection} />
      </Main>
   );
}
```

Somewhere in `ItemListView`, the handler is called when needed to:

```react
export default ItemListView = (props) => {
   // ....
   const itemClicked = (item) => {
      //...
      props.onSelectionChanged(item.id);
   }
   // ...
}
```

This method is easy and straightforward. However, if you have many data that need to be communicated, you may get into this:

```react
<ItemListView
   onSelectionChanged={setSelection}
   onItemChanged={handleItemChanged}
   onSomeEvent1={handleEvent1}
   onSomeEvent2={handleEvent2}
   ...
/>
```

You can write another function to consolidate them, with complicated state logic and management. But why bother? You can always use...

### Method 2: useReducer

*Redux*, and its functional component hook counterpart *useReducer*, is the perfect tool for such complex state management across different components. Let's see how we can use useReducer here.

(I once heard a speaker said: "...and then we use useState state management..." and I was surprised he didn't stutter.)

Simply put, a *reducer* is merely a function which takes two arguments: the current "state" object and an "action" object, and returns a new state object. When you feed a reducer to `useReducer()`, it returns an array, similar to `useState()`:

```react
const [ state, dispatch ] = useReducer(reducer);
```

Where `state` is the state object for you to consume, and `dispatch` a function for you to dispatch actions to the reducer.

Let's use our app again as an example. Suppose when we click an item, the `TopBar` goes into "select mode", such that there will be some UI changes, and at the same time store the selected item id. We can define our reducer like this:

```react
const reducer = (state, action) => {
   switch (action.type) {
      case 'TURN_ON_SELECT_MODE':
         return {...state, selectMode: true};
      case 'TURN_OFF_SELECT_MODE':
         return {...state, selectMode: false};
      case 'SET_ITEM':
         return {...state, selectedItem: action.value};
   }
}
```

Note:

* It is a common practice that the `action` object has two properties: `type` and `value`, although it doesn't have to be.
    
* You cannot change the state directly. State change is achieved by returning a *new* state object. Also, when you return, you must return the *entire* object, not just the value. Therefore, we use the [spread syntax ...](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax) to merge the new value with the existing ones, and return the new state object.
    

It's time to use it in our app:

```react
// App.js
export default App = () => {
   const [ state, dispatch ] = useReducer(reducer);
   return (
      <Main>
         <TopBar selectMode={state.selectMode} />
         <ItemListView dispatch={dispatch} />
      </Main>
   );
}

// ItemListView.js
export default ItemListView = (props) => {
   // ....
   const itemClicked = (item) => {
      //...
      props.dispatch({ type: 'TURN_ON_SELECT_MODE' });
      props.dispatch({ type: 'SET_ITEM', value: item.id });
   }
   // ...
}
```

The good thing about useReducer (and Redux) is that it packs state values/object and action together, so you do not need 10x props for different values and handlers. Also, since a state value can only be modified through action, it will never go into some incoherent state, which makes debugging state-related issues much, much easier.

But what if `ItemListView` consists of many many child components, where some, if not most, of them also need access to `dispatch`? We can certainly pass it down through props, but it's tedious. Hm... I wonder if there's some kind of *global* variables...

### Method 3: useContext

Well, turns out there is! But it's in the form of *context*.

None other than [the official React site](https://reactjs.org/docs/context.html) summarizes the use of context better:

> Context provides a way to pass data through the component tree without having to pass props down manually at every level.

Think of it like a magic, hyper-dimensional secret knapsack that you can put things (variables) in, and when needed, you use the hook `useContext` to retrieve the things back. Difficult to understand? Let's see it in action!

First of all, we define the context. Let's call it `TopBarContext`:

```react
// TopBarContext.js
export const TopBarContext = React.createContext(null);
```

Then, we wrap our whole app, or the highest node possible that will use this context, with the context's `Provider`. Also, in the process, we state the "things" we want to store, which in this case is the `dispatch` function:

```react
// App.js
import { TopBarContext } from './TopBarContext';

export default App = () => {
   const [ state, dispatch ] = useReducer(reducer);
   return (
      <TopBarContext.Provider value={dispatch}>
         <Main>
            <TopBar selectMode={state.selectMode} />
            <ItemListView />
         </Main>
      </TopBarContext.Provider>
   );
}
```

That's it! Note that we don't need to pass `dispatch` to `ItemListView` through props now.

In anywhere (anywhere under `TopBarContext.Provider`, that is) you want to use the `dispatch`, you use the `useContext` hook to retrieve it back:

```react
// ItemListView.js
import { TopBarContext } from './TopBarContext';

export default ItemListView = () => {
   // ....
   const dispatch = useContext(TopBarContext);
   const itemClicked = (item) => {
      //...
      dispatch({ type: 'TURN_ON_SELECT_MODE' });
      dispatch({ type: 'SET_ITEM', value: item.id });
   }
   // ...
}
```

### Which method should I use?

Each method has its pros and cons. Obviously, using props is the most straightforward, but can get tedious very quickly. Using context is the ideal way if your app is huge, and the value you wish to share is quite fundamental and universal (normally we see examples like theme, authentication status etc.) But it could be an overkill and lead to a difficult-to-understand codes. Reducer is somewhere in between... So, it really depends on the situation. Like I said at the beginning, it's a good thing to have different ways to do thing.

Hopefully this post can give you a basic understanding on how to use the props, reducer, and context to pass data between React components. If you want to dig deeper, these are some of the best tutorials on the topics:

* [reducer](https://www.robinwieruch.de/javascript-reducer) and [useReduce](https://www.robinwieruch.de/react-usereducer-hook)
* [useContext](https://daveceddia.com/usecontext-hook/)