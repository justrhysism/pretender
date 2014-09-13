# Pretender

Pretender is a mock server library in the style of Sinon (but built from microlibs. Because javascript)
that comes with an express/sinatra style syntax for defining routes and their handlers.

Pretender will temporarily replace the native XMLHttpRequest object, intercept all requests, and direct them
to little pretend service you've defined.

```javascript
var PHOTOS = {
  "10": {
    id: 10,
    src: 'http://media.giphy.com/media/UdqUo8xvEcvgA/giphy.gif'
  },
  "42": {
    id: 42,
    src: 'http://media0.giphy.com/media/Ko2pyD26RdYRi/giphy.gif'
  }
};

var server = new Pretender(function(){
  this.get('/photos', function(request){
    var all =  JSON.stringify(Object.keys(PHOTOS).map(function(k){return PHOTOS[k]}))
    return [200, {"Content-Type": "application/json"}, all]
  });

  this.get('/photos/:id', function(request){
    return [200, {"Content-Type": "application/json"}, JSON.stringify(PHOTOS[request.params.id])]
  });
});

$.get('/photos/12', {success: function(){ ... }})
```


## The Server DSL
The server DSL is inspired by express/sinatra. Pass a function to the Pretender constructor
that will be invoked with the Pretender instance as its context. Available methods are
`get`, `put`, `post`, `'delete'`, `patch`, and `head`. Each of these methods takes a path pattern
and a callback. The callback will be invoked with a single argument (the XMLHttpRequest instance that
triggered this request) and must return an array containing the HTTP status code, headers object, and body as a string.

```javascript
var server = new Pretender(function(){
  this.put('/api/songs/99', function(request){
    return [404, {}, ""];
  });
});

```

### Paths
Paths can either be hard-coded (`this.get('/api/songs/12')`) or contain dynamic segments
(`this.get('/api/songs/:song_id'`). If there were dynamic segments of the path,
these well be attached to the request object as a `params` property with keys matching
the dynamic portion and values with the matching value from the path.

```javascript
var server = new Pretender(function(){
  this.get('/api/songs/:song_id', function(request){
    request.params.song_id;
  });
});

$.get('/api/songs/871') // params.song_id will be '871'

```

Pretender will *only* handle requests to paths (`/some/kind/of/path`) and not fully qualified URLs
(`https://mydomain.tld/some/kind/of/path`). There are many
[CSP](http://www.html5rocks.com/en/tutorials/security/content-security-policy/) behaviors that cannot
be consistenly simulated when using full URLs (calls to `http` from an `https` page, requests to
external domains which may not implement CORS, etc).

### Query Parameters
If there were query parameters in the request, these well be attached to the request object as a `queryParams`
property.

```javascript
var server = new Pretender(function(){
  this.get('/api/songs', function(request){
    request.queryParams.sortOrder;
  });
});

// typical jQuery-style uses you've probably seen.
// queryParams.sortOrder will be 'asc' for both styles.
$.get({url: '/api/songs', data: {sortOrder: 'asc'});
$.get('/api/songs?sortOrder=asc');

```


### Responding
You must return an array from this handler that includes the HTTP status code, an object literal
of response headers, and a string body.

var server = new Pretender(function(){
  this.get('/api/songs', function(request){
    return [
      200,
      {'content-type': 'application/javascript'},
      '[{"id": 12}, {"id": 14}]'
    ];
  });
});

## Hooks
### Handled Requests
In addition to responding to the request, your server will call a `handledRequest` method with
the HTTP `verb`, `path`, and original `request`. By default this method does nothing. You can
override this method to supply your own behavior like logging or test framework integration:

```javascript
var server = new Pretender(function(){
  this.put('/api/songs/:song_id', function(request){
    return [202, {"Content-Type": "application/json"}, "{}"]
  });
});

server.handledRequest = function(verb, path, request) {
  console.log("a request was responded to");
}

$.getJSON("/api/songs/12");
```

### Unhandled Requests
Your server will call a `unhandledRequest` method with the HTTP `verb`, `path`, and original `request`,
object if your server receives a request for a route that doesn't have a handler. By default, this method
will throw an error. You can override this method to supply your own behavior:

```javascript
var server = new Pretender(function(){
  // no routes
});

server.unhandledRequest = function(verb, path, request) {
  console.log("what is this I don't even...");
}

$.getJSON("/these/arent/the/droids");
```


### Error Requests
Your server will call a `erroredRequest` method with the HTTP `verb`, `path`, original `request`,
and the original `error` object if your handler code causes an error:

By default, this will augment the error message with some information about which handler caused
the error and then throw the error again. You can override this method to supply your own behavior:

```javascript
var server = new Pretender(function(){
  this.get('/api/songs', function(request){
    undefinedWat("this is no function!");
  });
});

server.erroredRequest = function(verb, path, request, error) {
  SomeTestFramework.failTest();
  console.warn("There was an error", error);
}
```

### Mutating the body
Pretender is response format neutral, so you normally need to supply a string body as the
third part of a response:

```javascript
this.get('/api/songs', function(request){
  return [200, {}, "{'id': 12}"];
});
```

This can become tiresome if you know, for example, that all your responses are
going to be JSON. The body of a response will be passed through a
`prepareBody` hook before being passed to the fake response object.
`prepareBody` defaults to an empty function, but can be overriden:

```javascript
var server = new Pretender(function(){
  this.get('/api/songs', function(request){
    return [200, {}, {id: 12}];
  });
});

server.prepareBody = function(body){
  return body ? JSON.stringify(body) : '{"error": "not found"}';
}
```

### Mutating the headers
Response headers can be mutated for the entire service instance by implementing a
`prepareHeaders` method:

```javascript
var server = new Pretender(function(){
  this.get('/api/songs', function(request){
    return [200, {}, '{"id": 12}''];
  });
});

server.prepareHeaders = function(headers){
  headers['content-type'] = 'application/javascript';
  return headers;
};
```

## Tracking Requests
Your pretender instance will track handlers and requests on a few array properties.
All handlers are stored on `handlers` property and incoming requests will be tracked in one of
two properties: `handledRequests` and `unhandledRequests`. This is useful if you want to build
testing infrastructure on top of pretender and need to fail tests that have handlers without requests.

Each handler keeps a count of the number of requests is successfully served.

## Clean up
When you're done mocking, be sure to call `shutdown()` to restore the native XMLHttpRequest object:

```javascript
var server = new Pretender(function(){
 ... routing ...
});

server.shutdown(); // all done.
```
