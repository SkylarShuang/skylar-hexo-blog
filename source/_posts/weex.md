---
title: weex框架浅析
---

## 1.跨端框架 

### react-native、fluter和weex

React-native: 基于 js 引擎，通过 bridge 注入一些设备能力的 api，而渲染跨端则是使用安卓、ios 实现 react 的 virtual dom 的渲染。

Weex: js框架更类似于vue，其他架构与RN类似，也是通过bridge向安卓和ios发送消息实现api的调用

flutter: 渲染不是基于操作系统的组件，而是直接基于绘图库（skia）来绘制的，这样做到了渲染的跨端。逻辑的跨端也不是基于 js 引擎，而是自研的 dart vm 来跨端，通过 dart 语言来写逻辑

![image-20211009164255894](/Users/shuanghuili/Library/Application Support/typora-user-images/image-20211009164255894.png)



|              | **Flutter** | **React Native**  | **评价**                                                     |
| ------------ | ----------- | ----------------- | ------------------------------------------------------------ |
| **编程语言** | Dart        | JS                | JS语言生态更好  Dart学习成本也不高，支持JIT与AOT             |
| Native通信   | Skia        | JavaScript-bridge | RN在UI渲染路径较长  Flutter渲染性能在路径上堪比原生          |
| UI组件与API  | SDK         | 框架（framework） | Flutter：Material & Cupertino & testing  &  navigation、包的尺寸更大  RN：Dependent  on 3rd-party libraries |
|              |             |                   |                                                              |



## 2.weex

### 总体的框架图 

![img](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c0b8881d8d894efea24ba411376361d8~tplv-k3u1fbpfcp-watermark.awebp)

### weex-Jsfm

从一个简单的weex文件来讲解weex-jsfm做了哪些处理

打包前的文件

首先可以看一下框架的目录结构

![截屏2021-10-09 下午5.20.43](/Users/shuanghuili/Library/Application Support/typora-user-images/截屏2021-10-09 下午5.20.43.png)



#### entries

首先第一步为初始化前端框架（Vue，Vanilla，Rax和 Weex），注册前端全局方法供框架使用

```javascript
import { freezePrototype } from './env'
import setup from './setup'
import frameworks from '../frameworks'

setup(frameworks)
freezePrototype()
```

再看下是如何setUp框架的呢，首先注册框架的信息，然后用init函数初始化框架的配置，最后将方法注册为全局API供框架使用。

```javascript
export default function (frameworks) {
  const { init, config } = runtime
  config.frameworks = frameworks
  const { native, transformer } = subversion

  for (const serviceName in services) {
    runtime.service.register(serviceName, services[serviceName])
  }

  runtime.freezePrototype()

  // register framework meta info
  global.frameworkVersion = native
  global.transformerVersion = transformer

  // init frameworks
  const globalMethods = init(config)

  // set global methods
  for (const methodName in globalMethods) {
    global[methodName] = (...args) => {
      const ret = globalMethods[methodName](...args)
      if (ret instanceof Error) {
        console.error(ret.toString())
      }
      return ret
    }
  }
}
```

可以看到setup函数使用init函数来初始化框架，而init函数中使用initTaskHandler函数，这个函数初始化任务处理中心的方法，是brige/TaskCenter.js中的init函数

```javascript
export default function init (config) {
  runtimeConfig = config || {}
  frameworks = runtimeConfig.frameworks || {}
  // 初始化任务处理中心
  initTaskHandler()

  // Init each framework by `init` method and `config` which contains three
  // virtual-DOM Class: `Document`, `Element` & `Comment`, and a JS bridge method:
  // `sendTasks(...args)`.
  for (const name in frameworks) {
    const framework = frameworks[name]
    if (typeof framework.init === 'function') {
      try {
        framework.init(config)
      }
      catch (e) {}
    }
  }

  adaptMethod('registerComponents', registerComponents)
  adaptMethod('registerModules', registerModules)
  adaptMethod('registerMethods')

  ; ['destroyInstance', 'refreshInstance'].forEach(genInstance)

  return methods
}
```

#### brige 

js与native侧通信的桥梁

这个函数为TaskCenter类的原型上挂载了诸多对外的方法，这些方法最终是调用的原生方法处理。

```javascript
export function init () {
  const DOM_METHODS = {
    createFinish: global.callCreateFinish,
    updateFinish: global.callUpdateFinish,
    refreshFinish: global.callRefreshFinish,

    createBody: global.callCreateBody,

    addElement: global.callAddElement,
    removeElement: global.callRemoveElement,
    moveElement: global.callMoveElement,
    updateAttrs: global.callUpdateAttrs,
    updateStyle: global.callUpdateStyle,

    addEvent: global.callAddEvent,
    removeEvent: global.callRemoveEvent,
    __updateComponentData: global.__updateComponentData
  }
  const proto = TaskCenter.prototype

  for (const name in DOM_METHODS) {
    const method = DOM_METHODS[name]
    proto[name] = method ?
      (id, args) => method(id, ...args) :
      (id, args) => fallback(id, [{ module: 'dom', method: name, args }], '-1')
  }

  proto.componentHandler = global.callNativeComponent ||
    ((id, ref, method, args, options) =>
      fallback(id, [{ component: options.component, ref, method, args }]))

  proto.moduleHandler = global.callNativeModule ||
    ((id, module, method, args) =>
      fallback(id, [{ module, method, args }]))
}
```

然后将这些方法应用到removeChild方法上，当我们调用元素removeChild方法的时候就会对应到taskCenter.send方法，讲需要发送的指令发送给客户端从而进行删除子元素的方法，而像这样写元素appendChild，insertBefore和insertAfter等方法的文件都在vdom文件夹里面。

```javascript
 /**
   * Remove a child node, and decide whether it should be destroyed.
   * @param {object} node
   * @param {boolean} preserved
   */
  Element.prototype.removeChild = function removeChild (node, preserved) {
    if (node.parentNode) {
      removeIndex(node, this.children, true);
      if (node.nodeType === 1) {
        removeIndex(node, this.pureChildren);
        var taskCenter = getTaskCenter(this.docId);
        if (taskCenter) {
          taskCenter.send(
            'dom',
            { action: 'removeElement' },
            [node.ref]
          );
        }
      }
    }
    if (!preserved) {
      node.destroy();
    }
  };
```

#### vdom

virtual dom的实现，其中包括注释节点、weex元素，浏览器的document，自定义element节点。例如原生weex节点，就是直接向客户端发送消息来注册该元素。

```javascript
export function registerElement (type, methods) {
  // Skip when no special component methods.
  if (!Array.isArray(methods) || !methods.length) {
    return
  }

  // Init constructor.
  class WeexElement extends Element {}

  // Add methods to prototype.
  methods.forEach(methodName => {
    WeexElement.prototype[methodName] = function (...args) {
      const taskCenter = getTaskCenter(this.docId)
      if (taskCenter) {
        return taskCenter.send('component', {
          ref: this.ref,
          component: type,
          method: methodName
        }, args)
      }
    }
  })
```




最后总结流程图如下

以weex-vue-framework为例主要做了如下几件事：

1. createInstanceContext

   创建实例，处理weex实例

2. createVueModuleInstance

   创建Vue实例，并做对weex的适配

3. 挂载weex提供的api到Vue实例上，如document，taskCenter等

4. 当vue生成虚拟dom的时候就可以直接调用挂载在实例上的方法，进行组件的创建，然后向桥发送消息由客户端最终进行组件的创建

![image-20211013194158430](/Users/shuanghuili/Library/Application Support/typora-user-images/image-20211013194158430.png)

