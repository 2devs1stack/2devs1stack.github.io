---
layout: post
author: Christian Williams
title:  "Simple Code Coverage With nyc"
date:   2016-03-03 00:00:00 -0500
categories: react tape testing nyc coverage
---

## Adding more simplicity

In my [previous post]({% post_url 2016-03-02-react-testing-bliss %}), I talked about how I loved the [**tape**](https://github.com/substack/tape) testing library because of its simplicity. In this post, I'm going to talk about a module called [**nyc**](https://github.com/bcoe/nyc) which adds code coverage metrics into the test run output without adding any complexity.

---

## Code coverage?

Code coverage is a nice metric, because seeing a good report gives you that same *warm and fuzzy feeling* you get from seeing all your tests light up green, but it gives you an even bigger sense of confidence. A good code coverage number with passing tests, means quality is high and its safe to release.

---

 ## Example

 Imagine the same directory structure from my [react + tape post]({% post_url 2016-03-02-react-testing-bliss %}), but I've added a couple extra files:

 ```
 /src
  /components
    page.jsx
    search-filter.jsx
  util.js
 /tests
  /components
    page.test.jsx
    search-filter.tests.jsx
  util.test.js
```

util.js has a few helper functions in it.

```
export function sayHello (person) {
  return `Hello ${person}!`
}

export function sayGoodbye (person) {
  return `Goodbye ${person}!`
}

export function calculateInterest(principal, rate, numberOfTimePeriods) {
  return principal * (1 + rate * numberOfTimePeriods);
}

```

search-filter.js is the same as we left it... and page.jsx looks like this:

```
import React, { Component } from 'react';

class Page extends Component {
  constructor (props) {
    super(props);
  }

  render () {
    return <div></div>;
  }
}
```

page.test.jsx is importing page.jsx, but there are currently no tests being ran against it. util.test.js is running 2 out of the 3 tests.

Now let's add coverage reporting. To do that, we need to install the nyc module:

```
npm install nyc --save-dev
```

Then update the package.json test command to run nyc.

```
nyc tape 'tests/**/*.*'
```
