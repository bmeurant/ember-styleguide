# Ember Style Guide

## Table of contents:
* [General](#general)
* [Organizing](#organizing)
* [Controllers](#controllers)
* [Components](#components)
* [Ember Data](#ember-data)
* [Routing](#routing)
* [Templates](#templates)
* [Tests](#tests)

## General

### Create local version of Ember.* and DS.*
Ember can use new functionality of ES6 - `modules`. In the near future, Ember will use this convention and eventually we will have to import `computed` instead of `Ember.computed`. To make code more clear and ready for the future, we should create local versions of these modules.
```javascript
import Ember from 'ember';
import DS from 'ember-data';

// GOOD
const { Model, attr } = DS;
const { computed } = Ember;
const { alias } = computed;

export default Model.extend({
  name: attr('string'),
  degree: attr('string'),
  title: alias('degree'),

  fullName: computed('name', 'degree', function() {
    return `${this.get('degree')} ${this.get('name')}`;
  }),
});

// BAD
export default DS.Model.extend({
  name: DS.attr('string'),
  degree: DS.attr('string'),
  title: Ember.computed.alias('degree'),

  fullName: Ember.computed('name', 'degree', function() {
    return `${this.get('degree')} ${this.get('name')}`;
  }),
});
```

### Don’t use jQuery without Ember Run Loop
Using plain jQuery invokes actions out of the Ember Run Loop. In order to have a control on all operations in Ember it's good practice to trigger actions in run loop.
```javascript
/// GOOD
Ember.$('#something-rendered-by-jquery-plugin').on(
  'click',
  Ember.run.bind(this, this._handlerActionFromController)
);

// BAD
Ember.$('#something-rendered-by-jquery-plugin').on('click', () => {
  this._handlerActionFromController();
});
```


### Don't use observers
Usage of observers is very easy **BUT** it leads to hard to reason about consequences. Unless observers are necessary, it's better to avoid them.
```hbs
{{input value=text key-up="change"}}
```

```javascript
// GOOD
export default Controller.extend({
  actions: {
    change() {
      console.log(`change detected: ${this.get('text')}`);
    },
  },
});

// BAD
export default Model.extend({
  change: Ember.observer('text', function() {
    console.log(`change detected: ${this.get('text')}`);
  },
});
```

### Don't introduce side-effects in computed properties
When using computed properties do not introduce side effects. It will make reasoning about the origin of the change much harder.

```js
import Ember from 'ember';

const {
  Component,
  computed: { filterBy, alias },
} = Ember;

export default Component.extend({
  users: [
    { name: 'Foo', age: 15 },
    { name: 'Bar', age: 16 },
    { name: 'Baz', age: 15 }
  ],

  // GOOD:
  fifteen: filterBy('users', 'age', 15),
  fifteenAmount: alias('fifteen.length'),

  // BAD:
  fifteenAmount: 0,
  fifteen: computed('users', function() {
    const fifteen = this.get('users').filterBy('items', 'age', 15);
    this.set('fifteenAmount', fifteen.length); // SIDE EFFECT!
    return fifteen;
  })
});
```

### Use named functions defined on objects to handle promises
When you use promises and its handlers, use named functions defined on parent object. Thus, you will be able to test them in isolation using unit tests without any additional mocking.

```js
export default Component.extend({
  actions: {
    // BAD
    updateUser(user) {
      user.save().then(() => {
        return user.reload();
      }).then(() => {
        this.notifyAboutSuccess();
      }).catch(() => {
        this.notifyAboutFailure();
      });
    },
    // GOOD
    updateUser(user) {
      user.save()
        .then(this._reloadUser.bind(this))
        .then(this._notifyAboutSuccess.bind(this))
        .catch(this._notifyAboutFailure.bind(this));
    },
  },
  _reloadUser(user) {
    return user.reload();
  },
  _notifyAboutSuccess() {
    // ...
  },
  _notifyAboutFailure() {
    // ...
  },
});
```

And then you can make simple unit tests for handlers:
```
test('it reloads user in promise handler', function(assert) {
  const component = this.subject();
  // assuming that you have `user` defined with kind of sinon spy on its reload method
  component._reloadUser(user);
  assert.ok(userReloadSpy.calledOnce, 'user#reload should be called once');
});
```

### Do not use Ember's `function` prototype extensions
Use computed property syntax, observer syntax or module hooks instead of `.property()`, `.observe()` or `.on()` in Ember modules.
```js
export default Component.extend({
    // BAD
    abc: function() { /* custom logic */ }.property('xyz'),
    def: function() { /* custom logic */ }.observe('xyz'),
    ghi: function() { /* custom logic */ }.on('didInsertElement'),

    // GOOD
    abc: computed('xyz', function() { /* custom logic */ }),
    def: observer('xyz', function() { /* custom logic */ }),
    didInsertElement() { /* custom logic */ }
});
```

### Use `Ember.get` and `Ember.set`
This way you don't have to worry whether the object that you're trying to access is an `Ember.Object` or not. It also solves the problem of trying to wrap every object in `Ember.Object` in order to be able to use things like `getWithDefault`.
```javascript
// Bad
this.get('fooProperty');
this.set('fooProperty', 'bar');
this.getWithDefault('fooProperty', 'defaultProp');
object.get('fooProperty');
object.getProperties('foo', 'bar');
object.setProperties({ foo: 'bar', baz: 'qux' });

// Good

const {
  get,
  set,
  getWithDefault,
  getProperties,
  setProperties
} = Ember;
// ...
get(this, 'fooProperty');
set(this, 'fooProperty', 'bar');
getWithDefault(this, 'fooProperty', 'defaultProp');
get(object, 'fooProperty');
getProperties(object, 'foo', 'bar');
setProperties(object, { foo: 'bar', baz: 'qux' });
```

### Use brace expansion
This allows much less redundancy and is easier to read.

Note that **the dependent keys must be together (without space)** for the brace expansion to work.

```
// Good
fullName: computed('user.{firstName,lastName}', {
  // Code
})

// Bad
fullName: computed('user.firstName', 'user.lastName', {
  // Code
})
```

## Organizing

### Organize your components
You should write code grouped and ordered in this way:

1. Services
2. Default values
3. Single line computed properties
4. Multiline computed properties
5. Observers
6. Lifecycle Hooks
7. Actions
8. Custom / private methods

```javascript
const { Component, computed, inject: { service } } = Ember;
const { alias } = computed;

export default Component.extend({
  // 1. Services
  i18n: service(),

  // 2. Defaults
  role: 'sloth',

  // 3. Single line Computed Property
  vehicle: alias('car'),

  // 4. Multiline Computed Property
  levelOfHappiness: computed('attitude', 'health', function() {
    const result = this.get('attitude') * this.get('health') * Math.random();
    return result;
  }),

  // 5. Observers
  onVahicleChange: observer('vehicle', function() {
    // observer logic
  }),

  // 6. Lifecycle Hooks
  init() {
    // custom init logic
  },

  didInsertElement() {
    // custom didInsertElement logic
  },

  // 7. All actions
  actions: {
    sneakyAction() {
      return this._secretMethod();
    }
  },

  // 8. Custom / private methods
  _secretMethod() {
    // custom secret method logic
  }
});
```

### Organize your models
You should write code grouped and ordered in this way:

1. Attributes
2. Relations
3. Computed Properties

```javascript
// GOOD
export default Model.extend({
  // 1. Attributes
  shape: attr('string'),

  // 2. Relations
  behaviors: hasMany('behaviour'),

  // 3. Computed Properties
  mood: computed('health', 'hunger', function() {
    const result = this.get('health') * this.get('hunger');
    return result;
  })
});

// BAD
export default Model.extend({
  mood: computed('health', 'hunger', function() {
    const result = this.get('health') * this.get('hunger');
    return result;
  }),

  hat: attr('string'),

  behaviors: hasMany('behaviour'),

  shape: attr('string')
});
```

### Organize your routes
You should write code grouped and ordered in this way:

1. Services
2. Default route's properties
3. Custom properties
4. model() hook
5. Other route's methods (beforeModel etc.)
6. Actions
7. Custom / private methods


```javascript
const { Route, inject: { service }, get } = Ember;

export default Route.extend({
  // 1. Services
  currentUser: service(),

  // 2. Default route's properties
  queryParams: {
    sortBy: { refreshModel: true },
  },

  // 3. Custom properties
  customProp: 'test',

  // 4. Model hook
  model() {
    return this.store.findAll('article');
  },

  // 5. Other route's methods
  beforeModel() {
    if (!get(this, 'currentUser.isAdmin')) {
      this.transitionTo('index');
    }
  },

  // 6. All actions
  actions: {
    sneakyAction() {
      return this._secretMethod();
    },
  },

  // 7. Custom / private methods
  _secretMethod() {
    // custom secret method logic
  },
});
```

### Organize your controllers
You should write code grouped and ordered in this way:

1. Services
2. Default controller's properties
3. Custom properties
4. Single line computed properties
5. Multi line computed properties
6. Observers
7. Actions
8. Custom / private methods


```javascript
const { Controller, computed, inject: { service }, get } = Ember;

export default Controller.extend({
  // 1. Services
  currentUser: service(),

  // 2. Default route's properties
  queryParams: ['view'],

  // 3. Custom properties
  attitude: 10,

  // 4. Single line Computed Property
  health: alias('model.health'),

  // 5. Multiline Computed Property
  levelOfHappiness: computed('attitude', 'health', function() {
    return get(this, 'attitude') * get(this, 'health') * Math.random();
  }),

  // 6. Observers
  onVahicleChange: observer('vehicle', function() {
    // observer logic
  }),

  // 7. All actions
  actions: {
    sneakyAction() {
      return this._secretMethod();
    },
  },

  // 8. Custom / private methods
  _secretMethod() {
    // custom secret method logic
  },
});
```

## Controllers

### Alias your model
It makes code more readable if model has the same name as a subject. It’s more maintainable, and will conform to future  routable components. We can do this in two ways:
- set alias to model (in case when there is a `Nail Controller`):
```javascript
const { alias } = Ember.computed;
export default Ember.Controller.extend({
  nail: alias('model'),
});
```
- set it in `setupController` method:
```javascript
export default Ember.Route.extend({
  setupController(controller, model) {
    controller.set('nail', model);
  },
});
```

### Query params should always be on top
If you are using query params in your controller, those should always be placed on top. It will make spotting them much easier.

```js
import Ember from 'ember';

const { Controller} = Ember;

// BAD
export default Controller.extend({
  statusOptions: Ember.String.w('Accepted Pending Rejected'),
  status: [],
  queryParams: ['status'],
});

// GOOD
export default Controller.extend({
  queryParams: ['status'],
  status: [],
  statusOptions: Ember.String.w('Accepted Pending Rejected'),
});
```

## Ember Data

### Be explicit with Ember data attribute types
Ember Data could handle lack of specified types in model description. Nonetheless this could lead to ambiguity. Therefore always supply proper attribute type to ensure the right data transform is used.
```javascript
const { Model, attr } = DS;

// GOOD
export default Model.extend({
  name: attr('string'),
  points: attr('number'),
  dob: attr('date'),
});

// BAD
export default Model.extend({
  name: attr(),
  points: attr(),
  dob: attr(),
});
```

In case when you need a custom behavior it's good to write own [Transform](http://emberjs.com/api/data/classes/DS.Transform.html)


## Components

### Data Down Action Up
You should't change passed data in components instead trigger actions that should change this data.

```hbs
{{! GOOD }}
{{nice-component dataArray=data removeElement=(action "removeElement")}}
```
```javascript
export default Component.extend({
  actions: {
    removeElement(element) {
      this.attr.removeElement(element);
    },
  },
});

export default Controller.extend({
  actions: {
    removeElement(element) {
      this.get('data').removeObject(element);
    },
  },
});
```

```hbs
{{! BAD }}
{{uggly-component dataArray=data}}
```
```javascript
export default Component.extend({
  actions: {
    removeElement(element) {
      this.get('dataArray').removeObject(element);
    },
  },
});
```

### Service-backed Components
Sometimes you have some data that are not crucial for given page and can be
loaded after the page has been rendered. This approach has many advantages such
as improving perceived load time. One solution to this problem is using
service-backed components that fetch the data using i.e injected store. This
makes the component hard to test - therefore it's better to fetch the data in
the router's `setupController` hook (which doesn't block the render) and pass
the promise to the component.

```js

//BAD - service-backed component

import Ember from 'ember';

const {
  Component,
  inject: { service }
} = Ember;

export default Component.extend({
  store: service(),
  loadComments() {
    this.get('store').findAll('comment').then((comments) => {
      // do something with the comments
    });
  },
});

//GOOD

//route
import Ember from 'ember';

export default Route.extend({
  setupController(controller, model) {
    controller.set('commentsPromise', this.store.findAll('comment'));
  }
})

// you can now handle commentsPromise in your component or controller
```

### Closure Actions
Always use closure actions (according to DDAU convention). Exception: only when you need bubbling.

```javascript
export default Controller.extend({
  actions: {
    detonate() {
      alert('Kabooom');
    }
  }
});
```

```hbs
{{! GOOD }}
{{pretty-component boom=(action 'detonate')}}
```

```javascript
export default Component.extend({
  actions: {
    pushLever() {
      this.attr.boom();
    }
  }
})
```

```hbs
{{! BAD }}
{{awful-component detonate='detonate'}}
```
```javascript
export default Component.extend({
  actions: {
    pushLever() {
      this.sendAction('detonate');
    }
  }
})
```
### Don't use `.on()` calls as components values
Prevents using `.on()` in favour of component's lifecycle hooks.
```js
export default Component.extend({
  // BAD
  abc: on('didInsertElement', function () { /* custom logic */ }),

  // GOOD
  didInsertElement() { /* custom logic */ }
});
```
### Avoid leaking state
Don't use arrays and objects as default properties. More info here: https://dockyard.com/blog/2015/09/18/ember-best-practices-avoid-leaking-state-into-factories


```
// BAD
export default Ember.Component.extend({
  items: [],

  actions: {
    addItem(item) {
      this.get('items').pushObject(item);
    },
  },
});
```

```
// Good
export default Ember.Component.extend({
  init() {
    this._super(...arguments);
    this.items = [];
  },

  actions: {
    addItem(item) {
      this.get('items').pushObject(item);
    },
  },
});
```

## Routing

### Route naming
Dynamic segments in routes should use _snake case_, so Ember doesn't have to do extra serialization in order to resolve promises.

```javascript
// GOOD
this.route('tree', { path: ':tree_id'});

// BAD
this.route('tree', { path: ':treeId'});
```

## Templates

## Use components in `{{#each}}` blocks
When content of each block is larger than one line, use component to wrap this code. Ember convention is build app through divided to smaller modules (Components). This is more flexible and readable. Use simple rule of thumb - if you need more than one line in `#each` block, then use component.

```hbs
{{! GOOD }}
{{#each paintings as |paiting|}}
  {{paiting-details paiting=paiting}}
{{/each}}

{{! BAD }}
{{#each paintings as |paiting|}}
  <title>{{paiting.title}}</title>
  <author>{{paiting.author}}</author>
  <img src={{paiting.image}}>
  <div>{{paiting.cost}}<div>
{{/each}}
```

## Tests

### Use page objects in acceptance testing
In acceptance tests there is a tendency to repeat the same code many times (mainly selectors). It's also hard to reason about exact behavior based only on selector naes. To avoid this we can extract this to other abstract layer and then import them to tests.

```javascript
// GOOD
export default Ember.Object({
  expectGroceryHeader(msg) {
    andThen(() => {
      this.get('assert').equal(find('#grocery-header').text(), msg);
    });
    return this;
  }
})

import PageObject from 'my-project/tests/page-objects/base';
...
test('check changed message', function(assert) {
  PageObject.create({ assert })
            .expectGroceryHeader('cabbage');
});
```

## Credits
Huge thanks to guys from Dockyard. Big part of our styleguide is based on their [Dockyard Ember Style Guide](https://github.com/DockYard/styleguides/blob/master/engineering/ember.md). Keep up the good work!
