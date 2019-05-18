---
layout: post
title: "LZ4 decompression in pure PHP"
date: 2019-05-18 15:18:21 +0200
tags: php
---

For a PHP project I am working on, I need to access some data compressed using LZ4. Since I couldn't find the code I was looking for online, I decided to write my own. The result is just around 35 lines of code.

# The Code

* Parameters
  * *string* `$in`: Compressed data (binary).
  * *int* `$offset`: Start of first sequence (skip headers).
* Return Value
  * Returns the decompressed *string*.

```php
function lz4decode($in, $offset = 0) {
	$len = strlen($in);
	$out = '';
	$i = $offset;
	$take = function() use ($in, &$i) {
		return ord($in[$i++]);
	};
	while ($i < $len) {
		$token = $take();
		$nLiterals = $token >> 4;
		if ($nLiterals === 15) {
			while (true) {
				$nLiterals += $add = $take();
				if ($add !== 255) break;
			}
		}
		$out .= substr($in, $i, $nLiterals);
		$i += $nLiterals;
		if ($i === $len) break;
		$offset = $take() | $take() << 8;
		$matchLength = ($token & 0xF) + 4;
		if ($matchLength === 19) {
			while (true) {
				$matchLength += $add = $take();
				if ($add !== 255) break;
			}
		}
		$j = strlen($out) - $offset;
		while ($matchLength--) {
			$out .= $out[$j++];
		}
	}
	return $out;
}
```

# The Specs

The LZ4 specifications can be found in the [official documentation](https://github.com/lz4/lz4/blob/master/doc/lz4_Block_format.md).

![LZ4 Bytes Diagram](/res/LZ4_bytes.png)

The graphic above shows the bytes within a *sequence*. A LZ4 compressed *block* contains one or more sequences. A sequence contains new bytes to add to the output (literals included in the sequence) and a copy instruction (to output previous bytes again).

The first byte starts with a *token*, which consists of two 4-bit nibbles. The first (high) nibble contains the number of literals `nLiterals`. If the nibble is "full" (maximum value), the value "swaps over" to a new byte. As long as the nibble or the following bytes contain their maximum value, the next byte is also part of the sum.

The token and (optional) `nLiteral` summands are followed by the given number of byte literals, which are added to the output buffer. 

The copy instruction starts with a 16-bit little endian `offset`, indicating how far back in the output buffer we have to go to find a match for the next output. That value is followed by the overflow summands of the `matchLength` value, which is constructed like `nLiterals`, but with an additional constant summand of 4 (`minmatch`). The match is copied to the output buffer.

This process repeats up to the last sequence, which ends with a literal and contains no copy instruction.

```
// Example nLiterals overflow
20: (15, 0), [5]
525: (15, 0), [255, 255, 0]
// Example sequence & output
(4, 7), [1, 2, 3, 4], [2] => [1, 2, 3, 4], [3, 4, 3, 4, 3, 4]
```

# Annotations

```php
$take = function() use ($in, &$i) {
	return ord($in[$i++]);
};

```

This [anonymous function](https://www.php.net/manual/en/functions.anonymous.php) is used to get the next byte from the input sequence. The current byte sequence index `$i` from the outer scope is captured (or inherited) by-reference, so the advancement of the byte sequence persists.

This concept is inspired by Node.js, but works very well in PHP (even though it looks a bit unusual).

```php
$nLiterals = $token >> 4;
if ($nLiterals === 15) {
	while (true) {
		$nLiterals += $add = $take();
		if ($add !== 255) break;
	}
}
```

Get the high nibble for the `nLiterals` value. Add additional bytes when necessary.

```php
$out .= substr($in, $i, $nLiterals);
$i += $nLiterals;
```

Copy the literals to the output buffer. Adjust the index accordingly, since we didn't use `$take()`.


```php
if ($i === $len) break;
```

If the input ends, it is here (last sequence has no copy instruction). The `$i < $len` condition of the `while` statement should never be `false`, but is left in place for improved error recovery from invalid input.

```php
$offset = $take() | $take() << 8;
```

A poor man's `unpack` to read a little endian `uint16_t`.


```php
$matchLength = ($token & 0xF) + 4;
if ($matchLength === 19) {
	while (true) {
		$matchLength += $add = $take();
		if ($add !== 255) break;
	}
}
```

Then `matchLength` is calculated analogously to `nLiterals`, but with the low nibble and the minimum value of `4` added at the beginning.

```php
$j = strlen($out) - $offset;
while ($matchLength--) {
	$out .= $out[$j++];
}
```

`matchLength` bytes are copied from earlier in the buffer. Instead of using `substr`, we copy individual bytes using a loop. This allows common "looping" patterns like repeating the last byte many times by using an offset of `1`.

At this place, the processing of the sequence is completed, and we can move to the next one. I used to insert an echo statement to gain insights into the algorithm during development:

```php
echo "nLiterals: {$nLiterals}, offset: {$offset}, matchlength: {$matchLength}\n";
```

Side note: I generally try to avoid binary operations like `$take() | $take() << 8` when writing higher level code. It is hard to see what they do, and errors are often not immediately recognizable. I recommend moving them to appropriately named functions. That way, the desired effect can be understood without analyzing the implementation, and the implementation can be verified and tested in isolation. In this code however I prioritized compactness.

# License

If you want to use my code in your own project, please add the following note:

```
MIT License

Copyright (c) 2019 Stephan J. Müller

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

I am also considering uploading the project to composer. If you are interested, please let me know.
