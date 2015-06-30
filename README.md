[![npm version](https://img.shields.io/npm/v/rethinkdb-websocket-server.svg)](https://www.npmjs.com/package/rethinkdb-websocket-server)

# rethinkdb-websocket-server

Node.js WebSocket server that proxies to a RethinkDB instance. Supports query
validation.

## What is this?

This library attaches to a [node.js
http.Server](https://nodejs.org/api/http.html) and listens for incoming
WebSocket connections at a specified path. For each incoming WebSocket
connection, it opens a new TCP socket to a specified
[RethinkDB](http://rethinkdb.com/) instance, and forwards traffic in both
directions between the WebSocket and the RethinkDB socket until either side
disconnects.

Each query sent from the WebSocket is parsed and validated before being
forwarded to RethinkDB. This is done using a whitelist of pattern queries,
described in the involved example below.

The provided WebSocket server can be used in conjunction with any of the
following clients:
* [rethinkdb-websocket-client](https://github.com/mikemintz/rethinkdb-websocket-client)
  is a WebSocket wrapper around the RethinkDB JavaScript driver that works in
  the browser.
* [react-rethinkdb](https://github.com/mikemintz/react-rethinkdb) is a
  [React](http://facebook.github.io/react/) mixin that connects using
  rethinkdb-websocket-client.

## How do I use this?

This package should be installed with [npm](https://www.npmjs.com/) and should
run in a node.js http.Server.

#### Simple example

Below is a simple example of a script that listens at `ws://localhost:8000/`
for incoming WebSocket connections. It blindly forwards all queries to a
RethinkDB instance running locally at the default `28015` port.

```js
var http = require('http');
var wsListen = require('rethinkdb-websocket-server').listen;

var httpServer = http.createServer();
wsListen({httpServer: httpServer, unsafelyAllowAnyQuery: true});
httpServer.listen(8000);
```
#### Involved example

In this example, we listen for WebSocket connections at
`ws://localhost:8000/rethinkApi` and forward traffic to a RethinkDB instance
running at `rethink01.example.com:28015`. We also serve static files in the
`assets` directory over HTTP using [express](http://expressjs.com/).

Rather than enabling `unsafelyAllowAnyQuery`, we explicitly set up a query
whitelist. This one only allows two query patterns:
1. Queries that list `turtles` with the same `herdId` as the authenticated user
2. Queries that insert `turtles` with a non-empty `name` and a `herdId`
   referring to the primary key of an object in the `herds` table

In order to validate queries against the authenticated user, we create a
"session" object from the query params of the WebSocket URL. In this case, the
browser connects to `ws://localhost:8000/rethinkApi?userId=foo&authToken=bar`,
the `sessionCreator` function looks up that user in the database, and
`user.curHerdId` is stored in the custom session object that we have access to
when validating queries from this client.

**Note:** in production, you should enable secure websockets so sensitive
data is not vulnerable.

As you are developing, incoming queries that don't validate against the
whitelist will be logged to console in a format that you can copy and paste
directly into your JavaScript source file. For dynamic queries, you'll likely
want to generalize the pattern using `function(actual, refs, session)` terms,
`RQ.ref()` terms, and the `.validate()` method. Using [ES6 arrow
functions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions) can make this a bit less verbose.

```js
var express = require('express');
var http = require('http');
var Promise = require('bluebird');
var r = require('rethinkdb');
var RethinkdbWebsocketServer = require('rethinkdb-websocket-server');
var RQ = RethinkdbWebsocketServer.RQ;

var options = {};
options.dbHost = 'rethink01.example.com';
options.dbPort = 28015;

var rethinkConn = Promise.promisify(r.connect)({
  host: options.dbHost,
  port: options.dbPort,
  db: 'test',
});
function runQuery(query) {
  return rethinkConn.then(function(conn) {
    return query.run(conn);
  });
}

options.sessionCreator = function(urlQueryParams) {
  var userQuery = r.table('users').get(urlQueryParams.userId);
  return runQuery(userQuery).then(function(user) {
    if (user && user.authToken === urlQueryParams.authToken) {
      return {curHerdId: user.herdId};
    } else {
      return Promise.reject('Invalid auth token');
    }
  });
};

options.queryWhitelist = [
  // r.table('turtles').filter({herdId: curHerdId})
  RQ(
    RQ.FILTER(
      RQ.TABLE("turtles"),
      {"herdId": RQ.ref('herdId')}
    )
  ).opt("db", RQ.DB("test"))
  .validate(function(refs, session) {
    return session.curHerdId === refs.herdId;
  }),

  // r.table('turtles').insert({herdId: 'alpha-squadron', name: 'Speedy'})
  RQ(
    RQ.INSERT(
      RQ.TABLE("turtles"),
      {
        "herdId": RQ.ref('herdId'),
        "name": function(actual, refs, session) {
          return typeof actual === 'string' && actual.trim();
        },
      }
    )
  ).opt("db", RQ.DB("test"))
  .validate(function(refs) {
    var herdId = refs.herdId;
    if (typeof herdId !== 'string') return false;
    var validHerdQuery = r.table('herds').get(herdId).ne(null);
    return runQuery(validHerdQuery);
  }),
];

var app = express();
app.use('/', express.static('assets'));
var httpServer = http.createServer(app);
options.httpServer = httpServer;
options.httpPath = '/rethinkApi';

RethinkdbWebsocketServer.listen(options);
httpServer.listen(8000);
```
