---
layout: post
title: "Web Workers for Responsive Web Apps"
date: 2016-02-04 18:45:21 +0100
tags: javascript coding
---

Meet workerQueue.js, a queue based web worker API to simplify multithreading in JavaScript. 

# Background

I recently created an interactive data visualizer using HTML and JavaScript. It shows a heat map of player positions within a minecraft world. The map is generated from over 1 million player positions and the time a player has spent there. Creating a new map view by iterating over all data points takes about 100 ms. Since the map can be panned and zoomed, the map view needs to be updated very often. To people used to Google Maps and smartphones with very responsive direct screen manipulation, 10 frames per second feel very sluggish. Especially when they rely on visual feedback to control their zooming and panning. 

Chrome triggers around 60 events per second when dragging the map around. In order to increase responsiveness and perceived speed, I want to give immediate user feedback by just scaling and shifting the previously generated image and replace it with a newly generated picture later. But since JavaScript handles all events in a single thread, no new events will be triggered until the previous event has been dealt with. The existing map can't be scaled and translated while a new map is being rendered in the same event loop.  

First I tried to improve the frame rate by simply optimizing my code and data structures. But this would ultimately only solve my problems until I want to use a larger map. Since minecraft maps are practically infinite in size, pre-rendering the whole map isn't a viable solution either. Since the player position heat maps tend to be very sparse, this would also waste a lot of memory. Then I tried to just render some extra pixels around the visible part of the map and only update the picture when the user stops manipulating the viewport. But this led to seemingly random freezes of the user interface, which is (arguably) worse than a bad but constant frame rate. 

The obvious solution is to generate new map views in a background thread. This is also one of the first things that came to mind, but after some quick research about multithreading in JavaScript, I wasn't sure if this would be worth the effort. But ultimately, I justified the additional work by creating a generic solution that can be used in other projects also. 

# A Generic Web Worker

Multithreading in web apps is achieved using [web workers](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API) which can run in separate threads. To avoid common multithreading pitfalls, web workers have their own isolated namespaces and never share data. All communication happens by sending messages. The receiver will get its own copy of all objects referenced within a message. Some data types (transferable objects) can have their ownership transferred to the receiver instead of being copied. This saves memory and time when sending large arrays or images. Some objects like functions can't be sent to a worker because JavaScript doesn't know how to copy them. 

In addition to the separation of data, web workers also try to separate code. New web workers get initialized by passing them the path of a script file which is used as their individual global scope. This, combined with the inability to pass function, was the main reason I didn't want to use web workers initially. I want to separate my code by functionality, not by thread. 

