
### vue2原理

### 一、初始化

#### 1、new Vue(options) 发生了什么？

1）处理组件的配置项

  初始化根组件时进行了选项合并操作，即将全局的配置项合并到根组件的局部配置上
  初始化子组件的时候做了一些性能优化，将组件配置对象上的一些深层次的属性放到vm.$options 选项中，即拍平配置项 减少运行时原型链的查找，提高代码的执行效率。

2）初始化组件实例的关系属性（initLifeCycle）比如：$parent、$root、$children、$refs等

3）处理自定义事件
注：<comp @click="handleClick" /> 上注册的事件，实际上是子组件在监听，即遵循谁触发谁监听的原则，与父组件无关。

4）初始化插槽 获取this.$slots 定义 this._c 即createElement 方法 平时使用的h函数

5）调用 beforeCreate 钩子函数

6）初始化组件的inject配置项，得到result[key] = val 形式的配置对象，然后对该配置对象做响应式处理，并将每个key代理到vm实例上

7）响应式数据初始化，处理props、methods、data、computed、watch等选项

8）解析组件配置项上的provide对象，将其挂载到vm._provided属性上

9）调用created 钩子函数

10）如果发现配置项上有el选项，则自动调用$mount方法，反之需要手动调用$mount方法

接下来则进入挂载阶段

#### 2、beforeCreate中可以拿到数据吗？

不能， beforeCreate 生命周期执行的时候，数据还没有开始初始化，在它之前只做了一些关于属性的初始化和关于自定义事件的初始化还有插槽的初始化和_c 函数的初始化即h函数

#### 3、数据属性最早拿到的时间点是？

created
因为在 beforeCreate生命周期之后 created  之前 完成了 state数据的初始化 和 provide提供的数据的初始化


### 二、响应式原理

#### 1、Vue 响应式原理是怎么实现的？


#### 2、methods、computed 和 watch 有什么区别？


其实到这里也能看出，computed 和 watch 在本质是没有区别的，都是通过 watcher 去实现的响应式
非要说有区别，那也只是在使用方式上的区别，简单来说：
computed是懒执行，且不可更改，但是 watcher可配置
1、watch：适用于当数据变化时执行异步或者开销较大的操作时使用，即需要长时间等待的操作可以放在 watch 中
2、computed：其中可以使用异步方法，但是没有任何意义。所以 computed 更适合做一些同步计算


methods 一般用于封装一些较为复杂的处理逻辑（同步、异步）

computed 一般用于封装一些简单的同步逻辑，将经过处理的数据返回，然后显示在模版中，以减轻模版的重量

watch 一般用于当需要在数据变化时执行异步或开销较大的操作


##### 常见的面试题：computed和 methods有什么区别？ methods VS computed

通过示例会发现，如果在一次渲染中，有多个地方使用了同一个 methods 或 computed 属性，methods 会被执行多次，而 computed 的回调函数则只会被执行一次。

通过阅读源码我们知道，在一次渲染中，多次访问 computedProperty，只会在第一次执行 computed 属性的回调函数，后续的其它访问，则直接使用第一次的执行结果（watcher.value），而这一切的实现原理则是computed会缓存 缓存的原理是什么？ 通过对 watcher.dirty 属性的控制实现的。而 methods，每一次的访问则是简单的方法调用（this.xxMethods）。

computed VS watch

通过阅读源码我们知道，computed 和 watch 的本质是一样的，内部都是通过 Watcher 来实现的，其实没什么区别，非要说区别的化就两点：1、使用场景上的区别，2、computed 默认是懒执行的，切不可更改。

methods VS watch

methods 和 watch 之间其实没什么可比的，完全是两个东西，不过在使用上可以把 watch 中一些逻辑抽到 methods 中，提高代码的可读性。

#### 3、响应式数据的优先级
 props methods data computed watch
 




