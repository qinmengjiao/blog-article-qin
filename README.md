## 谈谈我对Promise的理解(一)


### 一、Promise是什么？

Promise是最早由社区提出和实现的一种解决异步编程的方案，比其他传统的解决方案（回调函数和事件）更合理和更强大。

ES6 将其写进了语言标准，统一了用法，原生提供了Promise对象。
ES6 规定，Promise对象是一个构造函数，用来生成Promise实例。

### 二、Promise是为解决什么问题而产生的？

promise是为解决异步处理回调金字塔问题而产生的

### 三、Promise的两个特点

1、Promise对象的状态不受外界影响

1）pending 初始状态

2）fulfilled 成功状态

3）rejected 失败状态

Promise 有以上三种状态，只有异步操作的结果可以决定当前是哪一种状态，其他任何操作都无法改变这个状态

2、Promise的状态一旦改变，就不会再变，任何时候都可以得到这个结果，状态不可以逆，只能由 pending变成fulfilled或者由pending变成rejected

### 四、Promise的三个缺点

1）无法取消Promise,一旦新建它就会立即执行，无法中途取消
2）如果不设置回调函数，Promise内部抛出的错误，不会反映到外部
3）当处于pending状态时，无法得知目前进展到哪一个阶段，是刚刚开始还是即将完成

### 五、Promise在哪存放成功回调序列和失败回调序列？

1）onResolvedCallbacks 成功后要执行的回调序列 是一个数组

2）onRejectedCallbacks 失败后要执行的回调序列 是一个数组

以上两个数组存放在Promise 创建实例时给Promise这个类传的函数中，默认都是空数组。
每次实例then的时候 传入 onFulfilled 成功回调 onRejected 失败回调，如果此时的状态是pending 则将onFulfilled和onRejected push到对应的成功回调序列数组和失败回调序列数组中，如果此时的状态是fulfilled 则onFulfilled立即执行，如果此时的状态是rejected则onRejected立即执行

上述序列中的回调函数执行的时候 是有顺序的，即按照顺序依次执行


### 六、Promise的用法

1、Promise构造函数接受一个函数作为参数，该函数的两个参数分别是resolve和reject。它们是两个函数，由 JavaScript 引擎提供，不用自己部署。

```
    const promise = new Promise(function(resolve, reject) {
      // ... some code

      if (/* 异步操作成功 */){
        resolve(value);
      } else {
        reject(error);
      }
    });
```

2、resolve函数的作用是，将Promise对象的状态从“未完成”变为“成功”（即从 pending 变为 resolved），在异步操作成功时调用，并将异步操作的结果，作为参数传递出去；reject函数的作用是，将Promise对象的状态从“未完成”变为“失败”（即从 pending 变为 rejected），在异步操作失败时调用，并将异步操作报出的错误，作为参数传递出去。

3、Promise实例生成以后，可以用then方法分别指定resolved状态和rejected状态的回调函数。

```
    promise.then(function(value) {
      // success
    }, function(error) {
      // failure
    });
```
then方法可以接受两个回调函数作为参数。第一个回调函数是Promise对象的状态变为resolved时调用，第二个回调函数是Promise对象的状态变为rejected时调用。其中，第二个函数是可选的，不一定要提供。这两个函数都接受Promise对象传出的值作为参数。

### 七、按照Promise A+规范写Promise的简单实现原理

```
// 第一步：Promise构造函数接受一个函数作为参数，该函数的两个参数分别是resolve和reject。它们是两个函数，由 JavaScript 引擎提供，不用自己部署。
    function Promise(task) {

        let that = this; // 缓存this
        that.status = 'pending'; // 进行中的状态
        that.value = undefined; //初始值

        that.onResolvedCallbacks = []; // 存放成功后要执行的回调函数的序列
        that.RejectedCallbacks = []; // 存放失败后要执行的回调函数的序列
        // 该方法是将Promise由pending变成fulfilled
        function resolve (value) {
            if (that.status == 'pending') {
                that.status = 'fulfilled';
                that.value = value;
                that.onResolvedCallbacks.forEach(cb => cd(that.value))
            }

        }
        // 该方法是将Promise由pending变成rejected
        function reject (reason) {
          if (that.status == 'pending') {
                that.status = 'rejected';
                that.value = reason;
                that.onRjectedCallbacks.forEach(cb => cd(that.value))
            }
        }

        try {
        // 每一个Promise在new一个实例的时候 接受的函数都是立即执行的
            task(resolve, reject)
        } catch (e) {
            reject(e)
        }
    }

// 第二部 写then方法，接收两个函数onFulfilled onRejected，状态是成功态的时候调用onFulfilled 传入成功后的值，失败态的时候执行onRejected，传入失败的原因，pending 状态时将成功和失败后的这两个方法缓存到对应的数组中，当成功或失败后 依次再执行调用
    Promise.prototype.then = function(onFulfilled, onRejected) {
        let that = this;
        if (that.status == 'fulfilled') {
            onFulfilled(that.value);
        }
        if (that.status == 'rejected') {
            onRejected(that.value);
        }
        if (that.status == 'pending') {
            that.onResolvedCallbacks.push(onFulfilled);
            that.onRjectedCallbacks.push(onRejected);
        }
    }
```
### 八、Promise 链式写法

