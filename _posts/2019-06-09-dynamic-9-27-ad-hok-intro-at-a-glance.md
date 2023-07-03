---
layout: post
excerpt_separator: <!--more-->
title: "From vanilla React hooks to ad-hok: At a glance"
permalink: ad-hok-intro-at-a-glance/
date: 2019-09-27
---

[`ad-hok`](https://github.com/helixbass/ad-hok) is a library that lets you use React hooks in a
[functional pipeline](https://martinfowler.com/articles/collection-pipeline/) style. If you're
unfamiliar with functional pipelines or the `flow()` helper from `lodash/fp`, take a look at
[this introduction](https://simonsmith.io/dipping-a-toe-into-functional-js-with-lodash-fp) first.

We'll go through examples from the React [Hooks at a Glance](https://reactjs.org/docs/hooks-overview.html) documentation
and look at how they could be written using `ad-hok` helpers instead of "vanilla hooks".

### Example 1: State Hook
The [first example](https://reactjs.org/docs/hooks-overview.html#state-hook) from the React docs just uses the
[`useState()`](https://reactjs.org/docs/hooks-state.html) hook:
```js
import React, { useState } from 'react';

function Example() {
  // Declare a new state variable, which we'll call "count"
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>
        Click me
      </button>
    </div>
  );
}
```
With `ad-hok`, rather than putting our hooks code at the beginning of a function component's body, we're going to express
the component as a `flow()`:
```js
import React from 'react'
import {flow} from 'lodash/fp'
import {addState} from 'ad-hok'

const Example = flow(
  // Declare a new state variable, which we'll call "count"
  addState('count', 'setCount', 0),
  ({count, setCount}) =>
    <div>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>
        Click me
      </button>
    </div>
  )
)
```
#### Following the flow of props
The thing that's "flowing down the pipeline" is a props object. At the beginning of the chain, it's just the props that got passed to your component. So for example, if we render `Example` like this:
```js
<Example message="woops" />
```
then `{message: 'woops'}` is the value fed into the `flow()` pipeline.

If we render `Example` with no props:
```js
<Example />
```
then an empty object `{}` is fed into the pipeline.

`ad-hok` helpers like `addState()` add additional props to the props object. Here, `addState()` adds both the state value `count` and the state setter `setCount` to the props object. Then the last
step in the pipeline uses those props when rendering.

### Example #2: [Effect Hook](https://reactjs.org/docs/hooks-overview.html#effect-hook)
```js
import React, { useState, useEffect } from 'react';

function Example() {
  const [count, setCount] = useState(0);

  // Similar to componentDidMount and componentDidUpdate:
  useEffect(() => {
    // Update the document title using the browser API
    document.title = `You clicked ${count} times`;
  });

  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>
        Click me
      </button>
    </div>
  );
}
```
This example introduces the [`useEffect()`](https://reactjs.org/docs/hooks-effect.html) hook. Notice that the effect
*depends on* the `count` state variable.

Here's the `ad-hok` version:
```js
import React from 'react'
import {flow} from 'lodash/fp'
import {addState, addEffect} from 'ad-hok'

const Example = flow(
  addState('count', 'setCount', 0),
  addEffect(({count}) => () => {
    // Update the document title using the browser API
    document.title = `You clicked ${count} times`;
  }),
  ({count, setCount}) => (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>
        Click me
      </button>
    </div>
  )
)
```
Instead of referencing `count` via a closure, we now access `count` via the props object that gets supplied to the
`addEffect()` callback.

#### So what?
It may not be obvious yet why this style might be preferable to the "vanilla hooks" version. Notice how each step of the
pipeline can be explicit about which props it depends on. Once you get used to the functional style, this can provide a
huge benefit in terms of keeping track of things when building components with complex logic and state.

### Example #3: Effect with cleanup
```js
import React, { useState, useEffect } from 'react';

function FriendStatus(props) {
  const [isOnline, setIsOnline] = useState(null);

  function handleStatusChange(status) {
    setIsOnline(status.isOnline);
  }

  useEffect(() => {
    ChatAPI.subscribeToFriendStatus(props.friend.id, handleStatusChange);

    return () => {
      ChatAPI.unsubscribeFromFriendStatus(props.friend.id, handleStatusChange);
    };
  });

  if (isOnline === null) {
    return 'Loading...';
  }
  return isOnline ? 'Online' : 'Offline';
}
```
And the `ad-hok` version:
```js
import React from 'react'
import {flow} from 'lodash/fp'
import {addState, addHandlers, addEffect} from 'ad-hok'

const FriendStatus = flow(
  addState('isOnline', 'setIsOnline', null),
  addHandlers({
    handleStatusChange: ({setIsOnline}) => status => {
      setIsOnline(status.isOnline)
    }
  }),
  addEffect(({friend, handleStatusChange}) => () => {
    ChatAPI.subscribeToFriendStatus(friend.id, handleStatusChange)

    return () => {
      ChatAPI.unsubscribeFromFriendStatus(friend.id, handleStatusChange)
    }
  }),
  ({isOnline}) => {
    if (isOnline === null) {
      return 'Loading...';
    }
    return isOnline ? 'Online' : 'Offline';
  }
)
```
What's new here?

1. we use `addHandlers()` to add a `handleStatusChange()` helper to the props object. Notice how `handleStatusChange()`
is also able to access its dependency `setIsOnline`

2. The `FriendStatus` component is expecting to be passed a `friend` prop. Notice how the effect handler is able to access
`friend` and `handleStatusChange` as part of the same props object even though `friend` came from outside of the component
and `handleStatusChange` came from inside the pipeline

3. Notice how the last step of the pipeline only asks for the `isOnline` prop even though it could also ask for `friend`,
`handleStatusChange`, and `setIsOnline`. This helps us see that when it comes to actually rendering this component, it's
only a function of the current `isOnline` status, nothing else

### Example #4: Multiple effects
```js
function FriendStatusWithCounter(props) {
  const [count, setCount] = useState(0);
  useEffect(() => {
    document.title = `You clicked ${count} times`;
  });

  const [isOnline, setIsOnline] = useState(null);
  useEffect(() => {
    ChatAPI.subscribeToFriendStatus(props.friend.id, handleStatusChange);
    return () => {
      ChatAPI.unsubscribeFromFriendStatus(props.friend.id, handleStatusChange);
    };
  });

  function handleStatusChange(status) {
    setIsOnline(status.isOnline);
  }
  // ...
```
Composing multiple effects and state is similar in `ad-hok`:
```js
const FriendStatusWithCounter = flow(
  addState('count', 'setCount', 0),
  addEffect(({count}) => () => {
    document.title = `You clicked ${count} times`
  }),
  addState('isOnline', 'setIsOnline', null),
  addHandlers({
    handleStatusChange: ({setIsOnline}) => status => {
      setIsOnline(status.isOnline)
    }
  }),
  addEffect(({friend, handleStatusChange}) => () => {
    ChatAPI.subscribeToFriendStatus(friend.id, handleStatusChange)
    return () => {
      ChatAPI.unsubscribeFromFriendStatus(friend.id, handleStatusChange)
    }
  }),
  // ...
```

### Keep learning!

You can do a lot with `addState()`, `addEffect()` and `addHandlers()`. But just as there are [more React hooks](https://reactjs.org/docs/hooks-reference.html), `ad-hok` has lots of other useful helpers. Explore the
[`ad-hok` documentation](https://github.com/helixbass/ad-hok), and [file an issue](https://github.com/helixbass/ad-hok/issues) if you have any questions or need any help :sparkles:
