## Javascript异步编程的前世今生


Javascript语言的执行环境是"单线程"（single thread）

所谓"单线程"，就是指一次只能完成一件任务。如果有多个任务，就必须排队，前面一个任务完成，再执行后面一个任务，以此类推

到这里肯能有人会疑惑，js既然是单线程的，就是只能在一个任务结束之后才能执行下一个任务。和异步的“后一个任务不等前一个任务结束就执行”是矛盾的吗？这个问题在下一篇文章 《JavaScript中的EventLoop》中会有详细的解读。

### 一、什么是异步编程？

1）同步编程：等待，串行；即一个任务执行完成之后第二个任务才开始执行，第二个任务要等第一个任务。

 缺点：阻塞。

2）异步编程：不等，并行；指每一个任务有一个或多个回调函数（callback），前一个任务结束后，不是执行后一个任务，而是执行回调函数，后一个任务则是不等前一个任务结束就执行，所以程序的执行顺序与任务的排列顺序是不一致的。

 优点：非阻塞。

### 二、异步编程的发展过程

所谓的异步编程，当然指的是函数的异步编程，在这里我们来了解一下js中的函数。

#### 在js中函数是一等公民，即可以作为参数，也可以做返回值。

在这里我们用提一下高阶函数，通过高阶函数的作用来说明上述的两个观点

1）高阶函数可以用来批量的生成函数

```
    // 判断一个参数是不是字符串
    function isString(str) {
        return Object.prototype.toString.call(str) == '[object String]'
    }
    console.log(isString('hello world'));
    // 判断一个参数是不是一个数组
    function isArray(arr) {
        return  Object.prototype.toString.call(arr) == '[object Array]'
    }
    console.log(isArray(['hello', 'world']));

    // 使用高阶函数批量生成函数
    function isType(type) {
        return function (params) {
            return Object.prototype.toString.call(params) == `[object ${type}]`
        }
    }
    let isString = isType('String')
    console.log(isString('hello world'));
    let isArray = isType('Array')
    console.log(isArray(['hello', 'world']));

    // 高阶函数的这个作用证实了上面的第二个观点，函数可以作为返回值
```

2）高阶函数可以用于生成需要调用多次才会执行的函数

```
    function eat() {
        console.log('吃完了')
    }
    let count = 0
    function nextTask(times, fn) {
        !function () {
            if (++count == times) {
                fn()
            }
        }()
    }
    nextTask(3, eat)
    nextTask(3, eat)
    nextTask(3, eat)
     // 高阶函数的这个作用证实了上面的第一个观点，/函数可以作为参数传到另外一个函数里面
```

#### 1) 异步编程的第一个实现--回调函数,这是最基本的方式

##### 1) 什么是会回调函数？

回调函数就是一个通过函数指针调用的函数。如果你把函数的指针（地址）作为参数传递给另一个函数，当这个指针被用来调用其所指向的函数时，我们就说这是回调函数。回调函数不是由该函数的实现方直接调用，而是在特定的事件或条件发生时由另外的一方调用的，用于对该事件或条件进行响应。

我对回调函数的解读就是现在不调，回头再调。

##### 2) 回调函数的特点

(1) 特点: error first,在调用回调函数的时候，第一个参数永远是错误对象

(2) 优点: 回调函数的优点是简单、容易理解和部署

(3) 缺点: 缺点是不利于代码的阅读和维护，各个部分之间高度耦合（Coupling），流程会很混乱，而且每个任务只能指定一个回调函数

1) 无法捕获错误 try catch

2) 不能return

3) 回调地狱 (
    1. 非常难看
    2. 非常难以维护
    3. 效率比较低，因为它们是串行的)


 ```
 // 回调金字塔
  fs.readFile('./1.txt', 'utf8', function (err, data1) {
    fs.readFile('./2.txt', 'utf8', function (err, data2) {
        fs.readFile('./3.txt', 'utf8', function (err, data3) {
          fs.readFile('./4.txt', 'utf8', function (err, data4) {
              console.log(data1 + data2 + data3 + data4);
          })
        })
    })
  })
 ```

#### 2) 异步编程的第二个实现--事件发布订阅

