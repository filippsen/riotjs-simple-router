# riotjs-simple-router

A simple but powerful router for RiotJS.

Simple Router is simple because it doesn't use any tags or components of its own, there are no wrapping of components or use of higher order components.

Instead it uses the `riot.install` feature to make the Router instance available to each component as `this.router`.

Simple Router is optionally dependent on the `window` object, so it can also be used in environments where `window` is not accessible, such as in browser plugins.

The class `RouterCtrl` can be used to manage the `Router` without `window`, either in environments who do not support `window` or for testing purposes.

Simple Router works on the hash (`#`) part of the URL. Anything in the URL before the hash is ignored by the Router.

The root route is registered as `/`, not as `/#`, also not as `/#/`.

The browser location `/`, `/#` and `/#` all match the configured route `/`.

Sub routes can be registered by setting the base field.

Technically registering a sub route using the base route argument is similar to registering the full route with base prepended to the registered `match` field in the route object.

There are no real concepts of parent routes and child routes, it is just sugar for when registering a route.

There is however a configuration field called `subGroup`. This can be used to require that at least one of the routes with the given subGroup is active. `subGroup` is only applicable when base matches
the route configuration.

Note that all registered routes are matched for each round and set as active when matching (there are no hierarchies or early quit).

Although, with one exception, if a matched route has `reroute` set then the location is changed to the given routes `pushURL` and match is run again.

```js
router.register({
    main: {
        main: {
            match: "^/main[/]?$",
            pushURL: "/#/main/1",
        }
    }
});

// Register sub route
//
router.register({
    main1: {
        match: "^/1[/]?$",  // Matched as "^/main/1[/]?$" thanks to the base field "/main".
        pushURL: "/#/main/1",
        base: "/main",
    },
});
```

Note that `pushURL` is any relative or absolute URL, you must include the hash when wanting
to have it routed.

`router.onPreRoute((active: Active) => boolean)` is used to validate the matched route(s) prior to updating the UI. If it returns `true` then `update()` is not called
to not update the UI unnecessarily if the function did a redirect and another match is queued.

If no routes are matched, then the route `404` is automatically set as an active route as fallback. The `404` route does not need to be registered as a route, if it is it is ignored in the matching process (but still used as intended).

Also, if no routes are matched then the `router.onFallback((url: string) => boolean)` is called. If the function returns `true` then `onPreRoute()` is not called.


## Usage
In `main.js`:  

```js
import {Router} from "riotjs-simple-router";

// Optionally pass window object to automatically hook event and control URL redirection.
// This is optional as the Router can be used in environments where the window object is not
// accessible by instead hooking events manually.
//
const router = new Router(window);

// Make the Router object available to every component as `this.router`.
//
riot.install(function(c) {
    c.router = router;
});

router.register({
    page1: {
        match: "^/page/1[/]?$",
        pushURL: "/#/page/1",
        group: "pages",
    }
});
```

`page1` in this case is the name of the route, which can be used to do `pushRoute()`.

`match="^/page1[/]?$"` will perform a JavaScript `string.match` on the hash part of the url (everything from and including first `/`). All captured fields are passed to the `args` variable.  

`unmatch` same as `match` but if positive match then skip this route.

`pushUrl` is an optional url (absolute or relative) which is the url to redirect to when doing `router.pushRoute("page1")`.  

`group` is an optional field which can be used to group urls and the effect is that `pages` is set as an active route with the same values as `page1`.

`subGroup` if set then at least one route must match who has the subGroup set. Only applicable if `base` matches (if base is set).
For a route using `nomatch` which is skipped due to positive `nomatch` `subGroup` is ignored.

`reroute` if set then immediately when the route is matched the URL is replaced with the pushURL of the reroute route.

In components:  

```html
<app>
        <page1 if={router.active.page1} route={router.active.page1}></page1>
    </route>
</app>

<app>
        <page1 if={router.active.pages} route={router.active.pages}></page1>
    </route>
</app>
```

In routed to component `page1`:  

```html
<page1>
    <script>
        export default {
            onBeforeUpdate(props, state) {
                const changed = this.router.hasChanged(state.oldRoute, props.route);

                state.oldRoute = props.route;

                if (changed) {
                    // either route.args and/or route.search changed.
                    console.log("Route Changed", props.route.args, props.route.search);
                }
            }
        }
    </script>
</page1>
```

## Details
The routed to component (`page1` in this text) has the property `props.route` passed to it on mount
and whenever the route object changes `props.route` is updated and can be checked in `onBeforeUpdate`, as shown above.

The `match` parameter can match many separate fields and are passed on the `args` array in RouteParams as `props.route.args`.

Search params can be given after the route as `/page1/default?page1=data`. This field is parsed as JSON and is available as `search` on RouteParams.

Note that the search field is NOT the traditional search field which comes after the path and before the hash (`#`). The following will NOT work as expected in this example: `/?page1=data#/page1/default`.
