---
layout: post
excerpt_separator: <!--more-->
title: "Using dynamic portals in React to \"play nice with\" autoplay policy"
permalink: dynamic-portal/
date: 2019-06-09
---

### The problem

On the current project, we have a video inside a "lightbox"-style modal. We got design feedback that it would be nice
for the video to autoplay when the modal is opened

However, by default when setting the video to autoplay, it was playing muted in Safari/iOS Safari (requiring an additional
tap to unmute). Since iPad is the primary deployment target for this project and the design feedback was that it should only
autoplay if it can play unmuted, we decided to try and figure out a way to achieve that

We felt like in theory it should be doable since even though browsers have been tightening up their autoplay policies, there
has to be a way to play a video unmuted based on explicit user action, and here there was an explicit user action taking place
before playing the video (clicking to open the modal)

So [Alex](https://github.com/amay) figured that (from a React perspective) effectively if the video were already mounted
before the modal was opened and then we did something more like an imperative `.play()` command when the modal was opened,
it was likely that the video would play unmuted

### The obstacle

If the modal itself could be mounted "up front" and simply kept visually hidden (via CSS) until it was toggled open, that
would be straightforward, as the video component inside the modal would also be pre-mounted along with the modal itself

However, on this project we're using [`react-bootstrap`](https://react-bootstrap.github.io/), which states in its [`Modal`
docs](https://react-bootstrap.github.io/components/modal/) that "Modals are *unmounted* when closed"

And it didn't seem viable to try and use a different type of modal just for this particular instance

So the only option for having the video pre-mounted before the modal opened seemed to be to somehow mount it somewhere and
then "move" it to inside the modal once the modal had mounted

### Portals to the rescue?

The React-y way to achieve that would be with a [portal](https://reactjs.org/docs/portals.html). If you haven't seen React
portals before, basically they allow you to render something inside any arbitrary DOM node (rather than inside its React
parent)

I had never seen a "dynamic portal" that changed its target parent DOM node "mid-flight", but that seemed like
the most obvious way to structure it - mount the video inside some visually hidden parent container initially (via a portal)
and then when the modal opened, *\*poof\** change the portal destination to the modal!

### tl;dr it worked

Using this dynamic portal to premount the video, it did in fact play unmuted in Safari!

### The code

So let's look at the specific code patterns used to create this "dynamic portal"

Portals expect to be given an actual DOM node (for the destination). So in React, when you hear "actual DOM node" you should
be thinking "ref"

Here, we need two refs since we need two DOM nodes for the portal destination - a visually hidden container and then the modal

Initially I tried using ["new-style refs"](https://reactjs.org/docs/refs-and-the-dom.html#creating-refs), since these days I'd
default to using that style of ref. But I wasn't seeing the portal rerendering reliably once the refs were populated

This was new to me, but apparently that's known/expected behavior when using new-style refs via the [`useRef` hook](https://reactjs.org/docs/hooks-reference.html#useref)
and if you need to trigger rerendering based on a ref changing, you need to use [callback-style refs](https://reactjs.org/docs/hooks-faq.html#how-can-i-measure-a-dom-node)
instead

So here's how I wired up the two callback-style refs:
```
import {flow} from 'lodash/fp'

const VideoAnswer = flow(
  ...
  addCallbackRefAndNode(
    'lightboxPortalTargetCallbackRef',
    'lightboxPortalTargetNode'
  ),
  addCallbackRefAndNode(
    'visuallyHiddenVideoContainerCallbackRef',
    'visuallyHiddenVideoContainerNode'
  ),
  ...
  ({
    ...
    lightboxPortalTargetCallbackRef,
    visuallyHiddenVideoContainerCallbackRef,
    ...
  }) =>
    ...
      <div
        css={a11yStyles.visuallyHidden}
        aria-hidden="true"
        ref={visuallyHiddenVideoContainerCallbackRef}
      />
      <Lightbox
        ...
        childPortalTargetRef={lightboxPortalTargetCallbackRef}
      />
    ...
)
```

First of all, this is an [`ad-hok`](https://github.com/helixbass/ad-hok)-style component using [`flow()`](https://simonsmith.io/dipping-a-toe-into-functional-js-with-lodash-fp#compose-yourself).
`ad-hok` is allowing us to build up component functionality in a highly composable way similar to [Recompose](https://github.com/acdlite/recompose),
but where you're using React hooks instead of higher-order components as your building blocks

`addCallbackRefAndNode()` is a helper for exposing a callback-style ref and the node it references:
```
import {flow} from 'lodash/fp'
import {upperFirst} from 'lodash'
import {addState, addCallback} from 'ad-hok'

const addCallbackRefAndNode = flow(
  (refPropName, nodePropName) => ({
    refPropName,
    nodePropName,
    setNodePropName: `set${upperFirst(nodePropName)}`,
  }),
  ({refPropName, nodePropName, setNodePropName}) =>
    flow(
      addState(nodePropName, setNodePropName),
      addCallback(
        refPropName,
        ({[setNodePropName]: setNode}) => node => {
          setNode(node)
        },
        []
      )
    )
)
```
Here, `addState()` and `addCallback()` are `ad-hok` helpers that wrap the `useState()` and `useCallback()` hooks, respectively.
So `addCallbackRefAndNode()` is encapsulating a state variable for the callback ref to assign the DOM node reference to

And the `childPortalTargetRef` is a new prop I added to our `<Lightbox>` component to allow wiring up the ref to the modal body:
```
const Lightbox = ({
  ...
  childPortalTargetRef,
}) => (
  <Modal
    ...
  >
    <Modal.Body
      ...
      ref={childPortalTargetRef}
    >
      ...
    </Modal.Body>
  </Modal>
)
```
So that takes care of getting references to the two DOM nodes we need

Then we want to dynamically change which one the portal uses as its destination based on whether the modal is open:
```
      <Portal
        to={showingModal ? lightboxPortalTargetNode : visuallyHiddenVideoContainerNode}
      >
        <Video
          ...
          playing={showingModal}
        />
      </Portal>
```
where `<Portal>` is just a simple wrapper around `React.createPortal()`:
```
import {createPortal} from 'react-dom'
import {flowMax, branch, renderNothing} from 'ad-hok'

import {childrenPropType, domNodePropType} from 'util/propTypes'

const Portal = flowMax(
  branch(({to}) => !to, renderNothing()),
  ({to, children}) => createPortal(children, to)
)

Portal.propTypes = {
  to: domNodePropType,
  children: childrenPropType.isRequired,
}
```
That's the gist of the dynamic portal implementation

For completeness' sake, I also ran into a little weird behavior in Chrome: for some reason when the modal was closed, the
video's audio was restarting and continuing to play (rather than the video stopping when the modal closed). We're using
[`react-player`](https://www.npmjs.com/package/react-player) as an abstraction around the actual HTML5 `<video>` inside our
`<Video>` component, so it's possible its internal state got wonky. But regardless, we can seize control of the situation by
making sure the video is fully unmounted after the modal has been closed

First, set up a `hasModalBeenClosed` state variable:
```
import {..., addStateHandlers} from 'ad-hok'

const addVideoUnmountingOnModalClose = flow(
  addStateHandlers(
    {hasModalBeenClosed: false},
    {
      onModalClose: () => () => ({
        hasModalBeenClosed: true,
      }),
    }
  ),
  addEffectOnPropChange(
    ['showingModal'],
    ({showingModal, onModalClose}, prevProps) => {
      if (!showingModal && prevProps.showingModal) {
        onModalClose()
      }
    }
  )
)
```
This uses the handy [`addEffectOnPropChange()` helper](./addEffectOnPropChange)

And then we'll consider the value of `hasModalBeenClosed` when deciding what the portal destination should be

Putting it all together:
```
const addVideoUnmountingOnModalClose = flow(
  addStateHandlers(
    {hasModalBeenClosed: false},
    {
      onModalClose: () => () => ({
        hasModalBeenClosed: true,
      }),
    }
  ),
  addEffectOnPropChange(
    ['showingModal'],
    ({showingModal, onModalClose}, prevProps) => {
      if (!showingModal && prevProps.showingModal) {
        onModalClose()
      }
    }
  )
)

const getVideoPortalTarget = ({
  showingModal,
  lightboxPortalTargetNode,
  hasModalBeenClosed,
  visuallyHiddenVideoContainerNode,
}) => {
  if (showingModal) return lightboxPortalTargetNode
  if (hasModalBeenClosed) return null
  return visuallyHiddenVideoContainerNode
}

const VideoAnswer = flow(
  ...
  addState('showingModal', 'setShowingModal', false),
  addCallbackRefAndNode(
    'lightboxPortalTargetCallbackRef',
    'lightboxPortalTargetNode'
  ),
  addCallbackRefAndNode(
    'visuallyHiddenVideoContainerCallbackRef',
    'visuallyHiddenVideoContainerNode'
  ),
  addVideoUnmountingOnModalClose,
  ({
    ...
    showingModal,
    lightboxPortalTargetCallbackRef,
    lightboxPortalTargetNode,
    visuallyHiddenVideoContainerCallbackRef,
    visuallyHiddenVideoContainerNode,
    hasModalBeenClosed,
  }) => (
    <>
      ...
      <div
        css={a11yStyles.visuallyHidden}
        aria-hidden="true"
        ref={visuallyHiddenVideoContainerCallbackRef}
      />
      <Portal
        to={getVideoPortalTarget({
          showingModal,
          lightboxPortalTargetNode,
          hasModalBeenClosed,
          visuallyHiddenVideoContainerNode,
        })}
      >
        <Video
          ...
          playing={showingModal}
        />
        ...
      </Portal>
      <Lightbox
        show={showingModal}
        ...
        childPortalTargetRef={lightboxPortalTargetCallbackRef}
      />
    </>
  )
)
```

#### Unresolved: a11y

This pattern seems to have achieved the desired behavior of getting unmuted autoplay to work cross-browser. But there's an
outstanding accessibility issue with it that I'm not sure if there's a great solution to:

When the video is initially mounted inside the visually-hidden container, it's hidden from screen readers using
`aria-hidden="true"`. But it's still keyboard-navigable (ie you can hit Tab and it will focus on the interactive elements of
the video component while it's offscreen)

I tried using the HTML5 `hidden` attribute instead of `aria-hidden="true"`, but that reverted the autoplay muted behavior

So I'm not sure if there's a general a11y technique for making all interactive elements under a given parent container element
non-keyboard-navigable?

Regardless, I think in this case achieving the desired autoplay behavior on the target iPad platform outweighs the potentially
confusing keyboard navigation. But it would be nice to cover our bases a11y-wise especially when thinking about reusing this
technique in the future
