---
title: 手写promise
date: {{ date }}
tags: JS
cover: https://ppoffice.github.io/hexo-theme-icarus/gallery/covers/vector_landscape_2.svg
excerpt: 手写promise也是面试的常客，我们在本篇文章中会理解如何手写promise以及几个静态方法。
toc: true
---
# 手写Promise

## 1. 简易版Promise

需要注意的几点问题：

1. 整体实现思路：依赖收集（pending时将onFullfilled方法保存在数组中），通知回调（当状态变化时，遍历数组，依次触发回调）
2. 微任务实现：`queueMicrotask`或者`MutationObserver`
3. 注意在promise的executor中触发resolve或reject后，promise的状态是同步改变的。只是then方法中注册的回调将会异步触发（微任务）。

```js
const PENDING = "PENDING";
const FULLFILLED = "FULLFILLED";
const REJECTED = "REJECTED";

class MyPromise {
  constructor(executor) {
    this.status = PENDING;
    this.value = null;
    this.reason = null;
    this.onFullfilledCallbacks = [];
    this.onRejectedCallbacks = [];

    let resolve = (value) => {
      if (this.status === PENDING) {
        // 根据测试，状态应该同步改变
        this.value = value;
        this.status = FULLFILLED;
        queueMicrotask(() => {
          this.onFullfilledCallbacks.forEach((func) => func());
        });
      }
    };

    let reject = (reason) => {
      if (this.status === PENDING) {
        this.reason = reason;
        this.status = REJECTED;
        queueMicrotask(() => {
          this.onRejectedCallbacks.forEach((func) => func());
        });
      }
    };

    try {
      executor(resolve, reject);
    } catch (err) {
      reject(err);
    }
  }

  // 调用then方法时可能会存在三种不同的状态，需要分别处理
  then(onFullfilled, onRejected) {
    onFullfilled =
      typeof onFullfilled === "function" ? onFullfilled : (value) => value;
    onRejected =
      typeof onRejected === "function"
        ? onRejected
        : (err) => {
            throw err;
          };
    if (this.status === FULLFILLED) {
      queueMicrotask(() => {
        onFullfilled(this.value);
      });
    }

    if (this.status === REJECTED) {
      queueMicrotask(() => {
        onRejected(this.reason);
      });
    }

    if (this.status === PENDING) {
      this.onFullfilledCallbacks.push(() => {
        onFullfilled(this.value);
      });
      this.onRejectedCallbacks.push(() => {
        onRejected(this.reason);
      });
    }
  }
}

let p = new Promise((resolve, reject) => {
  resolve(1000);
  // setTimeout(() => {
  //   resolve(1000);
  // }, 1000);
});
new Promise((resolve, reject) => {
  resolve(1200);
}).then((value) => {
  console.log(value);
});
console.log(1);
p.then((value) => {
  console.log(value);
});
console.log(2);

```

## 2. then的链式调用与值穿透

实现思路

1. then方法应该返回一个新的promise对象
2. 当onFullfilled返回的是值时，直接resolve；如果是promise,使用`promise.then(value => resolve(value))`

