---
layout: post
title: "手写一个符合Promises/A+规范的Promise"
date: 2019-10-26 
description: "前端技术"
tag: 前端技术
---   

往下读之前需要先了解[Promises/A+规范](https://www.ituring.com.cn/article/66566)
##开始实现规范
```
// 一般使用的例子
new Promise(function (resolve, reject) {
  resolve(1)
}).then(function (value) {
  console.log(value)
})



const PENDING = 'pending'
const RESOLVED = 'resolved'
const REJECTED = 'rejected'

function MyPromise(fn) {
  // 存储this指向，以供下面resolve和reject函数使用
  const that = this
  that.state = PENDING
  that.value = null
  that.resolvedCallbacks = []
  that.rejectedCallbacks = []

  function resolve(value) {
    // value可能接受到的不是一个具体的一个值，而是另一个promise实例，
    // 此时就到等到该promise的状态改变了才可以执行回调
    if (value instanceof MyPromise) {
      return value.then(resolve, reject)
    }
    // 为了使回调在当前脚本所有同步任务执行完才执行
    setTimeout(() => {
      if (that.state === PENDING) {
        that.state = RESOLVED
        that.value = value
        that.resolvedCallbacks.map(cb => {
          return cb(value)
        })
      }
    }, 0)
  }

  function reject(error) {
    setTimeout(() => {
      if (that.state === REJECTED) {
        that.state = REJECTED
        that.value = error
        that.rejectedCallbacks.map(cb => {
          return cb(error)
        })
      }
    }, 0)
  }

  try {
    fn(resolve, reject)
  } catch (e) {
    reject(e)
  }
}

MyPromise.prototype.then = function (onFulFilled, onRejected) {
  const that = this
  onFulFilled =
    typeof onFulFilled === 'function'
      ? onFulFilled
      : v => v
  onRejected =
    typeof onRejected === 'function'
      ? onRejected
      : v => v

  // Promises/A+规范中promise的解决过程
  function resolutionProcedure(promise2, x, resolve, reject) {
    if (promise2 === x) {
      return reject(new TypeError('Error'))
    }

    if (x instanceof MyPromise) {
      x.then(value => {
        resolutionProcedure(promise2, value, resolve, reject)
      })
    }

    let called = false
    if (x !== null && (typeof x === 'object' || typeof x === 'function')) {
      try {
        let then = x.then
        if (typeof then === 'function') {
          then.call(
            x,
            y => {
              if (called) return
              called = true
              resolutionProcedure(promise2, y, resolve, reject)
            }),
            e => {
              if (called) return
              called = true
              reject(e)
            }
        } else {
          resolve(x)
        }
      } catch (e) {
        if (called) return
        called = true
        reject(e)
      }
    } else {
      resolve(x)
    }
  }

  if (that.state === PENDING) {
    return (promise2 = new MyPromise((resolve, reject) => {
      that.resolvedCallbacks.push(() => {
        try {
          const x = onFulFilled(that.value)
          resolutionProcedure(promise2, x, resolve, reject)
        } catch (e) {
          reject(e)
        }
      })
      that.rejectedCallbacks.push(() => {
        try {
          const x = onRejected(that.value)
          resolutionProcedure(promise2, x, resolve, reject)
        } catch (e) {
          reject(e)
        }
      })
    }))
  }
  if (that.state === RESOLVED) {
    return (promise2 = new MyPromise((resolve, reject) => {
      setTimeout(() => {
        try {
          const x = onFulFilled(that.value)
          resolutionProcedure(promise2, x, resolve, reject)
        } catch (e) {
          reject(e)
        }
      })
    }))
  }
  if (that.state === REJECTED) {
    return (promise2 = new MyPromise((resolve, reject) => {
      setTimeout(() => {
        try {
          const x = onRejected(that.value)
          resolutionProcedure(promise2, x, resolve, reject)
        } catch (e) {
          reject(e)
        }
      })
    }))
  }
}

```



  
