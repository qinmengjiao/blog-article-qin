## Node Stream 入门与深入

#### 概念

Stream 是Node.js中最重要的组件和模式之一，之前在社区中看到这样一句格言“Stream all the things(流是一切)”。

具体的来说流是一组有序的，有起点和终点的字节数据传输手段，它是一个抽象的接口。

流是数据的集合 —— 就像数组或字符串一样。区别在于流中的数据可能不会立刻就全部可用，并且你无需一次性地把这些数据全部放入内存。这使得流在操作大量数据或是数据从外部来源逐段发送过来的时候变得非常有用。

每一个流对象都是EventEmitter类的一个实例，都有相应的on和emit方法，在下文中演示的代码示例中我们会看到。

stream 是node的核心模块，引入方式如下：

```
let Stream = require('stream')
```
#### 实现形式
流在node中有多种实现形式，例如：

```
1. http协议中的 请求req和响应res
2. tcp协议中的套接字对象sockets
3. fs文件模块中的可读流和可写流
4. process进程模块中的stdout stderr
5. zlib 中的streams
   .....
```

#### 类型

提供了以下四种类型：

###### 可读流（Readable）
```
let Readable = stream.Readable
```
###### 可写流（Writeable）
```
let Readable = stream.Writeable
```
###### 可读写流（Duplex）
```
let Duplex = stream.Duplex
```
###### 转换流（Transform）
```
let Transform = stream.Transform
```

### 原理

##### Readable

实现了stream.Readable接口的对象,将对象数据读取为流数据,当监听data事件后,开始发射数据.

###### 1. 创建

```
let rs = fs.createReadStream(path,{
    flags: 'r', // 打开文件要做的操作,默认为'r'
    encoding: null, // 默认为null
    start: '0', // 开始读取的索引位置
    end: '', // 结束读取的索引位置(包括结束位置)
    highWaterMark: '', // 读取缓存区默认的大小的阀值64kb
})
```
  
###### 2. 方法及作用

```
// 1.监听data事件 流自动切换到流动模式
// 2.数据会尽可能快的读出
rs.on('data', function (data) {
    console.log(data);
});
// 数据读取完毕后触发end事件
rs.on('end', function () {
    console.log('读取完成');
});
// 可读流打开事件
rs.on('open', function () {
    console.log(err);
});
// 可读流关闭事件
rs.on('close', function () {
    console.log(err);
});
// 指定编码 和上面创建流时的参数encoding意思相同
rs.setEncoding('utf8');


rs.on('data', function (data) {
// 可读流暂停读取
    rs.pause();
    console.log(data);
});
setTimeout(function () {
// 可读流恢复读取
    rs.resume();
},2000);
```

###### 3. 分类
可读流分为：*流动模式和暂停模式*

可读流对象readable中有一个维护状态的对象，readable._readableState，这里简称为state。
其中有一个标记，state.flowing， 可用来判别流的模式。
它有三种可能值：

true 流动模式。
false 暂停模式。
null 初始状态。

*1) 流动模式（flowing mode）*

流动模式下，数据会源源不断地生产出来，形成“流动”现象。
监听流的data事件便可进入该模式。

*2) 暂停模式（paused mode）*

暂停模式下，需要显示地调用read()，触发data事件。

在初始状态下，监听data事件，会使流进入流动模式。
但如果在暂停模式下，监听data事件并不会使它进入流动模式。
为了消耗流，需要显示调用read()方法。

*3）相互转化*

调用readable.resume()可使流进入流动模式，state.flowing被设为true。
调用readable.pause()可使流进入暂停模式，state.flowing被设为false。

###### 4. 原理


创建可读流时，需要继承Readable，并实现_read方法。

![](https://tech.meituan.com/img/stream-how-data-comes-out.png)

1) _read方法是从底层系统读取具体数据的逻辑，即生产数据的逻辑。

    在_read方法中，通过调用push(data)将数据放入可读流中供下游消耗。
    在_read方法中，可以同步调用push(data)，也可以异步调用。
    当全部数据都生产出来后，必须调用push(null)来结束可读流。
    流一旦结束，便不能再调用push(data)添加数据。
    可以通过监听data事件的方式消耗可读流。
    
    在首次监听其data事件后，readable便会持续不断地调用_read()，通过触发data事件将数据输出。
    第一次data事件会在下一个tick中触发，所以，可以安全地将数据输出前的逻辑放在事件监听后（同一个tick中）。
    当数据全部被消耗时，会触发end事件。
