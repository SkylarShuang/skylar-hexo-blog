---
title: Vue3异步组件的原理
cover: "https://cdn.pixabay.com/photo/2016/06/27/07/30/elvis-presley-1482026_960_720.jpg"
categories: 
     - Vue
---

[首先请参考Vue异步组件的用法](https://cn.vuejs.org/v2/guide/components-dynamic-async.html#%E5%BC%82%E6%AD%A5%E7%BB%84%E4%BB%B6)

## 1.用例分析

此处写一个Vue异步组件的例子：

```javascript
Vue.component('async-example', function (resolve, reject) {
  setTimeout(function () {
    // 向 `resolve` 回调传递组件定义
    resolve({
      template: '<div>I am async!</div>'
    })
  }, 1000)
})
```
可见此处传入了两个参数，一个是组件的名字，一个是工厂函数，工厂函数接受两个参数（resolve， reject），工厂函数会收到一个`resolve`回调，这个回调函数会在你从服务器得到组件定义的时候被调用。也可以调用`reject(reason)`来表示加载失败。
不想这个异步组件实现的具体的逻辑，先考虑一下大方向：

1.怎么判断这是一个异步组件

2.在确定是异步组件以后，要调用异步函数来渲染和生成组件并将结果保存，并创建异步组件的占位符

3.异步函数运行完以后，通过watcher来进行组件更新

## 2.具体逻辑解释

如果不是html标签，都会进入到createComponent函数中来创建VNode节点，那么异步组件肯定会进入到该函数中，该函数做三件事情

1）判断组件函数，然后进入不同的函数来进行相应的逻辑处理，如果传入的参数是对象，则通过Ctor = baseCtor.extend(Ctor)构造子类构造函数，如果是函数，说明传入的是异步组件，则进入到resolveAsyncComponent函数中

2）安装组件的钩子函数

3）通过以上函数完成相应组件的配置，从而根据这些配置来实例化VNode节点

```javascript
function createComponent(
        Ctor,
        data,
        context,
        children,
        tag
    ) {
        if (isUndef(Ctor)) {
            return
        }
        var baseCtor = context.$options._base;
        // 自定义组件(异步组件除外)均会传入一个对象
        if (isObject(Ctor)) {
            // 构造子类构造函数
            Ctor = baseCtor.extend(Ctor);
        }
        // 如果既不是对象也不是函数 则给出"无效组件"的提示
        if (typeof Ctor !== 'function') {
            {
                warn(("Invalid Component definition: " + (String(Ctor))), context);
            }
            return
        }
        var asyncFactory;
        // 通过cid属性是否未定义来判断是否为异步组件
        if (isUndef(Ctor.cid)) {
            asyncFactory = Ctor;
            // 正式进入处理异步组件的函数
            Ctor = resolveAsyncComponent(asyncFactory, baseCtor);
            if (Ctor === undefined) {
                // 创建异步组件的占位符
                return createAsyncPlaceholder(
                    asyncFactory,
                    data,
                    context,
                    children,
                    tag
                )
            }
        }
        // 安装组件的钩子函数
        installComponentHooks(data);
        // return a placeholder vnode
        var name = Ctor.options.name || tag;
        // 创建VNode节点
        var vnode = new VNode(
            ("vue-component-" + (Ctor.cid) + (name ? ("-" + name) : '')),
            data, undefined, undefined, undefined, context,
            { Ctor: Ctor, propsData: propsData, listeners: listeners, tag: tag, children: children },
            asyncFactory
        );
        return vnode
    }
```
由于此处重点讲解异步组件的逻辑，那么就重点看下resolveAsyncComponent的逻辑，
```javascript
function resolveAsyncComponent(
        factory,
        baseCtor
    ) {
        if (isTrue(factory.error) && isDef(factory.errorComp)) {
            return factory.errorComp
        }
        if (isDef(factory.resolved)) {
         // 此时返回缓存的组件
            return factory.resolved
        }
if (owner && !isDef(factory.owners)) {
            var owners = factory.owners = [owner];
            var sync = true;
            var timerLoading = null;
            var timerTimeout = null
                ; (owner).$on('hook:destroyed', function () { return remove(owners, owner); });
            var forceRender = function (renderCompleted) {
                for (var i = 0, l = owners.length; i < l; i++) {
                 // 触发组件强制更新的函数
                    (owners[i]).$forceUpdate();
                }
            };
// 保证resolve函数只运行一次
var resolve = once(function (res) {
// 创建异步组件并保存在resolved中
factory.resolved = ensureCtor(res, baseCtor);
// 如果不是同步的，则进行强制更新 
                if (!sync) {
                    forceRender(true);
                } else {
                    owners.length = 0;
                }
            });
 // 保证reject函数只运行一次
            var reject = once(function (reason) {
                warn(
                    "Failed to resolve async component: " + (String(factory)) +
                    (reason ? ("\nReason: " + reason) : '')
                );
                // 如果渲染错误切定义了加载失败的组件，则显示错误
                if (isDef(factory.errorComp)) {
                    factory.error = true;
                    forceRender(true);
                }
            });
            //执行工厂函数(异步函数)
            var res = factory(resolve, reject);
            if (isObject(res)) {
            // 如果函数返回的结果为一个promise对象
                if (isPromise(res)) {
                    // () => Promise
                    if (isUndef(factory.resolved)) {
                     // 如果加载成功运行定义的resolve函数
                        res.then(resolve, reject);
                    }
                } 
            }
            sync = false;
            // return in case resolved synchronously
            return factory.loading
                ? factory.loadingComp
                : factory.resolved
        }
    }
```
可见以上写法的逻辑已经梳理完毕了，但是还有高级异步组件的写法如下:

```javascript
const AsyncComponent = () => ({
  // 需要加载的组件 (应该是一个 `Promise` 对象)
  component: import('./MyComponent.vue'),
  // 异步组件加载时使用的组件
  loading: LoadingComponent,
  // 加载失败时使用的组件
  error: ErrorComponent,
  // 展示加载时组件的延时时间。默认值是 200 (毫秒)
  delay: 200,
  // 如果提供了超时时间且组件加载也超时了，
  // 则使用加载失败时使用的组件。默认值是：`Infinity`
  timeout: 3000
})
```
可以看到多了很多配置条件，可以在异步的不同状态下分别加载不同的页面，然后可以看一下Vue是怎么处理这一块的逻辑。
```javascript
if (isObject(res)) {
  if (isPromise(res)) {
    // () => Promise
    if (isUndef(factory.resolved)) {
      res.then(resolve, reject)
    }
    // 如果component是promise对象
  } else if (isPromise(res.component)) {
    //加载异步组件成功的话进入resolve回调，失败的话进入reject回调
    res.component.then(resolve, reject)
    // 如果定义了error
    if (isDef(res.error)) {
      // 调用ensureCtor方法传入错误组件和基本配置来创建组件并添加缓存
      factory.errorComp = ensureCtor(res.error, baseCtor)
    }
    // 如果定义了loading
    if (isDef(res.loading)) {
      // 创建loading组件并加入缓存
      factory.loadingComp = ensureCtor(res.loading, baseCtor)
      // 如果没有设置delay的时间，直接设置loading的状态为true
      if (res.delay === 0) {
        factory.loading = true
      } else {
        // 如果有delay的话，就用setTimeout函数使得在多少时间后触发强制更新来渲染loading组件
        timerLoading = setTimeout(function () {
          timerLoading = null
          if (isUndef(factory.resolved) && isUndef(factory.error)) {
            factory.loading = true
            // 强制更新并传入并未渲染完成的参数
            forceRender(false)
          }
        }, res.delay || 200)
      }
    }
    // 如果定义了timeout
    if (isDef(res.timeout)) {
      timerTimeout = setTimeout(function () {
        timerTimeout = null
        if (isUndef(factory.resolved)) {
          // 进入到reject回调函数 
          reject('timeout (' + (res.timeout) + 'ms)')
        }
      }, res.timeout)
    }
  }
}
```