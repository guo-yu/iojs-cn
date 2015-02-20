# Errors

<!--type=misc-->

由 io.js 生成的错误共有两类：JavaScript 错误与系统（system）错误。所有错误或是 JavaScript 的原生错误 [Error](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error) 类型的实例，或继承于此类型，这些错误都确定**至少具有** Error 类型的属性。

当某个操作由于语法错误或者语言运行时错误不被允许执行时，一个 **JavaScript 错误** 被生成，作为 **exception** 抛出。当某个操作由于系统级别的原因不被允许执行时，一个 **system 错误** 被生成，取决于具体 API 如何**广播**错误，客户端代码能够自由选择是否对此错误进行截断。

调用 API 的方法决定了如何传递之后生成的错误，比如**广播**错误给客户端代码，也进而通知客户端代码如何**截取**错误。通常来说，我们可以使用 `try / catch` 语句截取异常，其他错误传递的策略会在[以下文档](#errors_error_propagation_and_interception)中被提到。

## JavaScript Errors

<!--type=misc-->

通常，JavaScript errors 表示一个 API 正被不正确地使用，或者正在执行的程序本身有内部错误。

### Class: Error

<!--type=class-->

表示通用错误对象，与其他错误对象不同，`Error` 类型的实例并不表征该错误生成的具体环境。相反地，错误在生成时会捕捉一个「错误堆栈」（stack trace）详细地记录程序执行到生成错误的位点，也可能提供这个错误的介绍。

**注意**： 无论是对于 system 错误或 JavaScript 错误，io.js 都会从此基本类型派生错误实例。

#### new Error(message)

初始化一个新的错误对象，并且用给定的错误描述，设置这个对象的 `.message` 属性。这个错误实例的 `.stack` 属性将会标识出它在程序中，通过 `new Error` 语句初始化的位点。错误堆栈遵循 [V8 的 stack trace API](https://code.google.com/p/v8-wiki/wiki/JavaScriptStackTraceApi) 文档规范生成。一个错误堆栈只能延伸到程序代码同步执行的开始位置，*或* 通过设置 `Error.stackTraceLimit` 的数量决定追溯的最顶层数，当然，后者的追溯层数只能比前者更少。

#### error.message

在初始化 `Error()` 类型实例时传递的错误描述字符串，这段描述也同时会出现在错误堆栈的第一行，不过，改变这个属性 *并不一定会* 改变错误堆栈打印出的第一行。

#### error.stack

当**访问** `error.stack` 属性时，会返回当前程序运行时出现此错误的描述。以下是一个错误堆栈（stacktrace）的例子：

    Error: Things keep happening!
       at /home/gbusey/file.js:525:2
       at Frobnicator.refrobulate (/home/gbusey/business-logic.js:424:21)
       at Actor.<anonymous> (/home/gbusey/actors.js:400:8)
       at increaseSynergy (/home/gbusey/actors.js:701:6)

错误堆栈的第一行被格式化为 `<error class name>: <error message>` 在此之后的是一系列的错误层级（每一行由 "at " 单词开始）每一个错误层级描述了在此层级触发错误的调用语句。V8 引擎会尝试给每个函数展示一个有效的名字（由变量名，函数名，或者对象方法名称组成），不过偶尔会出现一个不太合适的名字。如果 v8 无法找到这个函数的名称，将会只有函数调用的位置信息展示出来，否则，会以函数名附加上错误位置信息的形式展示出来。

错误堆栈**只会** JavaScript 函数层面触发，比如在下边这个例子中：程序顺序执行经过一个 C++ addon 函数 `cheetahify`，此函数中内部调用了一个 JavaScript 函数，所以错误堆栈中将 **不会** 显示出此 C++ 函数的名称。

```javascript
var cheetahify = require('./native-binding.node');

function makeFaster() {
  // cheetahify *synchronously* calls speedy.
  cheetahify(function speedy() {
    throw new Error('oh no!');
  });
}

makeFaster(); // will throw:
// /home/gbusey/file.js:6
//     throw new Error('oh no!');
//           ^
// Error: oh no!
//     at speedy (/home/gbusey/file.js:6:11)
//     at makeFaster (/home/gbusey/file.js:5:3)
//     at Object.<anonymous> (/home/gbusey/file.js:10:1)
//     at Module._compile (module.js:456:26)
//     at Object.Module._extensions..js (module.js:474:10)
//     at Module.load (module.js:356:32)
//     at Function.Module._load (module.js:312:12)
//     at Function.Module.runMain (module.js:497:10)
//     at startup (node.js:119:16)
//     at node.js:906:3
```

错误堆栈中显示的位置信息会是以下三种之一：

* `native`, 原生错误。如果此堆栈表示错误出现在 v8 内部逻辑（比如 `[].forEach`）
* `plain-filename.js:line:column`, 如果此错误堆栈表示触发在 io.js 的内部调用中。
* `/absolute/path/to/file.js:line:column`, 当错误触发在用户编写的程序或者其依赖模块中时。

值得注意的一点是，只有当错误**被访问**时错误堆栈的描述才会生成，换句话说，这些错误描述并非预先生成。

错误堆栈的数量受限于 `Error.stackTraceLimit` 的最大数目，或者当前事件循环中的最大框架层级。

系统级别的错误以 augmented Error 的实例存在。以下文档将[详述](#errors_system_errors)

#### Error.captureStackTrace(targetObject[, constructorOpt])

在目标对象 `targetObject` 上创建 `.stack` 属性，当访问这个属性时，返回程序执行中调用 `Error.captureStackTrace` 的位置。

```javascript
var myObject = {};

Error.captureStackTrace(myObject);

myObject.stack  // similar to `new Error().stack`
```

不过，与直接使用 `new Error().stack` 不同的是，错误堆栈的第一行并非格式化为 `ErrorType:
message` 而是 `targetObject.toString()`. (?)

此函数的第二个可选参数 `constructorOpt` 接受一个函数。如果传入，错误堆栈中的所有在 `constructorOpt` 之上（包含 `constructorOpt`）都将被忽略。

这个接口在针对最终的模块使用者，隐藏错误具体实现描述的时候很有用处。一个通常的做法是将当前自定义 Error 的构造函数传给它：

```javascript

function MyError() {
  Error.captureStackTrace(this, MyError);
}

// without passing MyError to captureStackTrace, the MyError
// frame would should up in the .stack property. by passing
// the constructor, we omit that frame and all frames above it.
new MyError().stack

```

#### Error.stackTraceLimit

这个属性决定了错误堆栈将会收集的堆栈层数（无论该错误堆栈是以 `new Error().stack` 的方法生成还是以 `Error.captureStackTrace(obj)` 的方法生成）

此属性的缺省值是 `10`，可以将此属性设置为任何 JavaScript 数字类型，这将影响所有**修改之后**的错误堆栈的生成。如果将此属性设置为非数字类型，错误堆栈将不会捕捉任何层面的错误，并且在访问时输出 `undefined`

### Class: RangeError

RangeError 是 Error 类的一个子类，用以表示调用时提供的参数不在函数预设值范围之内的错误，无论是预设的是数字范围，还是一系列缺省配置参数，都可以用这个错误来标识，例子如下：

```javascript
require('net').connect(-1);  // throws RangeError, port should be > 0 && < 65536
```

io.js 此时会**立即**生成并抛出一个 RangeError 实例，这其实是一定意义上的参数校验。

### Class: TypeError

TypeError 是 Error 类的子类，用以表示此函数提供的参数不是给定的类型。比如，给一个指定传参类型为字符串的函数传递一个 function 会被认为是 TypeError:

```javascript
require('url').parse(function() { }); // throws TypeError, since it expected a string
```

io.js 将会生成比**立即**抛出一个 TypeError 的实例，这也是一定意义上的参数校验。

### Class: ReferenceError

ReferenceError 是 Error 类的子类，用以表示访问的变量未定义的错误。一般来说这种错误产生的原因是变量名称写错的笔误，或者程序写错。虽然客户端代码可以触发并抛出此类错误，但实际上只有 v8 会抛出此类错误。

```javascript
doesNotExist; // throws ReferenceError, doesNotExist is not a variable in this program.
```
ReferenceError 错误的实例均会有一个 `.arguments` 属性，此属性是一个数组，并包括一个字符串元素，表示了该未定义的变量名称:

```javascript
try {
  doesNotExist;
} catch(err) {
  err.arguments[0] === 'doesNotExist';
}
```

除非客户端代码是动态生成并运行，否则 ReferenceErrors 都应该被视为这段代码或其依赖的模块存在 bug

### Class: SyntaxError

A subclass of Error that indicates that a program is not valid JavaScript.
These errors may only be generated and propagated as a result of code
evaluation. Code evaluation may happen as a result of `eval`, `Function`,
`require`, or [vm](vm.html). These errors are almost always indicative of a broken
program.

```javascript
try {
  require("vm").runInThisContext("binary ! isNotOk");
} catch(err) {
  // err will be a SyntaxError
}
```

SyntaxErrors are unrecoverable from the context that created them – they may only be caught
by other contexts.

### Exceptions vs. Errors

<!--type=misc-->

A JavaScript "exception" is a value that is thrown as a result of an invalid operation or
as the target of a `throw` statement. While it is not required that these values inherit from
`Error`, all exceptions thrown by io.js or the JavaScript runtime *will* be instances of Error.

Some exceptions are *unrecoverable* at the JavaScript layer. These exceptions will always bring
down the process. These are usually failed `assert()` checks or `abort()` calls in the C++ layer.

## System Errors

System errors are generated in response to a program's runtime environment.
Ideally, they represent operational errors that the program needs to be able to
react to. They are generated at the syscall level: an exhaustive list of error
codes and their meanings is available by running `man 2 intro` or `man 3 errno`
on most Unices; or [online](http://man7.org/linux/man-pages/man3/errno.3.html).

In io.js, system errors are represented as augmented Error objects -- not full
subclasses, but instead an error instance with added members.

### Class: System Error

#### error.syscall

A string representing the [syscall](http://man7.org/linux/man-pages/man2/syscall.2.html) that failed.

#### error.errno
#### error.code

A string representing the error code, which is always `E` followed by capital
letters, and may be referenced in `man 2 intro`.

### Common System Errors

This list is **not exhaustive**, but enumerates many of the common system errors when
writing a io.js program. An exhaustive list may be found [here](http://man7.org/linux/man-pages/man3/errno.3.html).

#### EPERM: Operation not permitted

An attempt was made to perform an operation that requires appropriate
privileges.

#### ENOENT: No such file or directory

Commonly raised by [fs](fs.html) operations; a component of the specified pathname
does not exist -- no entity (file or directory) could be found by the given path.

#### EACCES: Permission denied

An attempt was made to access a file in a way forbidden by its file access
permissions.

#### EEXIST: File exists

An existing file was the target of an operation that required that the target
not exist.

#### ENOTDIR: Not a directory

A component of the given pathname existed, but was not a directory as expected.
Commonly raised by [fs.readdir](fs.html#fs_fs_readdir_path_callback).

#### EISDIR: Is a directory

An operation expected a file, but the given pathname was a directory.

#### EMFILE: Too many open files in system

Maxiumum number of [file descriptors](http://en.wikipedia.org/wiki/File_descriptor) allowable on the system has
been reached, and requests for another descriptor cannot be fulfilled until
at least one has been closed.

Commonly encountered when opening many files at once in parallel, especially
on systems (in particular, OS X) where there is a low file descriptor limit
for processes. To remedy a low limit, run `ulimit -n 2048` in the same shell
that will run the io.js process.

#### EPIPE: Broken pipe

A write on a pipe, socket, or FIFO for which there is no process to read the
data. Commonly encountered at the [net](net.html) and [http](http.html) layers, indicative that
the remote side of the stream being written to has been closed.

#### EADDRINUSE: Address already in use

An attempt to bind a server ([net](net.html), [http](http.html), or [https](https.html)) to a local
address failed due to another server on the local system already occupying
that address.

#### ECONNRESET: Connection reset by peer

A connection was forcibly closed by a peer. This normally results
from a loss of the connection on the remote socket due to a timeout
or reboot. Commonly encountered via the [http](http.html) and [net](net.html) modules.

#### ECONNREFUSED: Connection refused

No connection could be made because the target machine actively refused
it. This usually results from trying to connect to a service that is inactive
on the foreign host.

#### ENOTEMPTY: Directory not empty

A directory with entries was the target of an operation that requires
an empty directory -- usually [fs.unlink](fs.html#fs_fs_unlink_path_callback).

#### ETIMEDOUT: Operation timed out

A connect or send request failed because the connected party did not properly
respond after a period of time. Usually encountered by [http](http.html) or [net](net.html) --
often a sign that a connected socket was not `.end()`'d appropriately.

## Error Propagation and Interception

<!--type=misc-->

All io.js APIs will treat invalid arguments as exceptional -- that is, if passed
invalid arguments, they will *immediately* generate and throw the error as an
exception, even if they are an otherwise asynchronous API.

Synchronous APIs (like
[fs.readFileSync](fs.html#fs_fs_readfilesync_filename_options)) will throw the
error. The act of *throwing* a value (in this case, the error) turns the value
into an **exception**.  Exceptions may be caught using the `try { } catch(err)
{ }` construct.

Asynchronous APIs have **two** mechanisms for error propagation; one mechanism
for APIs that represent a single operation, and one for APIs that represent
multiple operations over time.

### Node style callbacks

<!--type=misc-->

Single operation APIs take "node style callbacks" -- a
function provided to the API as an argument. The node style callback takes
at least **one** argument -- `error` -- that will either be `null` (if no error
was encountered) or an `Error` instance.  For instance:

```javascript
var fs = require('fs');

fs.readFile('/some/file/that/does-not-exist', function nodeStyleCallback(err, data) {
  console.log(err)  // Error: ENOENT
  console.log(data) // undefined / null
});

fs.readFile('/some/file/that/does-exist', function(err, data) {
  console.log(err)  // null
  console.log(data) // <Buffer: ba dd ca fe>
})
```

Note that `try { } catch(err) { }` **cannot** intercept errors generated by
asynchronous APIs. A common mistake for beginners is to try to use `throw`
inside their node style callback:

```javascript
// THIS WILL NOT WORK:
var fs = require('fs');

try {
  fs.readFile('/some/file/that/does-not-exist', function(err, data) {
    // mistaken assumption: throwing here...
    if (err) {
      throw err;
    }
  });
} catch(err) {
  // ... will be caught here -- this is incorrect!
  console.log(err); // Error: ENOENT
}
```

This will not work! By the time the node style callback has been called, the
surrounding code (including the `try { } catch(err) { }` will have already
exited. Throwing an error inside a node style callback **will crash the process** in most cases.
If [domains](domain.html) are enabled, they may intercept the thrown error; similarly, if a
handler has been added to `process.on('uncaughtException')`, it will intercept
the error.

### Error events

<!--type=misc-->

The other mechanism for providing errors is the "error" event. This is
typically used by [stream-based](stream.html) and [event emitter-based](events.html#events_class_events_eventemitter) APIs, which
themselves represent a series of asynchronous operations over time (versus a
single operation that may pass or fail). If no "error" event handler is
attached to the source of the error, the error will be thrown. At this point,
it will crash the process as an unhandled exception unless [domains](domain.html) are
employed appropriately or [process.on('uncaughtException')](process.html#process_event_uncaughtexception) has a handler.

```javascript
var net = require('net');

var connection = net.connect('localhost');

// adding an "error" event handler to a stream:
connection.on('error', function(err) {
  // if the connection is reset by the server, or if it can't
  // connect at all, or on any sort of error encountered by
  // the connection, the error will be sent here.
  console.error(err);
});

connection.pipe(process.stdout);
```

The "throw when no error handlers are attached behavior" is not limited to APIs
provided by io.js -- even user created event emitters and streams will throw
errors when no error handlers are attached. An example:

```javascript
var events = require('events');

var ee = new events.EventEmitter;

setImmediate(function() {
  // this will crash the process because no "error" event
  // handler has been added.
  ee.emit('error', new Error('This will crash'));
});
```

As with node style callbacks, errors generated this way *cannot* be intercepted
by `try { } catch(err) { }` -- they happen *after* the calling code has already
exited.