2) 详解    
![](https://tech.meituan.com/img/stream-read.png)

###### doRead
流中维护了一个缓存，当缓存中的数据足够多时，调用read()不会引起_read()的调用，即不需要向底层请求数据。用doRead来表示read(n)是否需要向底层取数据.

```
var doRead = state.needReadable

if (state.length === 0 || state.length - n < state.highWaterMark) {
  doRead = true
}

if (state.ended || state.reading) {
  doRead = false
}

if (doRead) {
  state.reading = true
  state.sync = true
  if (state.length === 0) {
    state.needReadable = true
  }
  this._read(state.highWaterMark)
  state.sync = false
}

```
当缓存区的长度为0或者缓存区的数量小于state.highWaterMark这个阀值，则会调用 _read()去底层读取数据。
state.reading 标志上次从底层取数据的操作是否完成，一旦push 被调用，就会就会设置为false，表示此次_read()结束。

###### push
消耗方调用read(n)促使流输出数据，而流通过_read()使底层调用push方法将数据传给流。
如果调用push方法时缓存为空，则当前数据即为下一个需要的数据。
这个数据可能先添加到缓存中，也可能直接输出。
执行read方法时，在调用_read后，如果从缓存中取到了数据，就以data事件输出。

所以，如果_read异步调用push时发现缓存为空，则意味着当前数据是下一个需要的数据，且不会被read方法输出，应当在push方法中立即以data事件输出。

上图立即输出条件

```
state.flowing && state.length === 0 && !state.sync
```
###### end
由于流是分次向底层请求数据的，需要底层显示地告诉流数据是否取完。
所以，当某次（执行_read()）取数据时，调用了push(null)，就意味着底层数据取完。
此时，流会设置state.ended。

state.length表示缓存中当前的数据量。
只有当state.length为0，且state.ended为true，才意味着所有的数据都被消耗了。
一旦在执行read(n)时检测到这个条件，便会触发end事件。
当然，这个事件只会触发一次。
###### readable
在调用完_read()后，read(n)会试着从缓存中取数据。
如果_read()是异步调用push方法的，则此时缓存中的数据量不会增多，容易出现数据量不够的现象。

如果read(n)的返回值为null，说明这次未能从缓存中取出所需量的数据。
此时，消耗方需要等待新的数据到达后再次尝试调用read方法。

在数据到达后，流是通过readable事件来通知消耗方的。
在此种情况下，push方法如果立即输出数据，接收方直接监听data事件即可，否则数据被添加到缓存中，需要触发readable事件。
消耗方必须监听这个事件，再调用read方法取得数据。
###### 5. 手写
#### 流动模式
```
let EventEmitter = require('events');
let fs = require('fs');

class ReadStream extends EventEmitter {
    constructor(path, options) {
        super(path, options);
        
        // 初始化参数
        this.path = path; // 路径
        this.flags = options.flags || 'r'; // 打开文件要做的操作,默认为'r'
        this.mode = options.mode || 0o666; 
        this.pos = this.start = options.start || 0; // 偏移量 默认是开始位置
        this.end = options.end; // 结束为止
        this.encoding = options.encoding; // 编码形式
        this.highWaterMark = options.highWaterMark || 64 * 1024; // 可读流缓存区的阀值 默认64k
        
        
        this.flowing = null;// 标志形式 null初始状态 false 暂停模式 true 流动模式
        this.buffer = Buffer.alloc(this.highWaterMark); // 缓存区
        this.open();// //准备打开文件读取
        //当给这个实例添加了任意的监听函数时会触发newListener
        this.on('newListener', (type, listener) => {
            if (type == 'data') { // 当可读流监听data事件时初始态变成流动模式
                this.flowing = true;
                this.read();
            }
        });
    }
    
    read() {
        // 如果文件标示符不是数字的话 说明文件还没有打开，这时候我们要监听 文件的打开事件 一旦打开就开始读取
        if (typeof this.fd !== 'number') {
            return this.once('open', () => this.read());
        }
        // 每次读取多少个
        let howMuchToRead = this.end ? Math.min(this.end - this.pos + 1, this.highWaterMark) : this.highWaterMark;
        
        fs.read(this.fd, this.buffer, 0, howMuchToRead, this.pos, (err, bytes) => {
            if (err) {
                if (this.autoClose) {
                    this.destroy();
                }
                return this.emit('error', err);
            }
            if (bytes) {
                let data = this.buffer.slice(0, bytes);
                data = this.encoding ? data.toString(this.encoding) : data;
                this.emit('data', data);
                
                this.pos += bytes;
                
                if (this.end && this.pos > this.end) {
                    return this.endFn();
                } else {
                    if (this.flowing) {
                        this.read();
                    }
                }
            } else {
                return this.endFn();
            }
            
        })
    }
    
    endFn() {
        this.emit('end');
        this.destroy();
    }
    
    open() {
        fs.open(this.path, this.flags, this.mode, (err, fd) => {
            if (err) {
                if (this.autoClose) {
                    this.destroy();
                    return this.emit('error', err);
                }
            }
            this.fd = fd;
            this.emit('open');
        })
    }
    
    destroy() {
        fs.close(this.fd, () => {
            this.emit('close');
        });
    }
    
    pipe(dest) {
        this.on('data', data => {
            let flag = dest.write(data);
            if (!flag) {
                this.pause();
            }
        });
        dest.on('drain', () => {
            this.resume();
        });
    }
    
    pause() {
        this.flowing = false;
    }
    
    resume() {
        this.flowing = true;
        this.read();
    }
}

module.exports = ReadStream;
```

##### 测试代码

```
let ReadStream = require('./ReadStream');
let rs = new ReadStream('1.txt',{
   highWaterMark:3,
    encoding:'utf8'
});
//在真实的情况下，当可读流创建后会立刻进行暂停模式。其实会立刻填充缓存区
//缓存区大小是可以看到
rs.on('readable',function () {
    console.log(rs.length);//3
    //当你消费掉一个字节之后，缓存区变成2个字节了
    let char = rs.read(1);
    console.log(char);
    console.log(rs.length);
    //一旦发现缓冲区的字节数小于最高水位线了，则会现再读到最高水位线个字节填充到缓存区里
    setTimeout(()=>{
        console.log(rs.length);//5
    },500)
});
```

#### 暂停模式

```
let fs = require('fs');
let EventEmitter = require('events');
class ReadStream extends EventEmitter {
    constructor(path, options) {
        super(path, options);
     
        this.path = path;
        this.highWaterMark = options.highWaterMark || 64 * 1024;
        this.buffer = Buffer.alloc(this.highWaterMark);
        this.flags = options.flags || 'r';
        this.encoding = options.encoding;
        this.mode = options.mode || 0o666;
        this.start = options.start || 0;
        this.end = options.end;
        this.pos = this.start;
        this.autoClose = options.autoClose || true;
        this.bytesRead = 0;
        this.closed = false;
        this.flowing;
        this.needReadable = false;
        this.length = 0;
        this.buffers = [];
        this.on('end', function () {
            if (this.autoClose) {
                this.destroy();
            }
        });
        this.on('newListener', (type) => {
            if (type == 'data') {
                this.flowing = true;
                this.read();
            }
            if (type == 'readable') {
                this.read(0);
            }
        });
        this.open();
    }
    
    open() {
        fs.open(this.path, this.flags, this.mode, (err, fd) => {
            if (err) {
                if (this.autoClose) {
                    this.destroy();
                    return this.emit('error', err);
                }
            }
            this.fd = fd;
            this.emit('open');
        });
    }
    
    read(n) {
        if (typeof this.fd != 'number') {
            return this.once('open', () => this.read());
        }
        n = parseInt(n, 10);
        if (n != n) {
            n = this.length;
        }
        if (this.length == 0)
            this.needReadable = true;
        let ret;
        if (0 < n < this.length) {
            ret = Buffer.alloc(n);
            let b;
            let index = 0;
            while (null != (b = this.buffers.shift())) {
                for (let i = 0; i < b.length; i++) {
                    ret[index++] = b[i];
                    if (index == ret.length) {
                        this.length -= n;
                        b = b.slice(i + 1);
                        this.buffers.unshift(b);
                        break;
                    }
                }
            }
            ret = ret.toString(this.encoding);
        }
        
        let _read = () => {
            let m = this.end ? Math.min(this.end - this.pos + 1, this.highWaterMark) : this.highWaterMark;
            fs.read(this.fd, this.buffer, 0, m, this.pos, (err, bytesRead) => {
                if (err) {
                    return
                }
                let data;
                if (bytesRead > 0) {
                    data = this.buffer.slice(0, bytesRead);
                    this.pos += bytesRead;
                    this.length += bytesRead;
                    if (this.end && this.pos > this.end) {
                        if (this.needReadable) {
                            this.emit('readable');
                        }
                        
                        this.emit('end');
                    } else {
                        this.buffers.push(data);
                        if (this.needReadable) {
                            this.emit('readable');
                            this.needReadable = false;
                        }
                        
                    }
                } else {
                    if (this.needReadable) {
                        this.emit('readable');
                    }
                    return this.emit('end');
                }
            })
        }
        if (this.length == 0 || (this.length < this.highWaterMark)) {
            _read();
        }
        return ret;
    }
    
    destroy() {
        fs.close(this.fd, (err) => {
            this.emit('close');
        });
    }
    
    pause() {
        this.flowing = false;
    }
    
    resume() {
        this.flowing = true;
        this.read();
    }
    
    pipe(dest) {
        this.on('data', (data) => {
            let flag = dest.write(data);
            if (!flag) this.pause();
        });
        dest.on('drain', () => {
            this.resume();
        });
        this.on('end', () => {
            dest.end();
        });
    }
}

module.exports = ReadStream;
```

##### Writable

1. 创建
```
let rs = fs.createWriteStream(path,{
    flags: 'w', // 打开文件要做的操作,默认为'w'
    encoding: null, // 默认为null
    highWaterMark: '', // 读取缓存区默认的大小的阀值16kb
})
```
2. 方法及作用

```
let ws = fs.createWriteStream(path,{
    chunk: '',// 写入的数据buffer/string
    encoding: '', // 编码格式chunk为字符串时有用，可选
    callback: ()=>{} // 写入成功后的回调
});
// 返回值为布尔值，系统缓存区满时为false,未满时为true

ws.end(chunk,[encoding],[callback]);
// 表明接下来没有数据要被写入 Writable 通过传入可选的 chunk 和 encoding 参数，可以在关闭流之前再写入一段数据 如果传入了可选的 callback 函数，它将作为 'finish' 事件的回调函数


// 当一个流不处在 drain 的状态， 对 write() 的调用会缓存数据块， 并且返回 false。 一旦所有当前所有缓存的数据块都排空了（被操作系统接受来进行输出）， 那么 'drain' 事件就会被触发
建议， 一旦 write() 返回 false， 在 'drain' 事件触发前， 不能写入任何数据块
let fs = require('fs');
let ws = fs.createWriteStream('./2.txt',{
  flags:'w',
  encoding:'utf8',
  highWaterMark:3
});
let i = 10;
function write(){
 let  flag = true;
 while(i&&flag){
      flag = ws.write("1");
      i--;
     console.log(flag);
 }
}
write();
ws.on('drain',()=>{
  console.log("drain");
  write();
});
// 在调用了 stream.end() 方法，且缓冲区数据都已经传给底层系统之后， 'finish' 事件将被触发。
var writer = fs.createWriteStream('./2.txt');
for (let i = 0; i < 100; i++) {
  writer.write(`hello, ${i}!\n`);
}
writer.end('结束\n');
writer.on('finish', () => {
  console.error('所有的写入已经完成!');
});

// pipe用法 将数据的滞留量限制到一个可接受的水平，以使得不同速度的来源和目标不会淹没可用内存。
readStream.pipe(writeStream);
var from = fs.createReadStream('./1.txt');
var to = fs.createWriteStream('./2.txt');
from.pipe(to);

// pipe方法的原理
var fs = require('fs');
var ws = fs.createWriteStream('./2.txt');
var rs = fs.createReadStream('./1.txt');
rs.on('data', function (data) {
    var flag = ws.write(data);
    if(!flag)
    rs.pause();
});
ws.on('drain', function () {
    rs.resume();
});
rs.on('end', function () {
    ws.end();
});
```
3.原理

```
const Writable = require('stream').Writable

const writable = Writable()
// 实现`_write`方法
// 这是将数据写入底层的逻辑
writable._write = function (data, enc, next) {
  // 将流中的数据写入底层
  process.stdout.write(data.toString().toUpperCase())
  // 写入完成时，调用`next()`方法通知流传入下一个数据
  process.nextTick(next)
}

// 所有数据均已写入底层
writable.on('finish', () => process.stdout.write('DONE'))

// 将一个数据写入流中
writable.write('a' + '\n')
writable.write('b' + '\n')
writable.write('c' + '\n')

// 再无数据写入流时，需要调用`end`方法
writable.end()
```
    上游通过调用writable.write(data)将数据写入可写流中。write()方法会调用_write()将data写入底层。
    在_write中，当数据成功写入底层后，必须调用next(err)告诉流开始处理下一个数据。
    next的调用既可以是同步的，也可以是异步的。
    上游必须调用writable.end(data)来结束可写流，data是可选的。此后，不能再调用write新增数据。
    在end方法调用后，当所有底层的写操作均完成时，会触发finish事件。
    
4.手写

#### WriteStream

```
let fs = require('fs');
let EventEmitter = require('events');

class WriteStream extends EventEmitter {
    constructor(path, options) {
        super(path, options);
        this.path = path;
        this.flags = options.flags || 'w';
        this.mode = options.mode || 0o666;
        this.start = options.start || 0;
        this.pos = this.start;
        this.encoding = options.encoding || 'utf8';
        this.autoClose = options.autoClose;
        this.highWaterMark = options.highWaterMark || 16 * 1024;
        
        this.buffers = [];//缓存区
        this.writing = false;//表示内部正在写入数据
        this.length = 0;//表示缓存区字节的长度
        this.open();
    }
    
    open() {
        fs.open(this.path, this.flags, this.mode, (err, fd) => {
            if (err) {
                if (this.autoClose) {
                    this.destroy();
                }
                return this.emit('error', err);
            }
            this.fd = fd;
            this.emit('open');
        });
    }
    
    
    write(chunk, encoding, cb) {
        chunk = Buffer.isBuffer(chunk) ? chunk : Buffer.from(chunk, this.encoding);
        let len = chunk.length;
        
        this.length += len;
        
        let ret = this.length < this.highWaterMark;
                if (this.writing) {
            this.buffers.push({
                chunk,
                encoding,
                cb
            });
        } else {
            this.writing = true;
            this._write(chunk, encoding, () => this.clearBuffer());
        }
        return ret;
    }
    
    clearBuffer() {
        let data = this.buffers.shift();
        if (data) {
            this._write(data.chunk, data.encoding, () => this.clearBuffer())
        } else {
        
            this.writing = false;
            this.emit('drain');
        }
    }
    
    _write(chunk, encoding, cb) {
        if (typeof this.fd !== 'number') {
            return this.once('open', () => this._write(chunk, encoding, cb));
        }
        fs.write(this.fd, chunk, 0, chunk.length, this.pos, (err, bytesWritten) => {
            if (err) {
                if (this.autoClose) {
                    this.destroy();
                    this.emit('error', err);
                }
            }
            this.pos += bytesWritten;
            
            this.length -= bytesWritten;
            
            cb && cb();
        })
    }
    
    destroy() {
        fs.close(this.fd, () => {
            this.emit('close');
        })
    }
}

module.exports = WriteStream;
```


##### Duplex
Duplex实际上就是继承了Readable和Writable的一类流。
所以，一个Duplex对象既可当成可读流来使用（需要实现_read方法），也可当成可写流来使用（需要实现_write方法）。

```
var Duplex = require('stream').Duplex

var duplex = Duplex()

// 可读端底层读取逻辑
duplex._read = function () {
  this._readNum = this._readNum || 0
  if (this._readNum > 1) {
    this.push(null)
  } else {
    this.push('' + (this._readNum++))
  }
}

// 可写端底层写逻辑
duplex._write = function (buf, enc, next) {
  // a, b
  process.stdout.write('_write ' + buf.toString() + '\n')
  next()
}

// 0, 1
duplex.on('data', data => console.log('ondata', data.toString()))

duplex.write('a')
duplex.write('b')

duplex.end()
```
上面的代码中实现了_read方法，所以可以监听data事件来消耗Duplex产生的数据。
同时，又实现了_write方法，可作为下游去消耗数据。

因为它既可读又可写，所以称它有两端：可写端和可读端。
可写端的接口与Writable一致，作为下游来使用；可读端的接口与Readable一致，作为上游来使用。
##### Transform
在上面的例子中，可读流中的数据（0, 1）与可写流中的数据（'a', 'b'）是隔离开的，但在Transform中可写端写入的数据经变换后会自动添加到可读端。
Tranform继承自Duplex，并已经实现了_read和_write方法，同时要求用户实现一个_transform方法。

```
'use strict'

const Transform = require('stream').Transform

class Rotate extends Transform {
  constructor(n) {
    super()
    // 将字母旋转`n`个位置
    this.offset = (n || 13) % 26
  }

  // 将可写端写入的数据变换后添加到可读端
  _transform(buf, enc, next) {
    var res = buf.toString().split('').map(c => {
      var code = c.charCodeAt(0)
      if (c >= 'a' && c <= 'z') {
        code += this.offset
        if (code > 'z'.charCodeAt(0)) {
          code -= 26
        }
      } else if (c >= 'A' && c <= 'Z') {
        code += this.offset
        if (code > 'Z'.charCodeAt(0)) {
          code -= 26
        }
      }
      return String.fromCharCode(code)
    }).join('')

    // 调用push方法将变换后的数据添加到可读端
    this.push(res)
    // 调用next方法准备处理下一个
    next()
  }

}

var transform = new Rotate(3)
transform.on('data', data => process.stdout.write(data))
transform.write('hello, ')
transform.write('world!')
transform.end()

// khoor, zruog!
```
### 参考资料
[ 通过源码解析 Node.js 中导流（pipe）的实现](https://cnodejs.org/topic/56ba030271204e03637a3870)


