**首先来看看我要准备给大家写的静态文件服务器都要实现哪些功能，然后根据具体的功能我们来一一的介绍** 

支持以下功能
- 支持输出日志debug
- 读取静态文件
     - 文件的读取 MIME类型支持
     - 文件夹列表展示 handlebars模板引擎
- 缓存支持/控制
- Range支持，断点续传
- 支持gzip和deflate压缩
- 发布为可执行命令并可以后台运行,可以在全局通过下面的命令来设置根目录端口号和主机名：myselfsever -d指定静态文件的根目录 -p指定端口号 -o指定监听的主机

[下载地址](https://github.com/qinmengjiao/staticserver)

启动

```
    npm install // 安装依赖
    npm link // 创建连接
    myselfserver // 在任意目录下启动服务
    // 访问 localhost:8080
```

**其次我们先大致看一下我们的服务器的大致结构**

![image](http://upload-images.jianshu.io/upload_images/10097032-d3a5c95778e9c7fb?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

接下来我们来介绍我们这个服务器

要想实现一个服务器的功能，我们往往需要设置一下服务器的主机名，端口号，静态文件根目录等信息，在这里我们的config.js帮我们实现了这一配置，当然了这是基础的配置，后面我们还会讲到如何通过命令行来更改配置。基础配置如下：



	let path = require('path');
	let config = {
	    host: 'localhost',// 监听主机
	    port: 8080,// 主机端口号
	    root: path.resolve(__dirname, '..')// 静态文件根目录
	}
	module.exports = config;


配置好之后接下来我们就需要写一个我们的服务了，服务的代码会在我们的app.js中体现，其中包括我们上面列举的几乎所有的功能，我们先来看一下app.js的代码结构


```

	const fs = require('fs');
	const url = require('url');
	const http = require('http');
	const path = require('path');
	const config = require('./config');
	
	class Server {
	    constructor(argv) {
			// 生产handlebars模板
			this.list = list()
	        // 处理命令行中设置的参数，来重写基本参数
	        this.config = Object.assign({}, config, argv)
	    }
	    start() {
			/*启动http服务*/
	        let server = http.createServer();
	        server.on('request', this.request.bind(this));
	        let url = `http://${config.host}:${config.port}`;
	
	        server.listen(this.config.port, ()=>{
                // （1）支持输出日志debug 在下文中会讲一下使用它的注意事项
	            debug(`server started at ${chalk.green(url)}`);
	        })
	    }
	    /*（2）读取静态文件*/
	    async request(req,res) {
	      // 因为响应可能出错，所以在我们的代码中这里会用try catch包一下。
          // try
          // 文件夹： （2.1）模板引擎渲染展示文件列表  这里用handlebars模板引擎来渲染
          // 文件：-->sendFile
          // catch
          // 处理错误交给-->sendError
	    }
	    /* send file to browser*/
	    sendFile (req, res, filePath, statObj) {
	        // （2.2）处理文件并发送给浏览器  添加MIME类型支持
	    }
	    /* handle error*/
	    sendError (error, req, res) {
	        // 公用的的错误处理函数
	    }
	    /*cache 是否走缓存*/
	    isCache (req, res, filePath, statObj) {
	        // （3）处理缓存
	    }
	    /*broken-point continuingly-transferring  断点续传*/
	    rangeTransfer (req, res, filePath, statObj) {
	        // （4）支持断点续传
	    }
	    /*compression 压缩*/
	    compression (req, res) {
	       // （5）支持gzip和deflate压缩
	    }
	}
	module.exports = Server;
    （6）myselfsever -d指定静态文件的根目录 -p指定端口号 -o指定监听的主机 这个命令实现的脚本代码在 bin目录下的commond文件中
```
### 开始进入细节分析阶段

**注：每用到一个模块的时候记得提前安装到本地**

#### （1）如何打印错误日志---debug


```

const debug = require('debug'); // 安装引入debug包
debug('staticserver:app');
// debug 第三方模块返回的是一个函数，该函数执行需要传入两个参数，一个是你当前项目的名称，即package.json 中的name值，第二个参数是你的模块的服务入口文件的名称。在咱们的这个项目中是app.js

