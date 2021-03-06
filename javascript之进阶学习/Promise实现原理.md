
### Promise实现原理
* [1. 函数雏形](#id1)
* [2 引入状态值](#id2)
* [3 对象保护](#id3)
* [3.1 then的改变](#id31)
* [3.2 resolve的改变](#id32)
* [4 失败reject处理](#id4)
* [5 异常处理](#id5)

#### <h2 id='id1'>1 函数雏形</h2>
我们先来看个简单的例子,使用Promise调用resolve函数,然后在then相应
```
new Promise(function (resolve) {
    resolve('Hello World');
}).then(function (value) {
    console.log('then = ' + value);
})

```
根据这个我们先制作Promise的雏形,<br/>
那思路应该是怎么样的泥?我们发现在 promise 使用 resolve 引起 then 反应<br/>
我们想象下面的栗子,就如小明需要在教室内把东西给小红,但小明不知道小红什么时候进教室内,就需要小红进教室时签到,教室里用广播通知小明,小明再把东西给小红<br/>
在事件中,小红是then事件,需要先注册到数组里,小明给东西resolve(value),通过数组调用then事件,在由resolve中把值给then

```
function Promise(fn) {
    var value = null;
    var callbacks = [];

    this.then = function (onFulfilled) {
        // 添加事件
        callbacks.push(onFulfilled);
        return this;
    }

    function resolve(newValue) {
        setTimeout(function () {
            callbacks.forEach(function (callback) {
                //通知
                callback(newValue);
            });
        });
    }

    fn(resolve);

}

new Promise(function (resolve) {
    resolve('Hello World');
}).then(function (value) {
    console.log('then = ' + value);
})
// 输入then = Hello World
 
```

这样会得到一个可以输出的Promise,我们来解释几个问题<br/>
1.为什么在resolve里面使用setTimeout()?<br/>
如果不使用 setTimeout , resolve里面任务会在then之前就执行完毕, 无法存储then里面的事件,造成数组为空,
所以setTimeout 作为宏任务使事件滞后,使then事件可以被注册

#### <h2 id='id2'>2 引入状态值</h2>
promise 一共有三个状态 pending 进行中,fulfilled 完成,rejected 失败<br/>
我们先引入pending 进行中,fulfilled 完成,这两个状态

```
function Promise(fn) {
    var Progress = {
        Pending: 'pending',
        Fulfilled: 'fulfilled',
    }
    var state = Progress.Pending;
    var value = null;
    var callbacks = [];

    this.then = function (onFulfilled) {
        if (state === Progress.Pending) {
            callbacks.push(onFulfilled);
            return this;
        }
        // fulfilled 直接返回值
        onFulfilled(value);
        return this;
    }

    function resolve(newValue) {
        value = newValue;
        state = Progress.Fulfilled;
        setTimeout(function () {
            callbacks.forEach(function (callback) {
                callback(newValue);
            });
        });
    }

    fn(resolve);

}
new Promise(function (resolve) {
    resolve('Hello World');
}).then(function (value) {
    // 状态已经是fulfilled
    console.log('then = ' + value);
}).then(function (value) {
    // 状态已经是fulfilled
    console.log('then = ' + value);
})
```

#### <h2 id='id3'>3 串行Promise</h2>

串行Promise 是指在当前 promise 达到 fulfilled 状态后即开始进行下一个 promise（后邻 promise）<br/>
我们需要支持返回promise时then的返回,支持如下函数

```
new Promise(function (resolve) {
    resolve(1);
}).then(function (value) {
    // 返回Promise
    return new Promise(function (resolve) {
        resolve(value);
    });
}).then(function (value) {
    console.log('结束: ' + value);
})
```

这就需要在原来的基础上增加对promise的回调,我们全部贴上来在一个分析

```
function Promise(fn) {
    var Progress = {
        Pending: 'pending',
        Fulfilled: 'fulfilled',
    }
    var state = Progress.Pending;
    var value = null;
    var callbacks = [];

    function handle(callback) {
        if (state === Progress.Pending) {
            callbacks.push(callback);
            return;
        }
        var ret = callback.onFulfilled(value);

        // 如果是promise
        if (ret) {
            callback.resolve(ret);
        }
    }

    this.then = function (onFulfilled) {
        // 新建promise返回
        return new Promise(function (resolve) {
            handle({
                onFulfilled: onFulfilled || null,
                resolve: resolve
            });
        });
    }

    function resolve(newValue) {

        if (newValue &&
            (typeof newValue === 'object' || typeof newValue === 'function')) {
            var then = newValue.then;
            if (typeof then === 'function') {
                then.call(newValue, resolve);
                return;
            }
        }

        state = Progress.Fulfilled;
        value = newValue;
        setTimeout(function () {
            callbacks.forEach(function (callback) {
                handle(callback);
            });
        });
    }

    fn(resolve);

}      
```
#### <h2 id='id31'>3.1 then的改变</h2>

```
this.then = function (onFulfilled) {
    // 新建promise返回
    return new Promise(function (resolve) {
        handle({
            onFulfilled: onFulfilled || null,
            resolve: resolve
        });
    });
}

function handle(callback) {
    if (state === Progress.Pending) {
        callbacks.push(callback);
        return;
    }
    var ret = callback.onFulfilled(value);

    // 如果是promise
    if (ret) {
        callback.resolve(ret);
    }
}

```

#### <h2 id='id32'>3.2 resolve的改变</h2>
```
function resolve(newValue) {

    if (newValue &&
        (typeof newValue === 'object' || typeof newValue === 'function')) {
        var then = newValue.then;
        if (typeof then === 'function') {
            then.call(newValue, resolve);
            return;
        }
    }

    state = Progress.Fulfilled;
    value = newValue;
    setTimeout(function () {
        callbacks.forEach(function (callback) {
            handle(callback);
        });
    });
    
}

```

#### <h2 id='id4'>4 失败reject处理</h2>
增加reject函数,在handle增加对reject的处理

```
function Promise(fn) {
    var Progress = {
        Pending: 'pending',
        Fulfilled: 'fulfilled',
        Rejected: 'rejected'
    }
    var state = Progress.Pending,
        value = null,
        callbacks = [];

    this.then = function (onFulfilled, onRejected) {
        return new Promise(function (resolve, reject) {
            handle({
                onFulfilled: onFulfilled || null,
                onRejected: onRejected || null,
                resolve: resolve,
                reject: reject
            });
        });
    };

    function handle(callback) {
        if (state === Progress.Pending) {
            callbacks.push(callback);
            return;
        }
        // 是否事件执行完成 完成则获取then里面的事件
        // 如果不是promise
        var thenEvent = state === Progress.Fulfilled ? callback.onFulfilled : callback.onRejected;
        if (thenEvent === null) {
            thenEvent = state === Progress.Fulfilled ? callback.resolve : callback.reject;
            thenEvent(value);
            return;
        }
        // 是promise的话
        var promise = thenEvent(value);
        callback.resolve(promise);
    }

    function resolve(newValue) {
        // 是promise的话
        if (newValue && (typeof newValue === 'object' || typeof newValue === 'function')) {
            var then = newValue.then;
            if (typeof then === 'function') {
                then.call(newValue, resolve, reject);
                return;
            }
        }
        state = Progress.Fulfilled;
        value = newValue;
        handleCallback();
    }

    function reject(reason) {
        state = Progress.Rejected;
        value = reason;
        handleCallback();
    }

    function handleCallback() {
        setTimeout(function () {
            callbacks.forEach(function (deferred) {
                handle(deferred);
            });
        }, 0);
    }

    fn(resolve, reject);

}

new Promise(function (resolve, reject) {
    resolve('first Promise resolve');
}).then(function (value) {
    return new Promise(function (resolve, reject) {
        reject(value + '/second Promise reject');
    });
}).then(function (value) {
    console.log('value = ' + value);
}, function (error) {
    console.log('error = ' + error);
});

```

#### <h2 id='id5'>5 异常处理</h2>
在执行方法内中 try-catch,并在catch回调函数<br/>
```
try {
    fn(resolve, reject);
} catch (error) {
    reject(error);
}
```
 
```
new Promise(function (resolve, reject) {
    resolve('first Promise resolve');
}).then(function (value) {
    return new Promise(function (resolve, reject) {
        a = 1;
    });
}).then(function (value) {
    console.log('value = ' + value);
}, function (error) {
    console.log('error = ' + error);
});  
// error = ReferenceError: a is not defined

```


<br/><br/><br/><br/><br/>
参考资料:<br/>
https://tech.meituan.com/promise-insight.html

