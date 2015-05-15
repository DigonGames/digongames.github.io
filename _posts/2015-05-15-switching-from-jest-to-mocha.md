---
title:  "Making the switch from Jest to Mocha"
date:   2015-05-15 15:55:00
---
We're a React shop at Digon Games. Our client is written in React, using the Flux design pattern. You might even say we're Facebook fanboys when it comes to front-end technologies. So, when it came time to pick a test framework for our client, we turned straight to [Jest](https://facebook.github.io/jest/).

It didn't go very well. We struggled with Jest for a few months, and after yet another rage episode one afternoon, we decided enough was enough. We needed a better solution.

## What's the problem?

There were a few problems we had with Jest:

* The auto-mocking was often causing more problems than it solved. It was failing to mock out some modules we were using, causing obscure error messages, which told little about what was failing. This was usually solved with googling the problem and using the results to experiment with modules to exempt from auto-mocking. Often the problem was with some core node modules that Jest was trying to mock out. This led to an ever growing unmockedModulePathPatterns list in our package.json

* We found the Jest documentation to be a little scarce, and it feels like Jest community isn't very large. So when we were researching errors or general ways to do things, we sometimes found few answers.

* Tests ran very slow, especially when fiddling with the virtual DOM and rendering React components. With only a few dozen tests, which you'd expect to take a few hundred milliseconds to run, it was already taking 5+ seconds to run our tests.

* We came across bugs in Jest, which in some cases we couldn't get around (in particular: [https://github.com/facebook/jest/issues/211](https://github.com/facebook/jest/issues/211)), and didn't see much progress in fixing. The project even seemed to be stale on github for a while, but it's picking up now.

It is possible we jumped on early and these problems are related to the project being in it's early stages.

## Picking an alternative

In choosing an alternative, we wanted to be able to run the tests the same way we had been doing, in the shell. We needed a test framework, a test runner and an environment for our tests to run in. We looked at a few solutions for framework and runners, such as [Jasmine](http://jasmine.github.io/), [QUnit](http://qunitjs.com/) and [PhantomJS](http://phantomjs.org). In the end settled on [Mocha](http://mochajs.org/) + [jsdom](https://github.com/tmpvar/jsdom). This was the most obvious solution for us, as we're using Mocha to test our backend. So we know it well. We decided on jsdom for the environment because it seemed the easiest solution to set up. Even though jsdom is limited, and obviously not a real browser, it makes it easy to run our tests in the shell with DOM support. In the future we might look at solutions to run our tests in actual browsers for CI loops. On top of this we throw in [sinon](http://sinonjs.org/) for spys/stubs and [should](http://shouldjs.github.io) for assertions.

## The setup

When creating our setup these two resources were of much help: [Asbjørn Enge's Testing React Components](http://www.asbjornenge.com/wwc/testing_react_components.html) and [Hammer Lab's Testing React Web Apps with Mocha](http://www.hammerlab.org/2015/02/14/testing-react-web-apps-with-mocha/). Many thanks to the people behind those articles.

### Installation and setup

We need to start by installing our dependencies and setting up our environment. We're using JSX in our projects so we'll need a JSX transpiler. We opted for [Babel](http://babeljs.io/), we'll get free ES6 support thrown in as well.

```bash
npm install --save-dev mocha jsdom@3.1.2 babel sinon should
```

We're using jsdom 3xx branches because we're still on node.js, jsdom > 4 only supports io.js. Thankfully node.js and io.js have decided to converge in the Node foundation, so hopefully we'll be able to upgrade to jsdom latest soon.

Following Mocha conventions, we create a `/test` folder to put our tests in. We then add in our Makefile a command to run our tests:

```bash
test:
  @$(CURDIR)/node_modules/.bin/mocha --recursive --reporter spec --bail --compilers jsx:babel/register,js:babel/register
```

Now we can run our tests with `make test` in the shell. Great.

### Providing DOM

In order for us to test React components, we needed a DOM to mount them to. We created a file in the root of the test folder, call it `dom.js`:

```javascript
// make sure to always require this BEFORE requiring react
function createDOM() {
  // if DOM alredy exists, we don't need to do anything
  if (typeof document !== 'undefined') {
    return;
  }

  var baseDOM = '<!DOCTYPE html><html><head><meta charset="utf-8"></head><body></body></html>';

  var jsdom = require('jsdom').jsdom;
  global.document = jsdom(baseDOM);
  global.window = document.parentWindow;
  global.navigator = {
    userAgent: 'node.js'
  };
}

module.exports = createDOM();
```

Then we created another file in the root of the test folder, we called it `_init.js`. We know simply because of the name that Mocha will pick that file up first and run it. In there we do this:

```javascript
// always provide dom
require('./dom');
```

What we've done now is provide DOM to our tests, now we always have access to a global object called `document` and another one called `window`.

### Test isolation

One of the things Jest does transparently for you is provide test isolation, so that one test doesn't pollute another test. This mostly means two things: Stubbing/mocking dependencies before the test, and cleaning up after the test. This is something we have to solve manually in our setup.

For stubbing out dependencies, which Jest does by auto-mocking requires, we chose to explicitly stub what needs stubbing for each test. This is more verbose and laborious, but it's also less magic and more control.

We also needed to solve the issue of the DOM being global, so we need to clean it up after each test.

#### Testing React components in isolation

React provides test helpers called TestUtils. We started out using `TestUtils.renderIntoDocument` to mount our components, but quickly found that we wanted to be able to have a handle on the mounted components. So we wrote our own helper for this that returns the mounted node. In our `utils` helper file:

```javascript
var divs = [];
function renderIntoDocument(instance) {
  var div = document.createElement('div');
  divs.push(div);
  return React.render(instance, div);
}
```

Then in our tests, we do:

```javascript
var utils = require('utils');

describe('some test', {
  it('should render a component', function() {
      var SomeComponent = require('../components/somecomponent');

      var componentNode = utils.renderIntoDocument(<SomeComponent />);

      // do some work with componentNode
  });
});
```

Notice how we store the created elements in the utils file. We have another function in the utils file that looks like this:

```javascript
function unmountComponents() {
  divs.forEach(function(div) {
    React.unmountComponentAtNode(div);
  });
  divs = [];
}
```

This allows us to run this hook here in our tests:

```javascript
afterEach(function() {
  utils.unmountComponents();
});
```

Thus, we get automatic cleanup of tests after each test.

## Results

We've been using the setup for a few months now and we're very happy with switch. Writing tests takes much less time even though we have to manually stub stuff. And  they run with many fewer surprises and cryptic errors. Also they run in a fraction of the time - what took 5 seconds for only 30 tests before, now takes 4 seconds with 240 tests.
