---
layout: post
author: Matt Dickinson
title:  "Quickie: Node.js Cluster Performance"
date:   2016-04-30
tags:
  - code
  - nodejs
  - pumlhorse
  - load test
---

I’m currently working on a load test agent for [Pumlhorse](http://pumlhorse.com), which involves using the `cluster` library in Node.js. 
Most of the documentation is focused on using a cluster for HTTP server workers, but I’m using it a little differently. 
My master process is an HTTP server, but the workers are running the load test scripts.

My plan was to have a broker on the master process assign tasks to the child processes. As the inter-process communication (IPC) is non-blocking, 
this means async messages from the worker asking for work and the broker giving work. This also means that the broker might not have work for the worker until later, 
so I have to keep a queue of “hungry” workers.

I got wondering if maybe it would be easier to just keep a queue of work and spin up a new worker every time an item was dequeued. 
I knew there would be some amount of overhead in forking the process, but maybe it would be negated in favor of reducing code complexity.

It’s not.

I set up a sample application to test the two scenarios: `spawnUpFront` and `spawnOnDemand`. 
Both scenarios add a queue of 5000 items and start up a worker for every core. 
When the worker starts up, and sends two IPC requests to the master. 
Upon receiving a response to those requests, the worker then waits 30 ms and then checks if it should exit 
(yes for `spawnOnDemand`, no for `spawnUpFront`). If it receives a null response, then it exits. 
If a process exits in the `spawnOnDemand` scenario, then it starts up a new one (assuming there is still data in the queue).

I ran the `spawnUpFront` scenario first, and it ran around 19 seconds. I had to throw logging in so that on every request from a worker, it would spit out the queue size. 
That way I could watch it go from 5000 to 0, because otherwise I couldn’t tell anything was going on.

![CPU usage of upfront scenario]({{ site.contenturl }}nodejs-cluster-cpu-upfront.png)
*Note: The test is only running in the last 20 seconds*

“Well, that was pretty painless”, I thought. “I wonder what the on demand scenario is like”

![CPU usage of onDemand scenario]({{ site.contenturl }}nodejs-cluster-cpu-ondemand.png)
OH GOD NO MAKE IT STOP
And that’s not even the full duration. In all, the test took about 180 – 190 seconds…an order of magnitude above the first scenario. 
Now, this test was nowhere near scientific, but I’m comfortable saying that generating child processes on demand is **not** a viable approach.

Here’s [a gist of my test](https://gist.github.com/mdickin/03e67c263b6044848ae90e6aca0c2d6d), for anyone that is interested. 
Note that if you’re running Node.js v6 or above, you’ll need to fix the callback signature for `cluster.on(“message”, callback)`. 
Pre-v6, the documentation says it passes the worker, but it actually doesn’t.