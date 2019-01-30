<!--NodeJS-cheat-sheet-->

最近读《重学前端》，开篇就是让你拥有自己的知识体系图谱，后续学的东西补充到相应的模块，既可以加深对原有知识的理解，又可以强化记忆，很不错的学习方案。

这篇文章主要知识点来自：

- [《Node.js硬实战：115个核心技巧》](https://www.amazon.cn/dp/B01MYX8XG1)
- [i0natan/nodebestpractices](https://github.com/i0natan/nodebestpractices)
- 后续学习的一些知识点

<!--more-->

# 说明

比较好的 markdown 的查看方式是直接用 VSCode 打开大纲，这样整个脉络一目了然，后续补充知识点也很快定位到响应的位置：

![01.png](https://qiniu.chenng.cn/2019-01-28-09-44-31.png)

这个 markdown 文件已经丢到 Github：

https://github.com/ringcrl/node-point

# 安装

```sh
# 使用 nvm 安装
https://github.com/creationix/nvm#install-script # Git install

nvm install
nvm alias default

# 卸载 pkg 安装版
sudo rm -rf /usr/local/{bin/{node,npm},lib/node_modules/npm,lib/node,share/man/*/node.*}
```

# 全局变量

## require(id)

- 内建模块直接从内存加载
- 文件模块通过文件查找定位到文件
- 包通过 package.json 里面的 main 字段查找入口文件

### module.exports

```js
// 通过如下模块包装得到
(funciton (exports, require, module, __filename, __dirname) { // 包装头

}); // 包装尾
```

### JSON 文件

- 通过 `fs.readFileSync()` 加载
- 通过 `JSON.parse()` 解析

### 加载大文件

- require 成功后会缓存文件
- 大量使用会导致大量数据驻留在内存中，导致 GC 频分和内存泄露

## module.exports 和 exports

### 执行时

```js
(funciton(exports, require, module, __filename, __dirname) { // 包装头
  console.log('hello world!') // 原始文件
}); // 包装尾
```

### exports

- exports 是 module 的属性，默认情况是空对象
- require 一个模块实际得到的是该模块的 exports 属性
- exports.xxx 导出具有多个属性的对象
- module.exports = xxx 导出一个对象

### 使用

```js
// module-2.js
exports.method = function() {
  return 'Hello';
};

exports.method2 = function() {
  return 'Hello again';
};

// module-1.js
const module2 = require('./module-2');
console.log(module2.method()); // Hello
console.log(module2.method2()); // Hello again
```

## 路径变量

```js
console.log('__dirname:', __dirname); // 文件夹
console.log('__filename:', __filename); // 文件

path.join(__dirname, 'views', 'view.html'); // 如果不希望自己手动处理 / 的问题，使用 path.join
```

## console

| 占位符 |  类型  |                例子                 |
| :----: | :----: | :---------------------------------: |
|   %s   | String |     console.log('%s', 'value')      |
|   %d   | Number |       console.log('%d', 3.14)       |
|   %j   |  JSON  | console.log('%j', {name: 'Chenng'}) |

## process

### 查看 PATH

```js
node

console.log(process.env.PATH.split(':').join('\n'));
```

### 设置 PATH

```js
process.env.PATH += ':/a_new_path_to_executables';
```

### 获取信息

```js
// 获取平台信息
process.arch // x64
process.platform // darwin

// 获取内存使用情况
process.memoryUsage();

// 获取命令行参数
process.argv
```

### nextTick

process.nextTick 方法允许你把一个回调放在下一次时间轮询队列的头上，这意味着可以用来延迟执行，结果是比 setTimeout 更有效率。

```js
const EventEmitter = require('events').EventEmitter;

function complexOperations() {
  const events = new EventEmitter();

  process.nextTick(function () {
    events.emit('success');
  });

  return events;
}

complexOperations().on('success', function () {
  console.log('success!');
});
```

# Buffer

如果没有提供编码格式，文件操作以及很多网络操作就会将数据作为 Buffer 类型返回。

## toString

默认转为 `UTF-8` 格式，还支持 `ascii`、`base64` 等。

## data URI

```js
// 生成 data URI
const fs = require('fs');
const mime = 'image/png';
const encoding = 'base64';
const base64Data = fs.readFileSync(`${__dirname}/monkey.png`).toString(encoding);
const uri = `data:${mime};${encoding},${base64Data}`;
console.log(uri);

// data URI 转文件
const fs = require('fs');
const uri = 'data:image/png;base64,iVBORw0KGgoAAAANSUhEUgA...';
const base64Data = uri.split(',')[1];
const buf = Buffer(base64Data, 'base64');
fs.writeFileSync(`${__dirname}/secondmonkey.png`, buf);
```

# events

```js
const EventEmitter = require('events').EventEmitter;

const AudioDevice = {
  play: function (track) {
    console.log('play', track);
  },
  stop: function () {
    console.log('stop');
  },
};

class MusicPlayer extends EventEmitter {
  constructor() {
    super();
    this.playing = false; 
  }
}

const musicPlayer = new MusicPlayer();
musicPlayer.on('play', function (track) {
  this.playing = true;
  AudioDevice.play(track);
});
musicPlayer.on('stop', function () {
  this.playing = false;
  AudioDevice.stop();
});

musicPlayer.emit('play', 'The Roots - The Fire');
setTimeout(function () {
  musicPlayer.emit('stop');
}, 1000);

// 处理异常
// EventEmitter 实例发生错误会发出一个 error 事件
// 如果没有监听器，默认动作是打印一个堆栈并退出程序
musicPlayer.on('error', function (err) {
  console.err('Error:', err);
});
```

# 流

## 理解流

流是基于事件的 API，用于管理和处理数据。

- 流是能够读写的
- 是基于事件实现的一个实例

理解流的最好方式就是想象一下没有流的时候怎么处理数据：

- `fs.readFileSync` 同步读取文件，程序会阻塞，所有数据被读到内存
- `fs.readFile` 阻止程序阻塞，但仍会将文件所有数据读取到内存中
- 希望少内存读取大文件，读取一个数据块到内存处理完再去索取更多的数据

### 流的类型

- 内置：许多核心模块都实现了流接口，如 `fs.createReadStream`
- HTTP：处理网络技术的流
- 解释器：第三方模块 XML、JSON 解释器
- 浏览器：Node 流可以被拓展使用在浏览器
- Audio：流接口的声音模块
- RPC（远程调用）：通过网络发送流是进程间通信的有效方式
- 测试：使用流的测试库

## 使用内建流 API

### 静态 web 服务器

想要通过网络高效且支持大文件的发送一个文件到一个客户端。

#### 不使用流

```js
const http = require('http');
const fs = require('fs');

http.createServer((req, res) => {
  fs.readFile(`${__dirname}/index.html`, (err, data) => {
    if (err) {
      res.statusCode = 500;
      res.end(String(err));
      return;
    }

    res.end(data);
  });
}).listen(8000);
```

#### 使用流

```js
const http = require('http');
const fs = require('fs');

http.createServer((req, res) => {
  fs.createReadStream(`${__dirname}/index.html`).pipe(res);
}).listen(8000);
```

- 更少代码，更加高效
- 提供一个缓冲区发送到客户端

#### 使用流 + gzip

```js
const http = require('http');
const fs = require('fs');
const zlib = require('zlib');

http.createServer((req, res) => {
  res.writeHead(200, {
    'content-encoding': 'gzip',
  });
  fs.createReadStream(`${__dirname}/index.html`)
    .pipe(zlib.createGzip())
    .pipe(res);
}).listen(8000);
```

### 流的错误处理

```js
const fs = require('fs');
const stream = fs.createReadStream('not-found');

stream.on('error', (err) => {
  console.trace();
  console.error('Stack:', err.stack);
  console.error('The error raised was:', err);
});
```

## 使用流基类

### 可读流 - JSON 行解析器

可读流被用来为 I/O 源提供灵活的 API，也可以被用作解析器：

- 继承自 steam.Readable 类
- 并实现一个 `_read(size)` 方法

json-lines.txt

```
{ "position": 0, "letter": "a" }
{ "position": 1, "letter": "b" }
{ "position": 2, "letter": "c" }
{ "position": 3, "letter": "d" }
{ "position": 4, "letter": "e" }
{ "position": 5, "letter": "f" }
{ "position": 6, "letter": "g" }
{ "position": 7, "letter": "h" }
{ "position": 8, "letter": "i" }
{ "position": 9, "letter": "j" }
```

JSONLineReader.js

```js
const stream = require('stream');
const fs = require('fs');
const util = require('util');

class JSONLineReader extends stream.Readable {
  constructor(source) {
    super();
    this._source = source;
    this._foundLineEnd = false;
    this._buffer = '';

    source.on('readable', () => {
      this.read();
    });
  }

  // 所有定制 stream.Readable 类都需要实现 _read 方法
  _read(size) {
    let chunk;
    let line;
    let result;

    if (this._buffer.length === 0) {
      chunk = this._source.read();
      this._buffer += chunk;
    }

    const lineIndex = this._buffer.indexOf('\n');

    if (lineIndex !== -1) {
      line = this._buffer.slice(0, lineIndex); // 从 buffer 的开始截取第一行来获取一些文本进行解析
      if (line) {
        result = JSON.parse(line);
        this._buffer = this._buffer.slice(lineIndex + 1);
        this.emit('object', result); // 当一个 JSON 记录解析出来的时候，触发一个 object 事件
        this.push(util.inspect(result)); // 将解析好的 SJON 发回内部队列
      } else {
        this._buffer = this._buffer.slice(1);
      }
    }
  }
}

const input = fs.createReadStream(`${__dirname}/json-lines.txt`, {
  encoding: 'utf8',
});
const jsonLineReader = new JSONLineReader(input); // 创建一个 JSONLineReader 实例，传递一个文件流给它处理

jsonLineReader.on('object', (obj) => {
  console.log('pos:', obj.position, '- letter:', obj.letter);
});
```

### 可写流 - 文字变色

可写的流可用于输出数据到底层 I/O:

- 继承自 `stream.Writable`
- 实现一个 `_write` 方法向底层源数据发送数据

```sh
cat json-lines.txt | node stram_writable.js
```

stram_writable.js

```js
const stream = require('stream');

class GreenStream extends stream.Writable {
  constructor(options) {
    super(options);
  }

  _write(chunk, encoding, cb) {
    process.stdout.write(`\u001b[32m${chunk}\u001b[39m`);
    cb();
  }
}

process.stdin.pipe(new GreenStream());
```

### 双工流 - 接受和转换数据

双工流允许发送和接受数据：

- 继承自 `stream.Duplex`
- 实现 `_read` 和 `_write` 方法

### 转换流 - 解析数据

使用流改变数据为另一种格式，并且高效地管理内存：

- 继承自 `stream.Transform`
- 实现 `_transform` 方法

## 测试流

使用 Node 内置的断言模块测试

```js
const assert = require('assert');
const fs = require('fs');
const CSVParser = require('./csvparser');

const parser = new CSVParser();
const actual = [];

fs.createReadStream(`${__dirname}/sample.csv`)
  .pipe(parser);

process.on('exit', function () {
  actual.push(parser.read());
  actual.push(parser.read());
  actual.push(parser.read());

  const expected = [
    { name: 'Alex', location: 'UK', role: 'admin' },
    { name: 'Sam', location: 'France', role: 'user' },
    { name: 'John', location: 'Canada', role: 'user' },
  ];

  assert.deepEqual(expected, actual);
});
```

# 文件系统

## fs 模块交互

- POSIX 文件 I/O
- 文件流
- 批量文件 I/O
- 文件监控

## POSIX 文件系统

| fs 方法      | 描述                                                       |
| :----------- | :--------------------------------------------------------- |
| fs.truncate  | 截断或者拓展文件到制定的长度                               |
| fs.ftruncate | 和 truncate 一样，但将文件描述符作为参数                   |
| fs.chown     | 改变文件的所有者以及组                                     |
| fs.fchown    | 和 chown 一样，但将文件描述符作为参数                      |
| fs.lchown    | 和 chown 一样，但不解析符号链接                            |
| fs.stat      | 获取文件状态                                               |
| fs.lstat     | 和 stat 一样，但是返回信息是关于符号链接而不是它指向的内容 |
| fs.fstat     | 和 stat 一样，但将文件描述符作为参数                       |
| fs.link      | 创建一个硬链接                                             |
| fs.symlink   | 创建一个软连接                                             |
| fs.readlink  | 读取一个软连接的值                                         |
| fs.realpath  | 返回规范的绝对路径名                                       |
| fs.unlink    | 删除文件                                                   |
| fs.rmdir     | 删除文件目录                                               |
| fs.mkdir     | 创建文件目录                                               |
| fs.readdir   | 读取一个文件目录的内容                                     |
| fs.close     | 关闭一个文件描述符                                         |
| fs.open      | 打开或者创建一个文件用来读取或者写入                       |
| fs.utimes    | 设置文件的读取和修改时间                                   |
| fs.futimes   | 和 utimes 一样，但将文件描述符作为参数                     |
| fs.fsync     | 同步磁盘中的文件数据                                       |
| fs.write     | 写入数据到一个文件                                         |
| fs.read      | 读取一个文件的数据                                         |

```js
const fs = require('fs');
const assert = require('assert');

const fd = fs.openSync('./file.txt', 'w+');
const writeBuf = new Buffer('some data to write');
fs.writeSync(fd, writeBuf, 0, writeBuf.length, 0);

const readBuf = new Buffer(writeBuf.length);
fs.readSync(fd, readBuf, 0, writeBuf.length, 0);
assert.equal(writeBuf.toString(), readBuf.toString());

fs.closeSync(fd);
```

## 读写流

```js
const fs = require('fs');
const readable = fs.createReadStream('./original.txt');
const writeable = fs.createWriteStream('./copy.txt');
readable.pipe(writeable);
```

## 文件监控

`fs.watchFile` 比 `fs.watch` 低效，但更好用。

## 同步读取与 require

同步 fs 的方法应该在第一次初始化应用的时候使用。

```js
const fs = require('fs');
const config = JSON.parse(fs.readFileSync('./config.json').toString());
init(config);
```

require：

```js
const config = require('./config.json);
init(config);
```

- 模块会被全局缓冲，其他文件也加载并修改，会影响到整个系统加载了此文件的模块
- 可以通过 `Object.freeze` 来冻结一个对象

## 文件描述

文件描述是在操作系统中管理的在进程中打开文件所关联的一些数字或者索引。操作系统通过指派一个唯一的整数给每个打开的文件用来查看关于这个文件

| Stream | 文件描述 | 描述     |
| ------ | -------- | -------- |
| stdin  | 0        | 标准输入 |
| stdout | 1        | 标准输出 |
| stderr | 2        | 标准错误 |

`console.log('Log')` 是 `process.stdout.write('log')` 的语法糖。

一个文件描述是 open 以及 openSync 方法调用返回的一个数字

```js
const fd = fs.openSync('myfile', 'a');
console.log(typeof fd === 'number'); // true
```

## 文件锁

协同多个进程同时访问一个文件，保证文件的完整性以及数据不能丢失：

- 强制锁（在内核级别执行）
- 咨询锁（非强制，只在涉及到进程订阅了相同的锁机制）
    - `node-fs-ext` 通过 `flock` 锁住一个文件
- 使用锁文件
    - 进程 A 尝试创建一个锁文件，并且成功了
    - 进程 A 已经获得了这个锁，可以修改共享的资源
    - 进程 B 尝试创建一个锁文件，但失败了，无法修改共享的资源

Node 实现锁文件

- 使用独占标记创建锁文件
- 使用 mkdir 创建锁文件

### 独占标记

```js
// 所有需要打开文件的方法，fs.writeFile、fs.createWriteStream、fs.open 都有一个 x 标记
// 这个文件应该已独占打开，若这个文件存在，文件不能被打开
fs.open('config.lock', 'wx', (err) => {
  if (err) { return console.err(err); }
});

// 最好将当前进程号写进文件锁中
// 当有异常的时候就知道最后这个锁的进程
fs.writeFile(
  'config.lock',
  process.pid,
  { flogs: 'wx' },
  (err) => {
    if (err) { return console.error(err) };
  },
);
```

### mkdir 文件锁

独占标记有个问题，可能有些系统不能识别 `0_EXCL` 标记。另一个方案是把锁文件换成一个目录，PID 可以写入目录中的一个文件。

```js
fs.mkidr('config.lock', (err) => {
  if (err) { return console.error(err); }
  fs.writeFile(`/config.lock/${process.pid}`, (err) => {
    if (err) { return console.error(err); }
  });
});
```

### lock 模块实现

https://github.com/npm/lockfile

```js
const fs = require('fs');
const lockDir = 'config.lock';
let hasLock = false;

exports.lock = function (cb) { // 获取锁
  if (hasLock) { return cb(); } // 已经获取了一个锁
  fs.mkdir(lockDir, function (err) {
    if (err) { return cb(err); } // 无法创建锁

    fs.writeFile(lockDir + '/' + process.pid, function (err) { // 把 PID写入到目录中以便调试
      if (err) { console.error(err); } // 无法写入 PID，继续运行
      hasLock = true; // 锁创建了
      return cb();
    });
  });
};

exports.unlock = function (cb) { // 解锁方法
  if (!hasLock) { return cb(); } // 如果没有需要解开的锁
  fs.unlink(lockDir + '/' + process.pid, function (err) {
    if (err) { return cb(err); }

    fs.rmdir(lockDir, function (err) {
      if (err) return cb(err);
      hasLock = false;
      cb();
    });
  });
};

process.on('exit', function () {
  if (hasLock) {
    fs.unlinkSync(lockDir + '/' + process.pid); // 如果还有锁，在退出之前同步删除掉
    fs.rmdirSync(lockDir);
    console.log('removed lock');
  }
});
```

## 递归文件操作

一个线上库：[mkdirp](https://github.com/substack/node-mkdirp)

递归：要解决我们的问题就要先解决更小的相同的问题。

```
dir-a
├── dir-b
│   ├── dir-c
│   │   ├── dir-d
│   │   │   └── file-e.png
│   │   └── file-e.png
│   ├── file-c.js
│   └── file-d.txt
├── file-a.js
└── file-b.txt
```

查找模块：`find /asset/dir-a -name="file.*"`

```js
[
  'dir-a/dir-b/dir-c/dir-d/file-e.png',
  'dir-a/dir-b/dir-c/file-e.png',
  'dir-a/dir-b/file-c.js',
  'dir-a/dir-b/file-d.txt',
  'dir-a/file-a.js',
  'dir-a/file-b.txt',
]
```

```js
const fs = require('fs');
const join = require('path').join;

// 同步查找
exports.findSync = function (nameRe, startPath) {
  const results = [];

  function finder(path) {
    const files = fs.readdirSync(path);

    for (let i = 0; i < files.length; i++) {
      const fpath = join(path, files[i]);
      const stats = fs.statSync(fpath);

      if (stats.isDirectory()) { finder(fpath); }

      if (stats.isFile() && nameRe.test(files[i])) {
        results.push(fpath);
      }
    }
  }

  finder(startPath);
  return results;
};

// 异步查找
exports.find = function (nameRe, startPath, cb) { // cb 可以传入 console.log，灵活
  const results = [];
  let asyncOps = 0; // 2

  function finder(path) {
    asyncOps++;
    fs.readdir(path, function (er, files) {
      if (er) { return cb(er); }

      files.forEach(function (file) {
        const fpath = join(path, file);

        asyncOps++;
        fs.stat(fpath, function (er, stats) {
          if (er) { return cb(er); }

          if (stats.isDirectory()) finder(fpath);

          if (stats.isFile() && nameRe.test(file)) {
            results.push(fpath);
          }

          asyncOps--;
          if (asyncOps == 0) {
            cb(null, results);
          }
        });
      });

      asyncOps--;
      if (asyncOps == 0) {
        cb(null, results);
      }
    });
  }

  finder(startPath);
};

console.log(exports.findSync(/file.*/, `${__dirname}/dir-a`));
console.log(exports.find(/file.*/, `${__dirname}/dir-a`, console.log));
```

## 监视文件和文件夹

想要监听一个文件或者目录，并在文件更改后执行一个动作。

```js
const fs = require('fs');
fs.watch('./watchdir', console.log); // 稳定且快
fs.watchFile('./watchdir', console.log); // 跨平台
```

## 逐行地读取文件流

```js
const fs = require('fs');
const readline = require('readline');

const rl = readline.createInterface({
  input: fs.createReadStream('/etc/hosts'),
  crlfDelay: Infinity
});

rl.on('line', (line) => {
  console.log(`cc ${line}`);
  const extract = line.match(/(\d+\.\d+\.\d+\.\d+) (.*)/);
});
```

# 网络

## 获取本地 IP

```js
function get_local_ip() {
  const interfaces = require('os').networkInterfaces();
  let IPAdress = '';
  for (const devName in interfaces) {
    const iface = interfaces[devName];
    for (let i = 0; i < iface.length; i++) {
      const alias = iface[i];
      if (alias.family === 'IPv4' && alias.address !== '127.0.0.1' && !alias.internal) {
        IPAdress = alias.address;
      }
    }
  }
  return IPAdress;
}
```

## TCP 客户端

NodeJS 使用 `net` 模块创建 TCP 连接和服务。

### 启动与测试 TCP

```js
const assert = require('assert');
const net = require('net');
let clients = 0;
let expectedAssertions = 2;

const server = net.createServer(function (client) {
  clients++;
  const clientId = clients;
  console.log('Client connected:', clientId);

  client.on('end', function () {
    console.log('Client disconnected:', clientId);
  });

  client.write('Welcome client: ' + clientId);
  client.pipe(client);
});

server.listen(8000, function () {
  console.log('Server started on port 8000');

  runTest(1, function () {
    runTest(2, function () {
      console.log('Tests finished');
      assert.equal(0, expectedAssertions);
      server.close();
    });
  });
});

function runTest(expectedId, done) {
  const client = net.connect(8000);

  client.on('data', function (data) {
    const expected = 'Welcome client: ' + expectedId;
    assert.equal(data.toString(), expected);
    expectedAssertions--;
    client.end();
  });

  client.on('end', done);
}
```

## UDP 客户端

利用 `dgram` 模块创建数据报 `socket`，然后利用 `socket.send` 发送数据。

### 文件发送服务

```js
const dgram = require('dgram');
const fs = require('fs');
const port = 41230;
const defaultSize = 16;

function Client(remoteIP) {
  const inStream = fs.createReadStream(__filename); // 从当前文件创建可读流
  const socket = dgram.createSocket('udp4'); // 创建新的数据流 socket 作为客户端

  inStream.on('readable', function () {
    sendData(); // 当可读流准备好，开始发送数据到服务器
  });

  function sendData() {
    const message = inStream.read(defaultSize); // 读取数据块

    if (!message) {
      return socket.unref(); // 客户端完成任务后，使用 unref 安全关闭它
    }

    // 发送数据到服务器
    socket.send(message, 0, message.length, port, remoteIP, function () {
        sendData();
      }
    );
  }
}

function Server() {
  const socket = dgram.createSocket('udp4'); // 创建一个 socket 提供服务

  socket.on('message', function (msg) {
    process.stdout.write(msg.toString());
  });

  socket.on('listening', function () {
    console.log('Server ready:', socket.address());
  });

  socket.bind(port);
}

if (process.argv[2] === 'client') { // 根据命令行选项确定运行客户端还是服务端
  new Client(process.argv[3]);
} else {
  new Server();
}
```

## HTTP 客户端

使用 `http.createServer` 和 `http.createClient` 运行 HTTP 服务。

### 启动与测试 HTTP

```js
const assert = require('assert');
const http = require('http');

const server = http.createServer(function(req, res) {
  res.writeHead(200, { 'Content-Type': 'text/plain' }); // 写入基于文本的响应头
  res.write('Hello, world.'); // 发送消息回客户端
  res.end();
});

server.listen(8000, function() {
  console.log('Listening on port 8000');
});

const req = http.request({ port: 8000}, function(res) { // 创建请求
  console.log('HTTP headers:', res.headers);
  res.on('data', function(data) { // 给 data 事件创建监听，确保和期望值一致
    console.log('Body:', data.toString());
    assert.equal('Hello, world.', data.toString());
    assert.equal(200, res.statusCode);
    server.unref();
    console.log('测试完成');
  });
});

req.end();
```

### 重定向

HTTP 标准定义了标识重定向发生时的状态码，它也指出了客户端应该检查无限循环。

- 300：多重选择
- 301：永久移动到新位置
- 302：找到重定向跳转
- 303：参见其他信息
- 304：没有改动
- 305：使用代理
- 307：临时重定向

```js
const http = require('http');
const https = require('https');
const url = require('url'); // 有很多接续 URLs 的方法

// 构造函数被用来创建一个对象来构成请求对象的声明周期
function Request() {
  this.maxRedirects = 10;
  this.redirects = 0;
}

Request.prototype.get = function(href, callback) {
  const uri = url.parse(href); // 解析 URLs 成为 Node http 模块使用的格式，确定是否使用 HTTPS
  const options = { host: uri.host, path: uri.path };
  const httpGet = uri.protocol === 'http:' ? http.get : https.get;

  console.log('GET:', href);

  function processResponse(response) {
    if (response.statusCode >= 300 && response.statusCode < 400) { // 检查状态码是否在 HTTP 重定向范围
      if (this.redirects >= this.maxRedirects) {
        this.error = new Error('Too many redirects for: ' + href);
      } else {
        this.redirects++; // 重定向计数自增
        href = url.resolve(options.host, response.headers.location); // 使用 url.resolve 确保相对路径的 URLs 转换为绝对路径 URLs
        return this.get(href, callback);
      }
    }

    response.url = href;
    response.redirects = this.redirects;

    console.log('Redirected:', href);

    function end() {
      console.log('Connection ended');
      callback(this.error, response);
    }

    response.on('data', function(data) {
      console.log('Got data, length:', data.length);
    });

    response.on('end', end.bind(this)); // 绑定回调到 Request 实例，确保能拿到实例属性
  }

  httpGet(options, processResponse.bind(this))
    .on('error', function(err) {
      callback(err);
    });
};

const request = new Request();
request.get('http://google.com/', function(err, res) {
  if (err) {
    console.error(err);
  } else {
    console.log(`
      Fetched URL: ${res.url} with ${res.redirects} redirects
    `);
    process.exit();
  }
});
```

### HTTP 代理

- ISP 使用透明代理使网络更加高效
- 使用缓存代理服务器减少宽带
- Web 应用程序的 DevOps 利用他们提升应用程序性能

```js
const http = require('http');
const url = require('url');

http.createServer(function(req, res) {
  console.log('start request:', req.url);
  const options = url.parse(req.url);
  console.log(options);
  options.headers = req.headers;
  const proxyRequest = http.request(options, function(proxyResponse) { // 创建请求来复制原始的请求
    proxyResponse.on('data', function(chunk) { // 监听数据，返回给浏览器
      console.log('proxyResponse length:', chunk.length);
      res.write(chunk, 'binary');
    });

    proxyResponse.on('end', function() { // 追踪代理请求完成
      console.log('proxied request ended');
      res.end();
    });

    res.writeHead(proxyResponse.statusCode, proxyResponse.headers); // 发送头部信息给服务器
  });

  req.on('data', function(chunk) { // 捕获从浏览器发送到服务器的数据
    console.log('in request length:', chunk.length);
    proxyRequest.write(chunk, 'binary');
  });

  req.on('end', function() { // 追踪原始的请求什么时候结束
    console.log('original request ended');
    proxyRequest.end();
  });
}).listen(8888); // 监听来自本地浏览器的连接
```

### 封装 request-promise

```js
const https = require('https');
const promisify = require('util').promisify;

https.get[promisify.custom] = function getAsync(options) {
  return new Promise((resolve, reject) => {
    https.get(options, (response) => {
      response.end = new Promise((resolve) => response.on('end', resolve));
      resolve(response);
    }).on('error', reject);
  });
};
const rp = promisify(https.get);

(async () => {
  const res = await rp('https://jsonmock.hackerrank.com/api/movies/search/?Title=Spiderman&page=1');
  let body = '';
  res.on('data', (chunk) => body += chunk);
  await res.end;

  console.log(body);
})();
```

## DNS 请求

使用 `dns` 模块创建 DNS 请求。

- A：`dns.resolve`，A 记录存储 IP 地址
- TXT：`dns.resulveTxt`，文本值可以用于在 DNS 上构建其他服务
- SRV：`dns.resolveSrv`，服务记录定义服务的定位数据，通常包含主机名和端口号
- NS：`dns.resolveNs`，指定域名服务器
- CNAME：`dns.resolveCname`，相关的域名记录，设置为域名而不是 IP 地址

```js
const dns = require('dns');

dns.resolve('www.chenng.cn', function (err, addresses) {
  if (err) {
    console.error(err);
  }

  console.log('Addresses:', addresses);
});
```

## crypto 库加密解密

```js
const crypto = require('crypto')

function aesEncrypt(data, key = 'key') {
  const cipher = crypto.createCipher('aes192', key)
  let crypted = cipher.update(data, 'utf8', 'hex')
  crypted += cipher.final('hex')
  return crypted
}

function aesDecrypt(encrypted, key = 'key') {
  const decipher = crypto.createDecipher('aes192', key)
  let decrypted = decipher.update(encrypted, 'hex', 'utf8')
  decrypted += decipher.final('utf8')
  return decrypted
}
```

# 子进程

## 执行外部应用

### 基本概念

- 4个异步方法：exec、execFile、fork、spawn
    - Node
        - fork：想将一个 Node 进程作为一个独立的进程来运行的时候使用，是的计算处理和文件描述器脱离 Node 主进程
    - 非 Node
        - spawn：处理一些会有很多子进程 I/O 时、进程会有大量输出时使用
        - execFile：只需执行一个外部程序的时候使用，执行速度快，处理用户输入相对安全
        - exec：想直接访问线程的 shell 命令时使用，一定要注意用户输入
- 3个同步方法：execSync、execFileSync、spawnSync
- 通过 API 创建出来的子进程和父进程没有任何必然联系

### execFile

- 会把输出结果缓存好，通过回调返回最后结果或者异常信息

```js
const cp = require('child_process');

cp.execFile('echo', ['hello', 'world'], (err, stdout, stderr) => {
  if (err) { console.error(err); }
  console.log('stdout: ', stdout);
  console.log('stderr: ', stderr);
});
```

### spawn

- 通过流可以使用有大量数据输出的外部应用，节约内存
- 使用流提高数据响应效率
- spawn 方法返回一个 I/O 的流接口

#### 单一任务

```js
const cp = require('child_process');

const child = cp.spawn('echo', ['hello', 'world']);
child.on('error', console.error);
child.stdout.pipe(process.stdout);
child.stderr.pipe(process.stderr);
```

#### 多任务串联

```js
const cp = require('child_process');
const path = require('path');

const cat = cp.spawn('cat', [path.resolve(__dirname, 'messy.txt')]);
const sort = cp.spawn('sort');
const uniq = cp.spawn('uniq');

cat.stdout.pipe(sort.stdin);
sort.stdout.pipe(uniq.stdin);
uniq.stdout.pipe(process.stdout);
```

### exec

- 只有一个字符串命令
- 和 shell 一模一样

```js
const cp = require('child_process');

cp.exec(`cat ${__dirname}/messy.txt | sort | uniq`, (err, stdout, stderr) => {
  console.log(stdout);
});
```

### fork

- fork 方法会开发一个 IPC 通道，不同的 Node 进程进行消息传送
- 一个子进程消耗 30ms 启动时间和 10MB 内存
- 子进程：`process.on('message')`、`process.send()`
- 父进程：`child.on('message')`、`child.send()`

#### 父子通信

```js
// parent.js
const cp = require('child_process');

const child = cp.fork('./child', { silent: true });
child.send('monkeys');
child.on('message', function (message) {
  console.log('got message from child', message, typeof message);
})
child.stdout.pipe(process.stdout);

setTimeout(function () {
  child.disconnect();
}, 3000);
```

```js
// child.js
process.on('message', function (message) {
  console.log('got one', message);
  process.send('no pizza');
  process.send(1);
  process.send({ my: 'object' });
  process.send(false);
  process.send(null);
});

console.log(process);
```

## 常用技巧

### 退出时杀死所有子进程

- 保留对由 spawn 返回的 ChildProcess 对象的引用，并在退出主进程时将其杀死

```js
const spawn = require('child_process').spawn;
const children = [];

process.on('exit', function () {
  console.log('killing', children.length, 'child processes');
  children.forEach(function (child) {
    child.kill();
  });
});

children.push(spawn('/bin/sleep', ['10']));
children.push(spawn('/bin/sleep', ['10']));
children.push(spawn('/bin/sleep', ['10']));

setTimeout(function () { process.exit(0); }, 3000);
```

## Cluster 的理解

- 解决 NodeJS 单进程无法充分利用多核 CPU 问题
- 通过 master-cluster 模式可以使得应用更加健壮
- Cluster 底层是 child_process 模块，除了可以发送普通消息，还可以发送底层对象 `TCP`、`UDP` 等
- TCP 主进程发送到子进程，子进程能根据消息重建出 TCP 连接，Cluster 可以决定 fork 出合适的硬件资源的子进程数

# 综合应用

## watch 服务

```js
const fs = require('fs');
const exec = require('child_process').exec;

function watch() {
  const child = exec('node server.js');
  const watcher = fs.watch(__dirname + '/server.js', function () {
    console.log('File changed, reloading.');
    child.kill();
    watcher.close();
    watch();
  });
}

watch();
```

## RESTful web 应用

- REST 意思是表征性状态传输
- 使用正确的 HTTP 方法、URLs 和头部信息来创建语义化 RESTful API

- GET /gages：获取
- POST /pages：创建
- GET /pages/10：获取 pages10
- PATCH /pages/10：更新 pages10
- PUT /pages/10：替换 pages10
- DELETE /pages/10：删除 pages10

```js
let app;
const express = require('express');
const routes = require('./routes');

module.exports = app = express();

app.use(express.json()); // 使用 JSON body 解析
app.use(express.methodOverride()); // 允许一个查询参数来制定额外的 HTTP 方法

// 资源使用的路由
app.get('/pages', routes.pages.index);
app.get('/pages/:id', routes.pages.show);
app.post('/pages', routes.pages.create);
app.patch('/pages/:id', routes.pages.patch);
app.put('/pages/:id', routes.pages.update);
app.del('/pages/:id', routes.pages.remove);
```

## 中间件应用

```js
const express = require('express');
const app = express();
const Schema = require('validate');
const xml2json = require('xml2json');
const util = require('util');
const Page = new Schema();

Page.path('title').type('string').required(); // 数据校验确保页面有标题

function ValidatorError(errors) { // 从错误对象继承，校验出现的错误在错误中间件处理
  this.statusCode = 400;
  this.message = errors.join(', ');
}
util.inherits(ValidatorError, Error);

function xmlMiddleware(req, res, next) { // 处理 xml 的中间件
  if (!req.is('xml')) return next();

  let body = '';
  req.on('data', function (str) { // 从客户端读到数据时触发
    body += str;
  });

  req.on('end', function () {
    req.body = xml2json.toJson(body.toString(), {
      object: true,
      sanitize: false,
    });
    next();
  });
}

function checkValidXml(req, res, next) { // 数据校验中间件
  const page = Page.validate(req.body.page);
  if (page.errors.length) {
    next(new ValidatorError(page.errors)); // 传递错误给 next 阻止路由继续运行
  } else {
    next();
  }
}

function errorHandler(err, req, res, next) { // 错误处理中间件
  console.error('errorHandler', err);
  res.send(err.statusCode || 500, err.message);
}

app.use(xmlMiddleware); // 应用 XML 中间件到所有的请求中

app.post('/pages', checkValidXml, function (req, res) { // 特定的请求校验 xml
  console.log('Valid page:', req.body.page);
  res.send(req.body);
});

app.use(errorHandler); // 添加错误处理中间件

app.listen(3000);
```

## 通过事件组织应用

```js
// 监听用户注册成功消息，绑定邮件程序
const express = require('express');
const app = express();
const emails = require('./emails');
const routes = require('./routes');

app.use(express.json());

app.post('/users', routes.users.create); // 设置路由创建用户

app.on('user:created', emails.welcome); // 监听创建成功事件，绑定 email 代码

module.exports = app;
```

```js
// 用户注册成功发起事件
const User = require('./../models/user');

module.exports.create = function (req, res, next) {
  const user = new User(req.body);
  user.save(function (err) {
    if (err) return next(err);
    res.app.emit('user:created', user); // 当用户成功注册时触发创建用户事件
    res.send('User created');
  });
};
```

## WebSocket 与 session

```js
const express = require('express');
const WebSocketServer = require('ws').Server;
const parseCookie = express.cookieParser('some secret'); // 加载解析 cookie 中间件，设置密码
const MemoryStore = express.session.MemoryStore; // 加载要使用的会话存储
const store = new MemoryStore();

const app = express();
const server = app.listen(process.env.PORT || 3000);

app.use(parseCookie);
app.use(express.session({ store: store, secret: 'some secret' })); // 告知 Express 使用会话存储和设置密码(使用 session 中间件)
app.use(express.static(__dirname + '/public'));

app.get('/random', function (req, res) { // 测试测试用的会话值
  req.session.random = Math.random().toString();
  res.send(200);
});

// 设置 WebSocket 服务器，将其传递给 Express 服务器
// 需要传递已有的 Express 服务（listen 的返回对象）
const webSocketServer = new WebSocketServer({ server: server });

// 在连接事件给客户端创建 WebSocket
webSocketServer.on('connection', function (ws) {
  let session;

  ws.on('message', function (data, flags) {
    const message = JSON.parse(data);

    // 客户端发送的 JSON，需要一些代码来解析 JSON 字符串确定是否可用
    if (message.type === 'getSession') {
      parseCookie(ws.upgradeReq, null, function (err) {
        // 从 HTTP 的更新请求中获取 WebSocket 的会话 ID
        // 一旦 WebSockets 服务器有一个连接，session ID 可以用=从初始化请求中的 cookies 中获取
        const sid = ws.upgradeReq.signedCookies['connect.sid'];

        // 从存储中获取用户的会话信息
        // 只需要在初始化的请求中传递一个引用给解析 cookie 的中间件
        // 然后 session 可以使用 session 存储的 get 方法加载
        store.get(sid, function (err, loadedSession) {
          if (err) console.error(err);
          session = loadedSession;
          ws.send('session.random: ' + session.random, {
            mask: false,
          }); // session 加载后会把一个包含了 session 值的消息发回给客户端
        });
      });
    } else {
      ws.send('Unknown command');
    }
  });
});
```

```html
<!DOCTYPE html>
<html>

<head>
  <script>
    const host = window.document.location.host.replace(/:.*/, '');
    const ws = new WebSocket('ws://' + host + ':3000');

    setInterval(function () {
      ws.send('{ "type": "getSession" }'); // 定期向服务器发送消息
    }, 1000);

    ws.onmessage = function (event) {
      document.getElementById('message').innerHTML = event.data;
    };
  </script>
</head>

<body>
  <h1>WebSocket sessions</h1>
  <div id='message'></div><br>
</body>

</html>
```

## Express4 中间件

| package         | 描述                                                             |
| --------------- | ---------------------------------------------------------------- |
| body-parser     | 解析 URL 编码 和 JSON POST 请求的 body 数据                      |
| compression     | 压缩服务器响应                                                   |
| connect-timeout | 请求允许超时                                                     |
| cookie-parser   | 从 HTTP 头部信息中解析 cookies，结果放在 req.cookies             |
| cookie-session  | 使用 cookies 来支持简单会话                                      |
| csurf           | 在会话中添加 token，防御 CSRF 攻击                               |
| errorhandler    | Connect 中使用的默认错误处理                                     |
| express-session | 简单的会话处理，使用 stores 扩展来吧会话信息写入到数据库或文件中 |
| method-override | 映射新的 HTTP 动词到请求变量中的 _method                         |
| morgan          | 日志格式化                                                       |
| response-time   | 跟踪响应时间                                                     |
| serve-favicon   | 发送网站图标                                                     |
| serve-index     | 目录列表                                                         |
| whost           | 允许路由匹配子域名                                               |



# 项目管理

## 组件式构建

https://github.com/i0natan/nodebestpractices/blob/master/sections/projectstructre/breakintcomponents.chinese.md

## 多环境配置

- JSON 配置文件
- 环境变量
  使用第三方模块管理（nconf）

## 依赖管理

- dependencies：模块正常运行需要的依赖
- devDependencies：开发时候需要的依赖
- optionalDependencies：非必要依赖，某种程度上增强
- peerDependencies：运行时依赖，限定版本

# 异常处理

## 处理未捕获的异常

- 除非开发者记得添加.catch语句，在这些地方抛出的错误都不会被 uncaughtException 事件处理程序来处理，然后消失掉。
- Node 应用不会奔溃，但可能导致内存泄露

```js
process.on('uncaughtException', (error) => {
  // 我刚收到一个从未被处理的错误
  // 现在处理它，并决定是否需要重启应用
  errorManagement.handler.handleError(error);
  if (!errorManagement.handler.isTrustedError(error)) {
    process.exit(1);
  }
});

process.on('unhandledRejection', (reason, p) => {
  // 我刚刚捕获了一个未处理的promise rejection,
  // 因为我们已经有了对于未处理错误的后备的处理机制（见下面）
  // 直接抛出，让它来处理
  throw reason;
});
```

## 通过 domain 管理异常

- 通过 domain 模块的 create 方法创建实例
- 某个错误已经任何其他错误都会被同一个 error 处理方法处理
- 任何在这个回调中导致错误的代码都会被 domain 覆盖到
- 允许我们代码在一个沙盒运行，并且可以使用 res 对象给用户反馈

```js
const domain = require('domain');
const audioDomain = domain.create();

audioDomain.on('error', function(err) {
  console.log('audioDomain error:', err);
});

audioDomain.run(function() {
  const musicPlayer = new MusicPlayer();
  musicPlayer.play();
});
```

## Joi 验证参数

```js
const memberSchema = Joi.object().keys({
 password: Joi.string().regex(/^[a-zA-Z0-9]{3,30}$/),
 birthyear: Joi.number().integer().min(1900).max(2013),
 email: Joi.string().email(),
});
 
function addNewMember(newMember) {
 //assertions come first
 Joi.assert(newMember, memberSchema); //throws if validation fails
 
 //other logic here
}
```

## Kibana 系统监控

https://github.com/i0natan/nodebestpractices/blob/master/sections/production/smartlogging.chinese.md

# 上线实践

## 使用 winston 记录日记

```js
var winston = require('winston');
var moment = require('moment');

const logger = new (winston.Logger)({
  transports: [
    new (winston.transports.Console)({
      timestamp: function() {
        return moment().format('YYYY-MM-DD HH:mm:ss')
      },
      formatter: function(params) {
        let time = params.timestamp() // 时间
        let message = params.message // 手动信息
        let meta = params.meta && Object.keys(params.meta).length ? '\n\t'+ JSON.stringify(params.meta) : ''
        return `${time} ${message}`
      },
    }),
    new (winston.transports.File)({
      filename: `${__dirname}/../winston/winston.log`,
      json: false,
      timestamp: function() {
        return moment().format('YYYY-MM-DD HH:mm:ss')
      },
      formatter: function(params) {
        let time = params.timestamp() // 时间
        let message = params.message // 手动信息
        let meta = params.meta && Object.keys(params.meta).length ? '\n\t'+ JSON.stringify(params.meta) : ''
        return `${time} ${message}`
      }
    })
  ]
})

module.exports = logger

// logger.error('error')
// logger.warm('warm')
// logger.info('info')
```

## 委托反向代理

Node 处理 CPU 密集型任务，如 gzipping，SSL termination 等，表现糟糕。相反，使用一个真正的中间件服务像 Nginx 更好。否则可怜的单线程 Node 将不幸地忙于处理网络任务，而不是处理应用程序核心，性能会相应降低。

虽然 express.js 通过一些 connect 中间件处理静态文件，但你不应该使用它。Nginx 可以更好地处理静态文件，并可以防止请求动态内容堵塞我们的 node 进程。

```sh
# 配置 gzip 压缩
gzip on;
gzip_comp_level 6;
gzip_vary on;

# 配置 upstream
upstream myApplication {
  server 127.0.0.1:3000;
  server 127.0.0.1:3001;
  keepalive 64;
}

#定义 web server
server {
  # configure server with ssl and error pages
  listen 80;
  listen 443 ssl;
  ssl_certificate /some/location/sillyfacesociety.com.bundle.crt;
  error_page 502 /errors/502.html;

  # handling static content
  location ~ ^/(images/|img/|javascript/|js/|css/|stylesheets/|flash/|media/|static/|robots.txt|humans.txt|favicon.ico) {
  root /usr/local/silly_face_society/node/public;
  access_log off;
  expires max;
}
```

## 检测有漏洞的依赖项

https://docs.npmjs.com/cli/audit

## PM2 HTTP 集群配置

### 工作线程配置

- `pm2 start app.js -i 4`，`-i 4` 是以 cluster_mode 形式运行 app，有 4 个工作线程，如果配置 `0`，PM2 会根据 CPU 核心数来生成对应的工作线程
- 工作线程挂了 PM2 会立即将其重启
- `pm2 scale <app name> <n>` 对集群进行扩展

### PM2 自动启动

- `pm2 save` 保存当前运行的应用
- `pm2 startup` 启动

# 性能实践

## 避免使用 Lodash

- 使用像 lodash 这样的方法库这会导致不必要的依赖和较慢的性能
- 随着新的 V8 引擎和新的 ES 标准的引入，原生方法得到了改进，现在性能比方法库提高了 50%

使用 ESLint 插件检测：

```js
{
  "extends": [
    "plugin:you-dont-need-lodash-underscore/compatible"
  ]
}
```

## benchmark

```js
const _ = require('lodash'),
  __ = require('underscore'),
  Suite = require('benchmark').Suite,
  opts = require('./utils');
  //cf. https://github.com/Berkmann18/NativeVsUtils/blob/master/utils.js

const concatSuite = new Suite('concat', opts);
const array = [0, 1, 2];

concatSuite.add('lodash', () => _.concat(array, 3, 4, 5))
  .add('underscore', () => __.concat(array, 3, 4, 5))
  .add('native', () => array.concat(3, 4, 5))
  .run({ 'async': true });
```

## 使用 prof 进行性能分析

- 使用 tick-processor 工具处理分析

```sh
node --prof profile-test.js
```

```sh
npm install tick -g

node-tick-processor
```

## 使用 headdump 堆快照

- 代码加载模块进行快照文件生成
- Chrome Profiles 加载快照文件

```sh
yarn add heapdump -D
```

```js
const heapdump = require('heapdump');
const string = '1 string to rule them all';

const leakyArr = [];
let count = 2;
setInterval(function () {
  leakyArr.push(string.replace(/1/g, count++));
}, 0);

setInterval(function () {
  if (heapdump.writeSnapshot()) console.log('wrote snapshot');
}, 20000);
```

# 应用安全清单

## helmet 设置安全响应头

检测头部配置：[Security Headers](https://securityheaders.com/)。

应用程序应该使用安全的 header 来防止攻击者使用常见的攻击方式，诸如跨站点脚本攻击(XSS)、跨站请求伪造(CSRF)。可以使用模块 [helmet](https://www.npmjs.com/package/helmet) 轻松进行配置。

- 构造
    - X-Frame-Options：sameorigin。提供点击劫持保护，iframe 只能同源。
- 传输
    - Strict-Transport-Security：max-age=31536000; includeSubDomains。强制 HTTPS，这减少了web 应用程序中错误通过 cookies 和外部链接，泄露会话数据，并防止中间人攻击
- 内容
    - X-Content-Type-Options：nosniff。阻止从声明的内容类型中嗅探响应，减少了用户上传恶意内容造成的风险
    - Content-Type：text/html;charset=utf-8。指示浏览器将页面解释为特定的内容类型，而不是依赖浏览器进行假设
- XSS
    - X-XSS-Protection：1; mode=block。启用了内置于最新 web 浏览器中的跨站点脚本(XSS)过滤器
- 下载
    - X-Download-Options：noopen。
- 缓存
    - Cache-Control：no-cache。web 应中返回的数据可以由用户浏览器以及中间代理缓存。该指令指示他们不要保留页面内容，以免其他人从这些缓存中访问敏感内容
    - Pragma：no-cache。同上
    - Expires：-1。web 响应中返回的数据可以由用户浏览器以及中间代理缓存。该指令通过将到期时间设置为一个值来防止这种情况。
- 访问控制
    - Access-Control-Allow-Origin：not *。'Access-Control-Allow-Origin: *' 默认在现代浏览器中禁用
    - X-Permitted-Cross-Domain-Policies：master-only。指示只有指定的文件在此域中才被视为有效
- 内容安全策略
    - Content-Security-Policy：内容安全策略需要仔细调整并精确定义策略
- 服务器信息
    - Server：不显示。

## 使用 security-linter 插件

使用安全检验插件 [eslint-plugin-security](https://github.com/nodesecurity/eslint-plugin-security) 或者 [tslint-config-security](https://www.npmjs.com/package/tslint-config-security)。

## koa-ratelimit 限制并发请求

DOS 攻击非常流行而且相对容易处理。使用外部服务，比如 cloud 负载均衡, cloud 防火墙, nginx, 或者（对于小的，不是那么重要的app）一个速率限制中间件(比如 [koa-ratelimit](https://github.com/koajs/ratelimit))，来实现速率限制。

## 纯文本机密信息放置

存储在源代码管理中的机密信息必须进行加密和管理 (滚动密钥(rolling keys)、过期时间、审核等)。使用 pre-commit/push 钩子防止意外提交机密信息。

## ORM/ODM 库防止查询注入漏洞

要防止 SQL/NoSQL 注入和其他恶意攻击, 请始终使用 ORM/ODM 或 database 库来转义数据或支持命名的或索引的参数化查询, 并注意验证用户输入的预期类型。不要只使用 JavaScript 模板字符串或字符串串联将值插入到查询语句中, 因为这会将应用程序置于广泛的漏洞中。

库：

- TypeORM
- sequelize
- mongoose
- Knex
- Objection.js
- waterline

## 使用 Bcrypt 代替 Crypto

密码或机密信息(API 密钥)应该使用安全的 hash + salt 函数([bcrypt](https://www.npmjs.com/package/bcrypt))来存储, 因为性能和安全原因, 这应该是其 JavaScript 实现的首选。

```js
// 使用10个哈希回合异步生成安全密码
bcrypt.hash('myPassword', 10, function(err, hash) {
  // 在用户记录中存储安全哈希
});

// 将提供的密码输入与已保存的哈希进行比较
bcrypt.compare('somePassword', hash, function(err, match) {
  if(match) {
   // 密码匹配
  } else {
   // 密码不匹配
  } 
});
```

## 转义 HTML、JS 和 CSS 输出

发送给浏览器的不受信任数据可能会被执行, 而不是显示, 这通常被称为跨站点脚本(XSS)攻击。使用专用库将数据显式标记为不应执行的纯文本内容(例如:编码、转义)，可以减轻这种问题。

## 验证传入的 JSON schemas

验证传入请求的 body payload，并确保其符合预期要求, 如果没有, 则快速报错。为了避免每个路由中繁琐的验证编码, 您可以使用基于 JSON 的轻量级验证架构，比如 [jsonschema](https://www.npmjs.com/package/jsonschema) 或 [joi](https://www.npmjs.com/package/joi)

## 支持黑名单的 JWT

当使用 JSON Web Tokens(例如, 通过 [Passport.js](https://github.com/jaredhanson/passport)), 默认情况下, 没有任何机制可以从发出的令牌中撤消访问权限。一旦发现了一些恶意用户活动, 只要它们持有有效的标记, 就无法阻止他们访问系统。通过实现一个不受信任令牌的黑名单，并在每个请求上验证，来减轻此问题。

```js
const jwt = require('express-jwt');
const blacklist = require('express-jwt-blacklist');
 
app.use(jwt({
  secret: 'my-secret',
  isRevoked: blacklist.isRevoked
}));
 
app.get('/logout', function (req, res) {
  blacklist.revoke(req.user)
  res.sendStatus(200);
});
```

## 限制每个用户允许的登录请求

一类保护暴力破解的中间件，比如 express-brute，应该被用在 express 的应用中，来防止暴力/字典攻击；这类攻击主要应用于一些敏感路由，比如 `/admin` 或者 `/login`，基于某些请求属性, 如用户名, 或其他标识符, 如正文参数等。否则攻击者可以发出无限制的密码匹配尝试, 以获取对应用程序中特权帐户的访问权限。

```js
const ExpressBrute = require('express-brute');
const RedisStore = require('express-brute-redis');

const redisStore = new RedisStore({
  host: '127.0.0.1',
  port: 6379
});

// Start slowing requests after 5 failed 
// attempts to login for the same user
const loginBruteforce = new ExpressBrute(redisStore, {
  freeRetries: 5,
  minWait: 5 * 60 * 1000, // 5 minutes
  maxWait: 60 * 60 * 1000, // 1 hour
  failCallback: failCallback,
  handleStoreError: handleStoreErrorCallback
});

app.post('/login',
  loginBruteforce.getMiddleware({
    key: function (req, res, next) {
      // prevent too many attempts for the same username
      next(req.body.username);
    }
  }), // error 403 if we hit this route too often
  function (req, res, next) {
    if (User.isValidLogin(req.body.username, req.body.password)) {
      // reset the failure counter for valid login
      req.brute.reset(function () {
        res.redirect('/'); // logged in
      });
    } else {
      // handle invalid user
    }
  }
);
```

## 使用非 root 用户运行 Node.js

Node.js 作为一个具有无限权限的 root 用户运行，这是一种普遍的情景。例如，在 Docker 容器中，这是默认行为。建议创建一个非 root 用户，并保存到 Docker 镜像中（下面给出了示例），或者通过调用带有"-u username" 的容器来代表此用户运行该进程。否则在服务器上运行脚本的攻击者在本地计算机上获得无限制的权利 (例如，改变 iptable，引流到他的服务器上)

```dockerfile
FROM node:latest
COPY package.json .
RUN npm install
COPY . .
EXPOSE 3000
USER node
CMD ["node", "server.js"]
```

## 使用反向代理或中间件限制负载大小

请求 body 有效载荷越大, Node.js 的单线程就越难处理它。这是攻击者在没有大量请求(DOS/DDOS 攻击)的情况下，就可以让服务器跪下的机会。在边缘上（例如，防火墙，ELB）限制传入请求的 body 大小，或者通过配置 `express body parser` 仅接收小的载荷，可以减轻这种问题。否则您的应用程序将不得不处理大的请求, 无法处理它必须完成的其他重要工作, 从而导致对 DOS 攻击的性能影响和脆弱性。

express：

```js
const express = require('express');

const app = express();

// body-parser defaults to a body size limit of 300kb
app.use(express.json({ limit: '300kb' })); 

// Request with json body
app.post('/json', (req, res) => {

    // Check if request payload content-type matches json
    // because body-parser does not check for content types
    if (!req.is('json')) {
        return res.sendStatus(415); // Unsupported media type if request doesn't have JSON body
    }

    res.send('Hooray, it worked!');
});

app.listen(3000, () => console.log('Example app listening on port 3000!'));
```

nginx：

```
http {
    ...
    # Limit the body size for ALL incoming requests to 1 MB
    client_max_body_size 1m;
}

server {
    ...
    # Limit the body size for incoming requests to this specific server block to 1 MB
    client_max_body_size 1m;
}

location /upload {
    ...
    # Limit the body size for incoming requests to this route to 1 MB
    client_max_body_size 1m;
}
```

## 防止 RegEx 让 NodeJS 过载

匹配文本的用户输入需要大量的 CPU 周期来处理。在某种程度上，正则处理是效率低下的，比如验证 10 个单词的单个请求可能阻止整个 event loop 长达6秒。由于这个原因，偏向第三方的验证包，比如[validator.js](https://github.com/chriso/validator.js)，而不是采用正则，或者使用 [safe-regex](https://github.com/substack/safe-regex) 来检测有问题的正则表达式。

```js
const saferegex = require('safe-regex');
const emailRegex = /^([a-zA-Z0-9])(([\-.]|[_]+)?([a-zA-Z0-9]+))*(@){1}[a-z0-9]+[.]{1}(([a-z]{2,3})|([a-z]{2,3}[.]{1}[a-z]{2,3}))$/;

// should output false because the emailRegex is vulnerable to redos attacks
console.log(saferegex(emailRegex));

// instead of the regex pattern, use validator:
const validator = require('validator');
console.log(validator.isEmail('liran.tal@gmail.com'));
```

## 在沙箱中运行不安全代码

当任务执行在运行时给出的外部代码时(例如, 插件), 使用任何类型的沙盒执行环境保护主代码，并隔离开主代码和插件。这可以通过一个专用的过程来实现 (例如:cluster.fork()), 无服务器环境或充当沙盒的专用 npm 包。

- 一个专门的子进程 - 这提供了一个快速的信息隔离, 但要求制约子进程, 限制其执行时间, 并从错误中恢复
- 一个基于云的无服务框架满足所有沙盒要求，但动态部署和调用Faas方法不是本部分的内容
- 一些 npm 库，比如 [sandbox](https://www.npmjs.com/package/sandbox) 和 [vm2](https://www.npmjs.com/package/vm2) 允许通过一行代码执行隔离代码。尽管后一种选择在简单中获胜, 但它提供了有限的保护。

```js
const Sandbox = require("sandbox");
const s = new Sandbox();

s.run( "lol)hai", function( output ) {
  console.log(output);
  //output='Synatx error'
});

// Example 4 - Restricted code
s.run( "process.platform", function( output ) {
  console.log(output);
  //output=Null
})

// Example 5 - Infinite loop
s.run( "while (true) {}", function( output ) {
  console.log(output);
  //output='Timeout'
})
```

## 隐藏客户端的错误详细信息

默认情况下, 集成的 express 错误处理程序隐藏错误详细信息。但是, 极有可能, 您实现自己的错误处理逻辑与自定义错误对象(被许多人认为是最佳做法)。如果这样做, 请确保不将整个 Error 对象返回到客户端, 这可能包含一些敏感的应用程序详细信息。否则敏感应用程序详细信息(如服务器文件路径、使用中的第三方模块和可能被攻击者利用的应用程序的其他内部工作流)可能会从 stack trace 发现的信息中泄露。

```
// production error handler
// no stacktraces leaked to user
app.use(function(err, req, res, next) {
    res.status(err.status || 500);
    res.render('error', {
        message: err.message,
        error: {}
    });
});
```

## 对 npm 或 Yarn，配置 2FA

开发链中的任何步骤都应使用 MFA(多重身份验证)进行保护, npm/Yarn 对于那些能够掌握某些开发人员密码的攻击者来说是一个很好的机会。使用开发人员凭据, 攻击者可以向跨项目和服务广泛安装的库中注入恶意代码。甚至可能在网络上公开发布。在 npm 中启用两层身份验证（2-factor-authentication）, 攻击者几乎没有机会改变您的软件包代码。

https://itnext.io/eslint-backdoor-what-it-is-and-how-to-fix-the-issue-221f58f1a8c8

## session 中间件设置

每个 web 框架和技术都有其已知的弱点，告诉攻击者我们使用的 web 框架对他们来说是很大的帮助。使用 session 中间件的默认设置, 可以以类似于 `X-Powered-Byheader` 的方式向模块和框架特定的劫持攻击公开您的应用。尝试隐藏识别和揭露技术栈的任何内容(例如:Nonde.js, express)。否则可以通过不安全的连接发送cookie, 攻击者可能会使用会话标识来标识web应用程序的基础框架以及特定于模块的漏洞。

```js
// using the express session middleware
app.use(session({  
 secret: 'youruniquesecret', // secret string used in the signing of the session ID that is stored in the cookie
 name: 'youruniquename', // set a unique name to remove the default connect.sid
 cookie: {
   httpOnly: true, // minimize risk of XSS attacks by restricting the client from reading the cookie
   secure: true, // only send cookie over https
   maxAge: 60000*60*24 // set cookie expiry length in ms
 }
}));
```

## csurf 防止 CSRF

路由层：

```js
var cookieParser = require('cookie-parser');  
var csrf = require('csurf');  
var bodyParser = require('body-parser');  
var express = require('express');

// 设置路由中间件
var csrfProtection = csrf({ cookie: true });  
var parseForm = bodyParser.urlencoded({ extended: false });

var app = express();

// 我们需要这个，因为在 csrfProtection 中 “cookie” 是正确的
app.use(cookieParser());

app.get('/form', csrfProtection, function(req, res) {  
  // 将 CSRFToken 传递给视图
  res.render('send', { csrfToken: req.csrfToken() });
});

app.post('/process', parseForm, csrfProtection, function(req, res) {  
  res.send('data is being processed');
});
```

展示层：

```html
<form action="/process" method="POST">  
  <input type="hidden" name="_csrf" value="{{csrfToken}}">

  Favorite color: <input type="text" name="favoriteColor">
  <button type="submit">Submit</button>
</form>  
```
