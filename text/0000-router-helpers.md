- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Router Helpers & Modifiers

## Summary

This RFC introduces new router helpers and modifiers that represent a decomposition of functionality of what we commonly use `{{link-to}}` for. This RFC also deprecates the usage of `{{link-to}}` and `{{query-params}}` helpers. Below is a list of the new helpers:

```hbs
{{transition-to}}
{{root-url}}
{{url-for}}
{{is-active}}
{{is-loading}}
{{is-transitioning-in}}
{{is-transitioning-out}}
```

## Motivation

`{{link-to}}` is the primary way for Ember applications to transition from route to route in your application. While this works for a lot of cases there are some use cases that are not well supported or supported at all by the framework. Below is an enumeration of cases that `{{link-to}}` does not address.

### Anchor Tags

We currently do not have a good solution for transitioning solely based on HTML anchors defined in the templating layer. For instance let say you are using [Ember Intl](https://github.com/ember-intl/ember-intl) to do internationalization for your application. Ember Intl uses the [ICU message format](http://userguide.icu-project.org/formatparse/messages) for the actual translation strings and supports having HTML within the string. Now lets say you want to put a link in a translation string and have it work like `{{link-to}}` works. In that case you either have role your own solution or use something like [Ember-href-to](https://github.com/intercom/ember-href-to). API's like `RouterService#transitionTo` can transition an application using relative URLs and we have an oppurtunity to leverage this functionality to support this use case.

### Extensibility Of `{{link-to}}`

In 2.11 we moved `LinkComponent` from `private` to `public` largely because there was no other way to modify the behavior of `{{link-to}}` and it had effecively become de-facto public API. That being said, it is less than desirable to `reopen` or `extend` framework objects to gain access to the functionality to create some application specific primitive. For example [Ember Bootsrap](https://github.com/kaliber5/ember-bootstrap/) extends the [`LinkComponent`](https://github.com/kaliber5/ember-bootstrap/blob/master/addon/components/base/bs-dropdown/menu/link-to.js) and then layers more functionality on top of it. Addons would be better served if they had access to more primitive functionality.


### CSS Class Magic

`{{link-to}}` adds some convient, yet not obvious, classes to the element. These classes are:

- `active`: applied to any `{{link-to}}` that is on the "active" path
- `disabled`: applied depending on the evaluation of `disabled=someBool`
- `loading`: applied if one or more of the models passed are `undefined`
- `ember-transitioning-in`: applied to links that are about to be `active`
- `ember-transitioning-out`: applied to links that are about to be deactivated

The issue with these class names is that they are not declared anywhere in your templated and are provided by the `LinkComponent` as `classNameBindings`. This effectively creates a set of reserved class names that are highly prone to colissions in your typical application.

Furthermore, addons like [ember-cli-active-link-wrapper](https://github.com/alexspeller/ember-cli-active-link-wrapper) and [ember-bootstarp-nav-link](https://github.com/zoltan-nz/ember-bootstrap-nav-link) do a ton of work arounds to get things like the `.active` class to show up on wrapping elements instead of the element directly. This is a great example that shows we are missing some primitives.

### Default Query Param Serialization

Lastly, `{{link-to}}` has very strange behavior when it comes to serializing query params. On a controller you declare the query params for a specific route. These query params can have defaults for them. For example if you have a controller that looks like:

```js
// app/controllers/profile.js
import Controller from '@ember/controller';

export default Controller.extend({
  queryParams: ['someBool'],
  someBool: true,
})
```

and you to link to it like this:

```hbs
{{#link-to 'profile'}}Profile{{/link-to}}
```

In the DOM you will have an `href` on the anchor that gets serializes as:

```html
<a href="/profile?someBool=true" class="active ember-view">Profile</a>
```

Looking at a template you would have no idea that rendering the `{{link-to}}` would result in the query params being serialized. From an implementation point of view, this is problematic as we are forced to `lookup` the `Route` and the associated `Controller` to grab the query params. This can add a non-trival amount of overhead during rendering, especially if you have many `{{link-to}}`s on a route that link many different parts of your application. As a sidenote, this is one of the things  that needs to be delt with if we are ever to kill controllers.

## Detailed design

Below is a detailed design of all of the template helpers and modifiers.

### {{transition-to}} Element Modifier

```hbs
<a {{transition-to 'people.index' model queryParams=(hash a=a)}}>Profile</a>
```

`{{transition-to}}` is an element modifier that transitions the router to a new route when a user interacts with it. When installed on an anchor tag it will generate the url and set it as the `href` exactly how `{{url-for}}` would generate a url. When a user clicks or taps on the link, the event will be handled by Ember's global [`EventDispatcher`](https://www.emberjs.com/api/ember/release/classes/Ember.EventDispatcher).

By default `{{transition-to}}` adds to the browser's history when transitioning between routes. However, to replace the current entry in the browser's history you can use the `replace=true` option. This will behave exactly how `{{link-to ... replace=true}}` works.

```hbs
<a {{transition-to 'people.index' model queryParams=(hash a=a) replace=true}}>Profile</a>
```

`{{transition-to}}` takes a new option `data` that allows you to add metadata to the `Transition` object. For instance if you were instrumenting your application with Google Analytics you could use this in conjection with the routing service to understand the attribution of a transition.

```hbs
<a {{transition-to 'people.index' data=(hash attribution="edit.continue.button")}}>Continue >></a>
```

Then using the routing service's `routeDidChange` events you can intercept this data on the transition.

```js
import Route from '@ember/routing/route';
import { inject as service } from '@ember/service';

export default Route.extend({
  router: service('router'),
  init() {
    this._super(...arguments);

    this.router.on('routeDidChange', transition => {
      ga.send('pageView', {
        from: transition.from ? transition.from.name : 'initial',
        to: transition.to.name,
        attribution: transition.data.attribution
      });
    });
  }
})
```

#### `transitionTo` updates

`transitionTo` has always synchrounously returned a new `Transition` instance however it was the responsibilty of the caller to instert `data` onto the `Transition` instance. To align the declaritive API with the imperative API, this RFC proposes that `transitionTo` allows you to pass data in the options of `transitionTo` that will be set on the `Transition` during construction time.

```ts
interface Options {
  queryParams?: Dict<string|number>,
  data?: Dict<string|number>
}

interface Router /* Route, RouterService */ {
  //...
  transitionTo(routeName: string, models?: string|number|object, options?: Options): Transition;
}
```

#### `Transition.data` integrity

The data in a `Transition` is gurnateed to be carried through the completion of the route transition. This includes `abort`s, `redirect`s and `retry`s of the transition.

### URL Generation Helpers

### `{{root-url}}` Helper

`{{root-url}}` simply returns the value from `Application.rootURL`. It can be used to prefix any `href` values you wish to hard code.

```hbs
<a href="{{root-url}}profile">Profile</a>
```

Will result in the following for the default configuration:

```html
<a href="/profile">Profile</a>
```

#### Signature Explainer

`{{root-url}}` does not take any parameters.

### `{{url-for}}` Helper

```hbs
{{url-for routeName model queryParams=(hash a=a)}}
```

`{{url-for}}` generates a root-relative URL as a string (which will include the application's rootUrl).

#### Signature Explainer

Using the example above:

_Positional Params_
- **`routeName`** _String_: A fully-qualified name of this route, like `"people.index"`
- **`model`** _...Object|Array|Number|String_: Optionally pass an arbitrary amount of models to use for generation

_Named Params_
- **`queryParms`** _Object_: Optionally pass key value pairs that will be serialized

_Returns_
- _String_: a root-relative URL as a string (which will include the application's `rootUrl`)

### Route State Helpers

### `{{is-active}}` Helper

```hbs
{{is-active 'people.index' model queryParams=(hash a=a)}}
```

The arguments to `{{is-active}}` have the same semantics as `{{url-for}}`, however the return value is a boolean. This should provide the same logic that determines whether to put an `active` class on a `{{link-to}}`.

#### Signature Explainer

Using the example above:

_Positional Params_
- **`routeName`** _String_: A fully-qualified name of this route, like `"people.index"`
- **`model`** _...Object|Array|Number|String_: Optionally pass an arbitrary amount of models to use for generation

_Named Params_
- **`queryParms`** _Object_: Optionally pass key value pairs that will be used to determine if the route is active.

_Returns_
- _Boolean_: Determines if the route is active or not.

### `{{is-loading}}` Helper

```hbs
{{is-loading 'people.index' model queryParams=(hash a=a)}}
```

The arguments to `{{is-loading}}` have the same semantics as `{{url-for}}` and `{{is-active}}`, however if any of the model(s) passed to it are unresolved e.g. evaluate to `undefined` the helper will return `true`, otherwise the helper will return `false`. This should provide the same logic that determines whether to put an `loading` class on a `{{link-to}}`.

#### Signature Explainer

Using the example above:

_Positional Params_
- **`routeName`** _String_: A fully-qualified name of this route, like `"people.index"`
- **`model`** _...Object|Array|Number|String_: Optionally pass an arbitrary amount of models to use for generation

_Named Params_
- **`queryParms`** _Object_: Optionally pass key value pairs that will be used to determine if the route is loading.

_Returns_
- _Boolean_: Determines if the route is loading or not.

### `{{is-transitioning-in}}` Helper

```hbs
{{is-transitioning-in 'people.index' model queryParams=(hash a=a)}}
```

The arguments to `{{is-transitioning-in}}` have the same semantics as all the other route state helpers, however `{{is-transitioning-in}}` only returns `true` when the route is going from an non-active to an active state. This should provide the same logic that determines whether to put an `ember-transition-in` class on a `{{link-to}}`.

#### Signature Explainer

Using the example above:

_Positional Params_
- **`routeName`** _String_: A fully-qualified name of this route, like `"people.index"`
- **`model`** _...Object|Array|Number|String_: Optionally pass an arbitrary amount of models to use for generation

_Named Params_
- **`queryParms`** _Object_:  Optionally pass key value pairs that will be used to determine if the route is transitioning in.

_Returns_
- _Boolean_: Determines if the route is transitioning in.

### `{{is-transitioning-out}}` Helper

```hbs
{{is-transitioning-out 'people.index' model queryParams=(hash a=a)}}
```

`{{is-transitioning-out}}` is just the inverse of `{{is-transitioning-in}}`.

#### Signature Explainer

Using the example above:

_Positional Params_
- **`routeName`** _String_: A fully-qualified name of this route, like `"people.index"`
- **`model`** _...Object|Array|Number|String_: Optionally pass an arbitrary amount of models to use for generation

_Named Params_
- **`queryParms`** _Object_:  Optionally pass key value pairs that will be used to determine if the route is transitioning out.

_Returns_
- _Boolean_: Determines if the route is transitioning out.

### Event Dispatcher Changes

In the past, only `HTMLAnchorElement`s that were produced by `{{link-to}}`s would produce a transition when a user clicked on them. This RFC changes to the global `EventDispatcher` to allow for any `HTMLAnchorElement` with a valid root relative `href` to cause a transition. This will allow for us to not only allows us to support use cases like the ones described in the [motivation](#anchor-tags), it makes teaching easier since people who know HTML don't need know an Ember specific API to participate in routing transitions.

#### Route Globs And Route Blacklisting

While the vast majority of the time developers want root relative URLs to cause a transition there are cases where you want root relative urls to cause a normal HTTP navigation. In the router map you can define [wildcard / globbing](https://guides.emberjs.com/release/routing/defining-your-routes/#toc_wildcard--globbing-routes) that makes this problematic as any root relative url can be catched by a wildcard route. To solve this issue this RFC proposes expanding the route options to allow for a black list of urls that are allowed to cause a normal HTTP navigation.


```js
Router.map(function() {
  this.route('not-found', { path: '/*path', blacklist: ['/contact-us', '/order/:order_id'] });
});
```

When an event comes into the `EventDispatcher` we will cross check the blacklist to see if the event should be let through to the browser or if it should be handled internally.


## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

> How should this feature be introduced and taught to existing Ember
users?

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?