I was able to work around those limitations by using the [data URI scheme](https://en.wikipedia.org/wiki/Data_URI_scheme) to eliminate the need for additional script files and `eval` in combination with `Function.prototype.toString` to move code to the web worker thread. 

```javascript
// message handler for generic worker
var onmessage = function(e) {
  var d = e.data;
  var func = eval(d.code);
  var transferList = [];
  var res = func(d.args, transferList);
  postMessage({res: res, id: d.id}, transferList);
};
// create worker with the given message handler
var code = "this.onmessage = " + onmessage.toString();
var blob = new Blob([code], {type: "text/javascript"});
var worker = new Worker(window.URL.createObjectURL(blob));
```

This web worker is extremely simple, yet very powerful. It executes any code it receives as a string with the given arguments and sends the result back to the sender. `transferList` enables the code to specify parts of its result as transferable by appending them to the list. The message handler can even be used to define new functions within the worker namespace.

```javascript
// define the `say` function within the worker
var setup = function() {
  this.say = function(msg) {
    console.log(msg);
  };
};
// call the defined function
var test = function(args) {
  this.say(args[0]);
};
// send both functions to the worker for execution
worker.postMessage({code: '('+setup.toString()+')'});
worker.postMessage({code: '('+test.toString()+')', args: ["hi world"]});
```

I use `toString` instead of defining the functions within a string directly in order to get syntax highlighting and proper indentation handling by text editors. But since the function is converted to a string first, setting breakpoints won't work in Chrome. 

Calling `eval` for each function call might not be the most efficient solution, but since the thread is mainly used for relatively long tasks, the additional computing time is negligible. Doing the main part of the work within functions previously defined by `this.functionName` might help the JavaScript engine to optimize the code at execution time. In the example, the `say` function is only compiled once, and then reused. Many JavaScript engines try to speed up functions that get executed many times. [Some](https://webkit.org/blog/3362/introducing-the-webkit-ftl-jit/) even compile them to native code. 

# Queues

Our generic web worker is already quite useful by itself, but packing it into a nice API could certainly make it easier to use. We also don't have a good way to handle return values yet. 

I decided to create an interface similar to Apple's [Grand Central Dispatch](https://developer.apple.com/library/ios/documentation/Performance/Reference/GCD_libdispatch_Ref/). Conceptually, there are queues where functions (blocks, code) can be added. The framework then takes those functions and executes them at some point. Either one after another or multiple in parallel. 

On iOS, there is a serial (single threaded) main queue responsible for all UI updates. To avoid blocking the UI, a long task triggered within the main queue can be sent to a background queue. When finished, the background task then adds a new task to the main queue which shows the result on the UI. Using this model, thread synchronization, semaphores and other potential headache inducing shenanigans can be avoided. I simply use the JavaScript event loop as the main queue. 

```javascript
// workerQueue interface
function workerQueue() {
  this.worker = {}; // worker prototype
  this.maxWorkers = 1;
  this.delay = 0;
  this.add = function(func, args, callback);
};
```

`add` enqueues a function to the worker queue with the given argument. The function is then executed by a web worker in the background. After the function returns, `callback` is executed back in the main event loop with the function's return value. `maxWorkers` can be increased to execute multiple functions in parallel by creating additional workers when necessary.  

Before using the queue, functions and other properties can be assigned to `worker`. They are automatically copied to the worker thread when a new worker is initialized. If the function `worker.init` is defined, it is called in the web worker context after copying all the properties. 

```javascript
var wq = new workerQueue();
// build worker prototype (optional)
wq.worker.foo = 21;
wq.worker.init = function() {
  this.foo *= 2;
};
// run a job
var job = function (args, transferList) {
  this.foo += args;
  return this.foo;
};
var callback = function(result, job) {
  console.log("foo: " + result);
};
wq.add(job, 0, callback);
```

`args` and `result` can be any object, list or value. Transferable objects can be transferred by adding them to the optional `transferList` argument using `transferList.push(object)` inside the job function. The optional `job` parameter passed to `callback` is an object containing a unique number called `id` that is incremented after every `add` call and `start` which contains the execution start date to calculate execution time. 

# Throttling

In my interactive Minecraft map visualizer, I create a worker prototype with the map data in JSON, but as a string. The string is almost 20 MB and transferring the string to the worker is much faster than doing a deep copy of the parsed object. The worker's `init` function then parses the JSON string and deletes the string representation afterwards to free up some memory. The data is now accessible by the worker as a regular object. 

I also assign the function `fetchPixels` to the prototype which takes a viewport frame and some other parameters as argument and returns a RGBA buffer array that can be put into a HTML canvas element. This function does the main part of the work and defining it in the prototype helps preserving optimizations between function calls. 

To update the map, a job is added to the worker queue which calls `fetchPixels` inside the worker thread and returns the resulting frame buffer. The buffer is added to `transferList` so it is transferred instead of copied. The job's callback then places the new frame at the appropriate position inside the map canvas. 

This whole process works really well and panning and zooming now works smoothly with about 60 FPS while missing data at the edge of the map or with better resolution is added about 10 times per second. When updating the viewport (which happens 60 times per second), I remove all pending jobs by purging the job queue. This ensures that only the most current frames are rendered when a worker becomes available. 

When moving the map at 60 frames per second, about every 10th frame will lead to an actual data update when using a single worker thread. Unfortunately, setting `maxWorkers` to 2 does not double the experienced update rate. Instead, frame 1 and 2 are being generated while frame 3 to 10 are skipped. This process keeps repeating, which leads to two content updates within one millisecond, followed by 9 milliseconds of just rescaling the old content. Or even worse, a new frame could finish processing before a previous frame, which will lead to the map jumping back and forward between previous panning positions.  

The jumping problem can be solved relatively easy. Just save `job.id` of the most recently finished job, and if a job with a higher number did finish before, discard the frame. To spread out the updates more evenly among those 60 frames, adaptive throttling can be implemented using the `delay` property. If `delay` is greater than 0, a new job is started no earlier than `delay` milliseconds after starting the previous job. 

```javascript
var wq = new workerQueue();
wq.maxWorkers = 4;
var lastID = -1;
var job = function(args) {
  console.log("ping");
}
var callback = function(results, job) {
  // enforce sequence
  if (job.id<=lastID) return;
  lastID = job.id;
  // throttling
  var dt = new Date().getTime() - job.start;
  var delay = dt / wq.maxWorkers;
  if (wq.delay==0) {
    wq.delay = delay;
  } else {
    wq.delay = 0.2*delay + 0.8*wq.delay;
  }
};
// call this onmousemove (60 FPS)
function update() {
  // remove pending jobs
  wq.queue.splice(0,wq.queue.length);
  wq.add(job, false, callback);
}
```

I'm using a nice little first order IIR lowpass filter (the term with the 0.2 and 0.8) so the system will find an update rate it can keep up with without rapidly reacting to sudden performance drops. Good thing I studied signal theory for years so I know how it is called. 

# Source Code and Demo

The source code is available on [GitHub](https://github.com/stepmuel/WorkerQueue). There is also an [interactive demo](http://heapcraft.net/mapminerdemo/) of the [Map Miner](http://heapcraft.net/?p=mapminer) data visualizer I initially created the API for. The page also contains a short user manual and the source code. The demo might take a couple of seconds to load. It only uses a single worker thread. 




