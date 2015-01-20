Cosmos [![Build Status](https://travis-ci.org/skidding/cosmos.svg)](https://travis-ci.org/skidding/cosmos)
===
A foundation for maintainable web applications.

Cosmos glues [React](http://facebook.github.io/react/) components together and
creates a uniform relationship between them.

There's no such thing as *controllers* or *pages* in Cosmos, just *components.*
The UI is a tree of components consisting of a root component and its
descendants. Any component can be loaded full-screen as the root element and
a route is just an alias to component _props._

Check out [**Flatris**](http://skidding.github.io/flatris/), a demo app built
with Cosmos.

Jump to:

- [Manifesto](#manifesto)
- [Installation](#installation)
- [Specs](#specs)
- [Problem](#problem)
- [Contributing](#contributing)

## Manifesto

_cos·mos<sup>1</sup> `/ˈkäzməs,-ˌmōs,-ˌmäs/` noun — 1. The universe seen as
a well-ordered whole._

- Zero bootstrap
- Can be plugged into any other framework
- Everything is a Component
- Components are oblivious of ancestors
- Any component can be represented by a URI
- The state of the entire UI can be serialized at any given point in time
- Components can implement any data sync mechanism

## Installation

Either install the `cosmos-js` [npm package](https://www.npmjs.org/package/cosmos-js)
or include the build directly in your project.

```html
<!-- Development build -->
<script src="http://skidding.github.io/cosmos/build/cosmos.js"></script>
<!-- Production build -->
<script src="http://skidding.github.io/cosmos/build/cosmos.min.js"></script>
```

## Specs

In Cosmos, any component in any state can be represented and reproduced by a
persistent JSON. This goes hand in hand with React's declarative nature.

The input data of a component is a JSON object and the role of a component is
to transform its input data into HTML output. Easy to follow and assert
behavior. See [React Component](http://facebook.github.io/react/docs/component-api.html)
for in-depth specs.

### Core concepts

Reserved props when working with Cosmos: `component`, `componentLookup` and
`state`.

#### Component lookup

In order to make component definitions serializable, Cosmos doesn't work with
classes directly. The `component` prop holds the name of the component, while
the `componentLookup` prop is used for passing a mapping function that returns
corresponding component classes.

Working with the component lookup also results in cleaner component files,
especially when using CommonJS modules with relative paths.

First, we need a component namespace.

```js
var components = {};

// The component classes can be drawn from any global namespace or
// require() mechanism
var componentLookup = function(name) {
  return components[name];
};
```

Then, we attach components to that namespace.

```js
components.Boy = React.createClass({
  mixins: [Cosmos.mixins.PersistState],

  getInitialState: function() {
    return {
      mood: this.props.initialMood
    };
  },

  render: function() {
    return <span>a {this.state.mood} {this.props.eyes}-eyed boy</span>;
  }
});
```

Once that is in place, we are able to render components without holding
references to their class object.

```js
var props = {
  component: 'Boy',
  componentLookup: componentLookup,
  eyes: 'blue',
  initialMood: 'happy'
};

var boy = Cosmos.render(props, document.body);
```

> "a happy blue-eyed boy"

#### Component snapshot

The props and state of a component can be joined into a unified snapshot. The
`state` prop holds the state of the component.

```js
// Why do people sleep at night?
boy.setState({mood: 'curious'});

var boySnapshot = boy.generateSnapshot();
```

This is what `boySnapshot` will look like:

```js
{
  component: 'Boy',
  componentLookup: [function Function],
  eyes: 'blue',
  initialMood: 'happy',
  state: {
    mood: 'curious'
  }
}
```

Serializing the snapshot is as easy as excluding the `componentLookup` key.

#### State injection

Serializing the state of components is no fun if we can't load it back later.

```js
// The clone will be created in the identical state of the original component
// (with a "curious" mood)
var boyClone = Cosmos.render(boySnapshot);
```

#### Children

Cosmos gets interesting when dealing with nested components. The entire state
of a component tree can be serialized recursively, as well as injected top-down
from the root component to the tree leaves.

This is achieved through the `loadChild` API of the `PersistState` mixin.

```js
components.Father = React.createClass({
  mixins: [Cosmos.mixins.PersistState],

  children: {
    son: function() {
      return {
        component: 'Boy',
        eyes: this.props.eyes,
        initialMood: this.props.mood,
      };
    }
  },

  render: function() {
    return <p>I am the {this.props.mood} father of {this.loadChild('son')}.</p>;
  }
});
```

The parent's props will propagate to its child upon rendering.

```js
var props = {
  component: 'Father',
  componentLookup: componentLookup,
  eyes: 'blue',
  mood: 'happy'
};

var father = Cosmos.render(props, document.body);
```

> "I am the happy father of a happy blue-eyed boy."

However, the child manages its own state from that point on.

```js
var son = father.refs.son;

// We missed the icecream truck :(
son.setState({mood: 'sad'});
```

> "I am the happy father of a sad blue-eyed boy."

We can now generate a recursive snapshot and take a of the capture the nested 
state.

```js
var familySnapshot = father.generateSnapshot(true);
```

This is what the nested snapshot will look like:

```js
{
  component: 'Father',
  componentLookup: [function Function],
  eyes: 'blue',
  mood: 'happy',
  state: {
    children: {
      son: {
        mood: 'sad'
      }
    }
  }
}
```

This makes it possible to capture the entire state of an application, persist
it and then reproduce it in a different session.

### Mixins

Core mixins are placed under the `Cosmos.mixins` namespace. Read more in the
[Mixins wiki page.](https://github.com/skidding/cosmos/wiki/Mixins)

## Problem

Most web frameworks start out clean and friendly, but at some point after you
go on to build a real-life application on them they stop cooperating and start
turning against you. Finding an honorable route for solving a problem is now a
luxury, workarounds are the norm.

This can happen over and over. Slick at first, unmaintainable in 2 years. But
why is that, why is complexity proportional to the number of features added?

Two reasons:

1. **Interdependence.** Tie a number of units together, rely on one to change
the state of another and you successfully gave birth to
an unpredictable ecosystem
1. **Data obscurity.** Mixing data with logic turns any transparent river of
data into a muddy sewage network and generates a big
pile of opinionated, disposable code.

Avoiding these pitfalls upfront is difficult. They occur only after you've
reached a certain maturity in a program. Easy to notice once the code is no
longer pleasant to work with, but very hard to predict.

Cosmos fights against this uncertainty by surfacing the elements that don't
benefit from a natural tendency to scale.

### Vertical encapsulation

Working with so many entities builds complex relationships. Models,
Controllers, Views, Helpers, etc., they're all connected to each other in
various ways. Your application is the outcome of all sorts of objects with
different roles and behaviors depending on one another.

Scaling you app linearly requires a flat infrastructure. **Responsibilities
should translate into domain logic instead of low-level roles (data
modeling, rendering, etc.)**

Cosmos only has one entity: The Component. They are autonomous, have end-to-end
capabilities and each can function as a complete application by itself,
excluding interdependence from the start. Because they can describe their
output without relying on any external logic, components are declarative and
predictable.

Isolating the source of a bug now has O(1) complexity.

### Transparent data structures

> Bad programmers worry about the code. Good programmers
> worry about data structures and their relationships.

Cosmos is partly inspired by a Linus Torvalds
[comment](http://lwn.net/Articles/193245/) about **designing your code
around your data and not the other way around.**

But Cosmos does not impose any specific data structures, it simply makes them
surface and hard to miss. All component logic wraps around a single JSON
object, which starts as the component input and updates along with state
changes. That object will always be a visible, persistent data structure.

## Contributing

Until the Cosmos project becomes more solid and concrete, I urge you to recall
a past experience of building a web application that became harder and harder
to work on as time passed and development progressed. Attempt to answer this
question: If you enforced the Cosmos principles onto the infrastructure you're
picturing, would it: a) improve, b) complicate or c) not influence the
situation?

Thank you for your interest!
