
背景：当我们同时引入一个包中的两个方法，有两种形式


> 第一种形式 
> 
```
import {flatten,join} from 'lodash';
```
>第二种形式
>
```
import flatten from 'lodash/flatten';
import join from 'lodash/join';
```

对比两种形式，我们可以看出：

第一种方式的引入会把整个lodash包引进来
第二种方式是指引入整个包中的两个方法

显然我们要用第二种

但是一般的项目中 大部分都是以import解构的形式，所以在这里我们就写一个插件，当我们写成第一种形式引入的时候，利用插件转化成第二种形式


这是我们要写的插件的功能


第一步：初始化一个webpack的项目

```
 npm i webpack webpack-cli babel-core babel-loader babel-preset-env babel-preset-stage-0 -D
```

在Webpack中，提供了mode变量，用于配置运行环境，mode的值可以为development，表示的是开发模式，或者是production，表示的是生产模式。

在package.json中写入编译的命令

```
"scripts":{
  "build":"webpack --mode production/development"
}
```

第二步：创建一个项目结构

```
webpack-plugin // 项目名称
    dist // 打包输出目录
        bundle.js // 打包输出的文件
    src // 主要逻辑
        index.js // 项目的入口文件
    ./babelrc // 语法解析配置
    package.json
    webpack.config.js

```

第三步：写webpack的配置文件

```
const path = require('path');
module.exports = {
    entry: './src/index.js',// 入口文件
    output: {
        path: path.join(__dirname, 'dist'), // 输出路径
        filename: 'bundle.js' // 输出的文件名称
    },
    // 配置
    module: {
        // 配置加载器
        rules: [
            {
                test: /\.js$/,
                loader: 'babel-loader'
            }
        ]
    }
}

```
第四步：配置.babelrc

```
{
  "presets": [
    "env",
    "stage-0"
  ],
  "plugins": [
    [
      // "demand-loading", // 注：这个是我们自己写的插件名 显示先不放，等我们写好插件后再加上
      {
         "library": "lodash", // 我们在引用哪个库的时候使用我们写的这个插件，这里的意思是当我们引用lodash库的时候使用我们写的这个插件
      },
      "syntax-decorators"
    ]
  ]
}
```


第五步：index.js中写入对比脚本


```
 // import {flatten,join} from 'lodash'
import flatten from 'lodash/flatten'
import join from 'lodash/join'
```

先引入后面的这两句，然后打包一下

```
Hash: fcb0bd5d9734b5f56676
Version: webpack 4.2.0
Time: 346ms
Built at: 2018-3-27 21:24:33
    Asset      Size  Chunks             Chunk Names
bundle.js  21.3 KiB    main  [emitted]  main
Entrypoint main = bundle.js
[./node_modules/webpack/buildin/global.js] (webpack)/buildin/global.js 823 bytes {main} [built]
[./src/index.js] 286 bytes {main} [built]
    + 15 hidden modules


```

看到这样打包后的代码，我们发现这种方式引入 打包后的大小是。21.3k

然后注释掉后两行，只引入第一行

```
    Hash: aa8b689e1072463fc1cd
Version: webpack 4.2.0
Time: 3277ms
Built at: 2018-3-27 21:30:22
    Asset     Size  Chunks             Chunk Names
bundle.js  483 KiB    main  [emitted]  main
Entrypoint main = bundle.js
[./node_modules/webpack/buildin/amd-options.js] (webpack)/buildin/amd-options.js 82 bytes {main} [built]
[./node_modules/webpack/buildin/global.js] (webpack)/buildin/global.js 823 bytes {main} [built]
[./node_modules/webpack/buildin/module.js] (webpack)/buildin/module.js 521 bytes {main} [built]
[./src/index.js] 47 bytes {main} [built]
    + 1 hidden module

```

这次打包后的大小是 483k

通过对比 证实了我们所说的两种引入方式

第六步：咱们来写写这个插件 


先对比一下两种引入方式的抽象语法树的差别

![](http://a1.qpic.cn/psb?/V10n1LYN0FnrMc/61sxmy*JFl1EiOohpUYaMQUOOTO9ICrvAKggrtquKJc!/m/dEABAAAAAAAAnull&bo=LgXeAQAAAAADB9Y!&rf=photolist&t=5)


通过对比我们发现，只是ImportDeclaration不相同


```

	const babel = require('babel-core');
	const types = require('babel-types');
	
	let visitor = {
	    // 这里的ref是ImportDeclaration的第二个参数，这里的值是.babelrc中的 {
	       // "library": "lodash"
	    //}, 这里是指定 我们在引用哪个库的时候使用这个插件
	    ImportDeclaration(path, ref={options:{}}) {
	        let node = path.node;
	        let specifiers = node.secifiers
	        if (options.library == node.soure.value && !types.isImportDeclaration(specifiers[0])) {
	            let newImport = specifiers.map((specifier) => (
	                types.importDeclaration([types.ImportDefaultSpecifier(specifier.local)], types.stringLiteral(`${node.soure.value}/${specifier.local.name}`))
	            ));
	            path.replaceWithMultiple(newImport)
	        }
	    }     
	}
	
	const code = "import {flatten, join} from 'lodash';";
	
	let r = babel.transform(code, {
	    plugins: [
	        {visitor}
	    ]
	})

```

在创建替换逻辑的时候，types上的方法 用[github上的这个网址](https://github.com/jamiebuilds/babel-types)，哪个不会搜哪个，妈妈再也不用担心我的学习。嘻嘻

第七步：将上面的代码整理一下放到node_modules文件中

新建一个文件夹 babel-plugin-demand-loading 放到node_modules中，再新建一个index.js文件，将下面的代码放进去，再然后进入这个文件夹 npm init -y初始化一个package.json文件，里面的入口文件写成index.js


需要注意的事项：

> 第一： babel插件的文件夹命名，必须以 babel-plugin-xxx(你写的插件名)命名，否则引入不成功
> 第二： babel插件返回的是一个对象，里面有一个访问者模式的visitor对象，里面是我们的转化代码
> 


```

	const babel = require('babel-core');
	const types = require('babel-types');
	
	module.exports = {
	    visitor:  {
	        // 这里的ref是ImportDeclaration的第二个参数，这里的值是.babelrc中的 {
	        // "library": "lodash"
	        //}, 这里是指定 我们在引用哪个库的时候使用这个插件
	        ImportDeclaration(path, ref={}) {
	            let { opts } = ref
	            let node = path.node;
	            let specifiers = node.specifiers
	            if (opts.library == node.source.value && !types.isImportDeclaration(specifiers[0])) {
	                let newImport = specifiers.map((specifier) => (
	                    types.importDeclaration([types.ImportDefaultSpecifier(specifier.local)], types.stringLiteral(`${node.source.value}/${specifier.local.name}`))
	                ));
	                path.replaceWithMultiple(newImport)
	            }
	        }
	    }
	}


```

最后 npm run build 编译后，发现打包后的大小是。20多k说明我们的插件起作用了。

>不要厌烦熟悉的事物,每天都进步一点;不要畏惧陌生的事物,每天都学习一点;


