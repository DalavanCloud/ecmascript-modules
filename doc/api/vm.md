# VM (Executing JavaScript)

<!--introduced_in=v0.10.0-->

> Stability: 2 - Stable

<!--name=vm-->

The `vm` module provides APIs for compiling and running code within V8 Virtual
Machine contexts. **The `vm` module is not a security mechanism. Do
not use it to run untrusted code**. The term "sandbox" is used throughout these
docs simply to refer to a separate context, and does not confer any security
guarantees.

JavaScript code can be compiled and run immediately or
compiled, saved, and run later.

A common use case is to run the code in a sandboxed environment.
The sandboxed code uses a different V8 Context, meaning that
it has a different global object than the rest of the code.

One can provide the context by ["contextifying"][contextified] a sandbox
object. The sandboxed code treats any property in the sandbox like a
global variable. Any changes to global variables caused by the sandboxed
code are reflected in the sandbox object.

```js
const vm = require('vm');

const x = 1;

const sandbox = { x: 2 };
vm.createContext(sandbox); // Contextify the sandbox.

const code = 'x += 40; var y = 17;';
// `x` and `y` are global variables in the sandboxed environment.
// Initially, x has the value 2 because that is the value of sandbox.x.
vm.runInContext(code, sandbox);

console.log(sandbox.x); // 42
console.log(sandbox.y); // 17

console.log(x); // 1; y is not defined.
```

## Class: vm.Script
<!-- YAML
added: v0.3.1
-->

Instances of the `vm.Script` class contain precompiled scripts that can be
executed in specific sandboxes (or "contexts").

### new vm.Script(code, options)
<!-- YAML
added: v0.3.1
changes:
  - version: v5.7.0
    pr-url: https://github.com/nodejs/node/pull/4777
    description: The `cachedData` and `produceCachedData` options are
                 supported now.
  - version: v10.6.0
    pr-url: https://github.com/nodejs/node/pull/20300
    description: The `produceCachedData` is deprecated in favour of
                 `script.createCachedData()`
-->

* `code` {string} The JavaScript code to compile.
* `options`
  * `filename` {string} Specifies the filename used in stack traces produced
    by this script.
  * `lineOffset` {number} Specifies the line number offset that is displayed
    in stack traces produced by this script.
  * `columnOffset` {number} Specifies the column number offset that is displayed
    in stack traces produced by this script.
  * `cachedData` {Buffer|TypedArray|DataView} Provides an optional `Buffer` or
    `TypedArray`, or `DataView` with V8's code cache data for the supplied
     source. When supplied, the `cachedDataRejected` value will be set to
     either `true` or `false` depending on acceptance of the data by V8.
  * `produceCachedData` {boolean} When `true` and no `cachedData` is present, V8
    will attempt to produce code cache data for `code`. Upon success, a
    `Buffer` with V8's code cache data will be produced and stored in the
    `cachedData` property of the returned `vm.Script` instance.
    The `cachedDataProduced` value will be set to either `true` or `false`
    depending on whether code cache data is produced successfully.
    This option is deprecated in favor of `script.createCachedData()`.
  * `importModuleDynamically` {Function} Called during evaluation of this
    module when `import()` is called. This function has the signature
    `(specifier, module)` where `specifier` is the specifier passed to
    `import()` and `module` is this `vm.SourceTextModule`. If this option is
    not specified, calls to `import()` will reject with
    [`ERR_VM_DYNAMIC_IMPORT_CALLBACK_MISSING`][]. This method can return a
    [Module Namespace Object][], but returning a `vm.SourceTextModule` is
    recommended in order to take advantage of error tracking, and to avoid
    issues with namespaces that contain `then` function exports.

Creating a new `vm.Script` object compiles `code` but does not run it. The
compiled `vm.Script` can be run later multiple times. The `code` is not bound to
any global object; rather, it is bound before each run, just for that run.

### script.createCachedData()
<!-- YAML
added: v10.6.0
-->

