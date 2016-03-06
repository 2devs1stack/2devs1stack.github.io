---
layout: post
author: Christian Williams
title:  "Simple Code Coverage with nyc"
date:   2016-03-05 00:00:00 -0500
categories: react tape testing nyc coverage
---

## Setting Up nyc

In my [previous post]({% post_url 2016-03-02-react-testing-bliss %}), I talked about how I loved the [**tape**](https://github.com/substack/tape) testing library because of its simplicity. In this post, I'm going to talk about a module called [**nyc**](https://github.com/bcoe/nyc). nyc adds code coverage metrics into test run output without adding complexity to your test code or build.

Here is a summary of my setup steps for a project using nyc---this includes setup for the other packages I'm using as well.

```
npm install nyc tape skin-deep react-addons-test-utils --save-dev
npm install babel-preset-stage-0 babel-preset-react babel-preset-es2015 babel-register --save-dev
```

I updated the package.json `test` script to run nyc with tape. Notice the `-e` flag to tell nyc to report on .jsx files. I also setup babel-register with tape since I'm writing esnext.

```
nyc -e .jsx tape --require babel-register 'tests/**/*.*'
```

I added the following config in package.json because I don't want to get coverage reporting on my tests folder.

```
"nyc": {
  "include": ["src/**"]
}
```

Here's sample output from `npm test` showing tape test results followed by an nyc coverage report.

```
$ npm test

> coverage@1.0.0 test ...
> nyc -e .jsx tape --require babel-register 'tests/**/*.*'

TAP version 13
# components:Page
ok 1 title should equal "abc"
ok 2 title should be null
ok 3 title should be "title clicked"
# components:SearchFilter
ok 4 text filter value prop matches
ok 5 text filter change handler called
# object-graph-root
ok 6 val is true

1..6
# tests 6
# pass  6

# ok

--------------------------|----------|----------|----------|----------|----------------|
File                      |  % Stmts | % Branch |  % Funcs |  % Lines |Uncovered Lines |
--------------------------|----------|----------|----------|----------|----------------|
 src/                     |       50 |      100 |        0 |       50 |                |
  app.js                  |      100 |      100 |      100 |      100 |                |
  untested.js             |       50 |      100 |        0 |       50 |              2 |
 src/components/          |      100 |      100 |       60 |      100 |                |
  page.jsx                |      100 |      100 |      100 |      100 |                |
  panel.jsx               |      100 |      100 |        0 |      100 |                |
  search-filter.jsx       |      100 |      100 |       50 |      100 |                |
 src/util/                |    82.61 |       50 |    81.82 |    86.36 |                |
  a-class.js              |      100 |      100 |      100 |      100 |                |
  an-arrow.js             |      100 |      100 |      100 |      100 |                |
  an-object.js            |      100 |      100 |      100 |      100 |                |
  branches.js             |       75 |       50 |      100 |       75 |              3 |
  functions.js            |       75 |      100 |       50 |       75 |              6 |
  index.js                |      100 |      100 |      100 |      100 |                |
  lines-and-statements.js |    66.67 |      100 |       50 |       80 |              2 |
  object-graph-child.js   |      100 |      100 |      100 |      100 |                |
  object-graph-root.js    |      100 |      100 |      100 |      100 |                |
--------------------------|----------|----------|----------|----------|----------------|
All files                 |    85.29 |       50 |    70.59 |    87.88 |                |
--------------------------|----------|----------|----------|----------|----------------|
```


---


## Thinking About Coverage

Code coverage is nice, because seeing good coverage numbers gives you that same *warm and fuzzy feeling* you get from seeing all your tests light up green, but it gives you even more confidence. Good code coverage with passing tests means quality is high and your code is safe to release.

nyc outputs four different code coverage metrics:

* Line coverage - what percentage of lines are executed during the test.
* Statement coverage - what percentage of statements...
* Function coverage - what percentage of functions...
* Branch coverage - what percentage of branches (if/else, switch)...

There are some things to watch out for to make sure your coverage numbers reflect reality. Coverage is calculated based on code executed during a test run. There is no strict implication that assertions were made on the code when it was executed. Consider the following two files: `object-graph-child` and `object-graph-root`.

```
export class ObjectGraphChild {
  func () {
    return true;
  }
}

```

```
import { ObjectGraphChild } from './object-graph-child';

export class ObjectGraphRoot {
  constructor () {
    this.child = new ObjectGraphChild();
  }

  func () {
    return this.child.func();
  }
}
```

And this test:

```
import test from 'tape';
import ObjectGraphRoot from './object-graph-root';

test('object-graph-root', t => {
  t.plan(1);
  const objectGraphRoot = new ObjectGraphRoot();
  const val = objectGraphRoot.func();
  t.ok(val, 'val is true');
});
```

Here is my coverage output:

```
--------------------------|----------|----------|----------|----------|----------------|
File                      |  % Stmts | % Branch |  % Funcs |  % Lines |Uncovered Lines |
--------------------------|----------|----------|----------|----------|----------------|
 src/                     |      100 |      100 |      100 |      100 |                |
  object-graph-child.js   |      100 |      100 |      100 |      100 |                |
  object-graph-root.js    |      100 |      100 |      100 |      100 |                |
--------------------------|----------|----------|----------|----------|----------------|
All files                 |      100 |      100 |      100 |      100 |                |
--------------------------|----------|----------|----------|----------|----------------|
```

ObjectGraphChild shows 100% coverage since ObjectGraphRoot calls out to it, but that doesn't seem fair since I didn't make any assertions on ObjectGraphChild's behavior. This would be even more glaring if there were any side effects from the call to ObjectGraphChild.

There are a couple different approaches we could take to restore meaning to our code coverage numbers. We could make assertions about ObjectGraphChild in this same test. That strategy is fine up to a point, but it can get hairy if we have large object graphs with complex relationships (think - React component tree). We could mock ObjectGraphChild so we don't artificially report that file as covered. Additionally, we could (and should) avoid side effects in child components as much as possible, but its impractical to try to avoid all association invocations.

In the case of React, we can take advantage of [shallow rendering](https://facebook.github.io/react/docs/test-utils.html#shallow-rendering). This is a pretty awesome tactic that hits the sweet spot for meaningful code coverage. When using shallow rendering, invoking a component's render method does not render child components, which means a unit test against an owner component will not inadvertently cover child components. I'm using [skin-deep](https://github.com/glenjamin/skin-deep) to do my shallow rendering because it provides a nice API, and an easy way to recursively render child components ([dive](https://github.com/glenjamin/skin-deep/tree/one-point-oh#treedivepath)) in case you do want to test across units in a single test.

## React Example

Here's the folder structure for a basic React example illustrating the key concepts we've discussed:

```
/src
  /components
    page.jsx
    panel.jsx
  app.js
  untested.js
/tests
  /components
    page.test.jsx
    panel.test.jsx
  app.test.js
```

For now, app.js isn't even mounting anything, its just importing page.jsx and untested.js.

```
// app.js
import './components/page.jsx';
import './untested';
```

In my tests folder, I've added an `app.test.js` that just imports the app but doesn't contain any actual tests. This is just so all the imports can happen and nyc will know about files and show them in the coverage report, even if they aren't tested yet.

```
// app.test.js
import App from '../src/app.js';
```

The Page component renders a child Panel component, but since we are using shallow rendering in the test, thats fine. Our Panel component will not be rendered so we will still have to test it independently to cover it.

Its important to note, when passing down the onSetTitle event handler to the child panel, we are passing the delegate bound to `this` setup in the constructor. This allows us to cover that function by testing it separately. Had we used an arrow function to bind `this`, we wouldn't have been able to cover the resulting function by calling it from the test since we wouldn't be able to simulate a Panel header click with shallow rendering, so our coverage report for this file would be doomed to less than 100%. Since we are passing an already bound function instead, we can invoke the same function from our test and hit 100% coverage even without rendering the child panel.

```
// page.jsx
import React, { Component } from 'react';
import Panel from './panel';

class Page extends Component {
  constructor (props) {
    super(props);
    this.state = {
      title: null,
      subtitle: null
    };
    this.setSubTitle = this.setSubTitle.bind(this);
  }

  setSubTitle (text) {
    this.setState({ subtitle: text });
  }

  render () {
    return (
      <div>
        <h1 onClick={() => this.setState({title: 'title clicked'})}>{this.state.title}</h1>
        <Panel title={this.state.subtitle} onSetTitle={this.setSubTitle}>
          <p>Panel Content...</p>
        </Panel>
      </div>
    );
  }
}

export default Page;
```

This is the test for the Page component---notice the call to `page.setSubTitle`.

```
// page.test.jsx
import test from 'tape';
import React from 'react';
import sd from 'skin-deep';
import Page from '../../src/components/page.jsx';

test('components:Page', t => {
  t.plan(2);

  const tree = sd.shallowRender(<Page />);
  const page = tree.getMountedInstance();
  const h2 = tree.subTree('h1');

  h2.props.onClick();

  t.equal('title clicked', page.state.title, 'title should be "title clicked"');

  page.setSubTitle('new subtitle');

  t.equal('new subtitle', page.state.subtitle, 'subtitle should be "new subtitle"');
});
```

Here is the code for the Panel component:

```
// panel.jsx
import React, { Component } from 'react';

const Panel = ({ title, onSetTitle, children }) => (
  <div className="panel">
    <h2 onClick={() => onSetTitle('subtitle')}>{ title }</h2>
    { children }
  </div>
);

export default Panel;
```

This is our test of the panel component.

```
// panel.test.jsx
import test from 'tape';
import React from 'react';
import sd from 'skin-deep';
import Panel from '../../src/components/panel.jsx';

test('components:Panel', t => {
  t.plan(3);

  let calledSetTitle = false;

  const tree = sd.shallowRender(<Panel title="abc" onSetTitle={() => calledSetTitle = true} />);
  const panel = tree.getMountedInstance();

  const h2 = tree.subTree('h2');

  t.equal('abc', h2.text(), 'h2 text should equal "abc"');

  h2.props.onClick();

  t.equal('abc', h2.text(), 'h2 text should equal "subtitle"');
  t.ok(calledSetTitle, 'setTitle was called');
});
```

Here is the output from our test run:

```
$ npm test

> coverage@1.0.0 test ...
> nyc -e .jsx tape --require babel-register 'tests/**/*.*'

TAP version 13
# components:Page
ok 1 title should be "title clicked"
ok 2 subtitle should be "new subtitle"
# components:Panel
ok 3 h2 text should equal "abc"
ok 4 h2 text should equal "subtitle"
ok 5 setTitle was called

1..5
# tests 5
# pass  5

# ok

-----------------|----------|----------|----------|----------|----------------|
File             |  % Stmts | % Branch |  % Funcs |  % Lines |Uncovered Lines |
-----------------|----------|----------|----------|----------|----------------|
 src/            |       50 |      100 |        0 |       50 |                |
  app.js         |      100 |      100 |      100 |      100 |                |
  untested.js    |       50 |      100 |        0 |       50 |              2 |
 src/components/ |      100 |      100 |      100 |      100 |                |
  page.jsx       |      100 |      100 |      100 |      100 |                |
  panel.jsx      |      100 |      100 |      100 |      100 |                |
-----------------|----------|----------|----------|----------|----------------|
All files        |     87.5 |      100 |    83.33 |     87.5 |                |
-----------------|----------|----------|----------|----------|----------------|
```

Notice the metrics for `untested.js`. It shows 50% statement coverage which makes sense. There are two statements in the file. One is the export and one is the return statement within the function. The export is covered by the fact that the file is imported by app.js. It shows 0% function coverage since we did not test the one function.
