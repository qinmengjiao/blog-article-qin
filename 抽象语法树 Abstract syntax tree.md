
### 什么是抽象语法树？

在计算机科学中，抽象语法和抽象语法树其实是**源代码的抽象语法结构的树状表现形式**
[在线编辑器](https://astexplorer.net/)

我们常用的浏览器就是通过将js代码转化为抽象语法树来进行下一步的分析等其他操作。所以将js转化为抽象语法树更利于程序的分析。

如图：

![](http://img.zhufengpeixun.cn/ast.jpg)

如上图中的变量声明语句，转化为AST之后就是右图的样子。

先来分析一下左图：

> var 是一个关键字
> 
> AST是一个定义者
> 
> = 是Equal 等号的叫法有很多形式，在后面我们还会看到
> 
> “is tree” 是一个字符串
> 
> ；就是 Semicoion

再来对应一下右图：

首先一段代码转化成的抽象语法树是一个对象，该对象会有一个顶级的type属性'Program',第二个属性是body是一个数组。

body数组中存放的每一项都是一个对象，里面包含了所有的对于该语句的描述信息

	type:描述该语句的类型 --变量声明语句
	kind：变量声明的关键字 -- var
	declaration: 声明的内容数组，里面的每一项也是一个对象
		type: 描述该语句的类型 
		id: 描述变量名称的对象
			type：定义
			name: 是变量的名字
        init: 初始化变量值得对象
			type: 类型
 			value: 值 "is tree" 不带引号
			row: "\"is tree"\" 带引号

### 抽象语法树有哪些用途？

代码语法的检查，代码风格的检查，代码的格式化，代码的高亮，代码错误提示，代码自动补全等等

>如：JSLint、JSHint 对代码错误或风格的检查，发现一些潜在的错误
    IDE的错误提示，格式化，高亮，自动补全等等
代码的混淆压缩
>如：UglifyJS2等

优化变更代码，改变代码结构达到想要的结构

>代码打包工具webpack，rollup等等
>CommonJS、AMD、CMD、UMD等代码规范之间的转化
>CoffeeScript、TypeScript、JSX等转化为原生Javascript

### 通过什么工具或库来实现源码转化为抽象语法树？

那就是**javascript Parser** 解析器，他会把js源码转化为抽象的语法树。

浏览器会把js源码通过解析器转化为抽象语法树，再进一步转化为字节码或直接生成机器码

一般来说每一个js引擎都会有自己的抽象语法树格式，chrome的v8引擎，firefox的SpiderMonkey 引擎等等，MDN提供了详细SpiderMonkey AST format的详细说明，算是业界的标准。（SpiderMonkey是Mozilla项目的一部分，是一个用C语言实现的JavaScript脚本引擎，为了在SpiderMonkey中运行JavaScript代码，应用程序必须有三个要素：JSRuntime，JSContext和全局对象。）


**常用的javascript Parser**
>esprima

>traceur

>acorn

>shift

**我们主要拿esprima来举一个例子**

安装

```
 npm install esprima estraverse escodegen -S
```

esprima 涉及三个库名称和功能如下：

>esprima 把源码转化为抽象语法树

```

	let esprima = require('esprima'); // 引入esprima
	let jsOrigin = 'function eat(){};'; // 定义一个js源码
	
	let AST = esprima.parse(jsOrigin); // 通过esprima.parse将js源码转化为一个抽象语法树
	
	console.log(AST); // 打印生成的抽象语法树
   
	/*Script {
    type: 'Program',// 顶级的type属性
    body: [ FunctionDeclaration {
            type: 'FunctionDeclaration', // js源码的类型--是一个函数声明
            id: [Identifier],
            params: [],
            body: [BlockStatement],
            generator: false, // 是不是generator函数
            expression: false, // 是不是一个表达式
            async: false // 是不是一个异步函数
            },
            EmptyStatement { type: 'EmptyStatement' } 
          ],
    sourceType: 'script' 
    }*/

```


>estraverse 遍历并更新抽象语法树

在介绍用法之前我们先来[npm](https://www.npmjs.com/package/estraverse)上看一下这个库,这个库的下载量居然500多万，而且没有README说明文档，是不是很牛掰！

在举例子之前我们要遍历抽象语法树，首先我们要先了解一下他的遍历顺利

```
	
	let estraverse = require('estraverse');
	

	estraverse.traverse(AST, {
	    enter(node){
	        console.log('enter', node.type)
	        if(node.type === 'Identifier') {
	            node.name += '_enter'
	        }
	    },
	    leave(node){
	        console.log('leave', node.type)
	        if(node.type === 'Identifier') {
	            node.name += '_leave'
	        }
	    }
	})
	
	// enter Program
	// enter FunctionDeclaration
	// enter Identifier
	// leave Identifier
	// enter BlockStatement
	// leave BlockStatement
	// leave FunctionDeclaration
	// enter EmptyStatement
	// leave EmptyStatement
	// leave Program

```

通过上面节点类型的打印结果我们不难看出，我们的抽象语法树的每个节点被访问了2次，一次是进入的时候，一次是离开的时候，我们可以通过下面的图来更加清楚的理解抽象语法树的遍历顺序

![](http://a1.qpic.cn/psb?/V10n1LYN0FnrMc/miVuYlpPFv9fzCqqLmoeNfeZvbNllALGwGkHdhhVi0k!/m/dAgBAAAAAAAAnull&bo=XQIyAgAAAAADB00!&rf=photolist&t=5)
![](http://a1.qpic.cn/psb?/V10n1LYN0FnrMc/Aj.NkcM4Xl950ogZfgQCwY3LT6QVPbEQIGTv3fGZhJA!/m/dEABAAAAAAAAnull&bo=WQIGAQAAAAADB34!&rf=photolist&t=5)


看完遍历顺序之后，我们看到代码中的判断条件 如果是变量名的话，第一次进入访问时对这个变量的名称做了一次修改，当离开的时候也做了一次修改。那接下来我们要验证 抽象语法树种的这个节点的变量名称 是否修改成功了呢？我们有两种方案，方案一：直接打印抽象语法树，这个非常简单再这里就你介绍了。方案二： 我们将现有的抽象语法树转化成源码看一下变量名是否变成功 这样就一目了然了。那怎么将我们的抽象语法树还原成源码呢？这就要引入我们的第三个库了 escodegen

>escodegen 将抽象语法树还原成js源码

```

	let escodegen = require('escodegen');
	
	let originReback = escodegen.generate(AST);
	console.log(originReback);
    // function eat_enter_leave() {};

```

通过上面还原回来的源码我们看到变量名称确实被更改了。

### 接下来我们来探索一下如何用抽象语法树来将箭头函数转化为普通的函数

我们都知道es6语法转es5的语法我们用的是babel，让我们接下来就看一下 babel是如何将箭头函数转化为普通函数的。


##### 第一步需要使用babel的两个插件，babel-core 核心模块 babel-types 类型模块

```

	npm i babel-core babel-types -S

```

第一步：我们先来对比普通函数和箭头函数的抽象语法树，通过对比找出其中的不同之处，然后在节点可以复用的前提下，尽可能少的改变一下不同的地方，从而成功的将箭头函数转化为普通函数。

我们以这个箭头函数为例：

```javascript

	let sum = (a,b) => a+b; 
	------>
	var sum = function sum(a, b) {
	  return a + b;
	};
```

![](http://a2.qpic.cn/psb?/V10n1LYN0FnrMc/Lf20Mw96SVeIiGEjXZgWWuYITPKVmNmXJeMKzGU05tg!/m/dEUBAAAAAAAAnull&bo=igQPAgAAAAADB6E!&rf=photolist&t=5)

如上图所示，普通函数和箭头函数的AST的不同在于init，所以我们现在要做的是将箭头函数的arrowFunctionExpression 转换为FunctionExpression

利用babel-types生成新的部分的AST语法树，替换原有的。如果创建某个节点的语法树，那就在下面的网址上，需要哪个节点就搜哪个节点
[babel-types](https://github.com/jamiebuilds/babel-types)


```
	// babel 核心库，用来实现核心的转换引擎
	const babel = require('babel-core');
	// 实现类型转化 生成AST节点
	const types = require('babel-types');
	let code = 'let sum = (a,b) => a+b;';
	let es5Code = function (a,b) {
	    return a+b;
	};
	
	// babel 转化采用的是访问者模式Visitor 对于某个对象或者一组对象，不同的访问者，产生的结果不同，执行操作也不同
	
	// 这个访问者可以对特定的类型的节点进行处理
	let visitor = {
	    ArrowFunctionExpression(path) {
	        // 如果这个节点是箭头函数的节点的话，我们在这里进行处理替换工作
	        // 1.复用params参数
	        let params = path.node.params;
	        let blockStatement = types.blockStatement([types.returnStatement(path.node.body)])
	        let func = types.functionExpression(null, params, blockStatement, false,false);
	        path.replaceWith(func)
	
	    }
	};
	
	let arrayPlugin = {visitor};
	
	// babel内部先把代码转化成AST,然后进行遍历
	
	let result = babel.transform(code, {
	    plugins: [
	        arrayPlugin
	    ]
	});
	
	console.log(result.code);
    // let sum = function (a, b) {
    //     return a + b;
    // };

```


### 我们写一个babel的预计算插件

```

	let code = `const result = 1000 * 60 * 60 * 24`;
	//let code = `const result = 1000 * 60`;
	let babel = require('babel-core');
	let types = require('babel-types');
	//预计算
	let visitor = {
	    BinaryExpression(path){
	        let node = path.node;
	        if(!isNaN(node.left.value)&&!isNaN(node.right.value)){
	            let result = eval(node.left.value+node.operator+node.right.value);
	            result =  types.numericLiteral(result);
	            path.replaceWith(result);
	            //如果此表达式的父亲也是一个表达式的话,需要递归计算
	            if(path.parentPath.node.type == 'BinaryExpression'){
	                visitor.BinaryExpression.call(null,path.parentPath);
	            }
	        }
	    }
	}
	let r = babel.transform(code,{
	    plugins:[
	        {visitor}
	    ]
	});
	console.log(r.code);

```
以上就是我对抽象语法树的理解，有什么不正确的地方，恳求斧正。
