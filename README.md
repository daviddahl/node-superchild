[![Circle CI](https://circleci.com/gh/mayanklahiri/node-superchild.svg?style=svg)](https://circleci.com/gh/mayanklahiri/node-superchild)
[![GitHub issues](https://img.shields.io/github/issues/mayanklahiri/node-superchild.svg)](https://github.com/mayanklahiri/node-superchild/issues)
[![GitHub stars](https://img.shields.io/github/stars/mayanklahiri/node-superchild.svg)](https://github.com/mayanklahiri/node-superchild/stargazers)
[![GitHub license](https://img.shields.io/badge/license-MIT-blue.svg)](https://raw.githubusercontent.com/mayanklahiri/node-superchild/master/LICENSE)
[![npm version](https://badge.fury.io/js/superchild.svg)](https://badge.fury.io/js/superchild)
[![dependencies](https://david-dm.org/mayanklahiri/node-superchild.svg)](https://david-dm.org/mayanklahiri/node-superchild.svg)
[![bitHound Overall Score](https://www.bithound.io/github/mayanklahiri/node-superchild/badges/score.svg)](https://www.bithound.io/github/mayanklahiri/node-superchild)
[![bitHound Code](https://www.bithound.io/github/mayanklahiri/node-superchild/badges/code.svg)](https://www.bithound.io/github/mayanklahiri/node-superchild)
[![Twitter](https://img.shields.io/twitter/url/https/github.com/mayanklahiri/node-superchild.svg?style=social)](https://twitter.com/intent/tweet?text=Wow:&url=%5Bobject%20Object%5D)

**Superchild** is a POSIX-only (e.g., Linux, Mac OS X) wrapper around
node.js's built-in `child_process` module, which requires a lot of
care and attention to use correctly.

Links:

  * [Annotated source code](http://mayanklahiri.github.io/node-superchild/superchild.html)
  * [Current test coverage](http://mayanklahiri.github.io/node-superchild/coverage/lib/index.html)
  * [Github page](https://github.com/mayanklahiri/node-superchild)
  * [NPM page](https://www.npmjs.com/package/superchild)

The purpose of Superchild is to allow large node.js programs
to be split into independent processes (and sub-processes, resulting
in **process trees**), while handling the tedious parts of process
management and communication.

Superchild aims to be **compatible** with any program
that reads from and writes to their `stdin`, `stdout`, and `stderr`
streams, regardless of what language the program is written in.
This allows interesting hacks like using `ssh` to execute a
module on a remote host, written in another language, over an
encrypted link, while using the near-universal format of
line-delimited JSON messages over `stdio`.

**Features** that make Superchild different from node's built-in
`child_process` module include the following (many of these
are currently possible only due to restricting focus to POSIX
platforms, i.e., not Windows):

  1. A single function to replace `fork()`, `exec()`,
     and `spawn()` from the built-in `child_process` module.

  2. Waits for `stdout` and `stderr` streams to end before
     emitting an `exit` event, unlike `child_process.ChildProcess`.

  3. Handles isolating child process and its children in a
     separate, __detached process group__ that can be terminated
     as a subtree using the POSIX `kill` command. This means
     that calling `close()` on a Superchild instance will kill
     not just the child process, but all _its_ child processes
     and so on (i.e., the entire process group lead by the child).
     Note that if any processes in the sub-tree detach themselves
     into a new process group, they will not be part of our
     child's process group, and will not be killed.

  4. Handles __graceful termination__ of child's entire process group
     using `SIGTERM` -> `SIGKILL` signals with a configurable timeout.

  5. Handles __unexpected termination__ of the *current* process by
     killing the child's entire process group immediately with `SIGKILL`.

  6. Automatically serializes and deserializes __line-delimited JSON__
     values (LD-JSON) sent to and received from child, intermixed
     with `stdout`. `stderr` is passed through unbuffered. Effectively,
     this means that the child's `stdout` stream is demultiplexed
     into the child streams `stdout_line` (parsed raw text lines),
     `json_object` (parsed JSON objects), and `json_array` (parsed JSON
     arrays). Regular processes have 3 I/O streams (stdin, stdout,
     stderr); Superchildren have 6 streams (stdin, stdout, stderr,
     stdout_line, json_object, json_array).

## Install

      npm install superchild

## Usage

##### Run a shell command

Get a directory listing line-by-line using `ls`.

      var superchild = require('superchild');
      var child = superchild('ls -lh');
      child.on('stdout_line', function(line) {
        console.log('[stdout]: ', line);
      });

##### Spawn and communicate with a module

Spawn a node.js module in another process and communicate with it.
Note that the child uses the `superchild.unlogger` helper function
to parse its standard input for LD-JSON arrays and objects.

_master.js_

     var assert = require('assert');
     var superchild = require('superchild');
     var child = superchild('node echo.js');
     child.send({
       some: 'data',
     });
     child.on('json_object', function(jsonObj) {
       assert.equal(jsonObj.some, 'data');
     });

_echo.js_

     var unlogger = require('superchild').unlogger;
     unlogger().on('json_object', function(jsonObj) {
       // Echo JSON object from parent back to parent.
       console.log(JSON.stringify(jsonObj));
     });


## Events emitted

Superchild is an EventEmitter. The following events can be listened for
using `child.on()` and `child.once()` functions.

| Event          | Arguments                | Description                                             |
| ---------------| -------------------------|---------------------------------------------------------|
| `exit`         | `code`, `signal`         | Child process exited, identical to `child_process.exit` |
| `stderr_data`  | `dataStr`                | Received unbuffered data on child's `stderr` stream.    |
| `stdout_line`  | `lineStr`                | Received a full line of text from the child process.    |
| `json_object`  | `jsonObj`                | Parsed a line-delimited JSON object from child's `stdout` stream. |
| `json_array`   | `jsonArr`                | Parsed a line-delimited JSON array from child's `stdout` stream.  |

## Methods

| Method          | Description                                                                    |
| ----------------| -------------------------------------------------------------------------------|
| `send(jsonVal)` | Serialize and send a JSON-serializable object or array to the child as LD-JSON.|
| `close(cb)`     | Gracefully terminate the child, invoking the callback when the child has died. |

## Requirements

  * `node.js` version 0.11.13 or higher, due to the use of `spawnSync`.
  * POSIX-compliant platform, such as Linux or Mac OS.

## Source Code

The full annotated source code of `superchild.js` follows, generated
automatically by Docco.

#### Helper utilities

  * [`Class JSONSieve`](http://mayanklahiri.github.io/node-superchild/json-sieve.html),
    parses a readable stream into JSON objects and arrays, and stdout lines.
  * [`function unlogger()`](http://mayanklahiri.github.io/node-superchild/unlogger.html),
    an example of establishing bi-directional communication between a parent
    and child that can be easily ported to many other languages.
