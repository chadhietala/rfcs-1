- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Router Helpers & Modifiers

## Summary

This RFC introduces new router helpers and modifiers that represent a decomposition of functionality of what we commonly use `{{link-to}}` for. This RFC also deprecates the usage of `{{link-to}}` and `{{query-params}}` helpers. Below is a list of the new helpers:

```hbs
{{transition-to}}
{{url-for}}
{{external-url}}
{{is-active}}
{{is-loading}}
{{is-transitioning}}
```

For convience sake `{{transition-to}}` is a [macro](#macro) that can be used as an element modifier.

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

### Default QueryParam Serialization

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

> This is the bulk of the RFC.

> Explain the design in enough detail for somebody
familiar with the framework to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

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
