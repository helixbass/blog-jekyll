---
layout: post
excerpt_separator: <!--more-->
title: "Tip: \"Jiggle\" Webpack when it can't resolve a new import"
permalink: jiggle-webpack/
date: 2019-05-12
---

Sometimes [Webpack](https://webpack.js.org/) can get confused when you try and import a new module, failing (even though it should be able to resolve the module) with an error like:
```
Module not found: Can't resolve 'MyNewModule' in '/Users/jrosse/prj/myproject/src'
```

Specifically, I'm able to reproduce this by:
1. writing the new import statement *before* the new module actually exists
2. saving
3. creating the new module
4. re-saving the first module (the one doing the `import`ing of the new module)

This seems like a Webpack bug (and perhaps is tracked somewhere?), but here's the trick that [Matt Petrie](https://github.com/mattpetrie) taught me:

Just "jiggle" Webpack by eg changing the order of two of your `import` statements. This should get Webpack to recognize that the new module exists and build successfully :tada:

Strangely, if you then "un-jiggle" (eg restore the original `import` order), Webpack will fail again! Smells like a caching bug...?
