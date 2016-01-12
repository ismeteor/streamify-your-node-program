# Node Stream
对[Node.js]中[stream]模块的学习积累和理解。

## 什么是流
>A stream is an abstract interface implemented by various objects in Node.js.

这里所谓的“流”，或`stream`，指的是Node.js中`stream`模块提供的接口。

```js
var Stream = require('stream')
var Readable = Stream.Readable
var Writable = Stream.Writable
var Transform = Stream.Transform
var Duplex = Stream.Duplex

```

这些接口提供流式的数据处理功能。

流中具体的数据格式，数据如何产生，如何消耗，需要在实现流时，由用户自己定义。

譬如`fs.createReadStream`，其返回对象便是一个`ReadStream`类的实例。
`ReadStream`继承了`Readable`，并实现了`_read`方法，从而定义了一个具体的可读流。

## 为什么使用流
**场景**：解析日志，统计其中IE6+7的数量和占比。

给定的输入是一个大约400M的文件`ua.txt`。

可能首先想到的是用`fs.readFile`去读取文件内容，再调用`split('\n')`分隔成行，
进行检测和统计。

但在第一步便会碰到问题。

```js
var fs = require('fs')
fs.readFile('example/ua.txt', function (err, body) {
  console.log(body)
  console.log(body.toString())
})

```

输出：
```
⌘ node example/readFile.js
<Buffer 64 74 09 75 61 09 63 6f 75 6e 74 0a 0a 64 74 09 75 61 09 63 6f 75 6e 74 0a 32 30 31 35 31 32 30 38 09 4d 6f 7a 69 6c 6c 61 2f 35 2e 30 20 28 63 6f 6d ... >
buffer.js:382
    throw new Error('toString failed');
    ^

Error: toString failed
    at Buffer.toString (buffer.js:382:11)
    at /Users/zoubin/usr/src/zoub.in/stream-handbook/example/readFile.js:4:20
    at FSReqWrap.readFileAfterClose [as oncomplete] (fs.js:380:3)

```

可见，在读取大文件时，如果使用`fs.readFile`有以下两个问题：
* 很可能只能得到一个`Buffer`，而无法将其转为字符串。因为`Buffer`的`size`过大，`toString`报错了。
* 必须将所有内容都获取，然后才能从内存中访问。这会占用大量内存，同时， 处理的开始时刻被推迟了。

如果文件是几个G，甚至几十个G呢？显然`fs.readFile`无法胜任读取的工作。

在这样的场景下，应当使用`fs.createReadStream`，以流的行式来读取和处理文件：

```js
// example/parseUA.js

var fs = require('fs')
var split = require('split2')

fs.createReadStream(__dirname + '/ua.txt')  // 流式读取文件
  .pipe(split())                            // 分隔成行
  .pipe(createParser())                     // 逐行解析
  .pipe(process.stdout)                     // 打印

function createParser() {
  var Stream = require('stream')
  var csi = require('ansi-escape')
  var ie67 = 0
  var total = 0

  function format() {
    return csi.cha.eraseLine.escape(
      'Total: ' + csi.green.escape('%d') +
      '\t' +
      'IE6+7: ' + csi.red.escape('%d') +
      ' (%d%%)',
      total,
      ie67,
      Math.floor(ie67 * 100 / total)
    )
  }

  return Stream.Transform({
    transform: function (line, _, next) {
      total++
      line = line.toString()
      if (/MSIE [67]\.0/.test(line)) {
        ie67++
      }
      next(null, format())
    },
  })
}

```

最终结果：
```
⌘ node example/parseUA.js
Total: 2888380  IE6+7: 783730 (27%)

```

**要点**
* 大数据情况下必须使用流式处理

## Readable
可读流的功能是作为上游，提供数据给下游。

本文中用`readable`来指代一个`Readable`实例。

### 数据产生方式
可读流通过`push`方法产生数据，存入`readable`的缓存中。
当调用`push(null)`时，便宣告了流的数据产生的结束。

