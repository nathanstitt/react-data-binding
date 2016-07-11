# react-model-binding

A mixin for React classes that allows them to easily integrate with "model" and "collection" type objects. It's primary use is integrating with Backbone/Ampersand models and collections, but will also work with any object that supports listening to events via `on` and stopping via `off`.

## What does it do?

It reads objects from `props` and will automatically bind to any props named `model` or `collection`.  Additional objects can be given by implementing a `dataBindings` property on the Component.

A derived property getter is created for each object to facilitate access to objects.

## Example

```javascript
var React  = require('react');
var Person = require('models/person');
var Food   = require('models/food');
var ModelBinding = require('react-model-binding');

// Pairing uses the ModelBinding mixin to access objects and update when they emit events
// It has two objects, food is passed in via props, and person is created the mixin is initialized
// as part of the 'getInitialState' lifecycle method
var Pairing = React.createClass({
    mixins: [ModelBinding],
    propTypes: {
        personName: React.PropTypes.string.isRequired
    },
    dataBindings: {
        food: 'props',
        person: function(){
            return new Person({name: this.props.personName, food: this.props.food});
        }
    },
    bindEvents: {
        food: 'change:name' // specify that we only care about the change:name event
                            // person will default to listening for 'change'
    },
    render: function () {
        // `dataBindings` above allow us to access `food` and `person` from `this`
        return (
            <div>{this.person.name} eats {this.food.name} with a {this.person.utensil}</div>
        );
    }
});

// Menu is a plain React class (no mixin).
var Menu = React.createClass({
    getInitialState: function(){
        return { food: new Food({name: RandomFood()}) };
    },
    updateFood: function(){
        this.food.name = RandomFood(); // will emit a 'change:food' event
        // The "Pairing" component is listening and will re-render
    },
    render: function() {
        return (
            <div>
                <button onClick={this.updateFood}>Update</button>
                <Pairing food={this.state.food} personName={'Joan'} />
            </div>
        );
    }
});
```


## API

#### dataBindings

Specifies objects to be bound. A derived properties will be set for each property name, and the value will be set to either the object from 'props' or by the results of a function.

If not given, dataBindings will be created for any props named "model" or "collection"

**Example:**
```javascript
    dataBindings: {
        foo: 'props',  // will use whatever value comes from props
        bar: function(){ new Model(); }
    }
```

#### bindEvents

Events to listen for.  By default objects will listen for the `change` event, and 'collections' will listen for `add`, `remove`, and `reset`.  Since ReactModelBinding attempts to be tool agnostic, Collections can only be identified by an object with name of "collection or that has an `isCollection` property.

**Example:**
```javascript
    dataBindings: {
        foo: 'props',
        collection: function(){ new Collection(); },
        bar: function(){ new Model(); }
    }
    bindEvents: {
        foo: 'change:name change:title loading',  // will ONLY listen for changes to name & title, and the "loading" event
        // bar is not mentioned so it will listen to 'change'
        // collection is not mentioned but it's named "collection", hence it will listen to `add remove reset`
    }
```

#### setDataState

If given, a method that will be called whenever an event fires.  If this is implemented, it is responsible for triggering the change on the component via `setState` or `forceUpdate`.  If `setDataState` is not present, `forceUpdate` will be called on the component whenever a listened to event occurs.

 **Note:**  Using this method isn't very "Reacty". Ideally you should allow events to fire and deal with the state as it is during render.   The only really valid use-case for `setDataState` is to prevent a possibly expensive computation from occurring during render.


**Example:**
```javascript
    dataBindings: {
        person: 'props'
    }
    setDataState: function(){
        // only bother re-rendering if somethings out of component scope is different
        if ( GLOBAL_TEMP_CHANGED_HAS_CHANGED() ) {
            this.forceUpdate();
        }
    }
    render: function(){
        // somewhat contrived, but imagine that calculating
        // climateChangeImpact can only be done on-the-fly and is an expensive computation
        return (
            <div>{this.person.name} has {this.person.climateChangeImpact()} impact</div>
        );
    }

```

#### onAttributeBind(events, name, previousObject)

Will be called whenever an object is bound, either when it's initially configured or when changed.

The `events` argument is a reference to the internal event object.  Additional events can listened for by calling `events.listenTo(<event name>, callbackFunction)`   Event bindings added in this way will be automatically removed when the component unmounts or a different object is bound.

`name` is the property name of the object, as given in `dataBindings`

`previousObject` argument will be null when called during initial setup, otherwise refers to what was the previous value.  **Note**: There is no need to unbind events that may have been established on `previousObject` during previous calls to `onAttributeBind`, the mixin will have already done so before calling `onAttributeBind`.