* Returns: {Buffer}

Creates a code cache that can be used with the Script constructor's
`cachedData` option. Returns a Buffer. This method may be called at any
time and any number of times.

```js
const script = new vm.Script(`
function add(a, b) {
  return a + b;
}

const x = add(1, 2);
`);

const cacheWithoutX = script.createCachedData();

script.runInThisContext();

const cacheWithX = script.createCachedData();
```

### script.runInContext(contextifiedSandbox[, options])
<!-- YAML
added: v0.3.1
changes:
  - version: v6.3.0
    pr-url: https://github.com/nodejs/node/pull/6635
    description: The `breakOnSigint` option is supported now.
-->

* `contextifiedSandbox` {Object} A [contextified][] object as returned by the
  `vm.createContext()` method.
* `options` {Object}
  * `filename` {string} Specifies the filename used in stack traces produced
    by this script.
  * `lineOffset` {number} Specifies the line number offset that is displayed
    in stack traces produced by this script.
  * `columnOffset` {number} Specifies the column number offset that is displayed
    in stack traces produced by this script.
  * `displayErrors` {boolean} When `true`, if an [`Error`][] error occurs
    while compiling the `code`, the line of code causing the error is attached
    to the stack trace.
  * `timeout` {integer} Specifies the number of milliseconds to execute `code`
    before terminating execution. If execution is terminated, an [`Error`][]
    will be thrown. This value must be a strictly positive integer.
  * `breakOnSigint`: if `true`, the execution will be terminated when
    `SIGINT` (Ctrl+C) is received. Existing handlers for the
    event that have been attached via `process.on('SIGINT')` will be disabled
    during script execution, but will continue to work after that.
    If execution is terminated, an [`Error`][] will be thrown.

Runs the compiled code contained by the `vm.Script` object within the given
`contextifiedSandbox` and returns the result. Running code does not have access
to local scope.

The following example compiles code that increments a global variable, sets
the value of another global variable, then execute the code multiple times.
The globals are contained in the `sandbox` object.

```js
const util = require('util');
const vm = require('vm');

const sandbox = {
  animal: 'cat',
  count: 2
};

const script = new vm.Script('count += 1; name = "kitty";');

const context = vm.createContext(sandbox);
for (let i = 0; i < 10; ++i) {
  script.runInContext(context);
}

console.log(util.inspect(sandbox));

// { animal: 'cat', count: 12, name: 'kitty' }
```

Using the `timeout` or `breakOnSigint` options will result in new event loops
and corresponding threads being started, which have a non-zero performance
overhead.

### script.runInNewContext([sandbox[, options]])
<!-- YAML
added: v0.3.1
changes:
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/19016
    description: The `contextCodeGeneration` option is supported now.
-->

* `sandbox` {Object} An object that will be [contextified][]. If `undefined`, a
  new object will be created.
* `options` {Object}
  * `filename` {string} Specifies the filename used in stack traces produced
    by this script.
  * `lineOffset` {number} Specifies the line number offset that is displayed
    in stack traces produced by this script.
  * `columnOffset` {number} Specifies the column number offset that is displayed
    in stack traces produced by this script.
  * `displayErrors` {boolean} When `true`, if an [`Error`][] error occurs
    while compiling the `code`, the line of code causing the error is attached
    to the stack trace.
  * `timeout` {integer} Specifies the number of milliseconds to execute `code`
    before terminating execution. If execution is terminated, an [`Error`][]
    will be thrown. This value must be a strictly positive integer.
  * `contextName` {string} Human-readable name of the newly created context.
    **Default:** `'VM Context i'`, where `i` is an ascending numerical index of
    the created context.
  * `contextOrigin` {string} [Origin][origin] corresponding to the newly
    created context for display purposes. The origin should be formatted like a
    URL, but with only the scheme, host, and port (if necessary), like the
    value of the [`url.origin`][] property of a [`URL`][] object. Most notably,
    this string should omit the trailing slash, as that denotes a path.
    **Default:** `''`.
  * `contextCodeGeneration` {Object}
    * `strings` {boolean} If set to false any calls to `eval` or function
      constructors (`Function`, `GeneratorFunction`, etc) will throw an
      `EvalError`. **Default:** `true`.
    * `wasm` {boolean} If set to false any attempt to compile a WebAssembly
      module will throw a `WebAssembly.CompileError`. **Default:** `true`.