我们先来看一个例子，根据例子得出结论，然后再写源码的实现部分来验证结论

```
    let promise = new Promise(function (resolve, reject) {
        resolve(100);// reject(100)
    });

    promise.then(function (data) {
        return data+100;

    },function (err) {
       return 'ssss';

    }).then(function (data) {
        console.log(data);// 200  // undefined // sss
    })
```


从上面的例子可以看出:

当第一个promise的成功的回调里返回 200时，第二个promise的成功回调的参数就是200
当将resolve(100)改成reject(100)的时候，因为失败回调中什么也没有返回所以第二个promise的成功回调中的参数是undefined
当失败的回调中返回sss时，第二个promise的成功回调中的参数是sss

由此我们可以看出，第一个promise不管成功回调还是失败回调，他的返回值作为第二个promise中的成功时回调函数的参数值

链式写法能一直then下去的原因：链式调用靠的是返回新的promise，来保证可以一直走成功或失败

### 九、  Promise.catch

Promise.prototype.catch方法是.then(null, rejection)的别名，用于指定发生错误时的回调函数。

```
    //catch原理就是只传失败的回调
    Promise.prototype.catch = function(onRejected){
        this.then(null,onRejected);
    }
```

### 十、 Promise.all 方法

参数：接受一个数组，数组内都是Promise实例
返回值：返回一个Promise实例，这个Promise实例的状态转移取决于参数的Promise实例的状态变化。当参数中所有的实例都处于resolve状态时，返回的Promise实例会变为resolve状态。如果参数中任意一个实例处于reject状态，返回的Promise实例变为reject状态

```
    Promise.all = function(promises){
     return new Promise(function(resolve,reject){
       let done = gen(promises.length,resolve);
       for(let i=0;i<promises.length;i++){
         promises[i].then(function(data){
           done(i,data);
         },reject);
       }
     });
    }
```

### 十一、Promise.resolve

返回一个Promise实例，这个实例处于resolve状态。
根据传入的参数不同有不同的功能：

值(对象、数组、字符串等)：作为resolve传递出去的值
Promise实例：原封不动返回

```
    //返回一个立刻成功的promise
    //别人提供 给你一个方法，需要你传入一个promise,但你只有一个普通的值，你就可以通过这个方法把这个普通的值(string number object)转成一个promise对象
    Promise.resolve = function(value){
      return new Promise(function(resolve){
        resolve(value);
      });
    }
```

### 十二、  Promise.reject

返回一个Promise实例，这个实例处于reject状态。

参数一般就是抛出的错误信息。

```
    //返回一个立刻失败的promise
    Promise.reject = function(reason){
      return new Promise(function(resolve,reject){
        reject(reason);
      });
    }
```

### 十三、 Promise.race

参数：接受一个数组，数组内都是Promise实例
返回值：返回一个Promise实例，这个Promise实例的状态转移取决于参数的Promise实例的状态变化。当参数中任何一个实例处于resolve状态时，返回的Promise实例会变为resolve状态。如果参数中任意一个实例处于reject状态，返回的Promise实例变为reject状态。

```
    Promise.race = function(promises){
      return new Promise(function(resolve,reject){
        for(let i=0;i<promises.length;i++){
          promises[i].then(resolve,reject);
        }
      });
    }
```

[Promises/A+规范](https://promisesaplus.com/)
[完整的手写promise源码地址](http://www.cnblogs.com/qinmengjiao123-123/p/8280973.html)

