# 스트림-핸드북

이 문서는 how to write [node.js](http://nodejs.org/)
programs with [streams](http://nodejs.org/docs/latest/api/stream.html) 의 내용을 담고 있다.  
**[중국어 버전](https://github.com/jabez128/stream-handbook)** 도 제공되고 있다.

[![cc-by-3.0](http://i.creativecommons.org/l/by/3.0/80x15.png)](http://creativecommons.org/licenses/by/3.0/)

# 핸드북을 node 패키지로 설치하기

이 핸드북은 npm을 이용하여 설치할 수 있다.

```
npm install -g stream-handbook
```

`stream-handbook` 명령을 통해 이 README 파일의 내용을 확인할 수 있으며 `$PAGER` 프로그램들을 통해 읽을 수 있다.  
현재 당신이 이 글을 읽고 있는 방식으로도 계속 읽을 수 있음은 물론이다.

# 소개

```
"서로 다른 방식으로 데이터를 조작하는 것이 필요해졌을 때
프로그램들을 정원 호스의 이음새처럼 연결 해줄 수 있는 방법이 필요하다.  
이것은 입출력이 처리되는 방법이기도 하다."
```

[Doug McIlroy. October 11, 1964](http://cm.bell-labs.com/who/dmr/mdmpipe.html)

![doug mcilroy](http://substack.net/images/mcilroy.png)

***

Streams come to us from the
[earliest days of unix](http://www.youtube.com/watch?v=tc4ROCJYbm0)
and have proven themselves over the decades as a dependable way to compose large
systems out of small components that
[do one thing well](http://www.faqs.org/docs/artu/ch01s06.html).
In unix, streams are implemented by the shell with `|` pipes.
In node, the built-in
[stream module](http://nodejs.org/docs/latest/api/stream.html)
is used by the core libraries and can also be used by user-space modules.
Similar to unix, the node stream module's primary composition operator is called
`.pipe()` and you get a backpressure mechanism for free to throttle writes for
slow consumers.

Streams can help to
[separate your concerns](http://www.c2.com/cgi/wiki?SeparationOfConcerns)
because they restrict the implementation surface area into a consistent
interface that can be
[reused](http://www.faqs.org/docs/artu/ch01s06.html#id2877537).
You can then plug the output of one stream to the input of another and
[use libraries](http://npmjs.org) that operate abstractly on streams to
institute higher-level flow control.

Streams are an important component of
[small-program design](https://michaelochurch.wordpress.com/2012/08/15/what-is-spaghetti-code/)
and [unix philosophy](http://www.faqs.org/docs/artu/ch01s06.html)
but there are many other important abstractions worth considering.
Just remember that [technical debt](http://c2.com/cgi/wiki?TechnicalDebt)
is the enemy and to seek the best abstractions for the problem at hand.

![brian kernighan](http://substack.net/images/kernighan.png)

***

# 왜 스트림을 사용해야 하는가

node에서의 입출력은 비동기적이다. 따라서 디스크와 네트워크에 대한 상호작용은 콜백함수의 전달이 수반된다.  
당신이 디스크에 있는 한 파일을 제공하는 서버를 코딩한다면 다음과 같이 하고 싶을지도 모르겠다:

``` js
var http = require('http');
var fs = require('fs');

var server = http.createServer(function (req, res) {
    fs.readFile(__dirname + '/data.txt', function (err, data) {
        res.end(data);
    });
});
server.listen(8000);
```

위 코드는 잘 동작한다. 하지만 매 요청마다 `data.txt` 파일 전체를 메모리 버퍼에 올려야 하므로 효율적이지 않다.   
만약 `data.txt`가 아주 크다면 이 프로그램은 아주 많은 메모리를 잡아먹게 될 것이며   
동시에 이용하는 사용자가 늘어날수록, 특히 느린 처리속도를 가진 사용자가 늘어날수록 더욱 그럴 것이다.

뿐만아니라 사용자 경험도 형편없게 된다.  
서버에서 파일 전체가 메모리 버퍼에 올라갈때까지 기다린뒤에야 사용자에게 전송이 시작되기 때문이다. 

하지만 다행스럽게도 `(req, res)` 매개변수들은 모두 스트림이다.   
즉, `fs.readFile()` 대신 `fs.createReadStream()` 를 사용함으로써 훨씬 나은 코드를 작성할 수 있다:

``` js
var http = require('http');
var fs = require('fs');

var server = http.createServer(function (req, res) {
    var stream = fs.createReadStream(__dirname + '/data.txt');
    stream.pipe(res);
});
server.listen(8000);
```

`.pipe()` 함수는 `fs.createReadStream()` 에서 발생하는 `'data'` 와 `'end'` 이벤트들을 처리한다.
이 코드는 더 깔끔할 뿐만 아니라 `data.txt` 파일을 디스크에서 읽을 때마다 한 청크(chunk)씩 클라이언트에게 즉시 보내줄 수 있다.

`.pipe()` 를 이용할 시의 이점은 또 있다.  
느린 처리속도를 가진 클라이언트의 경우 백프레셔(backpressure)를 자동으로 처리해주어 node가 청크들을 과도하게 메모리 버퍼에 쌓아두지 않도록 해준다.

응답을 압축하고 싶다고? 서드파티 모듈을 이용하면 간단하다!

``` js
var http = require('http');
var fs = require('fs');
var oppressor = require('oppressor');

var server = http.createServer(function (req, res) {
    var stream = fs.createReadStream(__dirname + '/data.txt');
    stream.pipe(oppressor(req)).pipe(res);
});
server.listen(8000);
```

[oppressor](https://github.com/substack/oppressor) 를 통해 gzip이나 deflate 형태로 압축된 응답을 브라우저에게 보낼 수 있으며, 콘텐츠 인코딩처리도 쉽게 할 수 있다.

한번 스트림 API를 배우고 나면 시시한 non-streaming 커스텀 API 를 통해 데이터를 보내는 대신 다양한 스트리밍 모듈들을 레고블럭이나 정원 호스처럼 이리저리 연결해가며 사용할 수 있다.

이처럼 스트림은 node 프로그래밍을 간결하고 우아하게 해주며, 다양한 방식으로 조립할 수 있게 해준다.

# 스트림 기본

node에는 5가지 형태의 스트림들이 존재한다:  
읽기가능(readable), 쓰기가능(writable), 변형(transform), 이중(duplex) 그리고 고전(classic)

## 파이프(pipe)

모든 스트림들은 입력과 출력을 짝지을 때 `.pipe()`를 사용한다.

`.pipe()` 는 함수로써 읽기가능한 원본 스트림 `src` 의 출력을 가로채서 쓰기 가능한 대상 스트림 `dst` 로 보내준다:

```
src.pipe(dst)
```

`.pipe(dst)` 는 `dst` 스트림을 리턴한다. 이로 인해 우리는 여러개의 `.pipe()` 를 체이닝해서 호출할 수 있다:

``` js
a.pipe(b).pipe(c).pipe(d)
```
위 코드는 아래와 동일하다:

``` js
a.pipe(b);
b.pipe(c);
c.pipe(d);
```

이러한 방식은 커맨드라인 에서 여러 프로그램들을 파이핑할 수 있는 것과 매우 흡사하다:
```
a | b | c | d
```

shell 이 아니라 node 에서 할 수 있다는 점만 빼면 말이다!

## 읽기가능(readable) 스트림

읽기 가능 스트림은 데이터를 생산한다. 이 데이터는 쓰기가능(writable), 변형(transform) 또는 이중(duplex) 스트림에서 `.pipe()` 호출을 통해 소비할 수 있다:

``` js
readableStream.pipe(dst)
```

### 읽기가능 스트림 만들기

그럼 읽기가능 스트림을 만들어보자!

``` js
var Readable = require('stream').Readable;

var rs = new Readable;
rs.push('beep ');
rs.push('boop\n');
rs.push(null);

rs.pipe(process.stdout);
```

```
$ node read0.js
beep boop
```

`rs.push(null)` 는 소비자에게 `rs` 의 데이터 출력이 끝났다는 것을 말해준다.

우리가 읽기가능 스트림 `rs`에 메시지를 푸시한 것은 소비자인 `process.output` 으로 파이핑하기 전이지만, 파이핑후 메시지가 잘 출력되는 것을 확인했다.  
이것은 스트림 버퍼 덕분이다.  
우리가 읽기가능 스트림으로 메시지를 `.push()` 하면 푸시한 메시지 청크들은 소비자가 데이터를 읽기 전까지 버퍼에 저장된다.

우리가 모든 데이터를 버퍼에 쌓아두지 않고 소비자가 요청하는 만큼의 데이터만 생성할 수 있다면  
많은 경우 훨씬 유용하게 쓸 수 있다. 

우리는 `._read` 함수를 정의해줌으로써 요청에 의해 청크를 푸시할 수 있다:

``` js
var Readable = require('stream').Readable;
var rs = Readable();

var c = 97;
rs._read = function () {
    rs.push(String.fromCharCode(c++));
    if (c > 'z'.charCodeAt(0)) rs.push(null);
};

rs.pipe(process.stdout);
```

```
$ node read1.js
abcdefghijklmnopqrstuvwxyz
```

Here we push the letters `'a'` through `'z'`, inclusive, but only when the
consumer is ready to read them.

The `_read` function will also get a provisional `size` parameter as its first
argument that specifies how many bytes the consumer wants to read, but your
readable stream can ignore the `size` if it wants.

Note that you can also use `util.inherits()` to subclass a Readable stream, but
that approach doesn't lend itself very well to comprehensible examples.

To show that our `_read` function is only being called when the consumer
requests, we can modify our readable stream code slightly to add a delay:

```
var Readable = require('stream').Readable;
var rs = Readable();

var c = 97 - 1;

rs._read = function () {
    if (c >= 'z'.charCodeAt(0)) return rs.push(null);
    
    setTimeout(function () {
        rs.push(String.fromCharCode(++c));
    }, 100);
};

rs.pipe(process.stdout);

process.on('exit', function () {
    console.error('\n_read() called ' + (c - 97) + ' times');
});
process.stdout.on('error', process.exit);
```

Running this program we can see that `_read()` is only called 5 times when we
only request 5 bytes of output:

```
$ node read2.js | head -c5
abcde
_read() called 5 times
```

The setTimeout delay is necessary because the operating system requires some
time to send us the relevant signals to close the pipe.

The `process.stdout.on('error', fn)` handler is also necessary because the
operating system will send a SIGPIPE to our process when `head` is no longer
interested in our program's output, which gets emitted as an EPIPE error on
`process.stdout`.

These extra complications are necessary when interfacing with the external
operating system pipes but are automatic when we interface directly with node
streams the whole time.

If you want to create a readable stream that pushes arbitrary values instead of
just strings and buffers, make sure to create your readable stream with
`Readable({ objectMode: true })`.

### 읽기가능 스트림 소비하기

Most of the time it's much easier to just pipe a readable stream into another
kind of stream or a stream created with a module like
[through](https://npmjs.org/package/through)
or [concat-stream](https://npmjs.org/package/concat-stream),
but occasionally it might be useful to consume a readable stream directly.

``` js
process.stdin.on('readable', function () {
    var buf = process.stdin.read();
    console.dir(buf);
});
```

```
$ (echo abc; sleep 1; echo def; sleep 1; echo ghi) | node consume0.js 
<Buffer 61 62 63 0a>
<Buffer 64 65 66 0a>
<Buffer 67 68 69 0a>
null
```

When data is available, the `'readable'` event fires and you can call `.read()`
to fetch some data from the buffer.

When the stream is finished, `.read()` returns `null` because there are no more
bytes to fetch.

You can also tell `.read(n)` to return `n` bytes of data. Reading a number of
bytes is merely advisory and does not work for object streams, but all of the
core streams support it.

Here's an example of using `.read(n)` to buffer stdin into 3-byte chunks:

``` js
process.stdin.on('readable', function () {
    var buf = process.stdin.read(3);
    console.dir(buf);
});
```

Running this example gives us incomplete data!

```
$ (echo abc; sleep 1; echo def; sleep 1; echo ghi) | node consume1.js 
<Buffer 61 62 63>
<Buffer 0a 64 65>
<Buffer 66 0a 67>
```

This is because there is extra data left in internal buffers and we need to give
node a "kick" to tell it that we are interested in more data past the 3 bytes
that we've already read. A simple `.read(0)` will do this:

``` js
process.stdin.on('readable', function () {
    var buf = process.stdin.read(3);
    console.dir(buf);
    process.stdin.read(0);
});
```

Now our code works as expected in 3-byte chunks!

``` js
$ (echo abc; sleep 1; echo def; sleep 1; echo ghi) | node consume2.js 
<Buffer 61 62 63>
<Buffer 0a 64 65>
<Buffer 66 0a 67>
<Buffer 68 69 0a>
```

You can also use `.unshift()` to put back data so that the same read logic will
fire when `.read()` gives you more data than you wanted.

Using `.unshift()` prevents us from making unnecessary buffer copies. Here we
can build a readable parser to split on newlines:

``` js
var offset = 0;

process.stdin.on('readable', function () {
    var buf = process.stdin.read();
    if (!buf) return;
    for (; offset < buf.length; offset++) {
        if (buf[offset] === 0x0a) {
            console.dir(buf.slice(0, offset).toString());
            buf = buf.slice(offset + 1);
            offset = 0;
            process.stdin.unshift(buf);
            return;
        }
    }
    process.stdin.unshift(buf);
});
```

```
$ tail -n +50000 /usr/share/dict/american-english | head -n10 | node lines.js 
'hearties'
'heartiest'
'heartily'
'heartiness'
'heartiness\'s'
'heartland'
'heartland\'s'
'heartlands'
'heartless'
'heartlessly'
```

However, there are modules on npm such as
[split](https://npmjs.org/package/split) that you should use instead of rolling
your own line-parsing logic.

## 쓰기가능(writable) 스트림

A writable stream is a stream you can `.pipe()` to but not from:

``` js
src.pipe(writableStream)
```

### 쓰기가능 스트림 만들기

Just define a `._write(chunk, enc, next)` function and then you can pipe a
readable stream in:

``` js
var Writable = require('stream').Writable;
var ws = Writable();
ws._write = function (chunk, enc, next) {
    console.dir(chunk);
    next();
};

process.stdin.pipe(ws);
```

```
$ (echo beep; sleep 1; echo boop) | node write0.js 
<Buffer 62 65 65 70 0a>
<Buffer 62 6f 6f 70 0a>
```

The first argument, `chunk` is the data that is written by the producer.

The second argument `enc` is a string with the string encoding, but only when
`opts.decodeString` is `false` and you've been written a string.

The third argument, `next(err)` is the callback that tells the consumer that
they can write more data. You can optionally pass an error object `err`, which
emits an `'error'` event on the stream instance.

If the readable stream you're piping from writes strings, they will be converted
into `Buffer`s unless you create your writable stream with
`Writable({ decodeStrings: false })`.

If the readable stream you're piping from writes objects, create your writable
stream with `Writable({ objectMode: true })`.

### 쓰기가능 스트림에 쓰기

To write to a writable stream, just call `.write(data)` with the `data` you want
to write!

``` js
process.stdout.write('beep boop\n');
```

To tell the destination writable stream that you're done writing, just call
`.end()`. You can also give `.end(data)` some `data` to write before ending:

``` js
var fs = require('fs');
var ws = fs.createWriteStream('message.txt');

ws.write('beep ');

setTimeout(function () {
    ws.end('boop\n');
}, 1000);
```

```
$ node writing1.js 
$ cat message.txt
beep boop
```

If you care about high water marks and buffering, `.write()` returns false when
there is more data than the `opts.highWaterMark` option passed to `Writable()`
in the incoming buffer.

If you want to wait for the buffer to empty again, listen for a `'drain'` event.

## 변형(transform) 스트림

Transform streams are

You might also hear transform streams referred to as "through streams".

Through streams are simple readable/writable filters that transform input and
produce output.

## 이중(duplex) 스트림

Duplex streams are readable/writable and both ends of the stream engage
in a two-way interaction, sending back and forth messages like a telephone. An
rpc exchange is a good example of a duplex stream. Any time you see something
like:

``` js
a.pipe(b).pipe(a)
```

you're probably dealing with a duplex stream.

## 고전(classic) 스트림

Classic streams are the old interface that first appeared in node 0.4.
You will probably encounter this style of stream for a long time so it's good to
know how they work.

Whenever a stream has a `"data"` listener registered, it switches into
`"classic"` mode and behaves according to the old API.

### 고전 읽기가능 스트림

Classic readable streams are just event emitters that emit `"data"` events when
they have data for their consumers and emit `"end"` events when they are done
producing data for their consumers.

`.pipe()` checks whether a classic stream is readable by checking the truthiness
of `stream.readable`.

Here is a super simple readable stream that prints `A` through `J`, inclusive:

``` js
var Stream = require('stream');
var stream = new Stream;
stream.readable = true;

var c = 64;
var iv = setInterval(function () {
    if (++c >= 75) {
        clearInterval(iv);
        stream.emit('end');
    }
    else stream.emit('data', String.fromCharCode(c));
}, 100);

stream.pipe(process.stdout);
```

```
$ node classic0.js
ABCDEFGHIJ
```

To read from a classic readable stream, you register `"data"` and `"end"`
listeners. Here's an example reading from `process.stdin` using the old readable
stream style:

``` js
process.stdin.on('data', function (buf) {
    console.log(buf);
});
process.stdin.on('end', function () {
    console.log('__END__');
});
```

```
$ (echo beep; sleep 1; echo boop) | node classic1.js 
<Buffer 62 65 65 70 0a>
<Buffer 62 6f 6f 70 0a>
__END__
```

Note that whenever you register a `"data"` listener, you put the stream into
compatability mode so you lose the benefits of the new streams2 api.

You should pretty much never register `"data"` and `"end"` handlers yourself
anymore. If you need to interact with legacy streams, use libraries that you can
`.pipe()` to instead where possible.

For example, you can use [through](https://npmjs.org/package/through)
to avoid setting up explicit `"data"` and `"end"` listeners:

``` js
var through = require('through');
process.stdin.pipe(through(write, end));

function write (buf) {
    console.log(buf);
}
function end () {
    console.log('__END__');
}
```

```
$ (echo beep; sleep 1; echo boop) | node through.js 
<Buffer 62 65 65 70 0a>
<Buffer 62 6f 6f 70 0a>
__END__
```

or use [concat-stream](https://npmjs.org/package/concat-stream) to buffer up an
entire stream's contents:

``` js
var concat = require('concat-stream');
process.stdin.pipe(concat(function (body) {
    console.log(JSON.parse(body));
}));
```

```
$ echo '{"beep":"boop"}' | node concat.js 
{ beep: 'boop' }
```

Classic readable streams have `.pause()` and `.resume()` logic for provisionally
pausing a stream, but this was merely advisory. If you are going to use
`.pause()` and `.resume()` with classic readable streams, you should use
[through](https://npmjs.org/package/through) to handle buffering instead of
writing that yourself.

### 고전 쓰기가능 스트림

Classic writable streams are very simple. Just define `.write(buf)`, `.end(buf)`
and `.destroy()`.

`.end(buf)` may or may not get a `buf`, but node people will expect `stream.end(buf)`
to mean `stream.write(buf); stream.end()` and you shouldn't violate their
expectations.

## 읽을 거리

* [주요 스트림 문서](http://nodejs.org/docs/latest/api/stream.html#stream_stream)
* 우리는 [readable-stream](https://npmjs.org/package/readable-stream) 모듈을 통해 node 0.8 이하 버전에 대한 stream2 코드 호환을 지원할 수 있다. 단지 `npm install readable-stream` 로 설치한 뒤, `require('stream')` 대신 `require('readable-stream')` 를 사용하면 된다.

***

# 내장(built-in) 스트림

아래의 스트림들은 node 에 내장되어 있다.

## process

### [process.stdin](http://nodejs.org/docs/latest/api/process.html#process_process_stdin)

This readable stream contains the standard system input stream for your program.

It is paused by default but the first time you refer to it `.resume()` will be
called implicitly on the
[next tick](http://nodejs.org/docs/latest/api/process.html#process_process_nexttick_callback).

If process.stdin is a tty (check with
[`tty.isatty()`](http://nodejs.org/docs/latest/api/tty.html#tty_tty_isatty_fd))
then input events will be line-buffered. You can turn off line-buffering by
calling `process.stdin.setRawMode(true)` BUT the default handlers for key
combinations such as `^C` and `^D` will be removed.

### [process.stdout](http://nodejs.org/api/process.html#process_process_stdout)

This writable stream contains the standard system output stream for your program.

`write` to it if you want to send data to stdout

### [process.stderr](http://nodejs.org/api/process.html#process_process_stderr)

This writable stream contains the standard system error stream for your program.

`write` to it if you want to send data to stderr

## child_process.spawn()

## fs

### fs.createReadStream()

### fs.createWriteStream()

## net

### [net.connect()](http://nodejs.org/docs/latest/api/net.html#net_net_connect_options_connectionlistener)

This function returns a [duplex stream] that connects over tcp to a remote
host.

You can start writing to the stream right away and the writes will be buffered
until the `'connect'` event fires.

### net.createServer()

## http

### http.request()

### http.createServer()

## zlib

### zlib.createGzip()

### zlib.createGunzip()

### zlib.createDeflate()

### zlib.createInflate()

***

# 제어(control) 스트림들

## [through](https://github.com/dominictarr/through)

## [from](https://github.com/dominictarr/from)

## [pause-stream](https://github.com/dominictarr/pause-stream)

## [concat-stream](https://github.com/maxogden/node-concat-stream)

concat-stream will buffer up stream contents into a single buffer.
`concat(cb)` just takes a single callback `cb(body)` with the buffered
`body` when the stream has finished.

For example, in this program, the concat callback fires with the body string
`"beep boop"` once `cs.end()` is called.
The program takes the body and upper-cases it, printing `BEEP BOOP.`

``` js
var concat = require('concat-stream');

var cs = concat(function (body) {
    console.log(body.toUpperCase());
});
cs.write('beep ');
cs.write('boop.');
cs.end();
```

```
$ node concat.js
BEEP BOOP.
```

Here's an example usage of concat-stream that will parse incoming url-encoded
form data and reply with a stringified JSON version of the form parameters:

``` js
var http = require('http');
var qs = require('querystring');
var concat = require('concat-stream');

var server = http.createServer(function (req, res) {
    req.pipe(concat(function (body) {
        var params = qs.parse(body.toString());
        res.end(JSON.stringify(params) + '\n');
    }));
});
server.listen(5005);
```

```
$ curl -X POST -d 'beep=boop&dinosaur=trex' http://localhost:5005
{"beep":"boop","dinosaur":"trex"}
```

## [duplex](https://github.com/dominictarr/duplex)

## [duplexer](https://github.com/Raynos/duplexer)

## [emit-stream](https://github.com/substack/emit-stream)

## [invert-stream](https://github.com/dominictarr/invert-stream)

## [map-stream](https://github.com/dominictarr/map-stream)

## [remote-events](https://github.com/dominictarr/remote-events)

## [buffer-stream](https://github.com/Raynos/buffer-stream)

## [event-stream](https://github.com/dominictarr/event-stream)

## [auth-stream](https://github.com/Raynos/auth-stream)

***

# 메타(meta) 스트림들

## [mux-demux](https://github.com/dominictarr/mux-demux)

## [stream-router](https://github.com/Raynos/stream-router)

## [multi-channel-mdm](https://github.com/Raynos/multi-channel-mdm)

***

# 상태(state) 스트림들

## [crdt](https://github.com/dominictarr/crdt)

## [delta-stream](https://github.com/Raynos/delta-stream)

## [scuttlebutt](https://github.com/dominictarr/scuttlebutt)

[scuttlebutt](https://github.com/dominictarr/scuttlebutt) can be used for
peer-to-peer state synchronization with a mesh topology where nodes might only
be connected through intermediaries and there is no node with an authoritative
version of all the data.

The kind of distributed peer-to-peer network that
[scuttlebutt](https://github.com/dominictarr/scuttlebutt) provides is especially
useful when nodes on different sides of network barriers need to share and
update the same state. An example of this kind of network might be browser
clients that send messages through an http server to each other and backend
processes that the browsers can't directly connect to. Another use-case might be
systems that span internal networks since IPv4 addresses are scarce.

[scuttlebutt](https://github.com/dominictarr/scuttlebutt) uses a
[gossip protocol](https://en.wikipedia.org/wiki/Gossip_protocol)
to pass messages between connected nodes so that state across all the nodes will
[eventually converge](https://en.wikipedia.org/wiki/Eventual_consistency)
on the same value everywhere.

Using the `scuttlebutt/model` interface, we can create some nodes and pipe them
to each other to create whichever sort of network we want:

``` js
var Model = require('scuttlebutt/model');
var am = new Model;
var as = am.createStream();

var bm = new Model;
var bs = bm.createStream();

var cm = new Model;
var cs = cm.createStream();

var dm = new Model;
var ds = dm.createStream();

var em = new Model;
var es = em.createStream();

as.pipe(bs).pipe(as);
bs.pipe(cs).pipe(bs);
bs.pipe(ds).pipe(bs);
ds.pipe(es).pipe(ds);

em.on('update', function (key, value, source) {
    console.log(key + ' => ' + value + ' from ' + source);
});

am.set('x', 555);
```

The network we've created is an undirected graph that looks like:

```
a <-> b <-> c
      ^
      |
      v
      d <-> e
```

Note that nodes `a` and `e` aren't directly connected, but when we run this
script:

```
$ node model.js
x => 555 from 1347857300518
```

the value that node `a` set finds its way to node `e` by way of nodes `b` and
`d`. Here all the nodes are in the same process but because
[scuttlebutt](https://github.com/dominictarr/scuttlebutt) uses a
simple streaming interface, the nodes can be placed on any process or server and
connected with any streaming transport that can handle string data.

Next we can make a more realistic example that connects over the network and
increments a counter variable.

Here's the server which will set the initial `count` value to 0 and `count ++`
every 320 milliseconds, printing all updates to count:

``` js
var Model = require('scuttlebutt/model');
var net = require('net');

var m = new Model;
m.set('count', '0');
m.on('update', function (key, value) {
    console.log(key + ' = ' + m.get('count'));
});

var server = net.createServer(function (stream) {
    stream.pipe(m.createStream()).pipe(stream);
});
server.listen(8888);

setInterval(function () {
    m.set('count', Number(m.get('count')) + 1);
}, 320);
```

Now we can make a client that connects to this server, updates the count on an
interval, and prints all the updates it receives:

``` js
var Model = require('scuttlebutt/model');
var net = require('net');

var m = new Model;
var s = m.createStream();

s.pipe(net.connect(8888, 'localhost')).pipe(s);

m.on('update', function cb (key) {
    // wait until we've gotten at least one count value from the network
    if (key !== 'count') return;
    m.removeListener('update', cb);
    
    setInterval(function () {
        m.set('count', Number(m.get('count')) + 1);
    }, 100);
});

m.on('update', function (key, value) {
    console.log(key + ' = ' + value);
});
```

The client is slightly trickier since it should wait until it has an update from
somebody else to start updating the counter itself or else its counter would be
zeroed.

Once we get the server and some clients running we should see a sequence like this:

```
count = 183
count = 184
count = 185
count = 186
count = 187
count = 188
count = 189
```

Occasionally on some of the nodes we might see a sequence with repeated values like:

```
count = 147
count = 148
count = 149
count = 149
count = 150
count = 151
```

These values are due to
[scuttlebutt's](https://github.com/dominictarr/scuttlebutt)
history-based conflict resolution algorithm which is hard at work ensuring that the state of the system across all nodes is eventually consistent.

Note that the server in this example is just another node with the same
privledges as the clients connected to it. The terms "client" and "server" here
don't affect how the state synchronization proceeds, just who initiates the
connection. Protocols with this property are often called symmetric protocols.
See [dnode](https://github.com/substack/dnode) for another example of a
symmetric protocol.

## [append-only](http://github.com/Raynos/append-only)

***

# http streams

## [request](https://github.com/mikeal/request)

## [oppressor](https://github.com/substack/oppressor)

## [response-stream](https://github.com/substack/response-stream)

***

# io streams

## [reconnect](https://github.com/dominictarr/reconnect)

## [kv](https://github.com/dominictarr/kv)

## [discovery-network](https://github.com/Raynos/discovery-network)

***

# parser streams

## [tar](https://github.com/creationix/node-tar)

## [trumpet](https://github.com/substack/node-trumpet)

## [JSONStream](https://github.com/dominictarr/JSONStream)

Use this module to parse and stringify json data from streams.

If you need to pass a large json collection through a slow connection or you
have a json object that will populate slowly this module will let you parse data
incrementally as it arrives.

## [json-scrape](https://github.com/substack/json-scrape)

## [stream-serializer](https://github.com/dominictarr/stream-serializer)

***

# 브라우저(browser) 스트림들

## [shoe](https://github.com/substack/shoe)

## [domnode](https://github.com/maxogden/domnode)

## [sorta](https://github.com/substack/sorta)

## [graph-stream](https://github.com/substack/graph-stream)

## [arrow-keys](https://github.com/Raynos/arrow-keys)

## [attribute](https://github.com/Raynos/attribute)

## [data-bind](https://github.com/travis4all/data-bind)

***

# html 스트림들

## [hyperstream](https://github.com/substack/hyperstream)


# 오디오(audio) 스트림들

## [baudio](https://github.com/substack/baudio)

# rpc 스트림들

## [dnode](https://github.com/substack/dnode)

[dnode](https://github.com/substack/dnode) lets you call remote functions
through any kind of stream.

Here's a basic dnode server:

``` js
var dnode = require('dnode');
var net = require('net');

var server = net.createServer(function (c) {
    var d = dnode({
        transform : function (s, cb) {
            cb(s.replace(/[aeiou]{2,}/, 'oo').toUpperCase())
        }
    });
    c.pipe(d).pipe(c);
});

server.listen(5004);
```

then you can hack up a simple client that calls the server's `.transform()`
function:

``` js
var dnode = require('dnode');
var net = require('net');

var d = dnode();
d.on('remote', function (remote) {
    remote.transform('beep', function (s) {
        console.log('beep => ' + s);
        d.end();
    });
});

var c = net.connect(5004);
c.pipe(d).pipe(c);
```

Fire up the server, then when you run the client you should see:

```
$ node client.js
beep => BOOP
```

The client sent `'beep'` to the server's `transform()` function and the server
called the client's callback with the result, neat!

The streaming interface that dnode provides here is a duplex stream since both
the client and server are piped to each other (`c.pipe(d).pipe(c)`) with
requests and responses coming from both sides.

The craziness of dnode begins when you start to pass function arguments to
stubbed callbacks. Here's an updated version of the previous server with a
multi-stage callback passing dance:

``` js
var dnode = require('dnode');
var net = require('net');

var server = net.createServer(function (c) {
    var d = dnode({
        transform : function (s, cb) {
            cb(function (n, fn) {
                var oo = Array(n+1).join('o');
                fn(s.replace(/[aeiou]{2,}/, oo).toUpperCase());
            });
        }
    });
    c.pipe(d).pipe(c);
});

server.listen(5004);
```

Here's the updated client:

``` js
var dnode = require('dnode');
var net = require('net');

var d = dnode();
d.on('remote', function (remote) {
    remote.transform('beep', function (cb) {
        cb(10, function (s) {
            console.log('beep:10 => ' + s);
            d.end();
        });
    });
});

var c = net.connect(5004);
c.pipe(d).pipe(c);
```

After we spin up the server, when we run the client now we get:

```
$ node client.js
beep:10 => BOOOOOOOOOOP
```

It just works!™

The basic idea is that you just put functions in objects and you call them from
the other side of a stream and the functions will be stubbed out on the other
end to do a round-trip back to the side that had the original function in the
first place. The best thing is that when you pass functions to a stubbed
function as arguments, those functions get stubbed out on the *other* side!

This approach of stubbing function arguments recursively shall henceforth be
known as the "turtles all the way down" gambit. The return values of any of your
functions will be ignored and only enumerable properties on objects will be
sent, json-style.

It's turtles all the way down!

![turtles all the way](http://substack.net/images/all_the_way_down.png)

Since dnode works in node or on the browser over any stream it's easy to call
functions defined anywhere and especially useful when paired up with
[mux-demux](https://github.com/dominictarr/mux-demux) to multiplex an rpc stream
for control alongside some bulk data streams.

## [rpc-stream](https://github.com/dominictarr/rpc-stream)

***

# test streams

## [tap](https://github.com/isaacs/node-tap)

## [stream-spec](https://github.com/dominictarr/stream-spec)

***

# power combos

## distributed partition-tolerant chat

The [append-only](http://github.com/Raynos/append-only) module can give us a
convenient append-only array on top of
[scuttlebutt](https://github.com/dominictarr/scuttlebutt)
which makes it really easy to write an eventually-consistent, distributed chat
that can replicate with other nodes and survive network partitions.

TODO: the rest

## roll your own socket.io

We can build a socket.io-style event emitter api over streams using some of the
libraries mentioned earlier in this document.

First  we can use [shoe](http://github.com/substack/shoe)
to create a new websocket handler server-side and
[emit-stream](https://github.com/substack/emit-stream)
to turn an event emitter into a stream that emits objects.
The object stream can then be fed into
[JSONStream](https://github.com/dominictarr/JSONStream)
to serialize the objects and from there the serialized stream can be piped into
the remote browser.

``` js
var EventEmitter = require('events').EventEmitter;
var shoe = require('shoe');
var emitStream = require('emit-stream');
var JSONStream = require('JSONStream');

var sock = shoe(function (stream) {
    var ev = new EventEmitter;
    emitStream(ev)
        .pipe(JSONStream.stringify())
        .pipe(stream)
    ;
    ...
});
```

Inside the shoe callback we can emit events to the `ev` function.  Here we'll
just emit different kinds of events on intervals:

``` js
var intervals = [];

intervals.push(setInterval(function () {
    ev.emit('upper', 'abc');
}, 500));

intervals.push(setInterval(function () {
    ev.emit('lower', 'def');
}, 300));

stream.on('end', function () {
    intervals.forEach(clearInterval);
});
```

Finally the shoe instance just needs to be bound to an http server:

``` js
var http = require('http');
var server = http.createServer(require('ecstatic')(__dirname));
server.listen(8080);

sock.install(server, '/sock');
```

Meanwhile on the browser side of things just parse the json shoe stream and pass
the resulting object stream to `eventStream()`. `eventStream()` just returns an
event emitter that emits the server-side events:

``` js
var shoe = require('shoe');
var emitStream = require('emit-stream');
var JSONStream = require('JSONStream');

var parser = JSONStream.parse([true]);
var stream = parser.pipe(shoe('/sock')).pipe(parser);
var ev = emitStream(stream);

ev.on('lower', function (msg) {
    var div = document.createElement('div');
    div.textContent = msg.toLowerCase();
    document.body.appendChild(div);
});

ev.on('upper', function (msg) {
    var div = document.createElement('div');
    div.textContent = msg.toUpperCase();
    document.body.appendChild(div);
});
```

Use [browserify](https://github.com/substack/node-browserify) to build this
browser source code so that you can `require()` all these nifty modules
browser-side:

```
$ browserify main.js -o bundle.js
```

Then drop a `<script src="/bundle.js"></script>` into some html and open it up
in a browser to see server-side events streamed through to the browser side of
things.

With this streaming approach you can rely more on tiny reusable components that
only need to know how to talk streams. Instead of routing messages through a
global event system socket.io-style, you can focus more on breaking up your
application into tinier units of functionality that can do exactly one thing
well.

For instance you can trivially swap out JSONStream in this example for
[stream-serializer](https://github.com/dominictarr/stream-serializer)
to get a different take on serialization with a different set of tradeoffs.
You could bolt layers over top of shoe to handle
[reconnections](https://github.com/dominictarr/reconnect) or heartbeats
using simple streaming interfaces.
You could even add a stream into the chain to use namespaced events with
[eventemitter2](https://npmjs.org/package/eventemitter2) instead of the
EventEmitter in core.

If you want some different streams that act in different ways it would likewise
be pretty simple to run the shoe stream in this example through mux-demux to
create separate channels for each different kind of stream that you need.

As the requirements of your system evolve over time, you can swap out each of
these streaming pieces as necessary without as many of the all-or-nothing risks
that more opinionated framework approaches necessarily entail.

## html streams for the browser and the server

We can use some streaming modules to reuse the same html rendering logic for the
client and the server! This approach is indexable, SEO-friendly, and gives us
realtime updates.

Our renderer takes lines of json as input and returns html strings as its
output. Text, the universal interface!

render.js:

``` js
var through = require('through');
var hyperglue = require('hyperglue');
var fs = require('fs');
var html = fs.readFileSync(__dirname + '/static/row.html', 'utf8');

module.exports = function () {
    return through(function (line) {
        try { var row = JSON.parse(line) }
        catch (err) { return this.emit('error', err) }
        
        this.queue(hyperglue(html, {
            '.who': row.who,
            '.message': row.message
        }).outerHTML);
    });
};
```

We can use [brfs](http://github.com/substack/brfs) to inline the
`fs.readFileSync()` call for browser code
and [hyperglue](https://github.com/substack/hyperglue) to update html based on
css selectors. You don't need to use hyperglue necessarily here; anything that
can return a string with html in it will work.

The `row.html` used is just a really simple stub thing:

row.html:

``` html
<div class="row">
  <div class="who"></div>
  <div class="message"></div>
</div>
```

The server will just use [slice-file](https://github.com/substack/slice-file) to
keep everything simple. [slice-file](https://github.com/substack/slice-file) is
little more than a glorified `tail/tail -f` api but the interfaces map well to
databases with regular results plus a changes feed like couchdb.

server.js:

``` js
var http = require('http');
var fs = require('fs');
var hyperstream = require('hyperstream');
var ecstatic = require('ecstatic')(__dirname + '/static');

var sliceFile = require('slice-file');
var sf = sliceFile(__dirname + '/data.txt');

var render = require('./render');

var server = http.createServer(function (req, res) {
    if (req.url === '/') {
        var hs = hyperstream({
            '#rows': sf.slice(-5).pipe(render())
        });
        hs.pipe(res);
        fs.createReadStream(__dirname + '/static/index.html').pipe(hs);
    }
    else ecstatic(req, res)
});
server.listen(8000);

var shoe = require('shoe');
var sock = shoe(function (stream) {
    sf.follow(-1,0).pipe(stream);
});
sock.install(server, '/sock');
```

The first part of the server handles the `/` route and streams the last 5 lines
from `data.txt` into the `#rows` div.

The second part of the server handles realtime updates to `#rows` using
[shoe](http://github.com/substack/shoe), a simple streaming websocket polyfill.

Next we can write some simple browser code to populate the realtime updates
from [shoe](http://github.com/substack/shoe) into the `#rows` div:

``` js
var through = require('through');
var render = require('./render');

var shoe = require('shoe');
var stream = shoe('/sock');

var rows = document.querySelector('#rows');
stream.pipe(render()).pipe(through(function (html) {
    rows.innerHTML += html;
}));
```

Just compile with [browserify](http://browserify.org) and
[brfs](http://github.com/substack/brfs):

```
$ browserify -t brfs browser.js > static/bundle.js
```

And that's it! Now we can populate `data.txt` with some silly data:

```
$ echo '{"who":"substack","message":"beep boop."}' >> data.txt
$ echo '{"who":"zoltar","message":"COWER PUNY HUMANS"}' >> data.txt
```

then spin up the server:

```
$ node server.js
```

then navigate to `localhost:8000` where we will see our content. If we add some
more content:

```
$ echo '{"who":"substack","message":"oh hello."}' >> data.txt
$ echo '{"who":"zoltar","message":"HEAR ME!"}' >> data.txt
```

then the page updates automatically with the realtime updates, hooray!

We're now using exactly the same rendering logic on both the client and the
server to serve up SEO-friendly, indexable realtime content. Hooray!
