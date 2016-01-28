# Conflux [![Build Status](https://img.shields.io/circleci/project/ben-ng/conflux/master.svg)](https://circleci.com/gh/ben-ng/conflux/tree/master) [![Coverage Status](https://img.shields.io/coveralls/ben-ng/conflux/master.svg)](https://coveralls.io/github/ben-ng/conflux?branch=master) [![npm version](https://img.shields.io/npm/v/conflux.svg)](https://www.npmjs.com/package/conflux)

Conflux is [Redux](https://github.com/rackt/redux) for distributed systems.

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Contents**

- [Motivation](#motivation)
- [What Can I Build With Conflux?](#what-can-i-build-with-conflux)
- [Quick Example](#quick-example)
- [API](#api)
  - [Creating an instance](#creating-an-instance)
  - [Performing Methods](#performing-methods)
  - [Dispatching Actions](#dispatching-actions)
  - [Subscribing to changes](#subscribing-to-changes)
  - [Getting State](#getting-state)
  - [Deconstructing an instance](#deconstructing-an-instance)
- [Correctness](#correctness)
- [License](#license)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Motivation

Distributed systems are **hard**. Conflux is an attempt at making distributed systems understandable. It aims to do what [Redux](http://redux.js.org) did for [Flux](https://facebook.github.io/flux), and what [Raft](http://raft.github.io) did for [Paxos](https://en.wikipedia.org/wiki/Paxos_(computer_science)).

Natrually, it does this by composing the two ideas.

## What Can I Build With Conflux?

Conflux is *very* new, so I am working on a few example applications.

* [Distributed Mutex](https://github.com/ben-ng/mutex-js)

## Quick Example

```js
// Your methods, reducer, and other settings go in `opts`
// See the full docs for that; this example is meant to be short
var nodeA = conflux(opts)
var nodeB = conflux(opts)
var nodeC = conflux(opts)

// When the cluster comes to consensus about an action, this
// method is called.
nodeB.subscribe(function () {
  console.log(nodeB.getState().log.join(' '))
})

// Performs an RPC call on the leader node, causing actions
// to be dispatched on the entire cluster.
nodeA.perform('append', ['foo'])

nodeB.perform('append', ['bar'])

nodeC.perform('append', ['baz'])

// Output will be some permutation of:
//
// bar
// bar foo
// bar foo baz
```

If you're familiar with Redux this should seem quite familiar. You `subscribe()` to a Conflux instance, and call `getState()` inside to get the current state. Instead of dispatching actions directly, you `perform()` Methods that `dispatch()` them. Methods are declared when you construct a Conflux instance, and are the equivalent of Action Creators in Redux.

## API

### Creating an instance

```js
var conflux = require('conflux')
    // uuids are recommended, but you can use any string id
  , uuid = require('uuid')
  , c = conflux({
      /**
      * Required settings
      */

      id: uuid.v4()
    , clusterSize: 5
    , channel: {
        name: 'foobar'
        // ... additional Channel options specific to "foobar"
        // You can use any Gaggle Channel, like Redis.
      }

      // these take the place of "action creators" in redux parlance
    , methods: {
        foobar: function (foo, bar) {
          return {
            type: 'FOOBAR'
          , foo: foo
          , bar: bar
          }
        }
      }

      // this is the reducer function that redux stores are constructed with
    , reduce: function (state, action) {
        // return a new `state` using the information in `action`
      }

      // ... additional Gaggle options, see Gaggle's documentation
    })
```

Conflux is built on top of [Gaggle](https://github.com/ben-ng/gaggle), and therefore supports [any communication channel that Gaggle supports](https://github.com/ben-ng/gaggle#channels).

### Performing Methods (Dispatching Actions)

```txt
c.perform(String methodName, Array args, [Number timeout], [Function callback])
```

You never dispatch Actions directly in Conflux. Actions must be dispatched from the body of a Method. You declare Methods when constructing a Conflux instance, and call them with `perform()`. These Methods return the Action to be dispatched, `null` if nothing should be done, and an Error if the Action is invalid for the provisional state.

```js
var opts = {
  methods: {
    foobar: function (foo, bar) {
      // The perform callback will be called with no error and this return
      // value as the second argument
      return {
        type: 'FOOBAR'
      , foo: foo
      , bar: bar
      }
    }
  , bonk: function () {
      return new Error('Whoops')
    }
  , noop: function () {
      // The perform callback will be called with no error or response
      return null
    }
  }

  // ... other Conflux options
}

var c = conflux(opts)

// Callback API
c.perform('noop', [], function (err) {
  console.log(err) // => null
})

c.perform('noop', [], 5000, function (err) {
  console.log(err) // => null
})

// Promise API
c.perform('foobar', ['a', 'b'])
.then(function (action) {
  console.log(action) // => {type: 'FOOBAR', foo: 'a', bar: 'b'}
})

c.perform('bonk', [], 5000)
.catch(function (err) {
  console.log(err) // => Error: Whoops
})
```

### State Reducer

```js
// A starter template for your own reducer
function (state, action) {
  // Set initial state or clone existing state
  if (state == null) {
    state = {}
  }
  else {
    state = JSON.parse(JSON.stringify(state))
  }

  // Handle known actions
  if (action == null) {
    return state
  }

  switch (action.type) {
    case 'FOO':
      state.isFoo = true
    break
    case 'BAR':
      state.isBar = true
    break
  }

  // Return the new state
  return state
}
```

Reducers should obey a few rules:

1. Do not mutate `state`
2. Always return a new `state`
3. Be prepared to set an inital state if `state` is `undefined`
4. If the action is unrecognized (it might be `null` when `Conflux` initializes the state, for example), return the same `state`

### Subscribing to changes

```txt
c.subscribe(function() callback)
```

Calls `callback` whenever an action is committed. Returns the `unsubscribe` function for `callback.

### Getting State

```txt
c.getState()
c.getProvisionalState()
```

`getState` gets you the *committed* state of the node. All nodes are guaranteed to enter this state at some point, but it does not contain the effects uncommitted actions.

`getProvisionalState` gets you the state of the nodes *if all uncommitted actions are committed*. Since the leader might fail before these actions are committed, *it is possible that no nodes ever actually enter this state*.

You should use the provisional state in your Methods to determine the validity of an action. The committed state should be used just about everywhere else, like in your `subscribe()` callback.

### Deconstructing an instance

```txt
c.close([function(Error) callback])
```

When you're done, call `close` to remove event listeners and disconnect the channel.

```js
c.close(function (err) {})

c.close().then()
```

## Correctness

Distributed systems are really difficult to prove and test, and Conflux is no exception. I am still working on formal proofs of correctness, but in the meantime here is an incomplete list of things that are being done to

* Conflux has integration tests with [full statement *and* branch coverage](https://coveralls.io/github/ben-ng/conflux?branch=master)
* It is built on Gaggle, which has integration tests with [full statement coverage](https://coveralls.io/github/ben-ng/gaggle?branch=master)
* My [distributed mutex](https://github.com/ben-ng/mutex-js) is built on Conflux and has [full statement coverage](https://coveralls.io/github/ben-ng/mutex-js?branch=master) and a [fuzz test](https://github.com/ben-ng/mutex-js/tree/master/fuzz) you can run yourself

## License

Copyright (c) 2016 Ben Ng <me@benng.me>

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
