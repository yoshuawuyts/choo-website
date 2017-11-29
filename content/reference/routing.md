# Routing
Choo is built up out of two parts: stores and views. In order to render a view,
it must be added to the application through `app.route()`. This is the router.

In many other frameworks, routing is done by a separate library. We found that
routing is common in most applications, so making it part of the framework
makes sense.

Routing has a few parts to it. Routing must handle the browser's history API
(e.g.  going forward & backwards). It must handle anchor tags, and programmatic
actions. There's more than a few moving pieces.

To perform routing, Choo uses a [Trie](https://en.wikipedia.org/wiki/Trie) data
structure. This means that our routings is fast, and the order in which routes
are added doesn't matter.

_Note: It's recommended to read the [views]('/reference/views) chapter first, as
we'll assume that you're already familiar with how views work. This chapter is
intended to give an overview of how routing works in Choo._

## Static routing
Every application needs an entry point. Routes in Choo are defined relative to
the host. The route `'/'` maps to `www.mysite.com`. The route `'/foo'` maps to
`www.mysite.com/foo`.

```js
var html = require('choo/html')
var choo = require('choo')

var app = choo()                   // 1.
app.route('/', view)               // 2.
app.mount('body')                  // 3.

function view () {                 // 4.
  return html`
    <body>Hello World</body>
  `
}
```

1. We need an instance of Choo to add our routes to, so let's create that
   first.
2. We're going to add a view on the `'/` route. This means that if people
   navigate to `oursite.com`, this will be the route that is enabled.
3. Now that we have our view, we can start rendering our application.
4. We declare our view at the bottom of the page. Thanks to "Scope Hoisting",
   we can use it higher in the code. For now it doesn't really matter what's in
   here, just that we return some DOM node.

## Anchor tags
There's no point in routing if you can't navigate between them. The easiest way
to navigate between routes is to use `<a>` tags (anchor tags). Choo picks up
whenever a tag was clicked, and figures out which route to trigger on the
router.

```js
var html = require('choo/html')
var choo = require('choo')

var app = choo()
app.route('/', view)               // 1.
app.route('/second', second)       // 2.
app.mount('body')                  // 3.

function view () {
  return html`
    <body>
      <a href="/second">
        Navigate to the next route.
      </a>
    </body>
  `
}

function second () {
  return html`
    <body>
      <a href="/">
        Navigate back.
      </a>
    </body>
  `
}
```

1. We define our base view on route `/`. This is the first route that's loaded
   when a person visits our site. It contains a single anchor tag that points
   to `/second`.
2. We defined our second route as `/second`. This won't be shown unless someone
   navigates to `/second`. When it's rendered, it contains a single anchor tag
   that points to `/`.
3. We render our app to the DOM. Once it's loaded, people can click on anchor
   tags to switch between views.

## Fallback routes
Preparing for things to go wrong is a large part of programming. At some point,
someone using an application will land on an unexpected route. It's important
to not just crash the page, but to show something helpful to explain what just
happened. This is where fallback routes come in.

```js
var html = require('choo/html')
var choo = require('choo')

var app = choo()
app.route('/', view)               // 1.
app.route('/404', notFound)        // 2.
app.route('/*', notFound)          // 3.
app.mount('body')                  // 4.

function view () {
  return html`
    <body>
      <a href="/uh-oh">
        Click Click Click
      </a>
    </body>
  `
}

function not found () {
  return html`
    <body>
      <a href="/">
        Route not found. Navigate back.
      </a>
    </body>
  `
}
```

1. We define our base view on route `/`. This is the first route that's loaded
   when a person visits our site. It contains a single anchor tag that points
   to `/uh-oh`, which is a route that doesn't exist.
2. It's good practice to define a fallback route statically as `/404`. This
   helps with debugging, and is often treated specially when deploying to
   production.
3. We define our fallback route as `*`. The asterix symbol is pronounced
   "glob". Our glob route will now handle all routes that didn't match
   anything.
4. We mount the application on the DOM. If someone now clicks the link that's
   rendered in `/`, it will be handled by the fallback route.

## Dynamic routing
Sometimes there will be pages that have the same layout, but different data.
For example user pages, or blog entries. This requires _dynamic routing_. In
Choo we have two types of syntax for dynamic routing.

### Params
Params are declared with the `:` syntax. For example `/foo/:bar`. This means
that the `/foo` part is static, and `:bar` can be any value, up until the next
slash (`/`).

The value from a param, is exposed in Choo as `state.params`. So say we have
the route `/foo/:bar`, and we navigate to `/foo/beep`, the value of
`state.params.bar` will be `'beep'`.

```js
var html = require('choo/html')
var choo = require('choo')

var app = choo()
app.route('/', placeholder)
app.route('/:user', placeholder)
app.route('/:user/:repo', placeholder)
app.route('/:user/settings', placeholder)
app.route('/404', placeholder)

function placeholder (state) {
  console.log(state.params)
  return html`<body>placeholder</body>`
}
```

### Wildcards
In most cases params are the answer for dynamic routing. They're named, readily
available, and easy to traverse. However, they can't cover cases where the
amount of slashes in route will be unknown.

Take for example GitHub's code view. To navigate to Choo's `html/raw` API, the
route is: https://github.com/choojs/choo/blob/master/html/raw.js. In this case,
everything after `master/` is unknown. The depth and names of the route
segments depend entirely on the project itself. This is what wildcards are for.

If we were building GitHub's code view with Choo, we could express the route
as: `/:user/:repo/blob/:branch/*`. The value of `state.params.wildcard` would
then be: `'/html/raw.js'`.

Try and use wildcards sparingly. They're the most powerful tool in the routing
toolbox, which means that if you're not careful you might end up reimplementing
a router on top of it yourself.

### 404s
There is one last thing we should touch on with dynamic routing: what to do
when a route is not found.

If you're using params or wildcards, there can always be a case where a route
isn't found. For example if we have the route `/:users`, if a particular user
does not exist, we might want to show a fallback route instead.

There are generally two approaches to this: hard redirects, and soft redirects.
- A hard redirect is when we redirect to a new route. For example if `/foobar`
  doesn't match a known user, we'll navigate to `/404` instead.
- A soft redirect is when the URL is kept the same, but different content is
  shown. For example if `/foobar` doesn't match, we'll require and render the
  content of the `/404` view instead.

It's generally recommended to use soft redirects, as they interfere the least
with the browser, and allow users to recover from an error (e.g. fix typos in a
url).