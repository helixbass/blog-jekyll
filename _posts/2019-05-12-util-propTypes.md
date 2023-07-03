---
layout: post
excerpt_separator: <!--more-->
title: "React best practice: util/propTypes module for common prop types"
permalink: util-prop-types/
date: 2019-05-12
---

When building React apps, there are certain common types of objects (not specific to your app) that may get passed around as props, eg:
- styles
- `children`
- refs

Rather than rewriting the prop type for these types of objects manually each time you use them, a nice pattern is to export these "common prop types" from a helper module

For example, here's the `util/propTypes.js` module from the current project I'm working on:
```
import PropTypes from 'prop-types'

export const cssPropType = PropTypes.oneOfType([
  PropTypes.object,
  PropTypes.array,
])
export const classNamePropType = PropTypes.string
export const childrenPropType = PropTypes.node
export const imageSourcePropType = PropTypes.string
export const refPropType = PropTypes.object
```

So then if I'm defining a `Foo` component that accepts an optional `className` prop and a required `containerRef` prop, I can declare its prop types like:
```
Foo.propTypes = {
  className: classNamePropType,
  containerRef: refPropType.isRequired
}
```
This is DRYer and more declarative than doing it by hand would be:
```
Foo.propTypes = {
  className: PropTypes.string,
  containerRef: PropTypes.object.isRequired
}
```

### Prop types with style
This comes in particularly handy for styling-related props, which can often accept multiple different types

For example, this project uses [Emotion](https://emotion.sh/), so the `cssPropType` above is useful. But then on a React Native project, you might define `stylePropType` instead:
```
export const stylePropType = PropTypes.oneOfType([
  PropTypes.number,
  PropTypes.object,
  PropTypes.array,
  PropTypes.bool,
])
```

### Make it easy (on yourself) to be declarative
If the thought of writing the `import` statement for this existing prop type definition seems like more work than just writing it by hand, check out [`eslint-plugin-known-imports`](https://github.com/helixbass/eslint-plugin-known-imports) and let it do the work for you!

### Reuse between projects
Since these common prop types aren't project-specific, you can easily reuse most or all of an existing `util/propTypes` module when you start a new React project

### What about project-specific prop types?
I'd recommend keeping only non-project-specific prop types in `util/propTypes`, but there certainly can be common project-specific prop types as well (eg common data types that you're working with)

What do you think are good patterns for reusing project-specific prop types across different modules?
