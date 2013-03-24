---
title: Welcome to Oasis
---

<div class="blurb">
  <p>
    Oasis.js helps you structure communication between multiple
    <strong>untrusted sandboxes</strong>. It builds on the APIs
    in HTML5, backporting them to older browsers when necessary.
  </p>
  <p>
    Oasis.js uses the concepts in <strong>capability-based
    security</strong> to help you safely expose capabilities
    and data to untrusted code, secure in the knowledge that
    the sandboxes can only communicate with each other and the
    outside world through those capabilities.
  </p>
</div>

<h2>Getting Started</h2>

First, create a service that you would like to expose:

```javascript
var PingService = Oasis.Service.extend({
  
});
```

Next, create a sandbox with access to that service. The sandbox's
URL should be a JavaScript file. Oasis.js will take care of creating
an iframe (or WebWorker) and executing the JavaScript inside of it.

```javascript
Oasis.createSandbox('pingpong.js', {
  services: {
    ping: PingService
  }
});
```

This will give the sandbox access to the `PingService`.

In `pingpong.js`, we'll set up a consumer for the service:

```javascript
var PingConsumer = Oasis.Consumer.extend({
  
});
```

And connect it to the service in the parent:

```javascript
Oasis.connect({
  consumers: {
    ping: PingConsumer
  }
})
```

That gives you an idea of the basic communication structure. Now,
let's update the `PingService` to ping the sandbox.

```javascript
var PingService = Oasis.Service.extend({
  initialize: function() {
    this.send('ping');
  }
});
```

The `initialize` method on your service will be called once the
handshake with the sandbox is complete. In this case, the service
sends a `ping` message to the sandbox.

To respond back to the host environment, let's update our `PingConsumer`.

```javascript
var PingConsumer = Oasis.Consumer.extend({
  events: {
    ping: function() {
      this.send('pong');
    }
  }
});
```

Great! Now we have the host environment sending a message to the
sandbox, and the sandbox sending a message back. Let's close the
loop by listening for `pong` on the `PingService`.

```javascript
var PingService = Oasis.Service.extend({
  initialize: function() {
    this.send('ping');
  },

  events: {
    pong: function() {
      alert("Got a pong!");
    }
  }
});
```

This is the basic flow of events in Oasis. First, Oasis.js does the
handshake for you. Once the handshake is complete, you can send
events back and forth.

```sequence
Host-->Sandbox:
Note right of Host: Handshake
Sandbox-->Host:
Host->Sandbox: event:ping
Sandbox->Host: event:pong
```

## Requests

In the above example, the `ping` event was really sent from the host
to the sandbox as a **request** for a `pong`.

Because this is so common, Oasis.js provides a `request` API. Let's
take a look at what it looks like to make a request from the host to
the sandbox.

```javascript
var PingService = Oasis.Service.extend({
  initialize: function() {
    this.request('ping').then(function(data) {
      console.log(data);
    });
  }
});
```

The `request` API uses [Promises][1], a standard way of waiting for
a result for some asynchronous request.

In this case, the _Promise_ crosses the boundary into the sandbox, and
can be fulfilled in the _Consumer_.

[1]: http://promises-aplus.github.com/promises-spec/

```javascript
var PingConsumer = Oasis.Consumer.extend({
  requests: {
    ping: function(resolver) {
      resolver.resolve('pong');
    }
  }
});
```

Instead of listening for a `ping` **event**, we're listening for a `ping`
**request**. Request handlers get a resolver for the promise on the other side
of the boundary that they can resolve with some data.

The interaction between the host and sandbox looks very similar to the
interaction with events.

```sequence
Host-->Sandbox:
Note right of Host: Handshake
Sandbox-->Host:
Host->Sandbox: request:ping
Sandbox->Host: resolve:ping with 'pong'
```

Either side of the boundary (the host and the sandbox) can initiate a request
that is handled by the `requests` hash on the other side.

## Sending Data With an Event

When sending an event, you can send additional data. Simply pass additional
parameters to the `send` method.

```javascript
var PingService = Oasis.Service.extend({
  initialize: function() {
    this.send('ping', { additional: 'data' });
  }
});
```

The additional parameters will be available in the event handler on the other
side of the pipe.

```javascript
var PingConsumer = Oasis.Consumer.extend({
  events: {
    ping: function(options) {
      console.log(options.additional); // 'data'
    }
  }
});
```

The data gets passed along across the boundary.

```sequence
Host-->Sandbox:
Note right of Host: Handshake
Sandbox-->Host:
Host->Sandbox: event:ping with additional:data
```


## Sending Data With a Request

Similarly, you can send additional parameters across the boundary with a request.

```javascript
var PingService = Oasis.Service.extend({
  initialize: function() {
    this.request('ping', { name: 'tom' }).then(function(data) {
      console.log(data);
    });
  }
});
```

The additional parameters are available on the request handler after the `resolver`.

```javascript
var PingConsumer = Oasis.Consumer.extend({
  requests: {
    ping: function(resolver, options) {
      resolver.resolve('hello ' + options.name);
    }
  }
});
```

This will send a request to the sandbox with `{ name: 'tom' }` as options, which
the handler in the sandbox will use in its response.

```sequence
Host-->Sandbox:
Note right of Host: Handshake
Sandbox-->Host:
Host->Sandbox: request:ping with name:tom
Sandbox->Host: resolve:ping with 'hello tom'
```

## Cross Boundary Communication

When sending data with an **event** or in response to a **request**, you
are sending data through a [MessageChannel][2].

Modern browser support sending any of the following types across the boundary:

* Boolean
* Number
* String
* Date
* Regexp
* File**&ast;**
* Blob**&ast;**
* FileList**&ast;**
* ImageData**&ast;**
* ImageBitmap**&ast;**
* Object containing any of these
* Array containing any of these

[2]: http://www.whatwg.org/specs/web-apps/current-work/multipage/web-messaging.html#channel-messaging

The original version of [Web Messaging][3] only supported strings for
communication. In older browsers, Oasis.js will attempt to polyfill the
cloning algorithm used by newer browsers. However, cloning types marked with a
**&ast;** in the above list is not possible. The good news is that most older
browsers that don't support these new types also don't support "structured
cloning".

[3]: http://www.whatwg.org/specs/web-apps/current-work/multipage/web-messaging.html#web-messaging

For the broadest compatibility, you should limit the objects you send
across the boundary (using `send`, `request` or `resolve`) to the list
of types in the list above without an **&ast;**.