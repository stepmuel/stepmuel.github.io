---
layout: post
title: "A Simple Helper Function to get Default Values in PHP"
date: 2015-12-19 23:23:23 +0100
tags: php coding
---

Going back to PHP after a recent Python project, I was bothered by it's handling of undefined keys in dictionaries. A quick google search revealed that [I was not the only one](http://stackoverflow.com/questions/6696425/is-there-a-better-php-way-for-getting-default-value-by-key-from-array-dictionar/). Since I didn't like any of the existing workarounds, I decided to create my own solution. 

# Problem

Most modern programming languages provide convenient ways to handle undefined keys in dictionaries. In Objective C for example, accessing the element for an undefined key of an `NSDictionary` will return `nil` without causing any problems. Not having to make the dictionary access conditional can improve code readability and helps to avoid deeply nested if statements. 

While in Python an undefined dictionary key will raise a `KeyError` when using the bracket operator, the `get(key[, default])` method will simply return `None`, similar to Objective C. The optional `default` argument even allows to specify the value to return in case the key doesn't exist, which can often replace additional sanity checks. 

Unfortunately, none of those conveniences are available in PHP. There are a couple of tricks to achieve similar results, but none of them are particularly elegant.

# Solutions

```php?start_inline=1
// Solution 1: The standard way
if (array_key_exists($key, $dict)) {
  $foo = $dict[$key];
} else {
  $foo = null;
}

// Solution 2: The short way
$foo = isset($dict[$key]) ? $dict[$key] : null;

// Solution 3: The hacky way
$foo = @$dict[$key];
```

Solution 1 is the way it probably should be done. It behaves the same as Python's get method and `null` can be replaced by a default value. But it takes a lot of typing and understanding the code requires following the control flow. 

Solution 2 is much simpler. But it's still a pain to type, especially since `$dict[$key]` appears twice. Think `$longDictName[$keys['myKey']]`. `null` can be replaced by a default value, but `isset` returns `true` for the value `null`, which might lead to unexpected results. 

Solution 3 uses the `@` error control operator to suppress the any error messages by accessing an undefined element. If the key is undefined, `$foo` will be set to `null`. Not using the `@` will provide the same result, but generate a notice. Unfortunately, the `@` operator disables errors for the whole statement. So `$foo = @$dict[$undefined]` will not generate a warning about the use of an undefined variable. 

# Convenience

Since in PHP dictionaries are arrays and arrays are not objects, we can't define a get method. But we can define a global function to make solution 1 easier to use.

```php?start_inline=1
function array_get($dict, $key, $default=null) {
	return array_key_exists($key, $dict) ? $dict[$key] : $default;
}
```

Wonderful! But can we make it even simpler? The function has 3 arguments and their order has to be remembered. The name could be shorter, but since the function only works with arrays, this should be reflected somehow. 

```php?start_inline=1
function get(&$var, $default=null) {
    return isset($var) ? $var : $default;
}
```

`get` is based on solution 2 and solves both problems. The function is no longer limited to arrays so we can shorten the name. It has only 2 arguments, one of them optional. The functionality of `array_get` can be approached with `get($dict[$key], $default)`. `get($dict[$key])` is almost as simple as solution 3. But how does it work?

# Passing by Reference

The `&` in front of the `$var` argument causes the variable to be passed by reference. This feature is mostly used to allow a function to edit the content of a passed variable. But it can also be used to return additional values. [preg_match](http://php.net/manual/en/function.preg-match.php) for example returns the `$matches` array by reference. That's why using an undefined variable as by reference argument is totally valid and doesn't produce an error. 

# Discussion

The `get` helper function can make developing in PHP a little bit quicker and less error prone. While I try to avoid defining global functions in larger projects, I often use it in prototypes or simple scripts. 

```php?start_inline=1
// `get` usage examples
$test = array('foo'=>'bar');
get($test['foo'],'nope'); // bar
get($test['baz'],'nope'); // nope
get($test['spam']['eggs'],'nope'); // nope
get($undefined,'nope'); // nope
```

Unfortunately, using the `get` function will define and set the variable to `null` if it hasn't been defined before. `isset` will still return `false` and method 2 and 3 will still work afterwards, but `array_key_exists` will return `true`. This is usually not a problem but can lead to unexpected results.  

```php?start_inline=1
// Running the previous code left some residue:
json_encode($test); // {"foo":"bar","baz":null,"spam":{"eggs":null}}
$undefined===null; // true (no error; got defined by passing it to get)
isset($undefined) // false
get($undefined,'nope'); // nope
```

I hope you enjoyed the read! 