##### 该模式主要解决了上面回调金字塔嵌套的问题

 在讲事件发布订阅之前我们先来看一下node自带的事件监听模块EventEmitter，是node的核心模块

 里面有两个核心方法，一个叫on emit,on表示注册监听，emit表示发射事件

 第一步：引入这个模块

 ```
    let EventEmitter = require('events')
 ```

 第二步：创建一个该模块的实例

 ```
    let event = new EventEmitter()
    // 读到的内容对象
    let readedContent = {}
 ```

 第三步：监听一个事件

 ```
    event.on('ready', function(key, value) {
        readedContent[key] = value
        // 什么时候结束呢？到达指定读取的长度时停止 指定读两次
        if (Object.keys(readedContent.length) == 2) {
            console.log(readedContent)
        }
    })
 ```
 第四步：触发事件

 ```
    fs.readFile('./1.txt', 'utf8', function (err, template) {
    //1事件名 2参数往后是传递给回调函数的参数
    eve.emit('ready','template',template);
    })
    fs.readFile('./2.txt', 'utf8', function (err, data) {
    eve.emit('ready','data',data);
    })
 ```

##### EventEmitter 的实现原理

```
    function EventEmitter(){
      this.events = {};
      this._maxListeners = 10;
    }
    EventEmitter.prototype.setMaxListeners = function(maxListeners){
      this._maxListeners = maxListeners;
    }
    EventEmitter.prototype.listeners = function(event){
      return this.events[event];
    }
    EventEmitter.prototype.on = EventEmitter.prototype.addListener = function(type,listener){
      if(this.events[type]){
        this.events[type].push(listener);
        if(this._maxListeners!=0&&this.events[type].length>this._maxListeners){
          console.error(`MaxListenersExceededWarning: Possible EventEmitter memory leak detected. ${this.events[type].length} ${type} listeners added. Use emitter.setMaxListeners() to increase limit`);
        }
      }else{
        this.events[type] = [listener];
      }
    }
    EventEmitter.prototype.once = function(type,listener){
     let  wrapper = (...rest)=>{
       listener.apply(this);
       this.removeListener(type,wrapper);
     }
     this.on(type,wrapper);
    }
    EventEmitter.prototype.removeListener = function(type,listener){
      if(this.events[type]){
        this.events[type] = this.events[type].filter(l=>l!=listener)
      }
    }
    EventEmitter.prototype.removeAllListeners = function(type){
      delete this.events[type];
    }
    EventEmitter.prototype.emit = function(type,...rest){
      this.events[type]&&this.events[type].forEach(listener=>listener.apply(this,rest));
    }
    module.exports = EventEmitter;
```

#### 3) 异步编程的第三个实现--哨兵变量

```
    function render(length,cb){
      let html={};
      return function(key,value){
        html[key] = value;
        if(Object.keys(html).length == length){
          cb(html);
        }
      }
    }
    let done = render(3,function(html){
      console.log(html);
    });
    fs.readFile('./template.txt', 'utf8', function (err, template) {
      done('template',template);
    })
    fs.readFile('./data.txt', 'utf8', function (err, data) {
      done('data',data);
    })

```

#### 4) 异步编程的第四个实现--promise

Promises对象是CommonJS工作组提出的一种规范，目的是为异步编程提供统一接口。
简单说，它的思想是，每一个异步任务返回一个Promise对象，该对象有一个then方法，允许指定回调函数

```
    let Promise = require('./Promise');
    let p1 = new Promise(function(resolve,reject){
      resolve(100);
    });

    let p2 = p1.then(function(value){
      console.log('成功1=',value);
    },function(reason){
      console.log('失败1=',reason);
    })

    p2.then(function(value){
       console.log('成功2=',value);
    },function(reason){
       console.log('失败2=',reason);
    })

```

优点：优点在于，回调函数变成了链式写法，程序的流程可以看得很清楚，而且有一整套的配套方法，可以实现许多强大的功能


#### 5) 异步编程的第五个实现-- async/await
 async/await号称异步的终级解决方案，是最简单的

 ```
     let Promise = require('bluebird');
     let readFile = Promise.promisify(require('fs').readFile);
     async function read() {
       //await后面必须跟一个promise,
       let a = await readFile('./1.txt','utf8');
       console.log(a);
       let b = await readFile('./2.txt','utf8');
       console.log(b);
       let c = await readFile('./3.txt','utf8');
       console.log(c);
       return 'ok';
     }

     read().then(data => {
       console.log(data);
     });
 ```

 优点：

 1.简洁

 2.有很好的语义

 3.可以很好的处理异步 throw error return try catch

 现在koa2里已经可以支持async/await







