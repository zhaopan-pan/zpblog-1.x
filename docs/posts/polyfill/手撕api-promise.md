---
sidebar: auto
date: "2021-7-4"
tag: polyfill 
category: 
 - frontEnd
title: 手撕api-promise
---

## 核心代码

废话少说，直接上代码

```js
const STATUS = [`pending`, `rejected`, `fulfilled`];
const PENDING = `pending`;
const REJECTED = `rejected`;
const FULFILLED = `fulfilled`;
const isMyPromise = (fn) => fn instanceof MyPromise;
class MyPromise {
    constructor(executeCallback) {
        this.status = PENDING;
        this.value = undefined;
        this.fulfilledList = [];
        this.rejectList = [];

        try {
            executeCallback(this._resolveFn.bind(this), this._rejectFn.bind(this));
        } catch (err) {
            // 有异常信息按照rejected状态处理
            this._rejectFn(err);
        }
    }
    // 利用settimeout模拟微任务,
    // 因为实例化Promise时传入的resolve、reject两个回调函数，即使调用他们也并不会立即执行，而是等到.then的时候，
    // 把then(fulfilledCallBack, rejectedCallBack)中这两个回调函数push到完成队列(fulfilledList)和者拒绝队列(rejectList)后，才会执行这两个队列
    _resolveFn(val) {
        if (this.status !== PENDING) return;
        const run = () => {
            // 依次执行成功队列中的函数，并清空队列
            const runFulfilled = (value) => {
                let cb;
                this.value = value;
                this.status = FULFILLED;
                while ((cb = this.fulfilledList.shift())) {
                    cb(value); //执行then中添加的回调函数
                }
            };
            // 依次执行失败队列中的函数，并清空队列
            const runRejected = (err) => {
                let cb;
                this.value = err;
                this.status = REJECTED;
                while ((cb = this.rejectList.shift())) {
                    cb(err);
                }
            };
            // 如果resolve的参数为Promise对象，则必须等待该Promise对象状态改变后,
            // 当前Promsie的状态才会改变，且状态取决于参数Promsie对象的状态
            if (isMyPromise(val)) {
                val.then(runFulfilled, runRejected);
            } else {
                runFulfilled(val);
            }
        };
        setTimeout(run);
    }

    _rejectFn(val) {
        if (this.status !== PENDING) return;
        const run = () => {
            this.status = REJECTED;
            this.value = val;
            let cb;
            while ((cb = this.rejectList.shift())) {
                cb(this.value);
            }
        };
        setTimeout(run);
    }

    then = (fulfilledCallBack, rejectedCallBack) => {
        const {
            value,
            status
        } = this;
        //保证两者为函数
        typeof fulfilledCallBack !== "function" ?
            (fulfilledCallBack = (result) => result) :
            null;
        typeof rejectedCallBack !== "function" ?
            (rejectedCallBack = (reason) => {
                throw new Error(reason instanceof Error ? reason.message : reason);
            }) :
            null;
        // 所谓的链式操作就是自己返回新的自己，
        return new MyPromise((resolve, reject) => {
            // 声明成功时的回调函数
            let fulfilled = (value) => {
                try {
                    // 执行then中传过来的resolve回调方法
                    let res = fulfilledCallBack(value);
                    // 支持在resolve或者reject中嵌套promise，如果是返回promise就执行then方法
                    // 如果不是promise就执行新promise的resolve回调
                    isMyPromise(res) ? res.then(resolve, reject) : resolve(res);
                } catch (err) {
                    reject(err); // 代码有异常直接reject
                }
            };

            // 声明失败时的回调函数
            let rejected = (error) => {
                try {
                    // 执行then中传过来的resolve回调方法
                    let res = rejectedCallBack(error);
                    // 支持在resolve或者reject中嵌套promise，如果是返回promise就执行then方法
                    // 如果不是promise就执行新promise的resolve回调
                    isMyPromise(res) ? res.then(resolve, reject) : resolve(res);
                } catch (err) {
                    reject(err);
                }
            };

            switch (status) {
                // 当状态为pending时，将then方法回调函数加入执行队列等待执行
                case PENDING:
                    // 在队列中添加回调函数，等待在微任务中执行
                    this.fulfilledList.push(fulfilled);
                    this.rejectList.push(rejected);
                    break;
                    // 当状态已经改变时，立即执行对应的回调函数
                case FULFILLED:
                    fulfilled(value); // 当resolve(MyPromise.resolve(1));会走这里
                    break;
                case REJECTED:
                    rejected(value); // // 当resolve(MyPromise.rejected(1));会走这里
                    break;
            }
        });
    };

    catch = (rejectCallback) => this.then(null, rejectCallback);

    /**
     * 在Promise实例上挂载finally 方法
     * p.finally(() => {})本质是一个then方法，所以在实现方法中要调用then方法
     * 入参f是一个函数，需要在then方法中执行这个函数
     * 使用Promise.resolve会等f()的函数执行完再返回结果，并将上一个then的value返回
     * reject方法中需要抛出错误信息
     * @param {*} fn
     */
    finally(fn) {
        return this.then(
            (value) => MyPromise.resolve(fn()).then(() => value),
            (err) =>
            MyPromise.resolve(fn()).then(() => {
                throw err;
            })
        );
    }

    // resolve的值是promise的时候直接返回就行
    static resolve = (val) => {
        return isMyPromise(val) ? val : new MyPromise((resolve) => resolve(val));
    };

    static reject = (val) => new MyPromise((resolve, reject) => reject(val));

    /**
     * 所有promise成功才可以resolve，如有一个失败则reject
     * @param promiseList promise[]
     */
    static all = (promiseList = []) => {
        if (!Array.isArray(promiseList) || promiseList.length === 0) return;
        let resList = [];
        return new MyPromise((resolve, reject) => {
            for (let i = 0; i < promiseList.length; i++) {
                if (!isMyPromise(promiseList[i])) {
                    reject(new TypeError("must be Promise type"));
                }
                promiseList[i] &&
                    promiseList[i]
                    .then((res) => {
                        resList.push(res);
                        // 连个队列长度相同说明所有的promise已运行完毕
                        if (resList.length === promiseList.length) {
                            resolve(resList);
                        }
                    })
                    // 有promise运行异常直接停止并reject
                    .catch((err) => reject(err));
            }
        });
    };

    /**
     * 竞赛执行，以队列中第一个MyPromise的执行结果为准，不论成功或失败
     * @param promiseList promise[]
     */
    static race = (promiseList = []) => {
        if (!Array.isArray(promiseList) || promiseList.length === 0) return;
        return new MyPromise((resolve, reject) => {
            for (let p of promiseList) {
                if (!isMyPromise(p)) {
                    reject(new TypeError("must be Promise type"));
                }
                this.resolve(p).then(resolve, reject);
            }
        });
    };
}
```

