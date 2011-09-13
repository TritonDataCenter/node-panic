```
________                    __    __________               .__        
\______ \   ____   ____ | _/  |_  \______   \_____    ____ |__| ____  
 |    |  \ /  _ \ /    \  \   __\  |     ___/\__  \  /    \|  |/ ___\ 
 |    `   (  <_> )   |  \  |  |    |    |     / __ \|   |  \  \  \___ 
/_______  /\____/|___|  /  |__|    |____|    (____  /___|  /__|\___  >
        \/            \/                          \/     \/        \/ 
```

node-panic
===============

This module provides a primitive postmortem debugging facility for Node.js.
Postmortem debugging is critical for root-causing issues that occur in
production from the artifacts of a single failure.  Without such a facility,
tracking down problems in production becomes a tedious process of adding
logging, trying to reproduce the problem, and repeating until enough information
is gathered to root-cause the issue.  For reproducible problems, this process is
merely painful for developers, administrators, and customers alike.  For
unreproducible problems, this is untenable.

The basic idea of this implementation is to maintain a global object that
references all of the internal state we would want for postmortem debugging.
Then when our application crashes, we dump this state to a file, and then exit.


The basics
----------

There are only a few functions you need to know about.  The first time this
module is loaded, it creates a global object called `caDbg` to manage program
debug state.

* `caDbg.set(key, value)`: registers the object `value` to be dumped under the
  key `key` when the program panics.  This function replaces the value from any
  previous call with the same key.
* `caDbg.add(keybase, value)`: like caDbg.set, but generates a unique key based
  on `keybase`.
* `mod_panic.panic(msg, err)`: dumps the given error message an optional
  exception as well as all registered debug state to a file called
  "cacore.<pid>" and then exits the program.
* `mod_panic.enablePanicOnCrash()`: sets up the program to automatically invoke
  `mod_panic.panic` when an uncaught exception bubbles to the event loop

When the program panics (crashes), it saves all debug state to a file called
"cacore.<pid>".  This file is pure JSON and is best read using the "json" tool at:

    https://github.com/trentm/json

In the example above, the program first invokes `enablePanicOnCrash` to set up
automatic panicking when the program crashes.  As each function is invoked, it
adds its argument to the global debug state.  After the program crashes, you
can see the saved state as "func1.arg" and "func2.arg" in the dump.


Example
--------

First, a simple program:

    $ cat examples/example-auto.js 
    /*
     * example-auto.js: simple example of automatically panicking on crash
     */
    
    var mod_panic = require('panic');
    
    function func1(arg1)
    {
    	/* include func1 arg in debug state */
    	caDbg.set('func1.arg', arg1);
    	func2(arg1 + 10);
    }
    
    function func2(arg2)
    {
    	/* include func2 arg in debug state */
    	caDbg.set('func2.arg', arg2);
    	/* crash */
    	(undefined).nonexistentMethod();
    }
    
    /*
     * Trigger a panic on crash.
     */
    mod_panic.enablePanicOnCrash();
    
    /*
     * The following line of code will cause this Node program to exit after dumping
     * debug state to cacore.<pid> (including func1's and func2's arguments).
     */
    func1(10);
    console.error('cannot get here');


Run the program:

    $ node examples/example-auto.js 
    [2011-09-12 22:37:36.410 UTC] CRIT   CA PANIC: panic due to uncaught exception: EXCEPTION: TypeError: TypeError: Cannot call method 'nonexistentMethod' of undefined
        at func2 (/home/dap/node-postmortem/examples/example-auto.js:19:14)
        at func1 (/home/dap/node-postmortem/examples/example-auto.js:11:2)
        at Object.<anonymous> (/home/dap/node-postmortem/examples/example-auto.js:31:1)
        at Module._compile (module.js:402:26)
        at Object..js (module.js:408:10)
        at Module.load (module.js:334:31)
        at Function._load (module.js:293:12)
        at Array.<anonymous> (module.js:421:10)
        at EventEmitter._tickCallback (node.js:126:26)
    [2011-09-12 22:37:36.411 UTC] CRIT   writing core dump to /home/dap/node-postmortem/cacore.22984
    [2011-09-12 22:37:36.413 UTC] CRIT   finished writing core dump


View the "core dump":

    $ json < /home/dap/node-postmortem/cacore.22984
    {
      "dbg.format-version": "0.1",
      "init.process.argv": [
        "node",
        "/home/dap/node-panic/examples/example-auto.js"
      ],
      "init.process.pid": 22984,
      "init.process.cwd": "/home/dap/node-panic",
      "init.process.env": {
        "HOST": "devel",
        "TERM": "xterm-color",
        "SHELL": "/bin/bash",
        "USER": "dap",
        "PWD": "/home/dap/node-panic",
        "MACHINE_THAT_GOES_PING": "1",
        "SHLVL": "1",
        "HOME": "/home/dap",
        "_": "/usr/bin/node"
      },
      "init.process.version": "v0.4.9",
      "init.process.platform": "sunos",
      "init.time": "2011-09-12T22:37:36.408Z",
      "init.time-ms": 1315867056408,
      "func1.arg": 10,
      "func2.arg": 20,
      "panic.error": "EXCEPTION: TypeError: TypeError: Cannot call method 'nonexistentMethod' of undefined\n    at func2 (/home/dap/node-postmortem/examples/example-auto.js:19:14)\n    at func1 (/home/dap/node-postmortem/examples/example-auto.js:11:2)\n    at Object.<anonymous> (/home/dap/node-postmortem/examples/example-auto.js:31:1)\n    at Module._compile (module.js:402:26)\n    at Object..js (module.js:408:10)\n    at Module.load (module.js:334:31)\n    at Function._load (module.js:293:12)\n    at Array.<anonymous> (module.js:421:10)\n    at EventEmitter._tickCallback (node.js:126:26)",
      "panic.time": "2011-09-12T22:37:36.408Z",
      "panic.time-ms": 1315867056408,
      "panic.memusage": {
        "rss": 13000704,
        "vsize": 73252864,
        "heapTotal": 3196160,
        "heapUsed": 1926592
      }
    }


What's in the dump
------------------

The dump itself is just a JSON object.  This module automatically fills in the following keys:

* dbg.format-version: file format version
* init.process.argv: value of process.argv (process arguments)
* init.process.pid: value of process.pid (process identifier)
* init.process.cwd: value of process.cwd (process working directory)
* init.process.env: value of process.env (process environment)
* init.process.version: value of process.version (Node.js version)
* init.process.platform: value of process.platform (operating system)
* init.time: time at which node-panic was loaded
* init.time-ms: time in milliseconds at which node-panic was loaded
* panic.error: string description of the actual error that caused the panic (includes stack trace)
* panic.time: time at which the panic occurred
* panic.time: time in milliseconds at which the panic occurred
* panic.memusage: memory used when the panic occurred

*plus* any information added with `caDbg.set` or `caDbg.add`.


Notes
-----

This facility was initially developed for Joyent's Cloud Analytics service.
For more information on Cloud Analytics, see http://dtrace.org/blogs/dap/files/2011/07/ca-oscon-data.pdf

Pull requests accepted, but code must pass style and lint checks using:

* style: https://github.com/davepacheco/jsstyle
* lint: https://github.com/davepacheco/javascriptlint

This facility has been tested on MacOSX and Illumos with Node.js v0.4.  It has
few dependencies on either the underlying platform or the Node version and so
should work on other platforms.
