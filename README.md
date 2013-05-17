Like Underscore, but lazier
===========================

**Lazy.js** it a utility library for JavaScript, similar to [Underscore](http://underscorejs.org/) and [Lo-Dash](http://lodash.com/) but with one important difference: **lazy evaluation** (also known as deferred execution). This can translate to superior performance in many cases, *especially* when dealing with large arrays and/or "chaining" together multiple methods. For simple cases (`map`, `filter`, etc.) on small arrays, Lazy's performance should be similar to underscore or Lo-Dash.

The following chart illustrates the performance of Lazy.js versus Underscore and Lo-Dash for several common operations using arrays with 10 elements each on Chrome:

![Lazy.js versus Underscore/Lo-Dash](http://dtao.github.io/lazy.js/specs/lib/LazyVsLodash10Elements.png)

You can see that the performance difference becomes much more significant for methods that don't require iterating an entire collection (e.g., `indexOf`, `take`) as the arrays get larger:

![Lazy.js versus Underscore/Lo-Dash](http://dtao.github.io/lazy.js/specs/lib/LazyVsLodash100Elements.png)

Intrigued? Great! Now let's look at how Lazy.js actually works.

Introduction
------------

We'll start with an containing the integers from 0 to 999. Incidentally, generating such an array using Lazy.js is quite trivial:

```javascript
var array = Lazy.range(1000).toArray();
```

Note the `toArray` call; without it, what you'll get from `Lazy.range` won't be an actual array but rather a *sequence* which you can iterate over using `each`. But that will make more sense in a moment.

Now let's say we want to take the *squares* of each of these numbers, increment them, and then check which results are even. We'll use these helper functions:

```javascript
function square(x) { return x * x; }
function inc(x) { return x + 1; }
function isEven(x) { return x % 2 === 0; }
```

Yes, this is admittedly a very arbitrary goal. (Later I'll get around to thinking of a more realistic scenario.) Anyway, here's one way you might accomplish it using Underscore:

```javascript
var result = _.chain(array).map(square).map(inc).filter(isEven).take(5).value();
```

This query creates lots of extra arrays:

- `map(square)`: an extra 1000-element array
- `map(inc)`: another 1000-element array
- `filter(isEven)`: another 500-element array
- `take(5)`: all that just for 5 elements!

So if performance and/or efficiency were a concern for you, you would probably *not* do things this way using Underscore. Instead, you'd likely go the procedural route:

```javascript
var results = [];
for (var i = 0; i < array.length; ++i) {
  var value = (array[i] * array[i]) + 1;
  if (value % 2 === 0) {
    results.push(value);
    if (results.length === 5) {
      break;
    }
  }
}
```

There&mdash;now we we haven't created have any extraneous arrays, and we did all of the work in one iteration. Any problems?

Well, yeah. The main problem is that this is one-off code, which isn't reusable and took a bit of time to write. If only we could somehow leverage the expressive power of Underscore but still get the performance of the hand-written procedural solution...

That's where Lazy.js comes in! Here's how we'd write the above query using Lazy.js:

```javascript
var result = Lazy(array).map(square).map(inc).filter(isEven).take(5);
```

Looks almost identical, right? That's the idea: Lazy.js aims to be completely familiar to experienced JavaScript devs. Every method from Underscore should have the same name and identical behavior in Lazy.js, except that instead of returning a fully-populated array on every call, it creates a *sequence* object with an `each` method.

What's important here is that **no iteration takes place until you call `each`**. Which means that, unlike the Underscore example above, the equivalent Lazy.js query creates no extra arrays. Essentially it combines all necessary operations into a sequence that behaves quite a bit like the procedural code we wrote a moment ago.

Of course, *unlike* the procedural approach, Lazy.js lets you keep your code clean and functional, and focus on buliding an application instead of optimizing array traversals.

So, cool. Is that all? I think not!

Indefinite sequence generation
------------------------------

The sequence-based paradigm of Lazy.js lets you do some pretty cool things that simply aren't possible with Underscore's array-based approach. One of these is the generation of **indefinite sequences**, which can go on forever, yet still support all of Lazy's built-in mapping and filtering capablities.

Want an example? Sure thing! Let's say we want 300 unique random numbers between 1 and 1000.

```javascript
var uniqueRandsFrom1To1000 = Lazy.generate(function() { return Math.random(); })
  .map(function(e) { return (e * 1000) + 1; })
  .uniq()
  .take(300);

// Output: see for yourself!
uniqueRandsFrom1To1000.each(function(e) { console.log(e); });
```

Pretty neat. How about a slightly more advanced example? Let's use Lazy.js to make a [Fibonacci sequence](http://en.wikipedia.org/wiki/Fibonacci_number).

```javascript
var fibonacci = Lazy.generate(function() {
  var x = 1,
      y = 1;
  return function() {
    var prev = x;
    x = y;
    y = prev + y;
    return prev;
  };
}());

// Output: undefined
var length = fibonacci.length();

// Output: [2, 2, 3, 4, 6, 9, 14, 22, 35, 56]
var firstTenFibsPlusOne = fibonacci.map(inc).take(10).toArray();
```

OK, what else can we do with Lazy.js?

Asynchronous iteration
----------------------

You've probably seen code snippets before that show how to iterate over an array asynchronously in JavaScript. But have you seen such an example packed full of map-y, filter-y goodness like this?

```javascript
// The second argument defines a 100-millisecond interval between each element.
var asyncSequence = Lazy.async(array, 100)
  .map(inc)
  .filter(isEven)
  .take(20);

asyncSequence.each(function(e) {
  console.log(new Date().getMilliseconds() + ": " + e);
});
```

All right... what else?

Event sequences
---------------

With indefinite sequences, we saw that unlike Underscore and Lo-Dash, Lazy.js doesn't actually need an in-memory collection to iterate over. And asynchronous sequences demonstrate that it also doesn't need to do all its iteration at once.

Now here's a really cool combination of these two features: with *lazy.dom.js* (a separate file to include in browser-based environments), you can apply all of the power of Lazy.js to **handling DOM events**. In other words, you can think of DOM events as a *sequence*&mdash;just like any other&mdash;and apply the usual `map`, `filter`, etc. functions on that sequence.

Here's an example. Let's say we want to handle all `mousemove` events on a given DOM element, and show their coordinates in another DOM element.

```javascript
// First we define our "sequence" of events.
var mouseEvents = Lazy.events(sourceElement, "mousemove");

// Map the Event objects to their coordinates, relative to the element.
var coordinates = mouseEvents.map(function(e) {
  var elementRect = sourceElement.getBoundingClientRect();
  return [
    Math.floor(e.clientX - elementRect.left),
    Math.floor(e.clientY - elementRect.top)
  ];
});

// For mouse events on one side of the element, display the coordinates in one place.
coordinates
  .filter(function(pos) { return pos[0] < sourceElement.clientWidth / 2; })
  .each(function(pos) { displayCoordinates(leftElement, pos); });

// For those on the other side, display them in a different place.
coordinates
  .filter(function(pos) { return pos[0] > sourceElement.clientWidth / 2; })
  .each(function(pos) { displayCoordinates(rightElement, pos); });
```

Anything else? Nope&mdash;that's it for now!

Available functions
-------------------

Currently the following functions are available (meaning you can call them on any sequence, such as what you get back from `Lazy(array)`, `Lazy.generate`, `Lazy.range`, or `Lazy.async(array)`).

- `map`
- `pluck`
- `reduce` (aka `inject` or `foldl`)
- `reduceRight` (aka `foldr`)
- `filter`
- `reject`
- `where`
- `invoke`
- `find`
- `findWhere`
- `first` (aka `head` or `take`)
- `rest` (aka `tail` or `drop`)
- `initial`
- `last`
- `sortBy`
- `groupBy`
- `countBy`
- `uniq`
- `zip`
- `concat`
- `without`
- `difference`
- `union`
- `intersection`
- `flatten`
- `compact`
- `shuffle`
- `every` (aka `all`)
- `some` (aka `any`)
- `indexOf`
- `lastIndexOf`
- `sortedIndex`
- `contains`
- `min`
- `max`

This library is experimental and still a work in progress.
