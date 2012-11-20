# Straw

Simplified realtime stream processing for Node.js

Straw helps you create a topology of connected worker nodes that
consume, process, generate and emit messages.

Each worker node is run in it's own process.

Messages are passed in and out of worker nodes as JSON.

Redis pubsub is used for message passing, but the worker nodes are
shielded from this.

There is nothing stopping a node receiving or sending outside the
topology, e.g. write to a database, fetch or listen for data.

You can also inject or receive messages by accessing Redis pubsub
directly so your topologies can play nicely with existing
infrastructure, for example having your Express server subscribe to
one of the channels and publish it out via socket.io.

## Installation

    $ npm install straw
    $ cd straw
    $ npm install -d

Run the tests (requires `grunt`):

    $ npm test

## Run some examples

    $ node examples/ping-count
   
## Usage

By convention you create your workers nodes in a folder called
`nodes`, then create the topology, passing in an object describing how
the nodes are wired up.

This example has a node that generates timestamps once a second, with
it's output going to another that counts the cumulative number of
pings.


```javascript
var straw = require('straw');
var topo = new straw.topology({
  'ping':{
    'node': __dirname + '/nodes/ping',
    'output':'ping-out'
  },
  'count':{
    'node': __dirname + '/nodes/count',
    'input':'passthru-out',
    'output':'count-out'
  }
});
```

Worker nodes extends the prototype worker node and override only the
methods needed to do their job.

```javascript
var straw = require('straw');
module.exports = straw.node.extend({
  title: 'Ping',
  timer: null,
  ops: {interval: 1000},
  initialize: function(opts, done){
    this.opts.interval = opts && opts.interval || 1000;
  },
  run: function() {
    var self = this;
    var fn = function() {
      self.ping();
    };
    this.timer = setInterval(fn, this.opts.interval);
  },
  stop: function(done) {
    clearInterval(this.timer);
    if ( done ) {
      done(false, null);
    }
  },
  ping: function() {
    this.output({'ping': new Date().getTime()});
  }
});
```

`process()` is called every time a message is passed in to a node.

Your code needs to call `output()` whenever you have a message to send
out from the node.

```javascript
var straw = require('straw');
module.exports = straw.node.extend({
  title: 'Count',
  total: 0,
  process: function(msg, done) {      
    this.total ++;
    this.output({total: this.total});    
    if ( done ) {
      done(false, {count: this.total});
    }
  }
});
```

Run the topology like this. This example subscribes to a Redis channel
to show you the outputs from the counter

    $ node examples/ping-count

Output:

```
2012-11-15 12:07:04 STARTING ping
2012-11-15 12:07:04 STARTING count
2012-11-15 12:07:04 STARTED  ping
2012-11-15 12:07:04 STARTED  count
{ count: 1 }
{ count: 2 }
{ count: 3 }
```

Press `^c` to stop.

Calling `console.log` from within a node will output timestamped
messages to the shell, showing you which node they came from.

    $ node examples/ping-count-print


```
2012-11-15 14:59:26 STDOUT   print                {"count":1}
2012-11-15 14:59:27 STDOUT   print                {"count":2}
2012-11-15 14:59:28 STDOUT   print                {"count":3}
```
    
If you make any changes to a node file it's process will be terminated
and respawned. This is really handy in development. try running the
ping-count-print example, edit `examples/nodes/print/index.js` (just
add a space somewhere) then save it. You will see output in the log
letting you know it's been stopped and restarted.

    
## Topology

Each worker node must be defined in the topology as such:

```javascript
'<your-key>':{
    'node': '<absolute-path-to-node>',
    'input':'<redis-channel-in>',
    'output':'<default-redis-channel-out>',
    'outputs': {
        'named-output':'<redis-channel-out>',
        'another-named-output':'<another-redis-channel-out>'
        },
    'log': '<file-to-log-output-to>'
}
```

`log` and `outputs` are optional. All other fields are required.

`STDOUT` from the node (e.g. `console.log`) will be captured to the log.

You must define named outputs in your Topology before using them in
the node.

Any other fields will be passed in to the worker node as options for
it to use as it sees fit.

### Options

You can pass options in to the Topology that will be passed in to all
node runners. These let you set the Redis host and enable StatsD.

```javascript
var straw = require('straw');
var topo = new straw.topology({
  'ping':{
    'node': __dirname + '/nodes/ping',
    'output':'ping-out'
  },
}, {
  redis: {
    host: '127.0.0.1',
    port: 6379
  },
  statsd: {
    prefix: 'straw',
    host: '127.0.0.1',
    port: 8125
  }
});
```

If no options are provide, or redis is not provided, the default shown
above will be used/

If StatsD is provided, all node inputs and outputs (summed, and by
name) will be counted using `node-statsd` `increment()`, using the
node's key as the identifier.

If you provide a prefix, it will be prepended to the nodes's key so
you can namespace your stats.

## Node

These methods can/must be overridden depending on the required
functionality of your node;

```javascript
module.exports = straw.node.extend({
    title: 'Human readable name',
    initialize: function(opts) {
        // process incoming options (from the topology definition
        // and set up anything you need (e.g. database connection).
    },
    process: function(msg, done) {
        // process an incoming message. msg will be JSON.

        // this example just passes thru msg. normally you would do
        // some work on it here.
        // ...
        // and send it via the default output

        this.output(msg);

        // or send it via a named output. The name must be configured
        // in your topology
        this.output('named-output', msg);
    },
    run: function() {
        // start some background processing here e.g. fetch or
        // generate data
    },
    stop: function() {
        // stop background processing. will be called when terminating.
    }
});
```

### Stats

Nodes accumulate counts of messages emitted. You can use the count
method to count arbitrary values also.

```javascript
this.count();
this.count('some-key');
this.count('some-key', howmany);

this.counts(); // {"messages": 5, "some-key":4}
this.counts("some-key"); //
```

## Installing as a service

Once you have your topology tested and working, you'll probably want
to install it as a service.

Place this something like `/etc/init/myapp.conf`. The path to your
node binary may be different, particularly if you are using nvm.

```
#!upstart
description "My Topology"
author      "simon"

start on startup
stop on shutdown

script
    export HOME="/path/to/app"
    echo $$ > /var/run/my-app.pid
    exec sudo -u sysop /usr/bin/node /path/to/app/myapp.js >> /var/log/app/myapp.log 2>&1
end script
```

     $ sudo service myapp start

## Issues

* no guaranteed delivery of messages
* no round-robin style processing across multiple instances of the same node

## Nice to have

* pipes between nodes abstracted out
* create and change topologies dynamically
* use lists and blocking gets instead of pubsub
* run workers on remote hosts

## Thanks

Straw takes some inspiration from
[Storm](https://github.com/nathanmarz/storm) and
[Max/MSP](http://cycling74.com). It uses some code and concepts from
[Backbone](http://backbonejs.org) for the node definitions and event
handling.

## Release History

15/11/2012 Initial release

## License
Copyright (c) 2012 Simon Swain  
Licensed under the MIT license.