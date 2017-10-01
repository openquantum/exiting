# Exiting

Safely shutdown [hapi.js](http://hapijs.com/) servers whenever the process exits.

[![Build Status](https://travis-ci.org/kanongil/exiting.svg?branch=master)](https://travis-ci.org/kanongil/exiting)

## Details

While it is simple to start and stop a server, ensuring proper shutdown on external, or internal,
triggers can be cumbersome to handle properly.
**exiting** makes this easy by managing your Hapi server, taking care of starting and stopping it
as appropriate.

Depending on the exit trigger, the server will either be gracefully stopped or aborted (by only
triggering `onPreStop` hooks).
The exit triggers are handled as detailed:

 * Graceful exit with code `0`:
   * `process.exit()` with exit code `0`.
   * `SIGINT` kill signal, through eg. `ctrl-c`.
   * `SIGTERM` kill signal.
   * `SIGQUIT` kill signal.
 * Aborted exit:
   * `process.exit()` with non-zero exit code.
   * `SIGHUP` kill signal (code `1`).
   * Any uncaught exception (code `255`).
   * Any closed connection listeners, eg. on worker disconnect (code `255`).

If the server shutdown is too slow, a timeout will eventually trigger an exit (exit code `255`).

The shutdown logic is programmed to handle almost any conceivable exit condition, and provides
100% test coverage.
The only instances that `onPreHook` code is not called, are uncatchable signals, like `SIGKILL`,
and fatal errors that trigger during shutdown.

## Example

Basic server example:

```js
const Hapi = require('hapi');
const Exiting = require('exiting);

const server = new Hapi.Server();
const manager = new Exiting.Manager(server);

server.events.on('stop', () => {

    console.log('Server stopped.');
});

const provision = async () => {

    server.route({
        method: 'GET',
        path: '/',
        handler() {

            return 'Hello';
        }
    });

    await manager.start();

    console.log('Server started at:', server.info.uri);
};

provision();
```

The server and process life-cycle will now be managed by **exiting**.

If you need to delay the shutdown for processing, you can install an extention function on the
`onPreStop` or `onPostStop` extension points, eg:

```js
server.ext('onPreStop', () => {

    return new Promise((resolve) => {

        setTimeout(resolve, 1000);
    });
});
```

## Installation

Install using npm: `npm install exiting`.

## Usage

To enable **exiting** for you server, replace the call to `server.start()` with `new Exiting.Manager(server).start()`.

### new Exiting.Manager(server, [options])

Create a new exit manager for a hapi.js server. The `options` object supports:

 * `exitTimeout` - When exiting, force process exit after this amount of ms has elapsed. Default: `5000`.

### manager.start()

Starts the manager and the server, as if `server.start()` is called.

Note that `process.exit()` is monkey patched to intercept such calls.
Starting also installs the signal handlers and an `uncaughtException` handler.

### manager.stop([options])

Stops the manager and the server, as if `server.stop()` is called.