First contextifies the given `sandbox`, runs the compiled code contained by
the `vm.Script` object within the created sandbox, and returns the result.
Running code does not have access to local scope.

The following example compiles code that sets a global variable, then executes
the code multiple times in different contexts. The globals are set on and
contained within each individual `sandbox`.

```js
const util = require('util');
const vm = require('vm');

const script = new vm.Script('globalVar = "set"');

const sandboxes = [{}, {}, {}];
sandboxes.forEach((sandbox) => {
  script.runInNewContext(sandbox);
});

console.log(util.inspect(sandboxes));

// [{ globalVar: 'set' }, { globalVar: 'set' }, { globalVar: 'set' }]
```

### script.runInThisContext([options])
<!-- YAML
added: v0.3.1
-->

* `options` {Object}
  * `filename` {string} Specifies the filename used in stack traces produced
    by this script.
  * `lineOffset` {number} Specifies the line number offset that is displayed
    in stack traces produced by this script.
  * `columnOffset` {number} Specifies the column number offset that is displayed
    in stack traces produced by this script.
  * `displayErrors` {boolean} When `true`, if an [`Error`][] error occurs
    while compiling the `code`, the line of code causing the error is attached
    to the stack trace.
  * `timeout` {integer} Specifies the number of milliseconds to execute `code`
    before terminating execution. If execution is terminated, an [`Error`][]
    will be thrown. This value must be a strictly positive integer.

Runs the compiled code contained by the `vm.Script` within the context of the
current `global` object. Running code does not have access to local scope, but
*does* have access to the current `global` object.

The following example compiles code that increments a `global` variable then
executes that code multiple times:

```js
const vm = require('vm');

global.globalVar = 0;

const script = new vm.Script('globalVar += 1', { filename: 'myfile.vm' });

for (let i = 0; i < 1000; ++i) {
  script.runInThisContext();
}

console.log(globalVar);

// 1000
```

## vm.compileFunction(code[, params[, options]])
<!-- YAML
added: v10.10.0
-->
* `code` {string} The body of the function to compile.
* `params` {string[]} An array of strings containing all parameters for the
  function.
* `options` {Object}
  * `filename` {string} Specifies the filename used in stack traces produced
    by this script. **Default:** `''`.
  * `lineOffset` {number} Specifies the line number offset that is displayed
    in stack traces produced by this script. **Default:** `0`.
  * `columnOffset` {number} Specifies the column number offset that is displayed
    in stack traces produced by this script. **Default:** `0`.
  * `cachedData` {Buffer|TypedArray|DataView} Provides an optional `Buffer` or
    `TypedArray`, or `DataView` with V8's code cache data for the supplied
     source.
  * `produceCachedData` {boolean} Specifies whether to produce new cache data.
    **Default:** `false`.
  * `parsingContext` {Object} The [contextified][] sandbox in which the said
    function should be compiled in.
  * `contextExtensions` {Object[]} An array containing a collection of context
    extensions (objects wrapping the current scope) to be applied while
    compiling. **Default:** `[]`.

Compiles the given code into the provided context/sandbox (if no context is
supplied, the current context is used), and returns it wrapped inside a
function with the given `params`.

## vm.createContext([sandbox[, options]])
<!-- YAML
added: v0.3.1
changes:
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/19398
    description: The `sandbox` option can no longer be a function.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/19016
    description: The `codeGeneration` option is supported now.
-->