正常情况下，需要为流实例提供一个`_read`方法，在这个方法中调用`push`产生数据。
既可以在同一个tick中（同步）调用`push`，也可以异步的调用（通常如此）。

在需要数据时，流内部会自动调用`_read`方法来往缓存中添加数据。

```js
var Stream = require('stream')

var readable = Stream.Readable()

var source = ['a', 'b', 'c']

readable._read = function () {
  this.push(source.shift() || null)
}

```

或
```js
var Stream = require('stream')

var source = ['a', 'b', 'c']
var readable = Stream.Readable({
  read: function () {
    this.push(source.shift() || null)
  },
})

```

在**数据被消耗完**时，会触发`readable`的`end`事件。
所谓“消耗完”，需要满足两个条件：
* 已经调用`push(null)`，声明不会再有任何新的数据产生
* 缓存中的数据也被读取完

```js
var Stream = require('stream')

var source = ['a', 'b', 'c']
var readable = Stream.Readable({
  read: function () {
    var data = source.shift() || null
    console.log('push', data)
    this.push(data)
  },
})

readable.on('end', function () {
  console.log('end')
})

readable.on('data', function (data) {
  console.log('data', data)
})

```

输出：
```
⌘ node example/event-end.js
push a
push b
data <Buffer 61>
push c
data <Buffer 62>
push null
data <Buffer 63>
end

```

**要点**
* 必须调用`push(null)`来结束流，否则下游会一直等待。
* `push`可以同步调用，也可异步调用。
* `end`事件表示可读流中的数据已被完全消耗。

### 两种数据消耗模式
下游通过监听`data`事件（`flowing`模式）或通过调用`read`方法（`paused`模式），
从缓存中获取数据进行消耗。

#### flowing模式
在`flowing`模式下，`readable`的数据会持续不断的生产出来，
每个数据都会触发一次`data`事件，通过监听该事件来获得数据。

以下条件均可以使`readable`进入`flowing`模式：
* 调用`resume`方法
* 如果之前未调用`pause`方法进入`paused`模式，则监听`data`事件也会调用`resume`方法。
* `readable.pipe(writable)`。`pipe`中会监听`data`事件。

[例子](example/flowing-mode.js)：

```js
var Stream = require('stream')

var source = ['a', 'b', 'c']

var readable = Stream.Readable({
  read: function () {
    this.push(source.shift() || null)
  },
})

readable.on('data', function (data) {
  console.log(data)
})

```

输出结果：

```
⌘ node example/flowing-mode.js
<Buffer 61>
<Buffer 62>
<Buffer 63>

```

