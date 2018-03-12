
缓存是node开发中一个很重要的概念，它应用在很多地方，例如：浏览器有缓存、DNS有缓存、包括服务器也有缓存。

### 一、缓存作用
那缓存是为了做什么呢？
> 1.为了提高速度，提高效率。

> 2.减少数据传输，节省网费。

> 3.减少服务器的负担，提高网站性能。

> 4.加快客户端加载网页的速度。

### 二、缓存分类
那缓存有几种策略呢？
#### 强制缓存：

##### 1、概念：
客户端访问服务器请求资源，请求成功之后客户端会缓存到本地，缓存到本地之后，如果以后客户端再请求该资源此时不需要请求服务器了，直接访问本地的就可以。
##### 2、特点：
强制缓存不需要与服务器发生交互
##### 3、客户端访问强制缓存的流程图解

![](https://user-gold-cdn.xitu.io/2018/3/10/1620f11d27631a3f?w=865&h=332&f=png&s=25340)
1）缓存命中
客户端请求数据，现在本地的缓存数据库中查找，如果本地缓存数据库中有该数据，且该数据没有失效。则取缓存数据库中的该数据返回给客户端。

2）缓存未命中
客户端请求数据，现在本地的缓存数据库中查找，如果本地缓存数据库中有该数据，且该数据失效。则向服务器请求该数据，此时服务器返回该数据和该数据的缓存规则返回给客户端，客户端收到该数据和缓存规则后，一起放到本地的缓存数据库中留存。以备下次使用。
##### 4、如何实现强制缓存？
>1、浏览器会将文件缓存到Cache目录，第二次请求时浏览器会先检查Cache目录下是否含有该文件，如果有，并且还没到Expires设置的时间，即文件还没有过期，那么此时浏览器将直接从Cache目录中读取文件，而不再发送请求
2、Expires是服务器响应消息头字段，在响应http请求时告诉浏览器在过期时间前浏览器可以直接从浏览器缓存取数据，而无需再次请求,这是HTTP1.0的内容，现在浏览器均默认使用HTTP1.1,所以基本可以忽略
3、Cache-Control与Expires的作用一致，都是指明当前资源的有效期，控制浏览器是否直接从浏览器缓存取数据还是重新发请求到服务器取数据,如果同时设置的话，其优先级高于Expires

 把资源缓存在客户端，如果客户端再次需要此资源的时候，先获取到缓存中的数据，看是否过期，如果过期了。再请求服务器
如果没过期，则根本不需要向服务器确认，直接使用本地缓存即可


Cache-Control
private 客户端可以缓存
public 客户端和代理服务器都可以缓存
max-age=60 缓存内容将在60秒后失效
no-cache 需要使用对比缓存验证数据,强制向源服务器再次验证. 禁用强制缓存
no-store 所有内容都不会缓存，强制缓存和对比缓存都不会触发。兼用强制缓存和对比缓存å

 
 ```
 /**
 * 1. 第一次访问服务器的时候，服务器返回资源和缓存的标识，客户端则会把此资源缓存在本地的缓存数据库中。
 * 2. 第二次客户端需要此数据的时候，要取得缓存的标识，然后去问一下服务器我的资源是否是最新的。
 * 如果是最新的则直接使用缓存数据，如果不是最新的则服务器返回新的资源和缓存规则，客户端根据缓存规则缓存新的数据。
 */
let http = require('http');
let url = require('url');
let path = require('path');
let fs = require('fs');
let mime = require('mime');
let crypto = require('crypto');
/**
 * 强制缓存
 * 把资源缓存在客户端，如果客户端再次需要此资源的时候，先获取到缓存中的数据，看是否过期，如果过期了。再请求服务器
 * 如果没过期，则根本不需要向服务器确认，直接使用本地缓存即可
 */
http.createServer(function (req, res) {
    let { pathname } = url.parse(req.url, true);
    let filepath = path.join(__dirname, pathname);
    console.log(filepath);
    fs.stat(filepath, (err, stat) => {
        if (err) {
            return sendError(req, res);
        } else {
            send(req, res, filepath);
}
});
}).listen(8080);
function sendError(req, res) {
    res.end('Not Found');
}
function send(req, res, filepath) {
    res.setHeader('Content-Type', mime.getType(filepath));
    //expires指定了此缓存的过期时间，此响应头是1.0定义的，在1.1里面已经不再使用了
    res.setHeader('Expires', new Date(Date.now() + 30 * 1000).toUTCString());
    res.setHeader('Cache-Control', 'max-age=30');
    fs.createReadStream(filepath).pipe(res);
}
 ```




#### 对比缓存：
##### 1、概念：
浏览器第一次请求数据时，服务器会将缓存标识与数据一起返回给客户端，客户端将二者备份至缓存数据库中。
再次请求数据时，客户端将备份的缓存标识发送给服务器，服务器根据缓存标识进行判断，判断成功后，返回304状态码，通知客户端比较成功，可以使用缓存数据。
##### 2、特点：需要进行比较判断是否可以使用缓存
##### 3、对比缓存流程图解
1）客户端第一次发请求
![](https://user-gold-cdn.xitu.io/2018/3/10/1620f11d34744937?w=501&h=412&f=png&s=17354)
客户端第一次请求数据，发现本地缓存中没有，就向服务器发起请求，然后服务器把请求的数据返回给客户端，并和客户端商量你要保存到本地缓存中的规则，即是否缓存 缓存时间 有没有标示 最后修改时间等信息。
2）客户端第二次发请求
![](https://user-gold-cdn.xitu.io/2018/3/10/1620f11d15571a64?w=1138&h=731&f=png&s=67761)
客户端发起请请求
--->查看本地的缓存数据库中是否有缓存---> 没有---> 向服务器发起请求--->服务器返回200和响应内容--->显示


--->查看本地的缓存数据库中是否有缓存---> 有 ---> 缓存没有过期（本地）---> 缓存中读取--->显示

--->查看本地的缓存数据库中是否有缓存---> 有 ---> 缓存已过期（本地）---> 本地的缓存中有没有Etag和Last-Modified --->有--->发给服务器对应的字段 if-none-match 和if-modified-since ---> 服务器策略。如果这两个字段和服务器上的这两个字段相同 ---> 说明数据没有更新--->返回304--->服务器从它的缓存库中获取到数据给客户端--->显示


--->查看本地的缓存数据库中是否有缓存---> 有 ---> 缓存已过期（本地）---> 本地的缓存中有没有Etag和Last-Modified --->有--->发给服务器对应的字段 if-none-match 和if-modified-since ---> 服务器策略。如果这两个字段和服务器上的这两个字段相同 --->说明数据有更新--->返回200--->重新获取--->显示
##### 4、如何实现对比缓存？

/**
 * 1. 第一次访问服务器的时候，服务器返回资源和缓存的标识，客户端则会把此资源缓存在本地的缓存数据库中。
 * 2. 第二次客户端需要此数据的时候，要取得缓存的标识，然后去问一下服务器我的资源是否是最新的。
 * 如果是最新的则直接使用缓存数据，如果不是最新的则服务器返回新的资源和缓存规则，客户端根据缓存规则缓存新的数据。
 */
我们通过标示字段来判断缓存中的数据是否有效
这个标示有两种形式：

##### 第一种是最后修改时间，Last-Modified
>
1、Last-Modified：响应时告诉客户端此资源的最后修改时间
2、If-Modified-Since：当资源过期时（使用Cache-Control标识的max-age），发现资源具有Last-Modified声明，则再次向服务器请求时带上头If-Modified-Since。
3、服务器收到请求后发现有头If-Modified-Since则与被请求资源的最后修改时间进行比对。若最后修改时间较新，说明资源又被改动过，则响应最新的资源内容并返回200状态码；
4、若最后修改时间和If-Modified-Since一样，说明资源没有修改，则响应304表示未更新，告知浏览器继续使用所保存的缓存文件。

```
let http = require('http');
let url = require('url');
let path = require('path');
let fs = require('fs');
let mime = require('mime');
// http://localhost:8080/index.html
http.createServer(function (req, res) {
    let {pathname} = url.parse(req.url);
    let filepath = path.join(__dirname,pathname);
    console.log(filepath);
    fs.stat(filepath,function (err, stat) {
          if (err) {
              return sendError(req,res)
          } else {
              // 再次请求的时候会问服务器自从上次修改之后有没有改过
              let ifModifiedSince = req.headers['if-modified-since'];
              console.log(req.headers);
              let LastModified = stat.ctime.toGMTString();
              console.log(LastModified);
              if (ifModifiedSince == LastModified) {
                  res.writeHead('304');
                  res.end('')
              } else {
                  return send(req,res,filepath,stat)
              }
          }
    })

}).listen(8080)

function send(req,res,filepath,stat) {
    res.setHeader('Content-Type', mime.getType(filepath));
    // 发给客户端之后，客户端会把此时间保存下来，下次再获取此资源的时候会把这个时间再发给服务器
    res.setHeader('Last-Modified', stat.ctime.toGMTString());
    fs.createReadStream(filepath).pipe(res)
}

function sendError(req,res) {
    res.end('Not Found')
}
```
###### 最后修改时间存在问题
>1、某些服务器不能精确得到文件的最后修改时间， 这样就无法通过最后修改时间来判断文件是否更新了。
2、某些文件的修改非常频繁，在秒以下的时间内进行修改. Last-Modified只能精确到秒。
3、一些文件的最后修改时间改变了，但是内容并未改变。 我们不希望客户端认为这个文件修改了。
4、如果同样的一个文件位于多个CDN服务器上的时候内容虽然一样，修改时间不一样。


###### 第二种是Etag
ETag是实体标签的缩写，根据实体内容生成的一段hash字符串,可以标识资源的状态。当资源发生改变时，ETag也随之发生变化。 ETag是Web服务端产生的，然后发给浏览器客户端。
>1、客户端想判断缓存是否可用可以先获取缓存中文档的ETag，然后通过If-None-Match发送请求给Web服务器询问此缓存是否可用。
2、服务器收到请求，将服务器的中此文件的ETag,跟请求头中的If-None-Match相比较,如果值是一样的,说明缓存还是最新的,Web服务器将发送304 Not Modified响应码给客户端表示缓存未修改过，可以使用。
3、如果不一样则Web服务器将发送该文档的最新版本给浏览器客户端


```
let http = require('http');
let url = require('url');
let path = require('path');
let fs = require('fs');
let mime = require('mime');
let crypto = require('let crypto = require(\'mime\');\n');
// http://localhost:8080/index.html
http.createServer(function (req, res) {
    let {pathname} = url.parse(req.url);
    let filepath = path.join(__dirname,pathname);
    console.log(filepath);
    fs.stat(filepath,function (err, stat) {
        if (err) {
            return sendError(req,res)
        } else {

            let ifNoneMatch = req.headers['if-none-match'];
            // 一、显然当我们的文件非常大的时候通过下面的方法就行不通来，这时候我们可以用流来解决,可以节约内存
            let out = fs.createReadStream(filepath);
            let md5 = crypto.createHash('md5');
            out.on('data',function (data) {
                md5.update(data)
            });
            out.on('end',function () {
                let etag = md5.update(content).digest('hex');
                // md5算法的特点 1. 相同的输入相同的输出 2.不同的输入不通的输出 3.不能根据输出反推输入 4.任意的输入长度输出长度是相同的
                if (ifNoneMatch == etag) {
                    res.writeHead('304');
                    res.end('')
                } else {
                    return send(req,res,filepath,stat, etag)
                }
            });
            
            // 二、再次请求的时候会问服务器自从上次修改之后有没有改过
            // fs.readFile(filepath,function (err, content) {
            //     let etag = crypto.createHash('md5').update(content).digest('hex');
            //     // md5算法的特点 1. 相同的输入相同的输出 2.不同的输入不通的输出 3.不能根据输出反推输入 4.任意的输入长度输出长度是相同的
            //     if (ifNoneMatch == etag) {
            //         res.writeHead('304');
            //         res.end('')
            //     } else {
            //         return send(req,res,filepath,stat, etag)
            //     }
            // };
            // 但是上面的一方案也不是太好，读一点缓存一点，文件非常大的话需要好长时间，而且我们的node不适合cup密集型，即不适合来做大量的运算，所以说还有好多其他的算法
            // 三、通过文件的修改时间减去文件的大小
            // let etag = `${stat.ctime}-${stat.size}`; // 这个也不是太好
            // if (ifNoneMatch == etag) {
            //     res.writeHead('304');
            //     res.end('')
            // } else {
            //     return send(req,res,filepath,stat, etag)
            // }
        }
    })

}).listen(8080)

function send(req,res,filepath,stat, etag) {
    res.setHeader('Content-Type', mime.getType(filepath));
    // 第一次服务器返回的时候，会把文件的内容算出来一个标示发送给客户端
    //客户端看到etag之后，也会把此标识符保存在客户端，下次再访问服务器的时候，发给服务器
    res.setHeader('Etag', etag);
    fs.createReadStream(filepath).pipe(res)
}

function sendError(req,res) {
    res.end('Not Found')
}
```
###### 存在问题
都需要向服务器端发请求与服务器端发生交互


