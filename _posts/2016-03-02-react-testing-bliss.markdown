---
layout: post
author: Christian Williams
title:  "React Testing Bliss with Tape and Shallow Rendering"
date:   2016-03-02 20:33:02 -0500
categories: react tape testing
---

## loveHate(Testing)

I've never been able to fully get on the TDD train, and I admit that tests are one of the first things I cut from my workflow when time is scarce. However, nothing feels as warm, safe and secure as a project with a good set of unit tests that light up green, letting you know you didn't break anything.

The number of testing libraries and tools to choose from can be a discouragement in itself. There are a lot of reasons for choosing any one over the other. I'm going to describe the setup I've come to love in my most recent projects.

A word about my most recent projects -- lately, I've been pretty focused on frontend, using:

* React
* Node JS for a build environment and NPM as the task runner
* Webpack and Babel

---

## Testing Simplicity

I have the major hots for a testing library called [**tape**](https://github.com/substack/tape). The number one reason I like it so much is because it behaves like an ordinary node module. Here is a super simple example:

```
var test = require('tape');

test('my test', function (t) {
  t.plan(1);
  t.equal(1 + 1, 2);
});
```

If you're familiar with test runners like mocha, the first thing you might notice is that tape is missing magical `describe` and `it` functions that find their way into test module scope. Tape is designed to use node as the test runner, so you use CommonJS to require the tape module, and then you use tape's API to plan your test and make assertions.

To run the above test, I can just run `node mytest.js`. I don't need a separate test runner, I don't need a config file and I don't need a headless browser.

Another thing I appreciate about tape is the [TAP protocol](https://testanything.org/). TAP is a testing protocol designed to be processed as a stream. Its well structured and super simple. Here is some example output:

```
1..4
ok 1 - Input file opened
not ok 2 - First line of the input valid
ok 3 - Read the rest of the file
not ok 4 - Summarized correctly # TODO Not written yet
```

Tape is the foundation for my simple testing setup. Since I use React and Webpack, and I prefer to write my code in esnext, things have to get at least a little more complicated -- but not much.

* I use [`babel-register`](http://babeljs.io/docs/usage/require/) to run a babel transform against my source files before they are executed.
* and I use [`skin-deep`](https://github.com/glenjamin/skin-deep/tree/one-point-oh) to do shallow renders of my React components.

---

Lets go ahead and run through an end-to-end example. I'll assume you created an empty folder and package.json prior to now. We'll start by installing the NPM packages.

## Install NPM packages

```
npm install react --save
npm install babel-register babel-preset-react babel-preset-es2015 babel-preset-stage0 --save-dev
npm install tape skin-deep react-addons-test-utils --save-dev
```

I will also update the package.json to contain our `npm test` script that looks like this:

```
"test": "tape -r babel-register tests/**/*.js"
```

Now when I run `npm test`, the `tape` executable will be invoked. Using the tape executable instead of plain ol' node adds in the glob file pattern support and the hook for babel-register. Speaking of babel-register, I will also add a .babelrc file to the project with the presets my project will use. Here's my standard .babelrc:

```
{
  "presets": ["es2015", "stage-0", "react"]
}
```

---

## React App Setup

This is what the directory structure looks like for this app. To run the tests, we actually won't be needing app.jsx, but I figured I'd throw it in there.

```
/react-tape-example
  /src
    /components
      /search-filter.jsx
    app.jsx
  /tests
    /components
      /search-filter.test.jsx
  .babelrc
  package.json
```

We won't setup Webpack in this example. I don't need it to start writing and running tests and components.

Let's implement search-filter.jsx...

```
import React, { Component } from 'react';

const SearchFilter = ({ textFilter, onTextFilterChange }) => (
  <div className="search-filter">
    <div className="search-container">
      <input type="text" placeholder="Search" value={textFilter} onChange={(e) => onTextFilterChange(e.target.value)} />
      <span className="search-icon">Search</span>
    </div>
  </div>
);

export default SearchFilter;
```

SearchFilter is a [dumb component](https://medium.com/@dan_abramov/smart-and-dumb-components-7ca2f9a7c7d0#.7ie1cq7ky). It takes in a prop for the text input that contains the user's entered search term, and an event handler to handle text change. The rendered input is [controlled](https://facebook.github.io/react/docs/forms.html#controlled-components), which means, the owner component will need to modify its own state and pass that down as a prop.

---

## Writing the Test

In our test, we don't need to bother implementing the state management logic described above because we are testing the dumb SearchFilter component. We just need to:

* Make sure SearchFilter invokes the text change handler we pass as a prop with the values we are expecting.
* Make sure SearchFilter's rendered input field contains the text we sent in as a prop.

```
import test from 'tape';
import React from 'react';
import sd from 'skin-deep';
import SearchFilter from '../../src/components/search-filter.jsx';

test('components:SearchFilter', t => {
  t.plan(2);

  const tree = sd.shallowRender(
    <SearchFilter
      textFilter="Text Filter"
      onTextFilterChange={text => {
        t.ok(text === 'Some Text');
      }}
    />
  );

  const searchFilter = tree.getMountedInstance();

  searchFilter.props.onTextFilterChange('Some Text');

  const textFilterInput = tree.subTree('.search-container')
    .subTree('input');

  t.equal(textFilterInput.props.value, 'Text Filter');
});
```

All we do is import the goodies we described above, use **skin deep** to do a shallow render, and start doing assertions and triggering events.

There you have it:

* No browser needed
* No separate test runner
* Super fast setup and execution time
* APIs are small - but everything you need

Tape is exactly what I needed to overcome the cognitive dissonance I experienced wading through the vast expanse of JavaScript unit testing tools.