* `sandbox` {Object}
* `options` {Object}
  * `name` {string} Human-readable name of the newly created context.
    **Default:** `'VM Context i'`, where `i` is an ascending numerical index of
    the created context.
  * `origin` {string} [Origin][origin] corresponding to the newly created
    context for display purposes. The origin should be formatted like a URL,
    but with only the scheme, host, and port (if necessary), like the value of
    the [`url.origin`][] property of a [`URL`][] object. Most notably, this
    string should omit the trailing slash, as that denotes a path.
    **Default:** `''`.
  * `codeGeneration` {Object}
    * `strings` {boolean} If set to false any calls to `eval` or function
      constructors (`Function`, `GeneratorFunction`, etc) will throw an
      `EvalError`. **Default:** `true`.
    * `wasm` {boolean} If set to false any attempt to compile a WebAssembly
      module will throw a `WebAssembly.CompileError`. **Default:** `true`.

If given a `sandbox` object, the `vm.createContext()` method will [prepare
that sandbox][contextified] so that it can be used in calls to
[`vm.runInContext()`][] or [`script.runInContext()`][]. Inside such scripts,
the `sandbox` object will be the global object, retaining all of its existing
properties but also having the built-in objects and functions any standard
[global object][] has. Outside of scripts run by the vm module, global variables
will remain unchanged.

```js
const util = require('util');
const vm = require('vm');

global.globalVar = 3;

const sandbox = { globalVar: 1 };
vm.createContext(sandbox);

vm.runInContext('globalVar *= 2;', sandbox);

console.log(util.inspect(sandbox)); // { globalVar: 2 }

console.log(util.inspect(globalVar)); // 3
```

If `sandbox` is omitted (or passed explicitly as `undefined`), a new, empty
[contextified][] sandbox object will be returned.

The `vm.createContext()` method is primarily useful for creating a single
sandbox that can be used to run multiple scripts. For instance, if emulating a
web browser, the method can be used to create a single sandbox representing a
window's global object, then run all `<script>` tags together within the context
of that sandbox.

The provided `name` and `origin` of the context are made visible through the
Inspector API.

## vm.isContext(sandbox)
<!-- YAML
added: v0.11.7
-->

* `sandbox` {Object}
* Returns: {boolean}

Returns `true` if the given `sandbox` object has been [contextified][] using
[`vm.createContext()`][].

## vm.runInContext(code, contextifiedSandbox[, options])

* `code` {string} The JavaScript code to compile and run.
* `contextifiedSandbox` {Object} The [contextified][] object that will be used
  as the `global` when the `code` is compiled and run.
* `options` {Object|string}
  * `filename` {string} Specifies the filename used in stack traces produced
    by this script.
  * `lineOffset` {number} Specifies the line number offset that is displayed
    in stack traces produced by this script.
  * `columnOffset` {number} Specifies the column number offset that is displayed
    in stack traces produced by this script.
  * `displayErrors` {boolean} When `true`, if an [`Error`][] error occurs
    while compiling the `code`, the line of code causing the error is attached
    to the stack trace.
  * `timeout` {integer} Specifies the number of milliseconds to execute `code`
    before terminating execution. If execution is terminated, an [`Error`][]
    will be thrown. This value must be a strictly positive integer.

The `vm.runInContext()` method compiles `code`, runs it within the context of
the `contextifiedSandbox`, then returns the result. Running code does not have
access to the local scope. The `contextifiedSandbox` object *must* have been
previously [contextified][] using the [`vm.createContext()`][] method.

If `options` is a string, then it specifies the filename.

The following example compiles and executes different scripts using a single
[contextified][] object:

```js
const util = require('util');
const vm = require('vm');

const sandbox = { globalVar: 1 };
vm.createContext(sandbox);

for (let i = 0; i < 10; ++i) {
  vm.runInContext('globalVar *= 2;', sandbox);
}
console.log(util.inspect(sandbox));

// { globalVar: 1024 }
```

## vm.runInNewContext(code[, sandbox[, options]])
<!-- YAML
added: v0.3.1
-->

