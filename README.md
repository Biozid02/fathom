# Fathom

Find meaning in the web.

## Introduction

Fathom is an experimental framework for extracting meaning from web pages, identifying parts like Previous/Next buttons, address forms, and the main textual content. Essentially, it scores DOM nodes and extracts them based on conditions you specify. A Prolog-inspired system of types and annotations expresses dependencies between scoring steps and keeps state under control. It also provides the freedom to extend existing sets of scoring rules without editing them directly, so multiple third-party refinements can be mixed together.

## Why?

A study of existing projects like Readability and Distiller suggests that purely imperative approaches to semantic extraction get bogged down in the mechanics of DOM traversal and state accumulation, obscuring the operative parts of the extractors and making new ones long and tedious to write. They are also brittle due to the promiscuous profusion of state. Fathom is an exploration of whether we can make extractors simpler and more extensible by providing a declarative framework around these weak points. In short, Fathom handles tree-walking, execution order, and annotation bookkeeping so you don't have to.

Here are some specific areas we address:

* Browser-native DOM nodes are mostly immutable, so storing intermediate data on them is clumsy. Fathom addresses this by providing the **fathom node** (confusing name—will change), a proxy behind each DOM node which we can scribble on.
* With imperative extractors, any experiments or site-specific customizations must be hard-coded in. Fathom's rulesets, on the other hand, are unordered and therefore decoupled, stitched together only by the **flavors** they consume and emit. External rules can thus be plugged into existing rulesets, making it easy to experiment (without maintaining a fork) or to provide dedicated rules for particularly intractable web sites.
* Flavors provide a convenient way of tagging DOM nodes as belonging to certain categories, typically narrowing as the extractor's work progresses. A typical complex extractor would start by assigning a broad flavor to a set of candidate nodes, then fine-tune by examining them more closely and assigning additional, more specific flavors in a later rule.
* The flavor system also makes explicit the division between an extractor's public and private APIs: the flavors are public, and the imperative stuff that goes on inside ranker functions is private. Third-party rules can use the flavors as hook points to interpose themselves.
* Persistent state is cordoned off in flavored **notes** on fathom nodes. Thus, when a rule declares that it takes such-and-such a flavor as input, it can rightly assume there will be a note of that flavor on the fathom nodes that are passed in.

## Status

Fathom is under heavy development, and its design is still in flux. If you'd like to use it at such an early stage, you should remain in close contact with us.

### Parts that work so far

* "Rank" phase: scoring of nodes found with a CSS selector
* Flavor-driven rule dispatch

### Not working yet

* Concise rule definitions
* "Yank" phase

## Example

Fathom recognizes the significant parts of DOM trees. But what is significant? You decide, by providing a declarative set of rules. This simple one finds DOM nodes that could contain a useful page title and scores them according to how likely that is:

```javascript
var titleFinder = ruleset(
    // Give any title tag a score of 1, and tag it as title-ish:
    rule(dom("title"), node => [{scoreMultiplier: 1, flavor: 'titley'}]),

    // Give any OpenGraph meta tag a score of 2, and tag it as title-ish as well:
    rule(dom("meta[og:title]") node => [{scoreMultiplier: 2, flavor: 'titley'}]),

    // Take all title-ish things, and punish them if they contain
    // navigational claptrap like colons or dashes:
    rule(flavor("titley"), node => [{scoreMultiplier: containsColonsOrDashes(node.element) ? 2 : 1}])
);
```

Each rule is shaped like `rule(condition, ranker function)`. A **condition** specifies what the rule takes as input: at the moment, either nodes from the DOM tree that match a certain CSS selector or else nodes tagged with a certain flavor by other rules. The **ranker function** is an imperative bit of code which decides what to do with a node: whether to scale its score, assign a flavor, make an annotation on it, or some combination thereof.

Please pardon the verbosity; we're waiting for patterns to shake out before choosing syntactic sugar.

```javascript
// Run the rules above over a DOM tree, and return a knowledgebase of facts
// about nodes which can be queried in various ways. This is the "rank" phase
// of Fathom's 2-phase rank-and-yank algorithm.
var knowledgebase = titleFinder.score(jsdom.jsdom("<html><head>...</html>"));

// "Yank" out interesting nodes based on their flavors and scores. For example,
// we might look for the highest-scoring node of a given flavor, or we might
// look for a cluster of high-scoring nodes near each other. (The yank phase
// has yet to be implemented.)
```
