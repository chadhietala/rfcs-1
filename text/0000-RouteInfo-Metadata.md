- Start Date: 2018-11-02
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# RouteInfo MetaData

## Summary

The RFC introduces the ability to associate application specific metadata with it's corresponding `RouteInfo` object. This also adds a `metadata` field to the `RouteInfo` which will be the return value of `buildRouteInfoMetadata` for it's corresponding `Route`.

```js
// app/route/profile.js
import Route from '@ember/routing/route';
import { inject } from '@ember/service';
export default Route.extend({
  user: inject('user'),
  buildRouteInfoMetadata() {
    return {
      trackingKey: 'page_profile',
      user: {
        id: this.user.id,
        type: this.user.type
      }
    }
  }
  // ...
});
```

```js
// app/services/analytics.js
import Service, { inject } from '@ember/service';

export default Service.extend({
  router: inject('router'),
  init() {
    this._super(...arguments);
    this.router.on('routeDidUpdate', (transition) => {
      let { to, from } = transition;
      let fromMeta = from.metadata;
      let toMeta = to.metadata;
      ga.sendEvent('pageView', {
        from: fromMeta,
        to: toMeta,
        timestamp: Date.now(),
      })
    })
  },
  // ...
});
```

## Motivation

While the `RouteInfo` object is sufficient in providing developers metadata about the `Route` itself, it is not sufficient in layering on application specific metadata about the `Route`. This metadata could be anything from just a more domain specific name for a `Route` e.g. `profile_page` vs `profile.index`, all the way to providing contextual data when the `Route` was visited.

## Detailed design

### `buildRouteInfoMetadata`

This optional hook is intended to be used as a way of letting the routing system know about any metadata associated with the route.

#### `Route` Interface Extension

```ts
interface Route {
  // ... existing public API
  buildRouteInfoMetadata(): unknown
}
```

#### Runtime Semantics

- **Always** called before the `beforeModel` hook is called
- **Maybe** be called more than once during a transition e.g. aborts, redirects.

### `RouteInfo.metadata`

The `metadata` optional field on the `RouteInfo` will be populated with the return value of `buildRouteInfoMetadata`. If there is no metadata associated with the `Route`, the `metadata` field will be `null`.

```ts
interface RouteInfo {
  // ... existing public API
  metadata: Maybe<unknown>;
}
```

This field will also be added to `RouteInfoWithAttributes` as it is just a super-set of `RouteInfo`.


## How we teach this

We feel that this a low-level primitive that will allow existing tracking addons to encapsulate. That being said the concept here is pretty simple: What gets returned from `buildRouteInfoMetadata` becomes the value of `RouteInfo.metadata` for that `Route`.

The guides and tutorial should be updated to encoporate an example on how these APIs could integrate with services like Google Analytics.

## Drawbacks

This adds an additional hook that is called during route activation, expands the surface area of the `Route` class. While this is true there is currently know good way to associate application specicific metadata with a route transition.

## Alternatives

There are numerous alternative to the proposal:

### `setRouteMetaData`

This API would be similar to `setComponentManager` and `setModifierManager`. For example:

```js
// app/route/profile.js
import Route, { setRouteMetaData } from '@ember/routing/route';

export default Route.extend({

  init() {
    this._super(...arguments);
    setRouteMetaData(this, {
      trackingKey: 'page_profile',
      profile: {
        viewing: this.userId,
        locale: this.userLocale
      }
    });
  }
  // ...
});
```

You would then use the a `RouteInfo` to lookup the value:


```js
// app/services/analytics.js
import { getRouteMetaData } from '@ember/routing/route';
import Service, { inject } from '@ember/service';
 export default Service.extend({
  router: inject('router'),
  init() {
    this._super(...arguments);
    this.router.on('routeDidUpdate', (transition) => {
      let { to, from } = transition;
      let { trackingKey: fromKey } = getRouteMetaData(from);
      let { trackingKey: toKey } = getRouteMetaData(to);
      ga.sendEvent('pageView', {
        from: fromKey,
        to: toKey,
        timestamp: Date.now(),
      })
    })
  },
  // ...
});
```

This could work but there are two things that are confusing here:

1. What happens if you call `setRouteMetaData` mutliple times. Do you clobber the existing metadata? Do you merge it?
2. It is very odd that you would use a `RouteInfo` to access the metadata when you set it on the `Route`.

### `Route.meta`

This would add a special field to the `Route` class that would be copied off on to the `RouteInfo`. For example:

```js
// app/route/profile.js
import Route, { setRouteMetaData } from '@ember/routing/route';

export default Route.extend({
  meta: {
    trackingKey: 'page_profile',
    profile: {
      viewing: this.userId,
      locale: this.userLocale
    }
  }
  // ...
});
```

The value would then be populated on `RouteInfo.meta`.


```js
// app/services/analytics.js
import { getRouteMetaData } from '@ember/routing/route';
import Service, { inject } from '@ember/service';
 export default Service.extend({
  router: inject('router'),
  init() {
    this._super(...arguments);
    this.router.on('routeDidUpdate', (transition) => {
      let { to, from } = transition;
      let fromMeta = from.metadata;
      let toMeta = to.metadata;
      ga.sendEvent('pageView', {
        from: fromKey,
        to: toKey,
        timestamp: Date.now(),
      })
    })
  },
  // ...
});
```

This could work but there are two things that are problematic here:

1. What happens to the this data if you subclass it? Do you merge or clobber the field?
2. This is a generic property name and may conflict in existing applications

### Return MetaData From `activate`

Today `activate` does not get called when the dynamic segments of the `Route` change, making it not well fit for this use case.

## Unresolved questions

TBD?
