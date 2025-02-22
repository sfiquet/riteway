# RITEway

Test assertions that always supply a good bug report when they fail.

* **R**eadable
* **I**solated/**I**ntegrated
* **T**horough
* **E**xplicit

RITEway forces you to write **R**eadable, **I**solated, and **E**xplicit tests, because that's the only way you can use the API. It also makes it easier to be thorough by making test assertions so simple that you'll want to write more of them.

There are [5 questions every unit test must answer](https://medium.com/javascript-scene/what-every-unit-test-needs-f6cd34d9836d). RITEWay forces you to answer them.

1. What is the unit under test (module, function, class, whatever)?
2. What should it do? (Prose description)
3. What was the actual output?
4. What was the expected output?
5. How do you reproduce the failure?


## Installing

```shell
npm install --save-dev riteway
```

Then add an npm command in your package.json:

```json
"test": "riteway test/**/*-test.js",
```

Now you can run your tests with `npm test`. RITEway also supports full TAPE-compatible usage syntax, so you can have an advanced entry that looks like:

```json
"test": "nyc riteway test/**/*-rt.js | tap-nirvana",
```

In this case, we're using [nyc](https://www.npmjs.com/package/nyc), which generates test coverage reports. The output is piped through an advanced TAP formatter, [tap-nirvana](https://www.npmjs.com/package/tap-nirvana) that adds color coding, source line identification and advanced diff capabilities.

### Troubleshooting

If you get an error like:

```shell
SyntaxError: Unexpected identifier
    at new Script (vm.js:79:7)
    at createScript (vm.js:251:10)
    at Object.runInThisContext (vm.js:303:10)
...
```

The problem is likely that you need a `.babeljs` configured with support for esm (standard JavaScript modules) and/or React. If you need React support, that might look something like:

```json
{
  "presets": [
    [
      "@babel/preset-env",
      {
        "targets": [
          "last 2 versions",
          "safari >= 7"
        ]
      }
    ],
    "@babel/preset-react"
  ]
}
```

To install babel devdependencies:

```shell
npm install --save-dev @babel/core @babel/polyfill @babel/preset-env @babel/register
```

And if you're using react:

```shell
npm install --save-dev @babel/preset-react
```

You can then update your test script in `package.json` to use babel:

```json
"test": "node -r @babel/register -r @babel/polyfill source/test"
```

If you structure your folders by type like this:

```shell
├──todos
│  ├── component
│  ├── reducer
│  └── test
└──user
   ├── component
   ├── reducer
   └── test
```

Update your test script to find all files with your custom ending:

```json
"test": "riteway -r @babel/register -r @babel/polyfill 'src/**/*.test.js' | tap-nirvana",
```

Another option if you don't want to transpile is to install the [`esm` package](https://dev.to/bennypowers/you-should-be-using-esm-kn3). The esm-only option won't work for you if you use JSX in your project. If you're building a React project, use Babel instead.

```shell
npm install --save-dev esm
```

and use it in you package.json:

```json
"test": "riteway -r esm test/**/*.test.js | tap-nirvana"
```

## Example Usage

```js
import { describe, Try } from 'riteway';

// a function to test
const sum = (...args) => {
  if (args.some(v => Number.isNaN(v))) throw new TypeError('NaN');
  return args.reduce((acc, n) => acc + n, 0);
};

describe('sum()', async assert => {
  const should = 'return the correct sum';

  assert({
    given: 'no arguments',
    should: 'return 0',
    actual: sum(),
    expected: 0
  });

  assert({
    given: 'zero',
    should,
    actual: sum(2, 0),
    expected: 2
  });

  assert({
    given: 'negative numbers',
    should,
    actual: sum(1, -4),
    expected: -3
  });

  assert({
    given: 'NaN',
    should: 'throw',
    actual: Try(sum, 1, NaN),
    expected: new TypeError('NaN')
  });
});
```

### Testing React Components

```js
import React from 'react';
import render from 'riteway/render-component';
import { describe } from 'riteway';

describe('renderComponent', async assert => {
  const $ = render(<div className="foo">testing</div>);

  assert({
    given: 'some jsx',
    should: 'render markup',
    actual: $('.foo').html().trim(),
    expected: 'testing'
  });
});
```

RITEway makes it easier than ever to test pure React components using the `riteway/render-component` module. A pure component is a component which, given the same inputs, always renders the same output.

I don't recommend unit testing stateful components, or components with side-effects. Write functional tests for those, instead, because you'll need tests which describe the complete end-to-end flow, from user input, to back-end-services, and back to the UI. Those tests frequently duplicate any testing effort you would spend unit-testing stateful UI behaviors. You'd need to do a lot of mocking to properly unit test those kinds of components anyway, and that mocking may cover up problems with too much coupling in your component. See ["Mocking is a Code Smell"](https://medium.com/javascript-scene/mocking-is-a-code-smell-944a70c90a6a) for details.

A great alternative is to encapsulate side-effects and state management in container components, and then pass state into pure components as props. Unit test the pure components and use functional tests to ensure that the complete UX flow works in real browsers from the user's perspective.

#### Isolating React Unit Tests

When you [unit test React components](https://medium.com/javascript-scene/unit-testing-react-components-aeda9a44aae2) you frequently have to render your components many times. Sometimes the component needs to be wrapped in context (e.g. React Router or Redux) and often you want different props for some tests.

RITEway makes it easy to isolate your tests while keeping them readable by using [factory functions](https://link.medium.com/WxHPhCc3OV) in conjunction with [block scope](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/block).

```js
import ClickCounter from '../click-counter/click-counter-component';

