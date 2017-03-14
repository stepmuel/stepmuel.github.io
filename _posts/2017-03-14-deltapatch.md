---
layout: post
title: "DeltaPatch: Minimalistic JSON Patches"
date: 2017-03-14 16:50:19 +0100
tags: json coding
---

While developing a web app some time ago, I needed a way to send incremental data store updates to the clients. The DeltaPatch format is my solution to that problem. It is simple, lightweight and human readable. It does not support `null`, but all other data types supported by JSON.

# Basics

A *patch* is a set of instructions describing how to turn one thing into something else. In the context of JSON objects, a patch `p` should be able to transform a known set of data `a` into the desired set of data `b`. Since a patch only has to contain the difference between the two objects, it is often very small and can be used in cases where transferring or storing the raw data would not be feasible.

To work with patches, we define the functions `diff` and `patch`. They work similarly to the programs provided by [GNU Diffutils](https://www.gnu.org/software/diffutils/) with the same name, but are applied to JSON objects instead of files.

`p = diff(a, b)` generates a *patch* from objects `a` to object `b`. (A *patch* might also be called the *difference*, *diff* or *delta*.) If the patch is applied to `a`, it will produce `b`: `b = patch(a, p)`.


# Merging Dictionaries

Many programming languages provide functions for adding (or replacing) key/value pairs from one dictionary to another. In Python, this is done using the member function [update](https://docs.python.org/2/library/stdtypes.html#dict.update), PHP calls it [array_merge](http://php.net/manual/en/function.array-merge.php) and the JavaScript community seems to prefer the name [assign](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/Object/assign).

The principle of those functions is very simple:

* `merge(target, source)` adds all key/value pairs from source to target, while overwriting existing keys.

With some small additions, a new function `patch(target, source)` can be defined to efficiently change any dictionary into any other dictionary:

* **deleting values**: If a key has a value of `null`, it will be removed from the target if present, or ignored otherwise.
* **recursion**: If the existing value and the new value are both dictionaries, the old value is not overwritten, but patched.

That's it. Our new patch format:

    // the initial data store
    var s = {a: "a", b: false, c: 36, d: {a: "a", b: false}};
    // the delta
    var d = {
      b: null, // remove key
      c: 37, // change value
      d: {b: null}, // patch dictionary
      e: true // add key
    };
    // result of patch(s, d)
    {a: "a", c: 37, d: {a: "a"}, e: true};

# Pitfalls

* `null` can not be used as a value. Generally missing keys can be assumed to have a value of `null`, but if your existing application relies on distinguishing between `null` and `undefined`, this might be a problem. This compromise was needed to keep the format as simple as possible.
* Arrays do not merge. New arrays always replace old values. Trying to merge arrays would create new problems and make the format more complicated. For that reason, arrays should be kept shallow (e.g. not contain large objects, since they will be resent after every change).
* Dictionaries should generally be copied/cloned before being assigned to the source to avoid the target affecting the source down the line (and vice versa).
* When assigning dictionaries from a source, all `null` values should be removed. The target should never contain `null` values.

# Implementation

A JavaScript implementation of DeltaPatch can be found in the [deltalib](https://github.com/stepmuel/deltalib) repository on github in `lib/utils.js`. If you want to implement the protocol yourself, the following pseudo code snippets might be helpful.

    # helpers
    function isMergeable(v)
    function clone(v)
    function cloneWithoutNull(v)
    function empty(o)
    function equals(a, b)

`isMergeable(v)` returns whether `v` is a dictionary.

`clone(v)` returns a deep clone of `v`.

`cloneWithoutNull(v)` returns a deep clone of `v` while removing `null` values from all dictionaries.

`empty(v)` returns whether `v` is an empty dictionary `{}`.

`equals(a, b)` returns wether `a` and `b` represent the same data (deep compare).

    function patch(d, p) {
      for (var k in p) {
        var v = p[k]
        if (v === null) {
          delete d[k]
        } else if (isMergeable(v) && isMergeable(d[k])) {
          patch(d[k], v)
        } else {
          d[k] = cloneWithoutNull(v)
        }
      }
    }

`patch(d, p)` applies patch `p` to mergeable `d`.

    function diff(objA, objB) {
      var out = {}
      var keys = <all keys either in objA or in objB>
      for (var key in keys) {
        var a = objA[key]
        var b = objB[key]
        if (typeof a == 'undefined') {
          out[key] = b
        } else if (typeof b == 'undefined') {
          out[key] = null
        } else {
          if (isMergeable(a) && isMergeable(b)) {
            var d = diff(a, b)
            if (!empty(d)) {
              out[key] = d
            }
          } else {
            if (!equals(a, b)) {
              out[key] = b
            }
          }
        }
      }
      return out
    }

`diff(a, b)` will return a patch which when applied to `a` will result in `b`.

# Alternatives

[JSON Patch](http://jsonpatch.com/) specifies a more complex patch format which can achieve the same task as DeltaPatch. Unlike DeltaPatch, it supports `null` values and provides additional operations like `copy` (which can make some patches much smaller) and `test` (to avoid applying a patch to the wrong data). JSON Patch is already a somewhat widely-used standard, and worth considering if the simplicity and compactness of DeltaPatch isn't required.
