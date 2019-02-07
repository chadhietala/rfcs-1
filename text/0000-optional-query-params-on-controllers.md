- Start Date: (fill me in with today's date, YYYY-MM-DD)
- Relevant Team(s): (fill this in with the [team(s)](README.md#relevant-teams) to which this RFC applies)
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# Optionally Move Query Params Off Controllers

## Summary

This RFC proposes an optional feature that makes query params to be treated as global state which can be consumed from `RouterService`. By setting the flag you opt out of all the controller based query param features and default serialization.

## Motivation

Query params was one of the most saught after APIs during the Ember 1.0.0 era. The whole idea behind the query params were to be able to serialize app state directly to the URL. When the url was shared and the user loaded the application the state of the application would be retained.

For very simple cases this API has worked great, but for more complex cases it has proven to be a rather large burden and in several cases just flat out broken. This RFC is intended to greatly simplify the usage of query params by allowing existing APIs to just work.

## Detailed design

This RFC proposes that the query params from the URL are simply global state that Ember deserializes that is either used directly or is source in which new state is derived from. This is best illustrated by the following example:

```js
import Component from '@ember/component';
import { tracked } from '@glimmer/tracking';
import { action } from 'ember-decorators';
import { inject as service } from '@ember/service';

export default class extends Component {
  @service router;

  get sortBy() {
    return this.router.currentQueryParams.sortBy || 'name';
  }

  @action
  updateSortBy(type) {
    this.router.transitionTo({ sortBy: type });
  }
}
```

```hbs
<table>
  {{#let (array 'name' 'age' 'city') as |headings|}}
    {{#each headings as |heading|}}
      <th class={{if (eq this.sortBy heading) 'active'}}>
        <button onclick={{this.updateSortBy heading}}>{{uppercase heading}}</button>
      </th>
    {{/each}}
  {{/let}}

  {{#each (sort-by this.sort @rows) as |row|}}
    <tr>
      <td>{{row.name}}</td>
      <td>{{row.age}}</td>
      <td>{{row.city}}</td>
    </tr>
  {{/each}}
</table>
```

In the follow example we would get a URL that was something like `/contacts/?sortBy=city`. This would render the UI with the table sorted by city. If the user clicks one of the buttons in the `<th>`s we will do a query-param only transition, updating the URL and causing the UI to re-render with new `currentQueryParams.sortBy` value.


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
