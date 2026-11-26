---
layout: post
title:  'JavaScript - Chained Promises Before & After Example'
date:   2016-03-01 07:20:58 -0400
category: blog
---
This is an experiment I worked through to understand and test chaining and exception handling with promises. This is relatively new for me. That means the example may be less than 100% perfect, but perhaps a little more accessible if this is new for you too.

The scenario is a JavaScript function to bake a cake. The ingredients – flour, eggs, sugar, and frosting – are all retrieved by Ajax calls. We request the ingredients, then mix the batter, bake it, and add the frosting. When it’s done we call `deliverTheCake()`. If it fails we call `makeExcuse()`.

Assume that any step in the process can fail, so each one needs exception handling to ensure that the error handler executes.

## Bad Example

First is the bad example. The callback for each request starts the next request. That’s how a lot of JavaScript ends up, either by design or by gradual change.

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
        console.log("The cake is delivered: " + cake);
    }

    function makeExcuse(error) {
        console.log("Sorry, can't finish the cake because: " + error);
    }

    function getCake(success, error) {
        $.ajax("/api/getFlour",
            {
                success: function (flour) {
                    $.ajax("/api/getEggs", {
                        success: function (eggs) {
                            $.ajax("/api/getSugar", {
                                success: function (sugar) {
                                    $.ajax("/api/getFrosting", {
                                        success: function (frosting) {
                                            try {
                                                var batter = mix(flour, eggs, sugar);
                                                var unfrostedCake = bake(batter);
                                                var finished = frost(unfrostedCake, frosting);
                                                success(finished);
                                            }
                                            catch (e) {
                                                error(e);
                                            }
                                        },
                                        error: function (jqXHR, status, errorThrown) {
                                            error(errorThrown);
                                        }
                                    });
                                },
                                error: function (jqXHR, status, errorThrown) {
                                    error(errorThrown);
                                }
                            });
                        },
                        error: function (jqXHR, status, errorThrown) {
                            error(errorThrown);
                        }
                    });
                },
                error: function (jqXHR, status, errorThrown) {
                    error(errorThrown);
                }
            });
    }
```

The usage would be

``` js
    getCake(deliverTheCake, makeExcuse);
```

What’s wrong with this?

*   Even though the individual steps are executed asynchronously, they’re still executed sequentially. We request the flour, wait for it, request the eggs, wait for them, etc. It would be much faster if we requested them all concurrently.
*   The code becomes a “pyramid of doom” – the sideways triangle of indentation. We could change all of the callbacks to named functions where each calls the next but that’s still hard to read, follow, and modify.

## Better Example

I won’t say “good” example because there’s undoubtedly lots of room for improvement. But this code does the exact same only better and with fewer lines using chained promises.

First, just one more function to return a promise for an Ajax call:

``` js
    function getAjaxPromise(url) {
        var deferred = Q.defer();
            $.ajax(url,{ success: deferred.resolve, error: deferred.reject})
        return deferred.promise;
    }
```

And then this is what combines the Ajax calls and the mixing and the baking. The main point is that this `getCake()` function replaces the one above and does a better job. The library I’m using is [q.js](https://github.com/kriskowal/q).

``` js
    function getCake() {
        var deferred = Q.defer();
        var getFrosting = getAjaxPromise("/api/getFrosting");
        Q.all([getAjaxPromise("/api/getFlour"), getAjaxPromise("/api/getEggs"), getAjaxPromise("/api/getSugar")])
            .spread(mix)
            .then(bake)
            .then(function (baked) {
                getFrosting.then(function (frosting) {
                    var finished = frost(baked, frosting);
                    deferred.resolve(finished);
                });
            }).catch(deferred.reject);
        return deferred.promise;
    }
```

Following the pattern, the function doesn’t return a cake. It returns a promise for a cake. The usage would be

``` js
    getCake().then(deliverTheCake).fail(makeExcuse);
```

How is this better?

*   The asynchronous steps (getting the ingredients) are all executed concurrently. Regardless of what order they return in, the mixing and baking begin as soon as the flour, eggs, and sugar are available. It doesn’t care whether the frosting is available until it’s time for that step.
*   It’s easier to read (once you get used to the syntax.)
*   There are far fewer lines of code and the main function only has one line for exception handling.

It’s also much easier to modify.

*   The `bake()` function can become asynchronous and return a promise and the `.then(bake)` line doesn’t change at all. In the previous example we’d have to add more callbacks and another level of indentation. That’s one more brick on our pyramid of doom.
*   What if we want to request frosting from two sources, use whichever comes first, and ignore the other? In the first example that would be so complicated we wouldn’t even contemplate it. In the second it’s easy. We could have multiple promises or an array of promises for frosting. This executes as soon as any one of them is fulfilled:

``` js
        Q.any(arrayOfFrostingPromises).then(function(frosting){…})
```

*   What if we want the whole thing to fail after 10 seconds if the cake isn’t delivered? Again, that’s difficult in the first example. In the second example we wouldn’t change the getCake() function at all. We’d change the function call to

``` js
        getCake().timeout(10000).then(deliverTheCake).fail(makeExcuse);
```