describe('ClickCounter component', async assert => {
  const createCounter = clickCount =>
    render(<ClickCounter clicks={ clickCount } />)
  ;

  {
    const count = 3;
    const $ = createCounter(count);
    assert({
      given: 'a click count',
      should: 'render the correct number of clicks.',
      actual: parseInt($('.clicks-count').html().trim(), 10),
      expected: count
    });
  }

  {
    const count = 5;
    const $ = createCounter(count);
    assert({
      given: 'a click count',
      should: 'render the correct number of clicks.',
      actual: parseInt($('.clicks-count').html().trim(), 10),
      expected: count
    });
  }
});
```

## Output

RITEway produces standard TAP output, so it's easy to integrate with just about any test formatter and reporting tool. (TAP is a well established standard with hundreds (thousands?) of integrations).

```shell
TAP version 13
# sum()
ok 1 Given no arguments: should return 0
ok 2 Given zero: should return the correct sum
ok 3 Given negative numbers: should return the correct sum
ok 4 Given NaN: should throw

1..4
# tests 4
# pass  4

# ok
```

Prefer colorful output? No problem. The standard TAP output has you covered. You can run it through any TAP formatter you like:

```shell
npm install -g tap-color
npm test | tap-color
```

![Colorized output](docs/tap-color-screenshot.png)


## API

### describe

```js
describe = (unit: String, cb: TestFunction) => Void
```

Describe takes a prose description of the unit under test (function, module, whatever), and a callback function (`cb: TestFunction`). The callback function should be an [async function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function) so that the test can automatically complete when it reaches the end. RITEWay assumes that all tests are asynchronous. Async functions automatically return a promise in JavaScript, so RITEWay knows when to end each test.

### describe.only

```js
describe.only = (unit: String, cb: TestFunction) => Void
```

Like Describe, but don't run any other tests in the test suite.  See [test.only](https://github.com/substack/tape#testonlyname-cb)

### describe.skip

```js
describe.skip = (unit: String, cb: TestFunction) => Void
```

Skip running this test. See [test.skip](https://github.com/substack/tape#testskipname-cb)

### TestFunction

```js
TestFunction = assert => Promise<void>
```

The `TestFunction` is a user-defined function which takes `assert()` and must return a promise. If you supply an async function, it will return a promise automatically. If you don't, you'll need to explicitly return a promise.

Failure to resolve the `TestFunction` promise will cause an error telling you that your test exited without ending. Usually, the fix is to add `async` to your `TestFunction` signature, e.g.:

```js
describe('sum()', async assert => {
  /* test goes here */
});
```


### assert

```js
assert = ({
  given = Any,
  should = '',
  actual: Any,
  expected: Any
} = {}) => Void, throws
```

The `assert` function is the function you call to make your assertions. It takes prose descriptions for `given` and `should` (which should be strings), and invokes the test harness to evaluate the pass/fail status of the test. Unless you're using a custom test harness, assertion failures will cause a test failure report and an error exit status.

Note that `assert` uses [a deep equality check](https://github.com/substack/node-deep-equal) to compare the actual and expected values. Rarely, you may need another kind of check. In those cases, pass a JavaScript expression for the `actual` value.

### createStream

```js
createStream = ({ objectMode: Boolean }) => NodeStream
```

Create a stream of output, bypassing the default output stream that writes messages to `console.log()`. By default the stream will be a text stream of TAP output, but you can get an object stream instead by setting `opts.objectMode` to `true`.

```js
import { describe, createStream } from 'riteway';

createStream({ objectMode: true }).on('data', function (row) {
    console.log(JSON.stringify(row))
});

describe('foo', async assert => {
  /* your tests here */
});
```

### countKeys

Given an object, return a count of the object's own properties.

```js
countKeys = (Object) => Number
```

This function can be handy when you're adding new state to an object keyed by ID, and you want to ensure that the correct number of keys were added to the object.


## Render Component

First, import `render` from `riteway/render-component`:

```js
import render from 'riteway/render-component';
```

```js
render = (jsx) => CheerioObject
```

Take a JSX object and return a [Cheerio object](https://cheerio.js.org/), a partial implementation of the jQuery core API which makes selecting from your rendered JSX markup easy.
