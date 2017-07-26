---
layout: post
title: Hot module replacement in Webpacker
header_title: HMR in Webpacker
date: 2017-07-03 18:43:23 +0200
categories: ruby
description: |
  Rails 5.1 brings us a great new feature - Webpacker. It seamlessly integrates
  Rails with Webpack and allows us to build React, Angular, Vue and Elm
  frontends without much configuration hassle. The initial setup for React
  works quite well, but it can be even better. We can use hot module
  replacement.
last_modified_at: 2017-07-11 22:14:16 +0200
image: img/home-bg.jpg
---

Rails 5.1 brings us a great new feature - Webpacker. It seamlessly integrates
Rails with Webpack and allows us to build React, Angular, Vue and Elm frontends
without much configuration hassle.

The initial setup for React works quite well. The dev server listens for
changes in our codebase and triggers page reload. It is convenient because we
do not have to reload page manually, but it can be even better. We can use hot
module replacement (HMR).

HMR swaps only JavaScript code and does not touch HTML and CSS. It makes
development much more pleasant and quicker.

## Configuration

Configuring HMR is quite straightforward. You need to add `--hot` option to
`bin/webpack-dev-server` command. If you use `foreman`, then your Procfile
should look like this:
```
rails: bundle exec rails s
webpacker: bin/webpack-dev-server --hot
```

The next step is to prepare your code for hot reloading. Add the following
snippet to your entry point:
```javascript
if (module.hot) {
  module.hot.accept('./components/App', () => {
    const NextApp = require('./components/App').default

    ReactDOM.render(
      <NextApp />,
      document.getElementById('root')
    )
  })
}
```

Cool, but what is going on here? Let me explain it line by line.

## Explanation

```javascript
if (module.hot) {
```
In this line, we test if the HMR is turned on. Hot module replacement is useful
on development, but on production, it does not work (Who would want to reload
code on production? That is insane!).

```javascript
module.hot.accept('./components/App', () => {
```
`accept` function takes two arguments. The first is a path to the module which
we want to listen for changes. After a change in the module or its
dependencies, it calls the function passed as the second argument. You can find
more detailed description in the [official documentation](https://webpack.js.org/api/hot-module-replacement/).

```javascript
const NextApp = require('./components/App').default
```
Here, we use `require` to dynamically reload the root component. Why not
`import`? `import` was designed to handle static imports, while `require` is
capable of doing dynamic and runtime imports.

```javascript
ReactDOM.render(
  <NextApp />,
  document.getElementById('root')
)
```
In the last lines of code, we just render the reloaded component in the root
element.

## Example

I have prepared a sample application as a proof of concept. You can see that
it works and experiment with it. The repository can be found [here](https://github.com/piotrleniec/webpacker-configuration/tree/hot-module-replacement-js).
Don't forget to give a star if you like it. :)

That is it! Instead of a long full page reload, code swaps instantly, but
there's one problem - React state does not persist between the reloads. Can we
do something about it? Of course! In the next post, I am going to show you how
to configure `React Hot Loader` which solves the state issue.
