---
layout: post
title:  'Second Look at Promises using Native ES6 vs. q.js'
date:   2016-03-03 07:20:58 -0400
category: blog
---
When I first began using promises it was with the [q.js](https://github.com/kriskowal/q) library. Now the latest version of JavaScript - ES6 - has [native support](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) for promises. So I rewrote a [previous example](/blog/2016/03/01/chained-promises-before-and-after.html) using native promises to see how that would work.

The example (for better or worse) is a function for baking a cake. It requires four ingredients: flour, eggs, sugar, and frosting. It “obtains” those ingredients via `$.ajax` calls. In the bad example (without promises) it creates a small-scale “callback heck.” (Family blog.) The callback for each `$.ajax` call makes the next `$.ajax` call. Not only does it make some ugly code (pyramid of doom) but it means that the calls are done sequentially. It would be quicker if we made all the calls at once (getting all of the ingredients) and baked the cake as soon as the last ingredient arrived.

You can [look back at that post](/blog/2016/03/01/chained-promises-before-and-after.html) to see the bad example. With error handling it’s about 30 lines, and it would be very difficult to modify. Here’s the same thing with ES6 promises. It’s much shorter, easier to read, and even a little bit simpler than the same thing done with q.js.

``` js
    function mix(flour, eggs, sugar) {
      return flour + "," + eggs + "," + sugar + "_mixed";
    }

    function bake(batter) {
      return batter + "_baked";
    }

    function frost(cake, frosting) {
      return cake + "_frosted_with_" + frosting;
    }

    function deliverTheCake(cake) {
      alert("The cake is delivered: " + cake);
    }

    function makeExcuse(error) {
      alert("Sorry, can't finish the cake because: " + error);
    }

    function getCake() {
      var getFrosting = fetch("/api/getFrosting");
      var bakeCake = Promise.all([fetch("/api/getFlour"), fetch("/api/getEggs"), fetch("/api/getSugar")])
        .then(function(ingredients) {
          return mix.apply(this, ingredients);
        })
        .then(bake);
      return Promise.all([bakeCake, getFrosting])
        .then(function(cakeAndFrosting) {
          return frost.apply(this, cakeAndFrosting);
        });
    }

    getCake().then(deliverTheCake).catch(makeExcuse);
```

The output, which is just for debugging purposes, is `The cake is delivered: flour,eggs,sugar_mixed_baked_frosted_with_frosting.` [Here’s the sample](https://jsfiddle.net/scotthannen/uxzx7bzg/9/) in a JSFiddle. I replaced `fetch` with my own mock function. It simulates the ingredients arriving in unpredictable sequence and random errors. (JSFiddle has a way of mocking api calls but I got tired of messing with it.)

## What’s Different?

The ES6 version is a little bit shorter. That’s mostly because in the q.js example I was using a deferred. ES6 promises don’t have a deferred. And as it turned out, I didn’t need it. Those were just a few extra lines of code. Here are a few differences with ES6:

### Fetch

Using q.js I had to create an additional function to return a promise for the result of a jQuery `$.ajax` call. Now I don’t need that function. I’m not even using jQuery. Instead I’m using another new ES6 function, [`fetch`](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch). It makes an HTTP request, like `XMLHttpRequest`, and returns a promise.

### No Deferred

Using q.js, one might create a deferred, return a promise for the deferred, and then resolve or reject the deferred to produce the result of the promise. Like this:

``` js
    function getAjaxPromise(url) {
      var deferred = Q.defer();
        $.ajax(url,{ success: deferred.resolve, error: deferred.reject})
      return deferred.promise;
    }
```

But ES6 promises don’t have a deferred like q.js. True, we won’t need them as often. But sometimes it’s useful to return a promise that isn’t responsible for its own fulfillment. Perhaps some other event will cause it to be fulfilled. In that case you can use this function to create a deferred, and it works just like a q.js deferred.

``` js
    Promise.deferred = function() {
      var result = {};
      result.promise = new Promise(function(resolve, reject) {
        result.resolve = resolve;
        result.reject = reject;
      });
      return result;
    };

    var myDeferred = new Promise.deferred();

    myDeferred.promise.then(function(value){alert(value);})

    //Then, at some later point:
    myDeferred.resolve("I am resolved!");
```

### No `Q.any()`

Q.any() takes an array of promises and returns a promise that resolves with whichever of those promises is resolved first. The closest thing in ES6 is `Promise.race().` It takes an array of promises and returns a promise that resolves or rejects with the first of that array that resolves or rejects. Big difference: you can’t wait for the first successful outcome.

Browsing [a blog](https://esdiscuss.org/topic/promise-any) I found this function to provide the same functionality as `Q.any().` I had to edit it just a little.

``` js
    Promise.any = function (promises) {
      return new Promise(function (resolve, reject) {
        var count = promises.length, resolved = false;
        var errors = [];
        promises.forEach(function (p) {
          Promise.resolve(p).then(function (value) {
            resolved = true;
            count--;
            resolve(value);
          }, function (e) {
            errors.push(e);
            count--;
            if (count === 0 &amp;& !resolved) {
              reject(errors);
            }
          });
        });
      });
    };
```

[Here’s a JSFiddle](https://jsfiddle.net/scotthannen/ss9fsfqv/4/) demonstrating how `deferred()` and `any()` can be used to create a promise that resolves when the first of an array of promises resolves.

### No `.spread()`

This is trivial. Using q.js, `Q.spread()` converts the results of an array of promises to a list of method arguments. In the ES6 example we can do the same thing using [`apply`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/apply).

## Conclusion

With a few minor adjustments we may not need an additional library like q.js to use promises - we can just use native ES6 promises instead. Also, there are a number of articles that refer to unnecessary deferreds as an antipattern, like [this one](https://github.com/petkaantonov/bluebird/wiki/Promise-anti-patterns). Omitting native support will help to reduce that. I found such an unnecessary deferred in my own example and I wonder how many I have in my code.
