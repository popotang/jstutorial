

## 概念

Stream是Node.js处理外部数据的一种方式。

读取外部数据的时候，有两种方式。一种方式是同步处理，即先将数据全部读入内存，然后处理。它的优点是符合直觉，流程非常自然，缺点是如果遇到大文件，要花很长时间读取数据，可能要过很久才能进入数据处理的步骤。另一种方式就是Stream方式，它是系统读取外部数据实际上的方式，即每次只读入数据的一小块，像“流水”一样。所以，Stream方式就是每当系统读入了一小块数据，就会触发一个事件，发出“新数据块”的信号，只要监听这个事件，就能掌握进展，做出相应处理，这样就提高了程序的性能。

Stream方法在Node中被定义成了一个抽象接口，具有readable、writable、drain、data、end、close等事件，既可以读取数据，也可以写入数据。读写数据时，每读入（或写入）一段数据，就会触发一次data事件，全部读取（或写入）完毕，触发end事件。如果发生错误，则触发error事件。

Node内部的很多IO处理都部署了Stream接口，比如HTTP连接、文件读写、标准输入输出等。

数据流通过pipe方法，可以方便地导向其他具有Stream接口的对象。

```javascript
var fs = require('fs');
var zlib = require('zlib');

fs.createReadStream('wow.txt')
  .pipe(zlib.createGzip())
  .pipe(process.stdout);
```

上面代码先打开文本文件wow.txt，然后压缩，再导向标准输出。

```javascript
fs.createReadStream('wow.txt')
  .pipe(zlib.createGzip())
  .pipe(fs.createWriteStream('wow.gz'));
```

上面代码压缩文件wow.txt以后，又将其写回压缩文件。

下面代码新建一个Stream实例，然后指定写入事件和终止事件的回调函数，再将其接到标准输入之上。

```javascript
var stream = require('stream');
var Stream = stream.Stream;

var ws = new Stream;
ws.writable = true;

ws.write = function(data) {
  console.log("input=" + data);
}

ws.end = function(data) {
  console.log("bye");
}

process.stdin.pipe(ws);
```

调用上面的脚本，会产生以下结果。

```bash
$ node pipe_out.js
hello
input=hello
^d
bye
```

上面代码调用脚本下，键入hello，会输出`input=hello`。然后按下ctrl-d，会输出bye。使用管道命令，可以看得更清楚。

```bash
$ echo hello | node pipe_out.js
input=hello

bye
```

## HTTP请求

data事件表示读取或写入了一块数据。

```javascript

req.on('data', function(buf){
  // Do something with the Buffer
});

```

使用req.setEncoding方法，可以设定字符串编码。

```javascript

req.setEncoding('utf8');
req.on('data', function(str){
    // Do something with the String
});

```

end事件，表示读取或写入数据完毕。

```javascript

var http = require('http');

http.createServer(function(req, res){
    res.writeHead(200);
    req.on('data', function(data){
        res.write(data);
    });
    req.on('end', function(){
        res.end();
    });
}).listen(3000);

```

上面代码相当于建立了“回声”服务，将HTTP请求的数据体，用HTTP回应原样发送回去。

system模块提供了pump方法，有点像Linux系统的管道功能，可以将一个数据流，原封不动得转给另一个数据流。所以，上面的例子也可以用pump方法实现。

```javascript

var http = require('http'),
    sys = require('sys');

http.createServer(function(req, res){
    res.writeHead(200);
    sys.pump(req, res);
}).listen(3000);

```

## fs模块

fs模块的createReadStream方法用于新建读取数据流，createWriteStream方法用于新建写入数据流。使用这两个方法，可以做出一个用于文件复制的脚本copy.js。

```javascript

// copy.js

var fs = require('fs');
console.log(process.argv[2], '->', process.argv[3]);

var readStream = fs.createReadStream(process.argv[2]);
var writeStream = fs.createWriteStream(process.argv[3]);

readStream.on('data', function (chunk) {
  writeStream.write(chunk);
});

readStream.on('end', function () {
  writeStream.end();
});

readStream.on('error', function (err) {
  console.log("ERROR", err);
});

writeStream.on('error', function (err) {
  console.log("ERROR", err);
});d all your errors, you wouldn't need to use domains.

```

上面代码非常容易理解，使用的时候直接提供源文件路径和目标文件路径，就可以了。

``` bash

node cp.js src.txt dest.txt

```

Streams对象都具有pipe方法，起到管道作用，将一个数据流输入另一个数据流。所以，上面代码可以重写成下面这样：

``` javascript

var fs = require('fs');
console.log(process.argv[2], '->', process.argv[3]);

var readStream = fs.createReadStream(process.argv[2]);
var writeStream = fs.createWriteStream(process.argv[3]);

readStream.on('open', function () {
  readStream.pipe(writeStream);
});

readStream.on('end', function () {
  writeStream.end();
});

```

## 错误处理

下面是压缩后发送文件的代码。

```javascript
http.createServer(function (req, res) {
  // set the content headers
  fs.createReadStream('filename.txt')
  .pipe(zlib.createGzip())
  .pipe(res)
})
```

上面的代码没有部署错误处理机制，一旦发生错误，就无法处理。所以，需要加上error事件的监听函数。

```javascript
http.createServer(function (req, res) {
  // set the content headers
  fs.createReadStream('filename.txt')
  .on('error', onerror)
  .pipe(zlib.createGzip())
  .on('error', onerror)
  .pipe(res)

  function onerror(err) {
    console.error(err.stack)
  }
})
```

上面的代码还是存在问题，如果客户端中断下载，写入的数据流就会收不到close事件，一直处于等待状态，从而造成内存泄漏。因此，需要使用[on-finished模块](https://github.com/jshttp/on-finished)用来处理这种情况。

```javascript
http.createServer(function (req, res) {
  var stream = fs.createReadStream('filename.txt')

  // set the content headers
  stream
  .on('error', onerror)
  .pipe(zlib.createGzip())
  .on('error', onerror)
  .pipe(res)

  onFinished(res, function () {
    // make sure the stream is always destroyed
    stream.destroy()
  })
})
```

## 参考链接

- James Halliday, [cs294-101-streams-lecture](https://github.com/substack/cs294-101-streams-lecture)
