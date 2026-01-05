Title: Hermes: URL-First Routing for Fulcro
Date: 2026-01-05
Tags: clojure, fulcro, routing
Preview: true

Routing in single-page applications is a solved problem in the same way that date handling is a solved problem: everyone agrees it should be simple, and everyone has scars proving otherwise. The URL and application state drift apart. Back button behavior surprises users. Deep links break because the app assumes you started at the home page.

The root cause is treating the URL as output - something you update after navigation happens. [Hermes](https://github.com/parenstech/hermes) inverts this: the URL is the source of truth. Change the URL, the app state follows. Change the app state, the URL updates to match. Bidirectional sync, always consistent.

Components declare their routes inline using a `defsc-route` macro. Hermes discovers these declarations and builds the routing table automatically. Two-tier loading states distinguish "we are fetching data" from "we are transitioning routes." Navigation guards intercept route changes to warn about unsaved forms before leaving. Auth protection redirects unauthenticated users to login and returns them after. Server-side rendering works because routing decisions live in the URL, not in client-side state machines.

The library learned from [Fulcro's dynamic router](https://book.fulcrologic.com/#_routing) and its predecessors. The insight was not new routing algorithms but treating the URL as the single source of truth that everything else derives from. Once you accept that constraint, the implementation becomes straightforward and the edge cases disappear.