```

当我们项目中要使用debug错误输出模块时，需要配置环境变量我们的debug日志才会被输出 


环境变量的配置方法
windows上配置环境变量

- 配置单个文件，即只有当前文件中的debug日志会输出 （优势：可以灵活的配置，输出自己想要输出的文件的debug日志）

```
   $ set DEBUG=staticserver:app
```

- 配置整个项目，整个项目中的debug日志都会被输出

```
   $ set DEBUG=staticserver:*
```

mac上配置环境变量

```
  $ export DEBUG=staticserver:app
  $ export DEBUG=staticserver:*
```


#### （2）读取静态文件


如果是文件夹的话显示文件列表，这里我们用到了[handlebars模板引擎](http://handlebarsjs.com/)

```
	const handlebars = require('handlebars'); // 模版引擎
    // 调用handlebars.compile方法生成一个模板
	function list () {
	    let tmpl = fs.readFileSync(path.resolve(__dirname,'template','list.html'),'utf8');
	    return handlebars.compile(tmpl)
	}

```

接下来我们来看访问的路径是文件的情况。这时候我们会根据文件的后缀的不同返回不同的响应类型，这时我们就用到了我们的[mime模块](https://www.npmjs.com/package/mime)

```

    /*静态文件服务器*/
	const { promisify, inspect } = require('util');
    const mime = require('mime');
	const stat = promisify(fs.stat);
	const readdir = promisify(fs.readdir);
    async request(req,res) {
        // 先取到客户端想访问的路径
        let { pathname } = url.parse(req.url);
        // 如果访问的是favicon.ico 的话返回一个错误信息
        if (pathname == '/favicon.ico') {
            return this.sendError('not favicon.ico',req,res)
        }
        // 获取访问的文件或文件夹的路径
        let filePath = path.join(this.config.root, pathname);
        try{
            // 判断访问的路径是文件夹 还是文件
           let statObj = await stat(filePath);
           if (statObj.isDirectory()) {// 如果是目录的话 应该显示目录下面的文件列表 否则显示文件内容
               let files = await readdir(filePath);
               files = files.map(file => ({
                   name: file,
                   url: path.join(pathname, file)
               }))
                let html = this.list({
                    title: pathname,
                    files,
                });
               res.setHeader('Content-Type', 'text/html');
               res.end(html)
           } else {
            // 如果是文件的话显示文件内容
               this.sendFile(req, res, filePath, statObj)
           }
        }catch (e) {
            debug(inspect(e)); // 把一个对象转化为字符串，因为有的tosting会生成object object
            this.sendError(e, req, res);
        }
    }

    // 接下来我们来看访问的路径是文件的情况。这时候我们会根据文件的后缀的不同返回不同的响应类型，这时我们就用到了我们的mime模块
  
      sendFile (req, res, filePath, statObj) {
        // 如果缓存存在的话走缓存
        if(this.isCache(req, res, filePath, statObj)){
            return;
        }
        res.statusCode = 200; // 可以省略
        res.setHeader('Content-Type', mime.getType(filePath) + ';charset=utf-8');
        let encoding = this.compression(req,res);
        // 是不是需要压缩
        if(encoding) {
            // 在这里使用断点续传
            this.rangeTransfer(req, res, filePath, statObj).pipe(encoding).pipe(res)
        } else {
           
            this.rangeTransfer(req, res, filePath, statObj).pipe(res)
        }
    }

