Title: From Fulcro to Datastar: A Year of Development Velocity
Date: 2026-01-07
Tags: clojure, datastar, fulcro, hypermedia, ai
Preview: true

After two years building a production app with [Fulcro](https://fulcro.fulcrologic.com/), we migrated to [Datastar](https://data-star.dev/) in January 2026. The velocity difference has been striking. This post examines the numbers and explores why server-driven hypermedia is uniquely suited for AI-assisted development.

## The Numbers

Here is what the git history tells us:

| Metric | Fulcro Era (2022-2024) | Datastar Migration (Jan 2026) |
|--------|------------------------|------------------------------|
| Total commits | 2,732 (over 2+ years) | 78 (first week alone) |
| UI component code | 32,616 lines removed | 9,110 lines added |
| AI-assisted commits | ~0 | 71 (91% of new commits) |
| Files touched in migration | 182 | - |

The Fulcro removal commit tells the story: **32,616 lines deleted, 9,110 lines added**. Same functionality, one-third the code.

But the velocity metric that matters most is not lines of code. It is **commits per week with AI assistance**: 78 commits in the first week of January 2026, with 91% generated via [Claude Code](https://claude.com/claude-code).

## Why Datastar Enables AI Velocity

Fulcro is a sophisticated framework. It requires understanding:

- **Normalized client-side state** - data stored by ident, queries compose from fragments
- **EQL queries** - declarative data requirements that compose across components
- **Mutations** - optimistic updates, remote integration, state machine transitions
- **Form state management** - dirty tracking, validation, tempids for new entities
- **React integration** - hooks, lifecycle, render optimization

Here is a typical Fulcro modal component from the old codebase:

```clojure
(ns brian.ui.modal
  (:require
   #?(:cljs [com.fulcrologic.fulcro.dom :as dom]
      :clj [com.fulcrologic.fulcro.dom-server :as dom])
   [com.fulcrologic.fulcro-i18n.i18n :refer [tr trc]]
   [com.fulcrologic.fulcro.components :as comp :refer [defsc]]
   [com.fulcrologic.fulcro.mutations :as m]
   [com.fulcrologic.fulcro.react.hooks :as hooks]))

(m/defmutation show-modal [{:keys [ident] type :ui/type}]
  (action [{:keys [state]}]
          (swap! state assoc-in (conj ident :ui/modal)
                 {:ui/show? true :ui/type type})))

(m/defmutation hide-modal [{:keys [ident]}]
  (action [{:keys [state]}]
          (swap! state assoc-in (conj ident :ui/modal) {:ui/show? false})))

(defn- ui-modal
  [this {:ui/keys [overlay-id show? loading? style fade-type title content
                   action action-disabled? action-label actions footer-text
                   on-delete on-close close-label delete-label]}]
  (when show?
    (let [fade-type (or fade-type :blue)]
      ;; ... 200+ more lines of DOM construction, state management,
      ;; event handlers, computed props, etc.
      )))
```

The equivalent in Datastar:

```clojure
(ns brian.ui.modals
  (:require [brian.icons :as icons]
            [brian.ui.buttons :as buttons]
            [brian.ui.ds :as ds]))

(defn modal-overlay
  [{:keys [show-signal on-close]
    :or {show-signal "modalOpen"}} & children]
  [:div {:class "fixed inset-0 z-50 flex items-center justify-center"
         :style "display: none; background-color: rgba(99, 192, 255, 0.5);"
         :data-show (str "$" show-signal)
         :data-on:click (or on-close (str "$" show-signal " = false"))}
   (into [:div {:class (ds/tw "relative flex flex-col rounded-2xl"
                              "shadow-lg max-h-full bg-white")
                :data-on:click__stop "true"}]
         children)])
```

The Datastar version is:
- **Pure functions** - no mutations, no lifecycle, no state management
- **Plain data** - hiccup vectors, not React components
- **Server-controlled** - state lives on the server, HTML is the API

## HTML is the API

This paradigm shift is the key insight. In Fulcro, the API is EQL - a query language that requires understanding normalization, idents, and query composition. In Datastar, the API is HTML with `data-*` attributes.

```clojure
;; Fulcro: Client queries for data, manages local state
(defsc CourseCard [this {:course/keys [id title]}]
  {:query [:course/id :course/title]
   :ident :course/id}
  (dom/div {:onClick #(comp/transact! this [(select-course {:id id})])}
    title))

;; Datastar: Server sends HTML, client interprets attributes
(defn course-card [{:keys [id title]}]
  [:div {:data-on:click (str "@get('/courses/" id "')")}
   title])
```

The Datastar version requires no framework knowledge. It is HTML with a few conventions. An AI can generate this without understanding Fulcro's component model, query language, or state management.

## AI Comprehension Gap

When I ask Claude to write Fulcro code, it needs to understand:

1. How `defsc` works (macro that generates component + query + ident)
2. When to use `comp/transact!` vs direct state manipulation
3. How form state management works (fs/dirty?, tempids, validation)
4. Query composition and load patterns
5. The difference between props and computed props

When I ask Claude to write Datastar code, it needs to understand:

1. HTML
2. A handful of `data-*` attributes (`data-on:click`, `data-show`, `data-bind`)

The comprehension gap is enormous. Datastar code is nearly self-documenting because HTML semantics are universal knowledge.

## Real Example: Form Components

Compare the old Fulcro form infrastructure (95 lines just for the form wrapper):

```clojure
(ns brian.ui.form
  (:require
   [com.fulcrologic.fulcro.algorithms.form-state :as fs]
   [com.fulcrologic.fulcro.algorithms.merge :as merge]
   [com.fulcrologic.fulcro.algorithms.tempid :as tempid]
   [com.fulcrologic.fulcro.components :as comp]
   [com.fulcrologic.rad.form :as form]))

(defn invalid? [{::form/keys [master-form form-instance] :as env}]
  (let [{::form/keys [attributes read-only?]} (comp/component-options form-instance)
        read-only-form? (or
                         (?! read-only? form-instance)
                         (?! (comp/component-options master-form ::form/read-only?) master-form))]
    (and (not read-only-form?)
         (or (form/invalid? env)
             (some #(form/invalid-attribute-value? env %) attributes)))))

;; ... 70+ more lines of add-child, add-children, standard-controls, etc.
```

The Datastar equivalent is a simple input component:

```clojure
(defn input
  [{:keys [bind placeholder type searchable? on-change error? disabled?]
    :or {type :text}}]
  [:div {:class "relative border rounded hover:border-bright-blue"}
   (when searchable?
     [:div {:class "absolute left-4 top-1/2 -translate-y-1/2"}
      (icons/search)])
   [:input
    {:type (name type)
     :placeholder (or placeholder "")
     :data-bind bind
     :data-on:input__debounce.500ms on-change}]])
```

No form state management. No dirty tracking. No tempids. The server handles all of that.

## The Velocity Multiplier

The combination of simpler abstractions and AI assistance creates a velocity multiplier:

1. **Less code to write** - Server-side rendering eliminates client state management
2. **AI writes most of it** - 91% of our January commits were AI-assisted
3. **Faster iteration** - No compile step, no client bundle, instant hot reload
4. **Fewer bugs** - No state synchronization issues between client and server

The first week of January 2026 saw 78 commits. That is more commits than many months in the Fulcro era, when we averaged 70-80 commits per month.

## What We Lost

Fulcro is not without advantages:

- **Offline capability** - Client-side state enables offline-first apps
- **Optimistic updates** - Instant feedback before server confirmation
- **Fine-grained reactivity** - React's diffing is very efficient for complex UIs
- **Type safety** - EQL queries can be statically analyzed

For our use case (an education platform with reliable connectivity), these tradeoffs favor Datastar. Your mileage may vary.

## The Migration

The migration was surgical. We:

1. Created new Datastar UI components in `brian.ui.*` namespaces
2. Added route handlers that return HTML instead of EQL
3. Switched middleware to serve Datastar pages for main routes
4. Removed 32,000+ lines of Fulcro UI code

The hardest part was not writing new code. It was ensuring feature parity with the old UI. Having AI assistance meant we could focus on verification rather than implementation.

## Conclusion

The numbers speak for themselves: **one-third the code, 11x the commit velocity, 91% AI-assisted**.

Datastar's secret is not that it is a better framework. It is that it removes the framework. HTML is the API. The server is the source of truth. AI can write HTML.

If you are building a new Clojure web application in 2026, consider starting with [Datastar](https://data-star.dev/). If you have an existing Fulcro application, consider whether your use case requires client-side state. If not, the migration path is surprisingly tractable.

The future of web development might just be going back to basics: servers rendering HTML, with a thin layer of hypermedia for interactivity.