* `code` {string} The JavaScript code to compile and run.
* `sandbox` {Object} An object that will be [contextified][]. If `undefined`, a
  new object will be created.
* `options` {Object|string}
  * `filename` {string} Specifies the filename used in stack traces produced
    by this script.
  * `lineOffset` {number} Specifies the line number offset that is displayed
    in stack traces produced by this script.
  * `columnOffset` {number} Specifies the column number offset that is displayed
    in stack traces produced by this script.
  * `displayErrors` {boolean} When `true`, if an [`Error`][] error occurs
    while compiling the `code`, the line of code causing the error is attached
    to the stack trace.
  * `timeout` {integer} Specifies the number of milliseconds to execute `code`
    before terminating execution. If execution is terminated, an [`Error`][]
    will be thrown. This value must be a strictly positive integer.
  * `contextName` {string} Human-readable name of the newly created context.
    **Default:** `'VM Context i'`, where `i` is an ascending numerical index of
    the created context.
  * `contextOrigin` {string} [Origin][origin] corresponding to the newly
    created context for display purposes. The origin should be formatted like a
    URL, but with only the scheme, host, and port (if necessary), like the
    value of the [`url.origin`][] property of a [`URL`][] object. Most notably,
    this string should omit the trailing slash, as that denotes a path.
    **Default:** `''`.

The `vm.runInNewContext()` first contextifies the given `sandbox` object (or
creates a new `sandbox` if passed as `undefined`), compiles the `code`, runs it
within the context of the created context, then returns the result. Running code
does not have access to the local scope.

If `options` is a string, then it specifies the filename.

The following example compiles and executes code that increments a global
variable and sets a new one. These globals are contained in the `sandbox`.

```js
const util = require('util');
const vm = require('vm');

const sandbox = {
  animal: 'cat',
  count: 2
};

vm.runInNewContext('count += 1; name = "kitty"', sandbox);
console.log(util.inspect(sandbox));

// { animal: 'cat', count: 3, name: 'kitty' }
```

## vm.runInThisContext(code[, options])
<!-- YAML
added: v0.3.1
-->

* `code` {string} The JavaScript code to compile and run.
* `options` {Object|string}
  * `filename` {string} Specifies the filename used in stack traces produced
    by this script.
  * `lineOffset` {number} Specifies the line number offset that is displayed
    in stack traces produced by this script.
  * `columnOffset` {number} Specifies the column number offset that is displayed
    in stack traces produced by this script.
  * `displayErrors` {boolean} When `true`, if an [`Error`][] error occurs
    while compiling the `code`, the line of code causing the error is attached
    to the stack trace.
  * `timeout` {integer} Specifies the number of milliseconds to execute `code`
    before terminating execution. If execution is terminated, an [`Error`][]
    will be thrown. This value must be a strictly positive integer.

`vm.runInThisContext()` compiles `code`, runs it within the context of the
current `global` and returns the result. Running code does not have access to
local scope, but does have access to the current `global` object.

If `options` is a string, then it specifies the filename.

The following example illustrates using both `vm.runInThisContext()` and
the JavaScript [`eval()`][] function to run the same code:

<!-- eslint-disable prefer-const -->
```js
const vm = require('vm');
let localVar = 'initial value';

const vmResult = vm.runInThisContext('localVar = "vm";');
console.log('vmResult:', vmResult);
console.log('localVar:', localVar);

const evalResult = eval('localVar = "eval";');
console.log('evalResult:', evalResult);
console.log('localVar:', localVar);

// vmResult: 'vm', localVar: 'initial value'
// evalResult: 'eval', localVar: 'eval'
```

Because `vm.runInThisContext()` does not have access to the local scope,
`localVar` is unchanged. In contrast, [`eval()`][] *does* have access to the
local scope, so the value `localVar` is changed. In this way
`vm.runInThisContext()` is much like an [indirect `eval()` call][], e.g.
`(0,eval)('code')`.