```


#### （3）缓存支持和控制

要想理解下面缓存是否存在，缓存是否有效来判断要不要走缓存，大家可以先看一下这篇文章[node中的缓存机制](https://juejin.im/post/5aa39a12f265da237c68834c)可以加深对缓存的理解

```

	/*cache 是否走缓存*/
    isCache (req, res, filePath, statObj) {
        let ifNoneMatch = req.headers['if-none-match'];
        let ifModifiedSince = req.headers['if-modified-since'];
        res.setHeader('Cache-Control','private,max-age=10');
        res.setHeader('Expires',new Date(Date.now() + 10*1000).toGMTString);
        let etag = statObj.size;
        let lastModified = statObj.ctime.toGMTString();
        res.setHeader('Etag',etag)
        res.setHeader('Last-Modified',lastModified);
        if(ifNoneMatch && ifNoneMatch != etag) {
            return false
        }

        if(ifModifiedSince && ifModifiedSince != lastModified){
            return false
        }
        if(ifNoneMatch || ifModifiedSince) {
            res.writeHead(304);
            res.end();
            return true
        } else {
            return false
        }
    }

```



#### （4）Range支持，断点续传

该选项指定下载字节的范围，常应用于分块下载文件

range的表示方式有多种，如100-500，则指定从100开始的400个字节数据；-500表示最后的500个字节；5000-表示从第5000个字节开始的所有字节

另外还可以同时指定多个字节块，中间用","分开

服务器告诉客户端可以使用range response.setHeader('Accept-Ranges', 'bytes')

Server通过请求头中的Range:bytes=0-xxx来判断是否是做Range请求，如果这个值存在而且有效，则只发回请求的那部分文件内容，响应的状态码变成206,如果无效，则返回416状态码，表明Request Range Not Satisfiable

```

     /*broken-point continuingly-transferring  断点续传*/
    rangeTransfer (req, res, filePath, statObj) {
        let start = 0;
        let end = statObj.size-1;
        let range = req.headers['range'];
        if(range){
            res.setHeader('Accept-range','bytes');
            res.statusCode=206// 返回整个内容的一块
            let result = range.match(/bytes=(\d*)-(\d*)/);
            start = isNaN(result[1]) ? start : parseInt(result[1]);
            end = isNaN(result[2]) ? end : parseInt(result[2]) - 1
        }
        return fs.createReadStream(filePath, {
            start,
            end
        })
    }

```

#### （5）支持gzip和deflate压缩

服务器会根据客户端请求中的req.headers['accept-encoding']这个字段中的值来判断客户端支持哪种解压类型来进行压缩的

```
	/*compression 压缩*/
    compression (req, res) {
        let acceptEncoding = req.headers['accept-encoding'];//163.com
        if(acceptEncoding) {
            if(/\bgzip\b/.test(acceptEncoding)){
                res.setHeader('Content-Encoding','gzip');
                return zlib.createGzip();
            } else if(/\bdeflate\b/.test(acceptEncoding)) {
                res.setHeader('Content-Encoding','deflate');
                return zlib.createDeflate();
            } else {
                return null
            }
        }
    }

```

#### （6）发布为可执行命令并可以后台运行,可以在全局通过下面的命令来设置根目录端口号和主机名：myselfsever -d指定静态文件的根目录 -p指定端口号 -o指定监听的主机


这个功能是在我们的bin目录下面的command 文件中实现的。
文件开头的#! /usr/bin/env node 这句命令是告诉该脚本用哪种程序来执行，详细可看这边文章[Node.js 命令行程序开发教程](http://www.cnblogs.com/qinmengjiao123-123/p/8520807.html)

```

	#! /usr/bin/env node
	let yargs = require('yargs');
	let Server = require('../src/app.js')
	let argv = yargs.option('d', {
	    alias: 'root',
	    demand: false,
	    description: '请配置监听静态文件目录',
	    default: process.cwd(),
	    type: 'string'
	}).option('o', {
	    alias: 'host',
	    demand: false,
	    description: '请配置监听的主机',
	    default: 'localhost',
	    type: 'string'
	}).option('p', {
	    alias: 'port',
	    demand: false,
	    description: '请配置监听的主机的端口号',
	    default: 8080,
	    type: 'number'
	}).usage('myselfsever -d / -p 8080 -o localhost')
	    .example(
	        'myselfsever -d / -p 9090 -o localhost', '在本机器的9090端口上监听客户端发来的请求'
	    ).help('h').argv;
	
	
	let server = new Server();
	server.start(argv);
	
```

每天都进步一点;不要畏惧陌生的事物,每天都学习一点;