通常，并不直接监听`data`事件去消耗流，而是通过[`pipe`](#pipe)方法去消耗。

#### paused模式
在`paused`模式下，通过`readable.read`去获取数据。

以下条件均可使`readable`进入`paused`模式：
* 流创建完成，即初始状态
* 在`flowing`模式下调用`pause`方法
* 通过`unpipe`移除所有下游

除了初始状态外，很少会在`paused`模式下使用流。
一般不会调用`pause`方法或是`unpipe`方法从`flowing`模式切换至`paused`模式。

[例子](example/paused-mode.js)：

```js
var Stream = require('stream')

var source = ['a', 'b', 'c']

var readable = Stream.Readable({
  read: function () {
    this.push(source.shift() || null)
  },
})

readable.on('readable', function () {
  var data
  while (data = this.read()) {
    console.log(data)
  }
})

```

输出结果：

```
⌘ node example/paused-mode.js
<Buffer 61 62>
<Buffer 63>

```

`readable`事件表示流中产生了新数据，或是到了流的尽头。
`read(n)`时，会从缓存中试图读取相应的字节数。
当`n`未指定时，会一次将缓存中的字节全部读取。

如果在`flowing`模式下调用`read()`，不指定`n`时，会逐个元素地将缓存读完，
而不像`paused`模式会一次全部读取。

#### 要点
两种最常见的消耗可读流的方法：
* 监听`data`事件和`end`事件
* 调用`pipe()`方法

## Writable
可写流的功能是作为下游，消耗上游提供的数据。

本文中用`writable`来指代一个`Writable`实例。

### 实现可写流与使用可写流
与`Readable`类似，需要为`writable`实现一个`_write`方法，
来实现一个具体的可写流。

在写入数据（调用`writable.write(data)`）时，
会调用`_write`方法来处理数据。

```js
var Stream = require('stream')

var writable = Stream.Writable({
  write: function (data, _, next) {
    console.log(data)
    process.nextTick(next)
  },
})

writable.write('a')
writable.write('b')
writable.write('c')
writable.end()

```

或：
```js
var Stream = require('stream')

var writable = Stream.Writable()

writable._write = function (data, _, next) {
  console.log(data)
  process.nextTick(next)
}

writable.write('a')
writable.write('b')
writable.write('c')
writable.end()

```

输出：
```
⌘ node example/writable.js
<Buffer 61>
<Buffer 62>
<Buffer 63>

```

**实现要点**
* `_write(data, _, next)`中调用`next(err)`来声明“写入”操作已完成，
  可以开始写入下一个数据。
* `next`的调用时机可以是异步的。

**使用要点**
* 调用`write(data)`方法来往`writable`中写入数据。将触发`_write`的调用，将数据写入底层。
* 必须调用`end()`方法来告诉`writable`，所有数据均已写入。

### 写操作完成事件
与`Readable`的`end`事件类似，`Writable`有两个事件来表示所有写入完成的状况：

```js
var Stream = require('stream')

var writable = Stream.Writable({
  write: function (data, _, next) {
    console.log(data)
    next()
  },
})

writable.on('finish', function () {
  console.log('finish')
})

writable.on('prefinish', function () {
  console.log('prefinish')
})

writable.write('a', function () {
  console.log('write a')
})
writable.write('b', function () {
  console.log('write b')
})
writable.write('c', function () {
  console.log('write c')
})
writable.end()

```

输出：
```
⌘ node example/event-finish.js
<Buffer 61>
<Buffer 62>
<Buffer 63>
prefinish
write a
write b
write c
finish

```

**要点**
* `prefinish`事件。表示所有数据均已写入底层系统，即最后一次`_write`的调用，其`next`已被执行。此时，不会再有新的数据写入，缓存中也无积累的待写数据。
* `finish`事件。在`prefinish`之后，表示所有在调用`write(data, cb)`时传入的`cb`均已执行完。
* 一般监听`finish`事件，来判断写操作完成。

## objectMode
在介绍[`Readable`](#readable)时，从例子中可以发现一个现象，
生产数据时传给`push`的`data`是字符串或`null`，
而消耗时拿到的却是`Buffer`类型。
接下来探讨一下流中数据类型的问题。

### Readable({ objectMode: true })
在创建可读流时，可指定`objectMode`选项为`true`。
此时，称为一个`objectMode`流。
否则，称其为一个非`objectMode`流。

这个选项将影响`push(data)`中`data`的类型，以及消耗时获得的数据的类型：
* 在非`objectMode`时，`data`只能是`String`, `Buffer`, `Null`, `Undefined`。
  同时，消耗时获得的数据一定是`Buffer`类型。
* 在`objectMode`时，`data`可以是任意类型，`null`仍然有其特殊含义。
  同时，消耗时获得的数据与`push`进来的一样。实际就是同一个引用。

所谓“缓存”，其实也就是一个数组。可以看看`readable._readableState.buffer`。

每次调用`push(data)`时，如果是`objectMode`，便直接调用`state.buffer.push(data)`。
这里，`state = readable._readableState`。

如果是非`objectMode`，会将`String`类型转成`Buffer`，再调用`state.buffer.push(chunk)`。
这里，`chunk`即转换后的`Buffer`对象。
默认会以`utf8`的编码形式进行转换。
设置方法查[文档](https://nodejs.org/api/stream.html#stream_new_stream_readable_options)即可。
一般不需要。

在消耗`objectMode`流时，不管是`flowing`模式，还是`paused`模式，
都等同于调用`state.buffer.shift()`拿到数据。
保证`push`产生的数据会被一一消耗。

在消耗非`objectMode`流时，`flowing`模式仍然等同于调用`state.buffer.shift()`。
但`paused`模式则会拼接字节数，以满足`readable.read(n)`中`n`的要求。
如果`n`没指定，则会一次将`state.buffer`所有字节拼起来消耗掉。

这里看一个比较有意思的例子来说明`objectMode`与非`objectMode`的区别。

非`objectMode`下`push('')`：
```js
var Stream = require('stream')

var source = ['a', '', 'c']
var readable = Stream.Readable({
  read: function () {
    var data = source.shift()
    data = data == null ? null : data
    this.push(data)
  },
})

readable.on('end', function () {
  console.log('end')
})

readable.on('data', function (data) {
  console.log('data', data)
})


```

输出：
```
⌘ node example/empty-string-non-objectMode.js
data <Buffer 61>
data <Buffer 63>
end

```

`objectMode`下`push('')`：
```js
var Stream = require('stream')

var source = ['a', '', 'c']
var readable = Stream.Readable({
  objectMode: true,
  read: function () {
    var data = source.shift()
    data = data == null ? null : data
    this.push(data)
  },
})

readable.on('end', function () {
  console.log('end')
})

readable.on('data', function (data) {
  console.log('data', data)
})

```

输出：
```
⌘ node example/empty-string-objectMode.js
data a
data
data c
end

```

可见，非`objectMode`下直接将`push('')`给忽略了，
而`objectMode`下在消耗时能拿到这个空字符串。

（注意，非`objectMode`时`push('')`实际是会修改内部状态的，
会有一定的副作用，一般不要如此。
见[这里](https://nodejs.org/api/stream.html#stream_stream_push)）

**要点**
* `objectMode`时，可`push`任意类型的数据，消耗时会逐个消耗同样的数据
* 非`objectMode`时，只能`push`以下数据类型：`String`, `Buffer`, `Null`, `Undefined`。
  在消耗时只能拿到`Buffer`类型的数据

### Writable({ objectMode: true })

## highWaterMark

## pipe
`pipe`方法用于连通上游和下游，使上游的数据能流到指定的下游：`upstream.pipe(downstream)`。
上游必须是可读的，下游必须是可写的。

### 可读流与可写流配合使用
再看看如何将前面创建的可读流与可写流结合使用。

事件关联：

```js
var Stream = require('readable-stream')

var readable = createReadable()
var writable = createWritable()

readable.on('data', function (data) {
  writable.write(data)
})
readable.on('end', function (data) {
  writable.end()
})

writable.on('finish', function () {
  console.log('done')
})

function createWritable() {
  return Stream.Writable({
    write: function (data, _, next) {
      console.log(data)
      next()
    },
  })
}

function createReadable() {
  var source = ['a', 'b', 'c']

  return Stream.Readable({
    read: function () {
      process.nextTick(this.push.bind(this), source.shift() || null)
    },
  })
}

```

输出：
```
⌘ node example/readable-with-writable.js
<Buffer 61>
<Buffer 62>
<Buffer 63>
done

```

还可以使用`pipe`关联：
```js
var Stream = require('readable-stream')

var readable = createReadable()
var writable = createWritable()

readable.pipe(writable).on('finish', function () {
  console.log('done')
})

function createWritable() {
  return Stream.Writable({
    write: function (data, _, next) {
      console.log(data)
      next()
    },
  })
}

function createReadable() {
  var source = ['a', 'b', 'c']

  return Stream.Readable({
    read: function () {
      process.nextTick(this.push.bind(this), source.shift() || null)
    },
  })
}

```


### 工作原理

#### 使数据从上游流向下游
监听`upstream`的`data`事件，将获得的数据写入`downstream`。

当数据耗尽时，给下游结束信号（即自动调用`downstream.end()`，可通过`pipe`的第二个参数修改这个行为）。

#### 响应下游反馈
前面一步中数据的生产速度取决于上游，但是下游的消耗速度不一定跟得上，
未能及时消耗的数据便会存储至`downstream`的缓存中。
如果不做任何处理，这个缓存可能会快速膨胀，耗掉大量内存，下游的压力就变得很大。

所以，需要将下游的消耗情况反馈给上游，在必要的时候让上游暂停数据的生产：

在调用`downstream.write(data)`时，如果返回`false`（表示`downstream`缓存已到临界点），
便调用`upstream.pause()`，使上游进入`paused`模式，暂停数据的生产，同时等待下游的继续信号（即监听`downstream`的`drain`事件），
一旦

正是因为这个反馈的功能的存在，所以很多情况下推荐直接使用`pipe`方法去连接两个流。

### 使用案例
`pipe`最简单的使用情况便是：

```js
a.pipe(b).pipe(c)

```

这种链式写法之所以正确，是因为`pipe`方法会返回下游的`stream`对象。
所以，当下游（如此处的`b`）是一个既可写双可读的流（如`Transform`或`Duplex`）时，
便可以链起来了。

#### 多个下游
从前面的原理介绍中，可以知道，`pipe`会通过监听`data`事件的方式，
在`flowing`模式下消耗上游的数据。
所以，我们还可以这样：

```js
a.pipe(b)
a.pipe(c)

```

这样，`a`产生的数据会出来两份，分别给两个下游`b`和`c`。

这里需要注意的是：
* `b`或`c`中有一个要求暂停时，`a`便会暂停。
* 所谓两份数据，在非字符串类型的情况下，实际是一份数据的两份引用。

#### 多个上游
自然，还可以这样：

```js
a.pipe(c)
b.pipe(c)

```

不过实际上不会这么用。前面在原理中介绍过，
`pipe`在上游结束时会自动调用下游的`end`方法。
当`c`被`a`结束时，`b`便不能再往`c`中写数据了（这是`Writable`的特性）。

但这种合并多个流的需求是真实存在的。可以这样去做：
```js
a.pipe(c, { end: false })
b.pipe(c, { end: false })

a.once('end', remove)
b.once('end', remove)

```

不让上游结束下游，同时维护一下还未结束的上游数目，
减少到为0时，再结束下游。

这个便是[`merge-stream`]这个工具所做的事。

```js
merge(a, b).pipe(c)

```

还可能碰到一种情况，即上游数是动态增加的。
可以这么使用：

```js
var wait = require('stream').PassThough()
var upstream = merge(wait)

upstream.pipe(c)

// 异步添加a作为上游
process.nextTick(upstream.add.bind(upstream), a)

// 异步添加b作为上游
process.nextTick(upstream.add.bind(upstream), b)

// 确定不会再增加上游时
process.nextTick(wait.end.bind(wait))

```

由于合并多个上游得到的`upstream`要等所有上游结束，
自己才会结束，所以便可以通过添加一个空的`stream`来控制`upstream`的结束时机。


### writable
* `objectMode`的含义
* `highWaterMark`的含义
* `.write()`返回值的含义
* `.end()`的含义
* 事件`drain`、`finish`、`prefinish`的含义

### readable
* flowing mode, paused mode
* `objectMode`的含义
* `highWaterMark`的含义
* `.read()`返回值的含义
* `.push()`的含义
* `.pipe()`, `unpipe()`的含义
* 事件`data`、`readable`、`end`的含义


[Node.js]: https://nodejs.org/
[stream]: https://nodejs.org/api/stream.html
[`stream-combiner2`]: https://github.com/substack/stream-combiner2
[`substack#stream-handbook`]: https://github.com/substack/stream-handbook
[`through2`]: https://github.com/rvagg/through2
[`concat-stream`]: https://github.com/maxogden/concat-stream
[`duplexer2`]: https://github.com/deoxxa/duplexer2
[`labeled-stream-splicer`]: https://github.com/substack/labeled-stream-splicer
[`merge-stream`]: https://github.com/grncdr/merge-stream
[`gulp`]: https://github.com/gulpjs/gulp
[`browserify`]: https://github.com/substack/node-browserify

## Duplex

## Transform