```js
const PENDING = "PENDING";
const FULLFILLED = "FULLFILLED";
const REJECTED = "REJECTED";

class MyPromise {
  constructor(executor) {
    this.status = PENDING;
    this.value = null;
    this.reason = null;
    this.onFullfilledCallbacks = [];
    this.onRejectedCallbacks = [];

    let resolve = (value) => {
      if (this.status === PENDING) {
        // 根据测试，状态应该同步改变
        this.value = value;
        this.status = FULLFILLED;
        // 使用queueMicrotask添加一个微任务
        queueMicrotask(() => {
          this.onFullfilledCallbacks.forEach((func) => func());
        });
      }
    };

    let reject = (reason) => {
      if (this.status === PENDING) {
        this.reason = reason;
        this.status = REJECTED;
        queueMicrotask(() => {
          this.onRejectedCallbacks.forEach((func) => func());
        });
      }
    };

    try {
      executor(resolve, reject);
    } catch (err) {
      reject(err);
    }
  }

  // 使用单独的resolvePromise来处理onFullfilled方法的返回值
  // x的值可能是promise,可能是普通值
  resolvePromise(x, resolve, reject) {
    if (x instanceof MyPromise) {
      // x.then((value) => {
      //   resolve(value)
      // }, (err) => {
      //   reject(err);
      // })

      // 简单的版本：
      x.then(resolve, reject);
    } else {
      return resolve(x);
    }
  }

  then(onFullfilled, onRejected) {
    onFullfilled =
      typeof onFullfilled === "function" ? onFullfilled : (value) => value;
    onRejected =
      typeof onRejected === "function"
        ? onRejected
        : (err) => {
            throw err;
          };
    let promise2 = new MyPromise((resolve, reject) => {
      if (this.status === FULLFILLED) {
        queueMicrotask(() => {
          let x = onFullfilled(this.value);
          this.resolvePromise(x, resolve, reject);
          // resolve(x);
        });
      }

      if (this.status === REJECTED) {
        queueMicrotask(() => {
          let x = onRejected(this.reason);
          this.resolvePromise(x, resolve, reject);
        });
      }

      if (this.status === PENDING) {
         // 后面的then方法的onFullfilled回调将会保存在上一个promise对象的onFullfilledCallbacks数组中
        this.onFullfilledCallbacks.push(() => {
          let x = onFullfilled(this.value);
          this.resolvePromise(x, resolve, reject);
        });
        this.onRejectedCallbacks.push(() => {
          let x = onRejected(this.reason);
          this.resolvePromise(x, resolve, reject);
        });
      }
    });

    return promise2;
  }
}

let p = new MyPromise((resolve, reject) => {
  resolve(1000);
});
new MyPromise((resolve, reject) => {
  resolve(1200);
}).then((value) => {
  console.log(value);
});
console.log(1);
p.then((value) => {
  console.log(value);
  return new MyPromise((resolve, reject) => {
    setTimeout(() => {
      resolve('async')
    }, 1000)
  })
}).then((value) => {
  console.log(value);
})

p.then(() => {
  console.log(213);
})
console.log(2);

```

## 3. 静态方法（resolve, reject, race, all）

```js
const PENDING = "PENDING";
const FULLFILLED = "FULLFILLED";
const REJECTED = "REJECTED";

class MyPromise {
    
  // ...
    
  static resolve(val) {
    return new MyPromise((resolve, reject) => {
      resolve(val);
    });
  }

  static reject(err) {
    return new MyPromise((resolve, reject) => {
      reject(err);
    });
  }

  // race: 谁先resolve，谁就被then处理，其他的忽略，返回最快的结果
  // Promise.race([promise1, promise2, promise3]).then()
  static race(promises) {
    return new MyPromise((resolve, reject) => {
      for (let i = 0; i < promises.length; i++) {
        const promise = promises[i];
        promise.then(resolve, reject);
      }
    });
  }

  // all: 等待所有传入的promise都满足条件，拿到所有成功的结果
  // 获取所有的promise，执行then 结果
  // 利用了闭包机制
  static all(promises) {
    let arr = []; // 存结果，将结果返回，需要按顺序
    let count = 0; // 计数器，累计有多少成功了，如果结果的个数===arr.length，满足条件

    const processData = (index, data, resolve) => {
      arr[index] = data; // 按顺序存结果
      count++;
      if (count === promises.length) {
        resolve(arr);
      }
    };
    return new MyPromise((resolve, reject) => {
      for (let i = 0; i < promises.length; i++) {
        promises[i].then((data) => {
          processData(i, data, resolve);
        }, reject);
      }
    });
  }
}

let p1 = new MyPromise((resolve, reject) => {
  setTimeout(() => {
    resolve(1000);
  }, 1000);
});

let p2 = new MyPromise((resolve, reject) => {
  setTimeout(() => {
    resolve(2000);
  }, 2000);
});

let p3 = new MyPromise((resolve, reject) => {
  setTimeout(() => {
    resolve(3000);
  }, 3000);
});

MyPromise.race([p1,p2,p3]).then(value => {
   console.log(value); // 1000
})

MyPromise.all([p1, p2, p3]).then((value) => {
  console.log(value); // [1000,2000,3000]
});

```

