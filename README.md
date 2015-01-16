rsmq-worker
============

[![Build Status](https://secure.travis-ci.org/mpneuried/rsmq-worker.png?branch=master)](http://travis-ci.org/mpneuried/rsmq-worker)
[![Build Status](https://david-dm.org/mpneuried/rsmq-worker.png)](https://david-dm.org/mpneuried/rsmq-worker)
[![NPM version](https://badge.fury.io/js/rsmq-worker.png)](http://badge.fury.io/js/rsmq-worker)

RSMQ helper to simply implement a worker around the message queue.

## Install

```sh
  npm install rsmq-worker
```

## Initialize

```js
  new RSMQWorker( queuename, options );

```

**Config** 

- **queuename**: *( `String` required )* The queuename to pull the messages
- **options** *( `Object` optional )* The configuration object
	- **options.interval**: *( `Number[]` optional; default = `[ 0, 1, 5, 10 ]` )* An Array of increasing wait times in seconds
	- **options.maxReceiveCount**: *( `Number` optional; default = `10` )* Receive count until a message will be exceeded
	- **options.invisibletime**: *( `Number` optional; default = `30` )* A time in seconds to hide a message after it has been received.
	- **options.autostart**: *( `Boolean` optional; default = `false` )* Autostart the worker on init
	- **options.customExceedCheck**: *( `Function` optional; )* A custom function, with the raw message *(see message format)* as argument to build a custom exceed check. If you return a `true` the message will not exceed. On return `false` the regular check for `maxReceiveCount` will be used.
	- **options.rsmq**: *( `RedisSMQ` optional; default = `null` )* A allready existing rsmq instance to use instead of creating a new client
	- **options.redis**: *( `RedisClient` optional; default = `null` )* A allready existing redis client instance to use if no `rsmq` instance has been defiend 
	- **options.redisPrefix**: *( `String` optional; default = `""` )* The redis Prefix for rsmq if  no `rsmq` instance has been defiend 
	- **options.host**: *( `String` optional; default = `"localhost"` )* Host to connect to redis if no `rsmq` or `redis` instance has been defiend 
	- **options.host**: *( `Number` optional; default = `6379` )* Port to connect to redis if no `rsmq` or `redis` instance has been defiend 
	- **options.options**: *( `Object` optional; default = `{}` )* Options to connect to redis if no `rsmq` or `redis` instance has been defiend 

**Example:**

```js
  var RSMQWorker = require( "rsmq-worker" );
  var worker = new RSMQWorker( "myqueue" );

  worker.on( "message", function( msg, next, id ){
  	// process your message
  	next()
  });

  worker.start();
```

## Raw message format

A message ( e.g. received by the event `data` or `customExceedCheck` ) contains the following keys:

- **msg.message** : *( `String` )* The queue message content. You can use complex content by using a stringfied JSON.
- **msg.id** : *( `String` )* The rsmq internal messag id
- **msg.sent** : *( `Number` )* Timestamp of when this message was sent / created.
- **msg.fr** : *( `Number` )* Timestamp of when this message was first received.
- **msg.rc** : *( `Number` )* Number of times this message was received.

## Methods

### `.start()`

If you haven't defined the config `autostart` to `true` you have to call the `.start()` method.

**Return**

*( Self )*: The instance itself for chaining.

### `.del( id )`

Helper function to simply delete a message after it has been processed.

**Arguments**

* `id` : *( `String` required )*: The rsmq message id.

**Return**

*( Self )*: The instance itself for chaining.

### `.stop()`

Stop the receive interval.

**Return**

*( Self )*: The instance itself for chaining.

### `.send( msg [, delay ] )`

Helper function to simply send a message in the configured queue.

**Arguments**

* `filename` : *( `String` required )*: The rsmq message. In best practice it's a stringified JSON with additional data.
* `delay` : *( `Number` optional; default = `0` )*: The message delay to hide this message for the next `x` seconds.

**Return**

*( Self )*: The instance itself for chaining.

## Events

### `message`

Main event to catch and process a message.
If you do not set a handler for this Event nothing wohld be happen.

**Example:**

```js
worker.on( "message", function( message, next, msgid ){
	// process message ... 
	next();
});
```

**Arguments** 

- **message** : *( `String` )* The queue message content to process. You can use complex content by using a stringfied JSON.
- **next** : *( `Function` )* A function you have to call when your message has been prcessed.  
  **Arguments** 
  * `delete`: *( `Boolean` optional; default = true )* It's possible to prevent the worker from autodelete the message on end. This is useful if you want to pop up a message multiple times. To implement this, please check the config `options.customExceedCheck`
- **msgid** : *( `String` )* The message id. This is useful if you want to delete a message manually.

### `ready`

Fired until the worker is connected to rsmq/redis and has been initialized with the given queuename.

### `data`

The raw event when a message has been received.

**Arguments** 

- **msg** : *( `String` )* The raw rsmq message. ( See section Raw message format )

### `deleted`

Fired after a message has been deleted.

**Arguments** 

- **id** : *( `String` )* The rsmq message id

### `exceeded`

Fired after a message has been exceeded and immediately will be deleted.

**Arguments** 

- **msg** : *( `String` )* The raw rsmq message. ( See section Raw message format )

## Advanced example

This is an advanced example showing some features in action.

```js
	var fs = require( "fs" );
	var RSMQWorker = require( "rsmq-worker" );

	var fnCheck = function( msg ){
		// check function to not exceed the message if the content is `createmessages`
		if( msg.message === "createmessages" ){
			return true
		}
		return false
	}

	var worker = new RSMQWorker( "myqueue", {
		interval: [ .1, 1 ],				// wait 100ms between every receive and step up to 1,3 on empty receives
		invisibletime: 2,						// hide received message for 5 sec
		maxReceiveCount: 2,					// only receive a message 2 times until delete
		autostart: true,						// start worker on init
		customExceedCheck: fnCheck	// set the custom exceed check
	});

	//
	worker.on( "message", function( message, next, id ){
		console.log( "message", message )
		if( message === "createmessages" ){
			next( false )
			worker.send( JSON.stringify( { type: "writefile", filename: "./test.txt", txt: "Foo Bar" } ) );
			worker.send( JSON.stringify( { type: "deletefile", filename: "./test.txt" } ) );
			return	
		}

		var _data = JSON.parse( message )
		switch( _data.type ){
			case "writefile": 
				fs.writeFile( _data.filename, _data.txt, function( err ){
					if( err ){
						console.error( err )
					}else{
						next()
					}
				});
				break;
			case "deletefile": 
				fs.unlink( _data.filename, function( err ){
					if( err ){
						console.error( err )
					}else{
						next()
					}
				});
				break;
		}
		
	});

	worker.send( "createmessages" );
```

## Todos/Ideas

> Currently no feature ideas

## Release History
|Version|Date|Description|
|:--:|:--:|:--|
|0.0.1|2015-1-14|Initial commit|

## The MIT License (MIT)

Copyright © 2013 Mathias Peter, http://www.tcs.de

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the “Software”), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED “AS IS”, WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.