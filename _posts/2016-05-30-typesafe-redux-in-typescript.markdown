---
layout: post
title:  "Typesafe Redux in TypeScript"
date:   2016-05-30 16:36:24 -0600
permalink: typesafe-redux-in-typescript
---

Redux is great. TypeScript is great. Redux isn't, at the outset, anywhere where near typesafe. This means Redux in TypeScript isn't so great on its own. However, given a little effort, it is possible to use Redux in an (almost) totally typesafe way using TypeScript. Here follows a complete guide to how I got it working. This approach uses some configuration which can't be statically verified by the typescript transpiler. However, that configuration is pretty minimal and with it the rest of your app can benefit from completely typesafe access to redux. I've separated it into three parts:

1. Typesafe Actions
2. Typesafe Store Configuration and Reducers
3. Connecting to components

## Part 1: Typesafe Actions

I found this clever implementation for typescript actions on a [redux github issue](https://github.com/reactjs/redux/issues/992) posted by [aikoven](https://github.com/aikoven), I've modified it slightly. If you're not using TypeScript 1.9 (and you're probably not since at the time of writing it was unreleased), just remove the `readonly` modifiers.

{% highlight javascript %}
//listItemsActions.ts

export interface Action<T> {
  readonly type: string
  readonly payload: T
}

interface ActionCreator<T> {
  readonly type: string
  (payload: T): Action<T>
}

const actionCreator = <T>(type: string): ActionCreator<T> =>
  Object.assign((payload: T):any => ({type, payload}), {type})

export const isType = <T>(action: Action<any>, actionCreator: ActionCreator<T>):
  action is Action<T> => action.type === actionCreator.type


//Example action creator:
export interface ListItem {
  readonly done: boolean
  readonly description: string
}

export const createListItemAction =
  actionCreator<ListItem>('CREATE_LIST_ITEM_ACTION_TYPE')
{% endhighlight %}

There's three things to notice here:

First, through the `Action` interface, we define a typesafe way to access redux actions before we know what action we're dealing with. 

Second, through the `isType<T>` function, we expose a typesafe way of accessing action payloads through use of a [type guard](https://github.com/Microsoft/TypeScript/wiki/What's-new-in-TypeScript#user-defined-type-guard-functions). It works by comparing the action type to the one which has been defined in the action creator. This is safe so long as the action types are globally unique. An example of how this function is used is included in part 2.

Third, the action creator parameters themselves have defined types. This closes the loop and makes the entire action type safe. This is especially effective with `strictNullChecks` and `noImplicitAny` enabled in your TypeScript config!   


## Part 2: Typesafe Store Configuration and Reducers

Here's an example of the above actions integrated into a typesafe redux reducer:

{% highlight javascript %}
//listItemsReducers.ts

import { ListItem, createListItemAction } from 'listItemsActions'
import Immutable = require('immutable')

type ListItems = Immutable.List<ListItem>

export interface ListItemsState {
  readonly listItems: ListItems 
}

export const listItemsReducers = (state: ListItems = Immutable.List<ListItem>(),
  action: Action<any>): ListItems => {
  
  if (isType(action, createListItemAction)) {
    const listItem = {
      done: action.payload.done,
      description: action.payload.description
    }
    return state.push(listItem)
  }
  
  return state
}
{% endhighlight %}

Beyond the use of the `isType<T>` type guard, take note of the way the reducer is defined. The type of the state parameter and return value are both defined to be `ListItems` which is `Immutable.List<ListItem>`. As far as I can tell, there's no way to get the TypeScript compiler to help you here. You have to add these two type declarations yourself, but that ensures that the rest of the reducer is statically verifiable.

The `ListItemsState` interface can be used to define the type of the global state demonstrated in the following example. This particular example defines a global singleton store. Whether you want to do this is up to you. The approach I suggest here certainly doesn't require it.
  
{% highlight javascript %}
// store.ts

import { createStore, combineReducers, Unsubscribe } from 'redux'
import { Action } from 'listItemsActions'
import { listItemsReducers, ListItemsState } from 'listItemsReducers'
import { someOtherReducers, SomeOthersState } from 'someOtherReducers'

const reducer = combineReducers({
  listItems: listItemsReducers,
  someOther: someOtherReducers
})
export interface State extends ListItemsState, SomeOtherState {}

let store = createStore(reducer)

export const getState = () => store.getState() as State
export const dispatch = (a:Action<any>) => store.dispatch(a)
export const subscribeToState = (callback: () => void) => store.subscribe(callback)

{% endhighlight %}

The store exposes three functions:

1. The `getState()` method. This method is the crux of the implementation: it returns redux's state defined as the `State` type. We've declared `State` as a union of state interfaces defined by individual reducers. In this case, it's the union of `StickiesState` we defined in `listItemsReducers.ts` and `SomeOtherState`.
2. A `dispatch()` method which wraps redux's `getState()` but accepts only actions of type `Action<>`
3. A `subscribeToState()` method which wraps redux for convenience.

It's important to realize that the compiler can't help you with your definition of `State`. If the types in `store.ts` are misconfigured the application will be misconfigured as well. However, get this right and the rest of your application will benefit from typesafe access to the redux store.

## Part 3: Connecting to components 

Finally, you'll also need to connect redux to your react components. I suspect most people will choose to react-redux to do this. It isn't trivial to use react-redux in a typesafe way, though. It's also not a terribly complicated library so I'd suggest you consider using something like the following:

{% highlight javascript %}
//stateConnector.ts

import { getState, State, subscribeToState } from 'state'
import { Component } from 'react'

const hijackComponentWillUnmount = (component: any, callback: () => void) => {
  const componentWillUnmount = component.componentWillUnmount
  component.componentWillUnmount = () => {
    callback()
    if (componentWillUnmount) {
      componentWillUnmount.apply(component);
    }
  }
}
export const stateConnector = <T>(mapState: (s:State) => T, component: Component<any, T>) => {
  let oldState = mapState(getState())
  let wasUnsubscribed = false
  
  const unsubscribe = subscribeToState(() => {
    if (wasUnsubscribed) {
      return;
    }
    const newState = mapState(getState())
    if (/* oldState deeply does not equal newState */) {
      component.setState(newState)
    }

    oldState = newState
  });
  
  hijackComponentWillUnmount(component, () => {
    wasUnsubscribed = true
    unsubscribe()
  }) 
  
  component.state = oldState
}

{% endhighlight %}

This implementation has the distinct advantage of making it easy for the typescript transpiler to verify the typesafety of your code. A simple use looks like this:

{% highlight javascript %}
//listItemComponent.ts

import * as React from 'react'
import { stateConnector } from 'stateConnector'
import { ListItem } from 'listItemActions'

interface ComponentProps {
  readonly index: number
}

interface ComponentState {
  readonly listItem: ListItem
}

export class ListItemComponent extends React.Component<ComponentPropsProps, ComponentState> {
  constructor(props) {
    super(props)
    stateConnector((state) => {listItem: state.listItems[this.props.index]}, this)
  }
  
  render() {
    const doneClass = this.state.listItem.done ? "done" : "" 
    return <li class={doneClass}>{this.state.listItem.description}</li>
  }
}
{% endhighlight %}

It's taken a little work to get here but the transpiler is capable of verifying the entire above example! Access to the store is completely typesafe for all components. Woo!

If you use this approach, you'll need to figure out how to do your own equality comparison in `stateConnector.ts`. This depends on your redux implementation (such as use of immutable.js, seamless-immutable, etc) and can have significant performance implications. For most projects a relatively simple implementation should be just fine, though.