---
layout: post
title: "JSON Compression"
date: 2016-06-19 13:45:21 +0100
tags: coding
---

I few months ago, I ran out of space on the server I use to collect data for the HeapCraft project. This led me to look into data compression. But the thing about exploring lossless compression is that you might just end up with the same data you started with. 

# How did I end up with that much data?

The [HeapCraft](http://heapcraft.net/) project records gameplay from Minecraft servers and sends it to a logging server for statistical evaluation. While building its infrastructure, I decided to use JSON to encode individual game events. Those events are collected for 20 seconds, compressed, and then sent to the server. In my [master's thesis](http://heapcraft.net/img/collaboration-thesis.pdf) (Appendix A) I describe some of the reasons I decided to use JSON:

> Using JSON provides great flexibility for adding new event properties. Being human readable facilitates debugging and data introspection. It is also widely supported among different programming languages. Since our data contains a lot of duplicate strings (keys, event names, player ids, material names, …) it compresses very well. The compression ratio was constantly around 10, which makes the communication protocol comparable to one based on binary data.

The server would then decompress the stream of JSON objects and append them, line by line, to a simple text file; one file per server and day. This worked pretty seamlessly for about a year, until we started to get data from more popular servers. One server produced several gigabytes worth of logs per day. And after a few weeks, the server ran out of disk storage. 

# Crunching Data

In order to keep the system running, I started to `gzip` old files, only leaving the to ones for the current day uncompressed. The HeapCraft server does some data processing to give the server admins real-time access to some statistics and player position heat maps. The data processing jobs can be paused and resume at a previously saved position in the log file, once new data is available. However, in some cases (e.g. when adding new data processing jobs), the old log files have to be processed again. I implemented a scheduler that would in that case give each job 1 second of processing time in a round robin manner. 

Unfortunately, the `seek` call to move the log file pointer to the right position to resume ended up taking more than 1 second towards the end of large compressed files. Simply skipping all previous data doesn't work, since the compression algorithm `gzip` uses relies on previous data and has to more or less decompress all data up to the desired position in the file. That's when I realized that I might have to fundamentally rethink my data storage strategy. 

# Duplicate String Elimination

One way most (if not all) lossless compression algorithms reduce data size, is by finding duplicate data and representing it in a more space efficient way. A simple way to achieve this is by using back-references. The very fast [LZ4](https://en.wikipedia.org/wiki/LZ4_(compression_algorithm)) algorithm for example would compress the word `banana` to something like `ban[-2,3]`, telling a decompressor to replace the brackets with 3 chars starting 2 chars before the brackets. 

The biggest chunk of data in our log files consists of a relatively small set of strings that are repeated many times. For demonstration, I will use a log file including 40004 events at a size of 9.6 MB. Almost 90% of the events are of the type `PlayerMoveEvent`, which will be generated at up to 20 times per second for a moving player. 

	{
	  "event": "PlayerMoveEvent",
	  "player": "uuuuuuuu-uuuu-uuuu-uuuu-uuuuuuuuuuuu",
	  "worldUUID": "uuuuuuuu-uuuu-uuuu-uuuu-uuuuuuuuuuuu",
	  "time": 1425697333615,
	  "x": -24.801777929768868,
	  "y": 70,
	  "z": 47.30000001192093,
	  "pitch": 8.100057,
	  "yaw": -208.02672
	}

The UUIDs have been anonymized and white spaces are added for readability. Excluding the newline character, this event uses 233 bytes; 122 bytes are used for strings, 68 for numbers and 43 for structure characters like quotes, commas and colons. 

The strings inside a JSON object can be extracted by a simple regular expression. (A lookbehind assertion could be added to support escaped double quotes.)

```php?start_inline=1
$dic = array();
$key = 1;
$out = preg_replace_callback('/"(.*?)"/', function ($m) use (&$dic, &$key) {
  $str = $m[1];
  $id = @$dic[$str];
  if ($id===null) {
    $id = $key++;
    $dic[$str] = $id;
  }
  return "@$id";
}, $in);
```

This code takes all strings (including their surrounding quotes) from the log file and replaces them with the index for a string lookup table. If the string doesn't exist in the table, it will be added. The table can be saved in some kind of key-value store, or the first occurrences of each string could be left untouched to mimic a back-reference. The result will look something like this:

	{
	  @1: @128,
	  @116: @18
	  @112: @46,
	  @21: 1425697333615,
	  @35: -24.801777929768868,
	  @36: 70,
	  @37: 47.30000001192093,
	  @38: 8.100057,
	  @39: -208.02672,
	}

The new file has a size of 5.1 MB, resulting in a compression ratio of 1.88. I'm not counting the size of the string table, since it is quite small and the same table can be used to compress several files. (From the 1'969'380 strings in the file, only 1'768 are unique, having a combined size of just 5'961 bytes.)

Notice how larger numbers require more bytes. The compression ratio could be increased further by reordering the table such that smaller numbers are used for the most common strings. This is the basic principle of [Huffman coding](https://en.wikipedia.org/wiki/Huffman_coding) which is used in many compression algorithms, but I won't investigate that idea any further at this time. 

# Down the Rabbit Hole

The `gzip` compressed file is only 1.1 MB in size (compression ratio: 8.7), which means we still have a long way to go. The largest amount of space is now occupied by numbers, so let's replace them also!

	{s:s,s:s,s:s,s:i,s:f,s:f,s:f,s:f,s:f}

The strings are now `s`, integers `i` and floats `f`. Now the bottleneck seems to be punctuation mark. Let's replace the `s:` with `k` to represent keys. `true` and `false` can be `y` for 'yes' and `n` for now. And commas in JSON are redundant anyway, so let's just remove them. And voilà!

	{kskskskikfkfkfkfkf}

Simple, beautiful, yet still somehow human readable. And of course it also works for more complex objects:

	{ksksk{kikiki}ksk[sss]kiksks}

And the new compression ratio? 11.21. Nice! And you know what? Since those structure tokens are by themselves just a small number of different strings, we can handle them like our other strings and replace them with an index for our string table. Boom!

	42

# Sobering Up

Before implementing our genius compression algorithm, there is one small problem left so solve: how do we store our data? While we can probably put all strings into an external key-value store and neglect its impact on size, most numbers won't be repeating too often. We could append our values to the structure string in binary, e.g. for `[iif]`, just interpret the following bytes as two integers and one float, then replace the corresponding letters with their actual value. 

	[structure_string_key] [string_key|integer|float]*

So how much space would this need? Our dataset contains 364'518 keys, 127'827 strings, 78'260 integers, 155'321 floats, 1'229 booleans and 40'004 structure strings. Assuming we can represent all values with 32 bit, this adds up to 2.93 MB. This basically means that the biggest compression ratio we can get by using such a binary format is 3.28. 

Of course, we can use fewer than 4 bytes to store integers in most cases. Let's assume 16 bits are enough for all integers and string indices; still over 1.5 MB, which is quite a bit worse than pure `gzip` compression. 

It seems like a significant reduction in data size will only be possible by using some kind of prefix code, e.g. using fewer bits for common strings and small integers. However, we have already crossed a point where the dangers of having a complex storage format might out-weight the benefits of saving storage. Coding errors might lead to data loss and everything will be obscured by a format nobody understands. 

Relying on a well-established data compression format like the one created by `gzip` is probably the way to go. In the end, I rewrote the data processing scheduler to process compressed files as a whole without time limit to avoid expensive `seek` operations. The server which caused our storage shortage in the first place stopped sending us data, and we might soon migrate to a larger server. 

# Take Away Points

Before trying to reduce the space used by duplicate strings, I could have asked myself: how realistic is it to represent the `PlayerMoveEvent` with less than 30 bytes, like `gzip` does? Using 32 bits for all the floats and integers already uses 24. I already did some calculations to compare compressed JSON to a binary format for my master thesis, but must have forgotten the results. 

On the other hand, I like to code, and sometimes it's just more fun to test an idea by implementing it instead of figuring out if it makes sense. In the end, I added some new knowledge and a few interesting concepts to my tool belt, and have some stuff to write about in my blog. 

Writing code to decompress the strings I produced is left as an exercise for the reader. 