## 复杂案例流程解析

```js
new MyPromise((resolve, reject) => {
        resolve(MyPromise.reject("1111"));
    })
    .then((res) => console.log("res:", res))
    .catch((err) => console.log("err:", err))
```

  + `new MyPromise()`开始初始化`MyPromise`，
  传入了回调函数 `(resolve, reject) => {resolve(MyPromise.reject("1111")); }` 进到 `constructor`

  + 其中`resolve` 传入了一个`MyPromise.reject('1111')`，所以又重新开始初始化一个只有reject的MyPromise实例，所以先执行了`_rejectFn`，此时`settimeout`定时器队列 中被推入第一个`_rejectFn`函数
  + 同步函数继续往下执行，开始 `resolve(MyPromise.reject("1111")` 执行 `resolve ` 到了@_resolveFn ，此时@settimeout定时器队列 被推入第二个 `resolveFn` 函数， 的参数是一个@MyPromise 对象
  + 继续同步执行到@then函数 根据目前状态 `pending` 会往 `fulfilledList` 和 `rejectList` 都push进调用@then 时传入的两个回调函数
  此刻同步函数执行完毕，最先进入 @settimeout定时器队列 的 `_rejectFn` 函数 `reject('1111')` 开始执行，过程中 `status` 变成了 `rejected` ， `value` 变成了1111
  + `reject('1111')` 执行完毕后，就到settimeout定时器队列 中的第二个进入的 `_resolveFn` 函数 执行了，r `esolve(MyPromise.reject("1111")` ，因为resolve的是一个MyPromise对象, 则调用该对象的 `then` 函数，因为此刻 `status` 为 `rejected` 所以在返回的 `MyPromise` 中直接执行 `rejected(value)` 函数 
  + 其中的 `rejectedCallBack` 就是上面调用 `then` 时传入的 `runRejected` 函数，也是就是直接执行了 `runRejected` 函数
 其中 `rejectList` 在最开始执行同步函数的时候已经push进了最外层 的 `(err)=>{cosnole.log('err', err)}` 回调, 所以错误现在会被打印出： `err 1111`
