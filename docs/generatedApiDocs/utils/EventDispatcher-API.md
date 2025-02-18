<!-- Generated by documentation.js. Update this documentation by updating the source code. -->

## utils/EventDispatcher

Implements a jQuery-like event dispatch pattern for non-DOM objects (works in web workers and phoenix node as well):

*   Listeners are attached via on()/one() & detached via off()
*   Listeners can use namespaces for easy removal
*   Listeners can attach to multiple events at once via a space-separated list
*   Events are fired via trigger()
*   The same listener can be attached twice, and will be called twice; but off() will detach all
    duplicate copies at once ('duplicate' means '===' equality - see [http://jsfiddle.net/bf4p29g5/1/][1])

But it has some important differences from jQuery's non-DOM event mechanism:

*   More robust to listeners that throw exceptions (other listeners will still be called, and
    trigger() will still return control to its caller).
*   Events can be marked deprecated, causing on() to issue warnings
*   Easier to debug, since the dispatch code is much simpler
*   Faster, for the same reason
*   Uses less memory, since $(nonDOMObj).on() leaks memory in jQuery
*   API is simplified:
    *   Event handlers do not have 'this' set to the event dispatcher object
    *   Event object passed to handlers only has 'type' and 'target' fields
    *   trigger() uses a simpler argument-list signature (like Promise APIs), rather than requiring
        an Array arg and ignoring additional args
    *   trigger() does not support namespaces
    *   For simplicity, on() does not accept a map of multiple events -> multiple handlers, nor a
        missing arg standing in for a bare 'return false' handler.

For now, Brackets uses a jQuery patch to ensure $(obj).on() and obj.on() (etc.) are identical
for any obj that has the EventDispatcher pattern. In the future, this may be deprecated.

To add EventDispatcher methods to any object, call EventDispatcher.makeEventDispatcher(obj).

## Usage

### Importing from an extension

```js
const EventDispatcher = brackets.getModule("utils/EventDispatcher");
```

### Using the global object

The EventDispatcher Object is available within the global context, be it phoenix or phoenix core web workers or node.

```js
window.EventDispatcher.makeEventDispatcher(exports); // within phoenix require module
self.EventDispatcher.makeEventDispatcher(object); // within web worker
global.EventDispatcher.makeEventDispatcher(exports); // within node module that has an export
```

If you wish to import event dispatcher to your custom web worker, use the following

```js
importScripts('<relative path from your extension>/utils/EventDispatcher');
// this will add the global EventDispatcher to your web-worker. Note that the EventDispatcher in the web worker
// and node is a separate domain and cannot raise or listen to events in phoenix/other workers. For triggering events
// between different domains like between node and phcode, see `nodeConnector.triggerPeer` or
// `WorkerComm.triggerPeer` API for communication between phcode and web workers.
self.EventDispatcher.trigger("someEvent"); // within web worker
```

### Sample Usage within extension

```js
// in your extension js file.
define (function (require, exports, module) \{
    const EventDispatcher     = brackets.getModule("utils/EventDispatcher");
    EventDispatcher.makeEventDispatcher(exports); // This extension triggers some events
    let eventHandler = function (event, paramObject, paramVal) \{
        console.log(event, paramObject, paramVal);
    };
    exports.on("sampleEvent", eventHandler); // listen to our own event for demo
    exports.trigger("sampleEvent", \{ // trigger a sample event. This will activate the above listener 'on' function.
            param: 1,
            param2: "sample"
    }, "value");
    // If needed, the event listener can be removed with `off`. But it is not a requirement at shutdown.
    exports.off("sampleEvent", eventHandler);
}
```

## splitNs

Split "event.namespace" string into its two parts; both parts are optional.

Type: [function][2]

### Parameters

*   `eventStr`  
*   `eventName` **[string][3]** Event name and/or trailing ".namespace"

Returns **!\{event: [string][3], ns: [string][3]}** Uses "" for missing parts.

## setLeakThresholdForEvent

By default, we consider any events having more than 15 listeners to be leaky. But sometimes there may be
genuine use cases where an event can have a large number of listeners. For those events, it is recommended
to increase the leaky warning threshold individually with this API.

Type: [function][2]

### Parameters

*   `eventName` **[string][3]** 
*   `threshold` **[number][4]** The new threshold to set. Will only be set if the new threshold is greater than
    the current threshold.

## on

Adds the given handler function to 'events': a space-separated list of one or more event names, each
with an optional ".namespace" (used by off() - see below). If the handler is already listening to this
event, a duplicate copy is added.

Type: [function][2]

### Parameters

*   `events` **[string][3]** 
*   `fn`  

## off

Removes one or more handler functions based on the space-separated 'events' list. Each item in
'events' can be: bare event name, bare .namespace, or event.namespace pair. This yields a set of
matching handlers. If 'fn' is omitted, all these handlers are removed. If 'fn' is provided,
only handlers exactly equal to 'fn' are removed (there may still be >1, if duplicates were added).

Type: [function][2]

### Parameters

*   `events` **[string][3]** 
*   `fn`  

## one

Attaches a handler so it's only called once (per event in the 'events' list).

Type: [function][2]

### Parameters

*   `events` **[string][3]** 
*   `fn`  

## trigger

Invokes all handlers for the given event (in the order they were added).

Type: [function][2]

### Parameters

*   `eventName` **[string][3]** 

## makeEventDispatcher

Adds the EventDispatcher APIs to the given object: on(), one(), off(), and trigger(). May also be
called on a prototype object - each instance will still behave independently.

Type: [function][2]

### Parameters

*   `obj` **\![Object][5]** Object to add event-dispatch methods to

## triggerWithArray

Utility for calling on() with an array of arguments to pass to event handlers (rather than a varargs
list). makeEventDispatcher() must have previously been called on 'dispatcher'.

Type: [function][2]

### Parameters

*   `dispatcher` **\![Object][5]** 
*   `eventName` **[string][3]** 
*   `argsArray` **\![Array][6]\<any>** 

## on_duringInit

Utility for attaching an event handler to an object that has not YET had makeEventDispatcher() called
on it, but will in the future. Once 'futureDispatcher' becomes a real event dispatcher, any handlers
attached here will be retained.

Useful with core modules that have circular dependencies (one module initially gets an empty copy of the
other, with no on() API present yet). Unlike other strategies like waiting for htmlReady(), this helper
guarantees you won't miss any future events, regardless of how soon the other module finishes init and
starts calling trigger().

Type: [function][2]

### Parameters

*   `futureDispatcher` **\![Object][5]** 
*   `events` **[string][3]** 
*   `fn`  

## markDeprecated

Mark a given event name as deprecated, such that on() will emit warnings when called with it.
May be called before makeEventDispatcher(). May be called on a prototype where makeEventDispatcher()
is called separately per instance (i.e. in the constructor). Should be called before clients have
a chance to start calling on().

Type: [function][2]

### Parameters

*   `obj` **\![Object][5]** Event dispatcher object
*   `eventName` **[string][3]** Name of deprecated event
*   `insteadStr` **[string][3]?** Suggested thing to use instead

[1]: http://jsfiddle.net/bf4p29g5/1/

[2]: https://developer.mozilla.org/docs/Web/JavaScript/Reference/Statements/function

[3]: https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/String

[4]: https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/Number

[5]: https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/Object

[6]: https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/Array
