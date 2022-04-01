---
title: vue nextTick的原理
cover: "https://media.istockphoto.com/photos/dog-dreaming-picture-id1323095288?k=20&m=1323095288&s=612x612&w=0&h=ZC2DNmAcpDlMlNof1ojUXpItMhXggVfXFqnIZHU3NX4="
---


# Vue nextTick的原理

## 使用方法 

在下次 DOM 更新循环结束之后执行延迟回调。在修改数据之后立即使用这个方法，获取更新后的 DOM

```javascript
// 修改数据
vm.msg = 'Hello'
// DOM 还没有更新
Vue.nextTick(function () {
  // DOM 更新了
})
// 作为一个 Promise 使用 (2.1.0 起新增，详见接下来的提示)
Vue.nextTick()
  .then(function () {
    // DOM 更新了
  })
```



## 源码逻辑

  Vue.nextTick在src/core/util/nextTick.js文件中，可以通过源码看出这个逻辑为，首先将回调函数加入到flushCallbacks队列中，如果监测到pending（是否正在执行）为false，则触发timerFun函数，来运行flushCallbacks队列中的回调函数。由于flushCallbacks队列中的回调函数是需要在DOM更新后执行的，那么有几种方法来实现这个想法。

1.可以把回调函数放在微任务或者宏任务中运行，那样在DOM更新后才会运行这些回调函数，Vue也是首先进行了平台是否支持Promise的判断，如果支持则用promise.then()来执行队列中的回调函数

2.判断浏览器是否支持MutationObserver ，如果支持就用[MutationObserver](https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver)监测DOM的更新状态，此时也是有个比较巧妙的做法是，新建了一个文本节点，当运行timerFun函数时则改变这个文本节点的值，那样MutationObserver监测到以后就运行队列中的回调函数

3.判断平台是否支持setImmediate函数，通过setImmediate函数运行队列中的回调函数

4.如果都不支持，则使用setTimeout宏任务来更新队列函数

```javascript
/* @flow */
/* globals MutationObserver */

import { noop } from 'shared/util'
import { handleError } from './error'
import { isIE, isIOS, isNative } from './env'

export let isUsingMicroTask = false

const callbacks = []
let pending = false

function flushCallbacks () {
  pending = false
  const copies = callbacks.slice(0)
  callbacks.length = 0
  for (let i = 0; i < copies.length; i++) {
  // 执行队列中的回调函数
    copies[i]()
  }
}

let timerFunc
if (typeof Promise !== 'undefined' && isNative(Promise)) {
// 如果平台支持promise函数 则用promise执行任务队列
  const p = Promise.resolve()
  timerFunc = () => {
    p.then(flushCallbacks)
    if (isIOS) setTimeout(noop)
  }
  isUsingMicroTask = true
} else if (!isIE && typeof MutationObserver !== 'undefined' && (
  // 如果浏览器支持MutationObserver 则用MutationObserver监测DOM的更新状态
  isNative(MutationObserver) ||
  // PhantomJS and iOS 7.x
  MutationObserver.toString() === '[object MutationObserverConstructor]'
)) {
  // 在浏览器中新建一个文本节点，然后通过观测文本节点的变化来决定监测DOM的更新状态
  let counter = 1
  const observer = new MutationObserver(flushCallbacks)
  const textNode = document.createTextNode(String(counter))
  observer.observe(textNode, {
    characterData: true
  })
  timerFunc = () => {
    counter = (counter + 1) % 2
    textNode.data = String(counter)
  }
  isUsingMicroTask = true
} else if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
  // 判断是否支持setImmediate函数
  timerFunc = () => {
    setImmediate(flushCallbacks)
  }
} else {
  // 否则使用setTimeout来更新队列函数
  // Fallback to setTimeout.
  timerFunc = () => {
    setTimeout(flushCallbacks, 0)
  }
}

export function nextTick (cb?: Function, ctx?: Object) {
  let _resolve
  // 把nextTick的回调函数加入到队列中，
  callbacks.push(() => {
    if (cb) {
      try {
      // 执行回调函数
        cb.call(ctx)
      } catch (e) {
        handleError(e, ctx, 'nextTick')
      }
    } else if (_resolve) {
      _resolve(ctx)
    }
  })
  if (!pending) {
    pending = true
    timerFunc()
  }
  // 当未传入回调函数时，提供一个promise的调用
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
}
```