## Example: Running an HTTP Server within a VM

When using either [`script.runInThisContext()`][] or
[`vm.runInThisContext()`][], the code is executed within the current V8 global
context. The code passed to this VM context will have its own isolated scope.

In order to run a simple web server using the `http` module the code passed to
the context must either call `require('http')` on its own, or have a reference
to the `http` module passed to it. For instance:

```js
'use strict';
const vm = require('vm');

const code = `
((require) => {
  const http = require('http');

  http.createServer((request, response) => {
    response.writeHead(200, { 'Content-Type': 'text/plain' });
    response.end('Hello World\\n');
  }).listen(8124);

  console.log('Server running at http://127.0.0.1:8124/');
})`;

vm.runInThisContext(code)(require);
```

The `require()` in the above case shares the state with the context it is
passed from. This may introduce risks when untrusted code is executed, e.g.
altering objects in the context in unwanted ways.

## What does it mean to "contextify" an object?

All JavaScript executed within Node.js runs within the scope of a "context".
According to the [V8 Embedder's Guide][]:

> In V8, a context is an execution environment that allows separate, unrelated,
> JavaScript applications to run in a single instance of V8. You must explicitly
> specify the context in which you want any JavaScript code to be run.

When the method `vm.createContext()` is called, the `sandbox` object that is
passed in (or a newly created object if `sandbox` is `undefined`) is associated
internally with a new instance of a V8 Context. This V8 Context provides the
`code` run using the `vm` module's methods with an isolated global environment
within which it can operate. The process of creating the V8 Context and
associating it with the `sandbox` object is what this document refers to as
"contextifying" the `sandbox`.

## Timeout limitations when using process.nextTick(), Promises, and queueMicrotask()

Because of the internal mechanics of how the `process.nextTick()` queue and
the microtask queue that underlies Promises are implemented within V8 and
Node.js, it is possible for code running within a context to "escape" the
`timeout` set using `vm.runInContext()`, `vm.runInNewContext()`, and
`vm.runInThisContext()`.

For example, the following code executed by `vm.runInNewContext()` with a
timeout of 5 milliseconds schedules an infinite loop to run after a promise
resolves. The scheduled loop is never interrupted by the timeout:

```js
const vm = require('vm');

function loop() {
  while (1) console.log(Date.now());
}

vm.runInNewContext(
  'Promise.resolve().then(loop);',
  { loop, console },
  { timeout: 5 }
);
```

This issue also occurs when the `loop()` call is scheduled using
the `process.nextTick()` and `queueMicrotask()` functions.

This issue occurs because all contexts share the same microtask and nextTick
queues.

[`ERR_VM_DYNAMIC_IMPORT_CALLBACK_MISSING`]: errors.html#ERR_VM_DYNAMIC_IMPORT_CALLBACK_MISSING
[`Error`]: errors.html#errors_class_error
[`URL`]: url.html#url_class_url
[`eval()`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/eval
[`script.runInContext()`]: #vm_script_runincontext_contextifiedsandbox_options
[`script.runInThisContext()`]: #vm_script_runinthiscontext_options
[`url.origin`]: url.html#url_url_origin
[`vm.createContext()`]: #vm_vm_createcontext_sandbox_options
[`vm.runInContext()`]: #vm_vm_runincontext_code_contextifiedsandbox_options
[`vm.runInThisContext()`]: #vm_vm_runinthiscontext_code_options
[Module Namespace Object]: https://tc39.github.io/ecma262/#sec-module-namespace-exotic-objects
[V8 Embedder's Guide]: https://github.com/v8/v8/wiki/Embedder's%20Guide#contexts
[contextified]: #vm_what_does_it_mean_to_contextify_an_object
[global object]: https://es5.github.io/#x15.1
[indirect `eval()` call]: https://es5.github.io/#x10.4.2
[origin]: https://developer.mozilla.org/en-US/docs/Glossary/Origin
