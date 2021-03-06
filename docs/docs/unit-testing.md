---
title: Unit testing
---

Unit testing is a great way to protect against errors in your code before you
deploy it. While Gatsby does not include support for unit testing out of the
box, it only takes a few steps to get up and running. However there are a few
features of the Gatsby build process that mean the standard Jest setup doesn't
quite work. This guide shows you how to set it up.

## Setting up your environment

The most popular testing framework for React is [Jest](https://jestjs.io/),
which was created by Facebook. While Jest is a general purpose JavaScript unit
testing framework, it has lots of features that make it work particularly well
with React.

_Note: For this guide, you will be starting with `gatsby-starter-blog`, but the
concepts should be the same or very similar for your site._

### 1. Installing dependencies

First you need to install Jest and some more required packages. You need to
install Babel 7 as it's required by Jest.

```sh
npm install --save-dev jest babel-jest react-test-renderer identity-obj-proxy babel-core@^7.0.0-bridge.0 @babel/core babel-preset-gatsby
```

### 2. Creating a configuration file for Jest

Because Gatsby handles its own Babel configuration, you will need to manually
tell Jest to use `babel-jest`. The easiest way to do this is to add a `jest.config.js`. You can set up some useful defaults at the same time:

```json:title=jest.config.js
module.exports = {
  "transform": {
    "^.+\\.jsx?$": "<rootDir>/jest-preprocess.js"
  },
  "moduleNameMapper": {
    ".+\\.(css|styl|less|sass|scss)$": "identity-obj-proxy",
    ".+\\.(jpg|jpeg|png|gif|eot|otf|webp|svg|ttf|woff|woff2|mp4|webm|wav|mp3|m4a|aac|oga)$": "<rootDir>/__mocks__/fileMock.js"
  },
  "testPathIgnorePatterns": ["node_modules", ".cache"],
  "transformIgnorePatterns": ["node_modules/(?!(gatsby)/)"],
  "globals": {
    "__PATH_PREFIX__": ""
  },
  "testURL": "http://localhost",
  "setupFiles": ["<rootDir>/loadershim.js"]
}
```

Let's go over the content of this configuration file:

- The `transform` section tells Jest that all `js` or `jsx` files need to be
  transformed using a `jest-preprocess.js` file in the project root. Go ahead and
  create this file now. This is where you set up your Babel config. You can start
  with the following minimal config:

```js:title=jest-preprocess.js
const babelOptions = {
  presets: ["babel-preset-gatsby"],
}

module.exports = require("babel-jest").createTransformer(babelOptions)
```

- The next option is `moduleNameMapper`. This
  section works a bit like webpack rules, and tells Jest how to handle imports.
  You are mainly concerned here with mocking static file imports, which Jest can't
  handle. A mock is a dummy module that is used instead of the real module inside
  tests. It is good when you have something that you can't or don't want to test.
  You can mock anything, and here you are mocking assets rather than code. For
  stylesheets you need to use the package `identity-obj-proxy`. For all other assets
  you need to use a manual mock called `fileMock.js`. You need to create this yourself.
  The convention is to create a directory called `__mocks__` in the root directory
  for this. Note the pair of double underscores in the name.

```js:title=__mocks__/fileMock.js
module.exports = "test-file-stub"
```

- The next config setting is `testPathIgnorePatterns`. You are telling Jest to ignore
  any tests in the `node_modules` or `.cache` directories.

- The next option is very important, and is different from what you'll find in other
  Jest guides. The reason that you need `transformIgnorePatterns` is because Gastby
  includes un-transpiled ES6 code. By default Jest doesn't try to transform code
  inside `node_modules`, so you will get an error like this:

```
/my-blog/node_modules/gatsby/cache-dir/gatsby-browser-entry.js:1
({"Object.<anonymous>":function(module,exports,require,__dirname,__filename,global,jest){import React from "react"
                                                                                            ^^^^^^
SyntaxError: Unexpected token import
```

This is because `gatsby-browser-entry.js` isn't being transpiled before running
in Jest. You can fix this by changing the default `transformIgnorePatterns` to
exclude the `gatsby` module.

- The `globals` section sets `__PATH_PREFIX__`, which is usually set by Gatsby,
  and which some components need.

- You need to set `testURL` to a valid URL, because some DOM APIs such as
  `localStorage` are unhappy with the default (`about:blank`).

> Note: if you're using Jest 23.5.0 or later, `testURL` will default to `http://localhost` so you can skip this setting.

- There's one more global that you need to set, but as it's a function you can't
  set it here in the JSON. The `setupFiles` array lets you list files that will be
  included before all tests are run, so it's perfect for this.

```js:title=loadershim.js
global.___loader = {
  enqueue: jest.fn(),
}
```

### 3. Useful mocks to complete your testing environment

#### Mocking `gastby`

Finally it's a good idea to mock the `gatsby` module itself. This may not be
needed at first, but will make things a lot easier if you want to test
components that use `Link` or GraphQL.

```js:title=__mocks__/gatsby.js
const React = require("react")
const gatsby = jest.requireActual("gatsby")

module.exports = {
  ...gatsby,
  graphql: jest.fn(),
  Link: jest.fn().mockImplementation(({ to, ...rest }) =>
    React.createElement("a", {
      ...rest,
      href: to,
    })
  ),
  StaticQuery: jest.fn(),
}
```

This mocks the `graphql()` function, `Link` component, and `StaticQuery` component.

#### Mocking `location` from `Router`

One more issue that you may encounter is that some components expect to be able
to use the `location` prop that is passed in by `Router`. You can fix this by
manually passing in the prop:

```js:title=src/__tests__/index.js
import React from "react"
import renderer from "react-test-renderer"
import BlogIndex from "../pages/index"

describe("BlogIndex", () => {
  it("renders correctly", () => {
    const location = {
      pathname: "/",
    }

    const tree = renderer.create(<BlogIndex location={location} />).toJSON()
    expect(tree).toMatchSnapshot()
  })
})
```

For more information on testing page components, be sure to read the docs on
[testing components with GraphQL](/docs/testing-components-with-graphql/)

## Writing tests

A full guide to unit testing is beyond the scope of this guide, but you can
start with a simple snapshot test to check that everything is working.

First, create the test file. You can either put these in a `__tests__`
directory, or put them elsewhere (usually next to the component itself), with
the extension `.spec.js` or `.test.js`. The decision comes down to your own
taste. For this guide you will be testing the `<Bio />` component, so create a
`Bio.test.js` file next to it in `src/components`:

```js:title=src/components/Bio.test.js
import React from "react"
import renderer from "react-test-renderer"
import Bio from "./Bio"

describe("Bio", () => {
  it("renders correctly", () => {
    const tree = renderer.create(<Bio />).toJSON()
    expect(tree).toMatchSnapshot()
  })
})
```

This is a very simple snapshot test, which uses `react-test-renderer` to render
the component, and then generates a snapshot of it on first run. It then
compares future snapshots against this, which means you can quickly check for
regressions. Visit [the Jest docs](https://jestjs.io/docs/en/getting-started) to
learn more about other tests that you can write.

## Running tests

If you look inside `package.json` you will probably find that there is already a
script for `test`, which just outputs an error message. Change this to simply
`jest`:

```json:title=package.json
  "scripts": {
    "test": "jest"
  }
```

This means you can now run tests by typing `npm run test`. If you want you could
also add a script that runs `jest --watchAll` to watch files and run tests when
they are changed.

Now, run `npm run test` and you should immediately get an error like this:

```sh
 @font-face {
    ^

    SyntaxError: Invalid or unexpected token

      2 |
      3 | // Import typefaces
    > 4 | import 'typeface-montserrat'
```

This is because the CSS mock doesn't recognize the `typeface-` modules. You can
fix this easily by creating a new manual mock. Back in the `__mocks__`
directory, create a file called `typeface-montserrat.js` and another called
`typeface-merriweather.js`, each with the content `{}`. Any file in the mocks
folder which has a name that matches that of a node_module is automatically used
as a mock.

Run the tests again now and it should all work! You should get a message about
the snapshot being written. This is created in a `__snapshots__` directory next
to your tests. If you take a look at it, you will see that it is a JSON
representation of the `<Bio />` component. You should check your snapshot files
into a source control system (for example, a GitHub repo) so that so that any changes are tracked in history.
This is particularly important to remember if you are using a continuous
integration system such as Travis to run tests, as these will fail if no
snapshot is present.

If you make changes that mean you need to update the snapshot, you can do this
by running `npm run test -- -u`.

## Using TypeScript

If you are using TypeScript, you need to make a couple of small changes to your
config. First install `ts-jest`:

```sh
npm install --save-dev ts-jest
```

Then update the configuration in `jest.config.js`, like so:

```json:title=jest.config.js
module.exports = {
  "transform": {
    "^.+\\.tsx?$": "ts-jest",
    "^.+\\.jsx?$": "<rootDir>/jest-preprocess.js"
  },
  "testRegex": "(/__tests__/.*\\.([tj]sx?)|(\\.|/)(test|spec))\\.([tj]sx?)$",
  "moduleNameMapper": {
    ".+\\.(css|styl|less|sass|scss)$": "identity-obj-proxy",
    ".+\\.(jpg|jpeg|png|gif|eot|otf|webp|svg|ttf|woff|woff2|mp4|webm|wav|mp3|m4a|aac|oga)$": "<rootDir>/__mocks__/fileMock.js"
  },
  "moduleFileExtensions": ["ts", "tsx", "js", "jsx", "json", "node"],
  "testPathIgnorePatterns": ["node_modules", ".cache"],
  "transformIgnorePatterns": ["node_modules/(?!(gatsby)/)"],
  "globals": {
    "__PATH_PREFIX__": ""
  },
  "testURL": "http://localhost",
  "setupFiles": ["<rootDir>/loadershim.js"]
}
```

You may notice that two other options, `testRegex` and `moduleFileExtensions`,
have been added. Option `testRegex` is the pattern telling Jest which files
contain tests. The pattern above matches any `.js`, `.jsx`, `.ts` or `.tsx`
file inside a `__tests__` directory, or any file elsewhere with the extension
`.test.js`, `.test.jsx`, `.test.ts`, `.test.tsx`, or `.spec.js`, `.spec.jsx`,
`.spec.ts`, `.spec.tsx`.

Option `moduleFileExtensions` is needed when working with TypeScript.
The only thing it is doing is telling Jest which file extensions you can
import in your files without making precise the file extension. By default,
it works with `js`, `json`, `jsx`, `node` file extensions so we just need
to add `ts` and `tsx`. You can read more about it in [Jest's documentation](https://jestjs.io/docs/en/configuration.html#modulefileextensions-array-string).

## Other resources

If you need to make changes to your Babel config, you can edit the config in
`jest-preprocess.js`. You may need to enable some of the plugins used by Gatsby,
though remember you may need to install the Babel 7 versions. See
[the Gatsby Babel config guide](/docs/babel) for some examples.

For more information on Jest testing, visit
[the Jest site](https://jestjs.io/docs/en/getting-started).

For an example encapsulating all of these techniques--and a full unit test suite with [react-testing-library][react-testing-library], check out the [using-jest][using-jest] example.

[using-jest]: https://github.com/gatsbyjs/gatsby/tree/master/examples/using-jest
[react-testing-library]: https://github.com/kentcdodds/react-testing-library
