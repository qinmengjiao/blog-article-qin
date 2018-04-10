
### webpack-loader是什么？

![](https://user-gold-cdn.xitu.io/2018/4/10/162b027a91fb2482?w=100&h=100&f=png&s=4791)

loader是一个函数，有一个参数是源文件内容，有内部的this，最后的loader必须有返回值否则会报错：Final loader didn't return a Buffer or String

原因：
>如果是处理顺序排在最后一个的 Loader，那么它的返回值将最终交给webpack的require，换句话说，它一定是一段用字符串来存储可执行的JavaScript脚本

### 如何配置webpack-loader？

##### 配置单个loader

```
    {
      test: /\.js$/
      use: [
        {
          loader: path.resolve(__dirname, 'src', 'loaders', 'first-loader.js'),
          options: {/*这里是给loader传入的参数会有一个专门的库来处理这个参数，待会下面会讲到*/}
        }
      ]
    }
```
##### 配置多个loader

 如果把我们自己写的loader当做一个模块来加载的时候，webpack默认的会去查找node_modules下面的模块，但是我们自己写的loader没有在node_modules中 这时候我们怎么样让他也查找我们自己写的loader呢？这时候我们就需要设置一个属性 resolveLoader他是一个对象，里面有一个modules属性 该属性的值是一个数组 数组中 的每个值就是要查找的loader存放地

```
    resolveLoader: {
       modules: [
    		path.resolve('node_modules'),
    	    path.resolve(__dirname, 'src', 'loaders')
    	]
    },
```
再配置loader

```
    {
        test: /\.less$/,
        use: ['style-loader','less-loader']
    }
```





### loader的用法准则

> 单一职责，一个loader只做一件事情，这样设计的原因是因为，职责越单一，组合性就强，可配置性就好。
> 
> 从右到左，链式执行，上一个loader的处理结果传给下一个loader接着处理
> 
> 模块化，一个loader就是一个模块，单独存在不依赖其他的任何模块
> 无状态，就类似于纯函数。
> 
> loader有实用工具loader-utils解析loader参数的和schema-utils校验格式的
> 
> loader的依赖

### loader相关API

1、缓存
webpack充分地利用缓存来提高编译效率

```
    this.cacheable();
```

2、异步
当一个 Loader 无依赖，可异步的时候我想都应该让它不再阻塞地去异步


```
    // 让 Loader 缓存
    module.exports = function(source) {
        var callback = this.async();
        // 做异步的事
        doSomeAsyncOperation(content, function(err, result) {
            if(err) return callback(err);
            callback(null, result);
        });
    };
```

3、传入的源文件以buffer的形式

默认的情况源文件是以 UTF-8 字符串的形式传入给 Loader,设置module.exports.raw = true可使用 buffer 的形式进行处理


```
    module.exports.raw = true
```

4、获取loader的options

```
    const loaderUtils = require('loader-utils');
    module.exports = function (source) {
    	// 获取当前用户给当前loader传入的参数对象options
        const options = loaderUtils.getOptions(this);
    	return source;
    }
```

5、校验传入的options的值是不是符合要求

```
    const loaderUtils = require('loader-utils');
    const validate = require('schema-utils');
    let json = {
        "type": "object",
        "properties": {
            "content": {
                "type": "string",
            }
        }
    }
    module.exports = function (source) {
        const options = loaderUtils.getOptions(this);
    	// 第一个参数是校验的json 第二个参数是loader传入的options 第三个参数是当前loader的名称
        validate(json, options, 'first-loader');
        console.log(options.content)
    }
    // 当前规定 传入的options必须是一个对象 他里面的content的值必须是一个字符串 如果不满足要求则会有好的报错  xxx应该是一个什么类型
```

6、loader的返回其他结果

Loader有些场景下还需要返回除了内容之外的东西。

```
    module.exports = function(source) {
      // 通过 this.callback 告诉 Webpack 返回的结果
      this.callback(null, source, sourceMaps);
      // 当你使用 this.callback 返回内容时，该 Loader 必须返回 undefined，
      // 以让 Webpack 知道该 Loader 返回的结果在 this.callback 中，而不是 return 中 
      return;
    };
```


```
    this.callback(
        // 当无法转换原内容时，给 Webpack 返回一个 Error
        err: Error | null,
        // 原内容转换后的内容
        content: string | Buffer,
        // 用于把转换后的内容得出原内容的 Source Map，方便调试
        sourceMap?: SourceMap,
        // 如果本次转换为原内容生成了 AST 语法树，可以把这个 AST 返回，
        // 以方便之后需要 AST 的 Loader 复用该 AST，以避免重复生成 AST，提升性能
        abstractSyntaxTree?: AST
    );
```

7、同步与异步

Loader 有同步和异步之分，上面介绍的 Loader 都是同步的 Loader，因为它们的转换流程都是同步的，转换完成后再返回结果。 但在有些场景下转换的步骤只能是异步完成的，例如你需要通过网络请求才能得出结果，如果采用同步的方式网络请求就会阻塞整个构建，导致构建非常缓慢。

```
    module.exports = function(source) {
        // 告诉 Webpack 本次转换是异步的，Loader 会在 callback 中回调结果
        var callback = this.async();
        someAsyncOperation(source, function(err, result, sourceMaps, ast) {
            // 通过 callback 返回异步执行后的结果
            callback(err, result, sourceMaps, ast);
        });
    };
```

8、处理二进制数据 

在默认的情况下，Webpack 传给 Loader 的原内容都是 UTF-8 格式编码的字符串。 但有些场景下 Loader 不是处理文本文件，而是处理二进制文件，例如 file-loader，就需要 Webpack 给 Loader 传入二进制格式的数据。 为此，你需要这样编写 Loader：

```
    module.exports = function(source) {
        // 在 exports.raw === true 时，Webpack 传给 Loader 的 source 是 Buffer 类型的
        source instanceof Buffer === true;
        // Loader 返回的类型也可以是 Buffer 类型的
        // 在 exports.raw !== true 时，Loader 也可以返回 Buffer 类型的结果
        return source;
    };
    // 通过 exports.raw 属性告诉 Webpack 该 Loader 是否需要二进制数据 
    module.exports.raw = true;
```

9、缓存


在有些情况下，有些转换操作需要大量计算非常耗时，如果每次构建都重新执行重复的转换操作，构建将会变得非常缓慢。 为此，Webpack 会默认缓存所有 Loader 的处理结果，也就是说在需要被处理的文件或者其依赖的文件没有发生变化时， 是不会重新调用对应的 Loader 去执行转换操作的。


```
    module.exports = function(source) {
     // 关闭该 Loader 的缓存功能
       this.cacheable(false);
       return source;
    };
```

10、其它 Loader API

>this.context：当前处理文件的所在目录，假如当前 Loader 处理的文件是 /src/main.js，则 this.context 就等于 /src。
>
>this.resource：当前处理文件的完整请求路径，包括 querystring，例如 /src/main.js?name=1。
>
>this.resourcePath：当前处理文件的路径，例如 /src/main.js。
>
>this.resourceQuery：当前处理文件的 querystring。 
>
>this.target：等于 Webpack 配置中的 Target
>
>this.loadModule：但 Loader 在处理一个文件时，如果依赖其它文件的处理结果才能得出当前文件的结果时， 就可以通过 this.loadModule(request: string, callback: function(err, source, sourceMap, module)) 去获得 request 对应文件的处理结果。
>
>this.resolve：像 require 语句一样获得指定文件的完整路径，使用方法为 resolve(context: string, request: string, callback: function(err, result: string))。
>
>this.addDependency：给当前处理文件添加其依赖的文件，以便再其依赖的文件发生变化时，会重新调用 Loader 处理该文件。使用方法为 addDependency(file: string)。
>
>this.addContextDependency：和 addDependency 类似，但 addContextDependency 是把整个目录加入到当前正在处理文件的依赖中。使用方法为 addContextDependency(directory: string)。
>
>this.clearDependencies：清除当前正在处理文件的所有依赖，使用方法为 clearDependencies()。

>this.emitFile：输出一个文件，使用方法为 emitFile(name: string, content: Buffer|string, sourceMap: {...})。
完整API



### 手写简单的style-loader 和less-loader


```
    // 创建一个style标签里面放上源文件的样式字符串 插到页面上
    module.exports = function (source) {
        let script = (`
            let style = document.createElement("style");
            style.innerText = ${JSON.stringify(source)};
            document.head.appendChild(style);
        `);
        return script;
    }
```




```
    let less = require('less');
    module.exports = function (source) {
        this.cacheable();
        less.render(source, (err,result)=>{
            console.log(result.css);
            this.callback(err,result.css)
        })
    }
```

>不要厌烦熟悉的事物,每天都进步一点;不要畏惧陌生的事物,每天都学习一点;
