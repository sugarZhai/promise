这篇文章主要是为了整理《你不知道的js》中的部分笔记，可是没想到promise要讲的东西太多了，所以单独写了一篇文章来介绍它，后期也会持续更新


[TOC]
#第三章 Promise
#Promise

 - 雏形分析

先看一个Promise的雏形

```
       function Promise(fn) {
            var value = null,
                callbacks = []; 
                 //callbacks为数组，因为可能同时有很多个回调
            this.then = function (onFulfilled) {
                callbacks.push(onFulfilled);
            };

            function resolve(value) {
                callbacks.forEach(function (callback) {
                    callback(value);
                });
            }
           fn(resolve);
        }
```

> 上述代码很简单，大致的逻辑是这样的：
> 1、调用then方法，将想要在Promise异步操作成功时执行的回调放入callbacks队列，其实也就是注册回调函数，可以向观察者模式方向思考；
>2、创建Promise实例时传入的函数会被赋予一个函数类型的参数，即resolve，它接收一个参数value，代表异步操作返回的结果，当一步操作执行成功后，用户会调用resolve方法，这时候其实真正执行的操作是将callbacks队列中的回调一一执行；

```
支持then链式调用的写法
this.then = function (onFulfilled) {
    callbacks.push(onFulfilled);
    return this;
};
```

```
// 例2
getUserId().then(function (id) {
    // 一些处理
}).then(function (id) {
    // 一些处理
});
```

 - 延时机制
上述代码可能还存在一个问题：如果在then方法注册回调之前，resolve函数就执行了，怎么办？比如promise内部的函数是同步函数：

```
// 例3
function getUserId() {
    return new Promise(function (resolve) {
        resolve(9876);
    });
}
getUserId().then(function (id) {
    // 一些处理
});
```

> 这显然是不允许的，Promises/A+规范明确要求回调需要通过异步方式执行，用以保证一致可靠的执行顺序。因此我们要加入一些处理，保证在resolve执行之前，then方法已经注册完所有的回调。我们可以这样改造下resolve函数:

```
function resolve(value) {
    setTimeout(function() {
        callbacks.forEach(function (callback) {
            callback(value);
        });
    }, 0)
} 
```

> 上述代码的思路也很简单，就是通过setTimeout机制，将resolve中执行回调的逻辑放置到JS任务队列末尾，以保证在resolve执行时，then方法的回调函数已经注册完成.

但是，这样好像还存在一个问题，可以细想一下：如果Promise异步操作已经成功，这时，在异步操作成功之前注册的回调都会执行，但是在Promise异步操作成功这之后调用的then注册的回调就再也不会执行了，这显然不是我们想要的

 - 加入状态
根据上一个问题的抛出，我们必须加入状态机制，也就是大家熟知的pending、fulfilled、rejected。

Promises/A+规范中的2.1Promise States中明确规定了，pending可以转化为fulfilled或rejected并且只能转化一次，也就是说如果pending转化到fulfilled状态，那么就不能再转化到rejected。并且fulfilled和rejected状态只能由pending转化而来，两者之间不能互相转换。一图胜千言：

 ![这里写图片描述](https://img-blog.csdn.net/20180721165456517?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L01pY2hlbGxlWmhhaQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

```
   function Promise(fn) {
    var state = 'pending',
        value = null,
        callbacks = [];

    this.then = function (onFulfilled) {
        if (state === 'pending') {
            callbacks.push(onFulfilled);
            return this;
        }
        onFulfilled(value);
        return this;
    };

    function resolve(newValue) {
        value = newValue;
        state = 'fulfilled';
        setTimeout(function () {
            callbacks.forEach(function (callback) {
                callback(value);
            });
        }, 0);
    }

    fn(resolve);
}
```

> 上述代码的思路是这样的：resolve执行时，会将状态设置为fulfilled，在此之后调用then添加的新回调，都会立即执行。

 - 链式Promise
> 那么这里问题又来了，如果用户再then函数里面注册的仍然是一个Promise，该如何解决？比如下面的例4

```
// 例4
getUserId()
    .then(getUserJobById)
    .then(function (job) {
        // 对job的处理
    });

function getUserJobById(id) {
    return new Promise(function (resolve) {
        http.get(baseUrl + id, function(job) {
            resolve(job);
        });
    });
}
```
链式Promise是指在当前promise达到fulfilled状态后，即开始进行下一个promise（后邻promise）。那么我们如何衔接当前promise和后邻promise呢？（这是这里的难点）。

其实也不是辣么难，只要在then方法里面return一个promise就好啦。Promises/A+规范中的2.2.7就是这么说

下面来看看这段暗藏玄机的then方法和resolve方法改造代码：

```

function Promise(fn) {
    var state = 'pending',
        value = null,
        callbacks = [];

    this.then = function (onFulfilled) {
        return new Promise(function (resolve) {
            handle({
                onFulfilled: onFulfilled || null,
                resolve: resolve
            });
        });
    };

    function handle(callback) {
        if (state === 'pending') {
            callbacks.push(callback);
            return;
        }
        //如果then中没有传递任何东西
        if(!callback.onFulfilled) {
            callback.resolve(value);
            return;
        }

        var ret = callback.onFulfilled(value);
        callback.resolve(ret);
    }

    
    function resolve(newValue) {
        if (newValue && (typeof newValue === 'object' || typeof newValue === 'function')) {
            var then = newValue.then;
            if (typeof then === 'function') {
                then.call(newValue, resolve);
                return;
            }
        }
        state = 'fulfilled';
        value = newValue;
        setTimeout(function () {
            callbacks.forEach(function (callback) {
                handle(callback);
            });
        }, 0);
    }

    fn(resolve);
}
```
> 1、then方法中，创建并返回了新的Promise实例，这是串行Promise的基础，并且支持链式调用。
>2、handle方法是promise内部的方法。then方法传入的形参onFulfilled以及创建新Promise实例时传入的resolve均被push到当前promise的callbacks队列中，这是衔接当前promise和后邻promise的关键所在（这里一定要好好的分析下handle的作用）。
>3、getUserId生成的promise（简称getUserId promise）异步操作成功，执行其内部方法resolve，传入的参数正是异步操作的结果id
>4、调用handle方法处理callbacks队列中的回调：getUserJobById方法，生成新的promise（getUserJobById promise）
>5、执行之前由getUserId promise的then方法生成的新promise(称为bridge promise)的resolve方法，传入参数为getUserJobById promise。这种情况下，会将该resolve方法传入getUserJobById promise的then方法中，并直接返回。
>6、在getUserJobById promise异步操作成功时，执行其callbacks中的回调：getUserId bridge promise中的resolve方法
>7、最后执行getUserId bridge promise的后邻promise的callbacks中的调。

 - 失败处理
```
//例5
function getUserId() {
    return new Promise(function(resolve) {
        //异步请求
        http.get(url, function(error, results) {
            if (error) {
                reject(error);
            }
            resolve(results.id)
        })
    })
}

getUserId().then(function(id) {
    //一些处理
}, function(error) {
    console.log(error)
})
```
有了之前处理fulfilled状态的经验，支持错误处理变得很容易,只需要在注册回调、处理状态变更上都要加入新的逻辑：

```
function Promise(fn) {
    var state = 'pending',
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
        if (state === 'pending') {
            callbacks.push(callback);
            return;
        }

        var cb = state === 'fulfilled' ? callback.onFulfilled : callback.onRejected,
            ret;
        if (cb === null) {
            cb = state === 'fulfilled' ? callback.resolve : callback.reject;
            cb(value);
            return;
        }
        ret = cb(value);
        callback.resolve(ret);
    }

    function resolve(newValue) {
        if (newValue && (typeof newValue === 'object' || typeof newValue === 'function')) {
            var then = newValue.then;
            if (typeof then === 'function') {
                then.call(newValue, resolve, reject);
                return;
            }
        }
        state = 'fulfilled';
        value = newValue;
        execute();
    }

    function reject(reason) {
        state = 'rejected';
        value = reason;
        execute();
    }

    function execute() {
        setTimeout(function () {
            callbacks.forEach(function (callback) {
                handle(callback);
            });
        }, 0);
    }

    fn(resolve, reject);
}
```
上述代码增加了新的reject方法，供异步操作失败时调用，同时抽出了resolve和reject共用的部分，形成execute方法。

错误冒泡是上述代码已经支持，且非常实用的一个特性。在handle中发现没有指定异步操作失败的回调时，会直接将bridge promise(then函数返回的promise，后同)设为rejected状态，如此达成执行后续失败回调的效果。这有利于简化串行Promise的失败处理成本，因为一组异步操作往往会对应一个实际功能，失败处理方法通常是一致的：

```
//例6
getUserId()
    .then(getUserJobById)
    .then(function (job) {
        // 处理job
    }, function (error) {
        // getUserId或者getUerJobById时出现的错误
        console.log(error);
    });
```

 - 异常处理
 如果在执行成功回调、失败回调时代码出错怎么办？对于这类异常，可以使用try-catch捕获错误，并将bridge promise设为rejected状态。handle方法改造如下：
 

```
function handle(callback) {
    if (state === 'pending') {
        callbacks.push(callback);
        return;
    }

    var cb = state === 'fulfilled' ? callback.onFulfilled : callback.onRejected,
        ret;
    if (cb === null) {
        cb = state === 'fulfilled' ? callback.resolve : callback.reject;
        cb(value);
        return;
    }
    try {
        ret = cb(value);
        callback.resolve(ret);
    } catch (e) {
        callback.reject(e);
    } 
}
```
如果在异步操作中，多次执行resolve或者reject会重复处理后续回调，可以通过内置一个标志位解决。

 - 总结
 
现在回顾下Promise的实现过程，其主要使用了设计模式中的观察者模式：
1、通过Promise.prototype.then和Promise.prototype.catch方法将观察者方法注册到被观察者Promise对象中，同时返回一个新的Promise对象，以便可以链式调用。
2、被观察者管理内部pending、fulfilled和rejected的状态转变，同时通过构造函数中传递的resolve和reject方法以主动触发状态转变和通知观察者。
 
 
 - 练习
1、怎么解决回调函数⾥⾯回调另⼀个函数，另⼀个函数的参数需要依赖这个回调函数。需要被解决的代码如下：

```
$http.get(url).success((res)=>{
          if(success){
              success(res)
          }
       }).error((err)=>{
           if(error){
               error(err)
           }
       })
       success=(data)=>{
         if(data.id!==0){
            let url="getdata/data?id="+data.id+"";
            $http.get(url).success((res)=>{
                showData(res)
            }).error((err)=>{
                if(error){
                    error(err)
                }
            })
         }
       }
```
解决方案：promise.then()链式调用

```
       $http.get(url).then((data)=>{
            if(res.id!==0){
                let url="getdata/res?id="+res.id+"";
                return $http.get(data)
            }
        }).then(success((res)=>{
            showData(res)
            console.log("resolved:",res);
        }),error((err)=>{
            console.log("rejected:",err)
        }))
```
2、以下代码依次输出的内容是?

```
     setTimeout(() => {
            console.log(1)
        }, 0);
        new Promise(executor((resolve)=>{
            console.log(2)
            for(let i=0;i<1000;i++){
                i===999&&resolve();
            }
            console.log(3)
        }).then(()=>{
            console.log(4)
        }));
        console.log(5)
        //2 3 5 4 1   
         resolve()=>同步其他流程=>then()=>setTimeout()
        //首先setTimeout()在定时结束后将传递这个函数放到任务队列,最后一步执行
        // 然后是一个Promise,直接执行 2、3
        //promise的then应当会放到tick的最后，但是还是在当前tick中
        //因此先 5后 4，最后到下一个tick 1
```
3、jQuery的ajax返回的是promise对象吗?

```
     //jquery的ajax返回的是deferred对象，
    //通过promise的resolve()方法将其转换为promise对象
   let jsPromise=Promise.resolve($.ajax('/whatever.json'));
```
4、promise只有2个状态，成功和失败，怎么让一个函数无论成功还是失败都能 被调用?  Promise.all()

 - Promise.all方法用于将多个Promise实例，包装成一个新的Promise实例
 - Promise.all()方法接收一个数组作为参数，数组的元素都是Promise对象的实例, 如果不是，就会先调用下面讲到的Promise.resolve方法，将参数转为Promise实例，再进一步处理 
 -  Promise.all()方法的参数可以不是数组，但必须具有Iterator接口，且返回的每个成员都是Promise实例

> let p=Promise.all([p1,p2,p3]);
>     p的状态有p1,p2,p3决定，分为两种情况。
>     1、当该数组的所有的Promise实例都进入Fullfilled状态：Promise.all**返回的实例才会变成Fullfilled状态
>     并将Promise实例数组的所有返回值组成一个数组，传递给Promise.all返回实例的回调函数**
>     2、当该数组里的某个Promise实例都进入Reject状态：Promise.all返回的实例会立即变成Reject状态，并将第一个rejected的实例返回值
>     传递给Promise.all返回实例的都进入Rejected状态：Promise.all返回的实例会立即变成Rejected状态。并将第一个rejected的实例
>     返回值传递给Promise.all返回实例的回调函数

在这里我还想补充一个知识点：
##promise.all()和promise.race()的区别：
promise.all()
使用场景：多个异步结果合并到一起
特点：如果有任何一个失败，该Promise全部失败，返回值是第一个失败的子Promise的结果

```
Promise.all([p1,p2,p3])
let p1 = new Promise((resolve, reject) => {
  resolve('成功了')
})

let p2 = new Promise((resolve, reject) => {
  resolve('success')
})

let p3 = Promse.reject('失败')

Promise.all([p1, p2]).then((result) => {
  console.log(result)               //['成功了', 'success']
}).catch((error) => {
  console.log(error)
})

Promise.all([p1,p3,p2]).then((result) => {
  console.log(result)
}).catch((error) => {
  console.log(error)      // 失败了，打出 '失败'
})
```
Promise.race()
使用场景：把异步操作和定时器放到一起，如果定时器先触发，认为超时，告知用户
特点：它有人以一个返回成功后，就算完成，但是进程不会立即停止

```
let p1 = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve('成功了')
  }, 2000);
})

let p2 = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve('success')    
  }, 5000);
})


Promise.race([p1, p2]).then((result) => {
  console.log(result)               //['成功了', 'success']
}).catch((error) => {
  console.log(error)
})
```

5、分析下列程序代码，得出运行结果，解释其原因 

```
     const promise1=new Promise((resolve,reject)=>{
            setTimeout(()=>{
                resolve('success')
            },1000)
        })
        const promise2=promise1.then(()=>{
            throw new Error('error!!!')
        })
        console.log('promise1',promise1)
        console.log('promise2',promise2)
        setTimeout(()=>{
          console.log('promise1',promise1)
          console.log('promise2',promise2)
        },2000)
```
![这里写图片描述](https://img-blog.csdn.net/20180721181735886?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L01pY2hlbGxlWmhhaQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

```
        const promise=new Promise((resolve,reject)=>{
            setTimeout(()=>{
                console.log('once')
                resolve('success')
            },1000)
        })

        const start=Date.now()
        promise.then((res)=>{
            console.log(res,Date.now()-start)
        })
        promise.then((res)=>{
            console.log(res,Date.now()-start)
        })
        //once
        //success 1001
        //success 1001
//promise的then或者.catch可以被调用多次，但这里Promise构造函数只执行一次，
//或者说Promise内部状态一经改变，并且有了一个值，
//那么后续每次调用.then或者catch都会直接拿到该值
```

```
        const promise=Promise.resolve()
        .then(()=>{
            return promise
        })
        promise.catch(console.error)
        //TypeError
        //.then或.catch返回的值不能是promise本身，否则会造成死循环
```

```
        Promise.resolve(1)
        .then(2)
        .then(Promise.resolve(3))
        .then(console.log)
        //1
        //.then或者.catch的参数期望是函数，传入非函数则会发生值穿透
```

```
        Promise.resolve()
        .then(success((res)=>{
            throw new Error('error')
        }),fail((e)=>{
            console.log('fail1:',e)
        }))
        .catch(fail2((e)=>{
            console.log('fail2:',e)
        }))
        //fail2:Error:error
        //at success(...)
        //原因：.then可以接收两个参数，第一个是处理成功的函数，第二个是处理错误的函数。
        //.catch是.then第二个参数的简便写法，但是它们用法上有一点需要注意：
        //.then的第二个处理错误的函数捕获不了第一个处理成功的函数抛出的错误，
        //而后续的.catch可以捕获之前的错误
```

```
       process.nextTick(()=>{
            console.log('nextTick')
        })
        Promise.resolve()
         .then(()=>{
             console.log('then')
         })
         setTmmediate(()=>{
             console.log('setTmmediate')
         })
         console.log('end')
         // end
         //nextTick
         //then
         //setTmmediate
         //原因：process.nextTick和promise.then都属于macrotask,在事件循环的check阶段执行。
         //事件循环的每个阶段(macrotask)之间都会执行microtask,事件循环的开始会限制性一次microtask
```
#Promise/A+规范
Promise表示一个异步操作的最终结果。与Promise最主要的交互方法是通过将函数传入它的then方法从而获取得Promise最终的值或Promise最终最拒绝（reject）的原因。
##术语
promise是一个包含了兼容promise规范then方法的对象或函数，
thenable 是一个包含了then方法的对象或函数。
value 是任何Javascript值。 (包括 undefined, thenable, promise等).
exception 是由throw表达式抛出来的值。
reason 是一个用于描述Promise被拒绝原因的值
##要求
###Promise状态
一个Promise必须处在其中之一的状态：pending, fulfilled 或 rejected.

 - 如果是pending状态,则promise：

 > 可以转换到fulfilled或rejected状态

 - 如果是fulfilled状态,则promise：
> 不能转换成任何其它状态
> 必须有一个值，且这个值不能被改变。

 - 如果是rejected状态,则promise可以：
> 不能转换成任何其它状态
> 必须有一个原因，且这个值不能被改变。

”值不能被改变”指的是其identity不能被改变，而不是指其成员内容不能被改变。
###then 方法
一个Promise必须提供一个then方法来获取其值或原因。
Promise的then方法接受两个参数：

> promise.then(onFulfilled, onRejected)


1、 onFulfilled 和 onRejected 都是可选参数：
      

 - 如果onFulfilled不是一个函数，则忽略之
 - 如果onRejected不是一个函数，则忽略之

2、如果onFulfilled是一个函数:
     

 - 它必须在promise fulfilled后调用， 且promise的value为其第一个参数。
 - 它不能在promise fulfilled前调用
 - 不能被多次调用

3、如果onRejected是一个函数：
      

 - 它必须在promise rejected后调用， 且promise的reason为其第一个参数。
 - 它不能在promise rejected前调用
 - 不能被多次调用

4、onFulfilled 和 onRejected 只允许在 execution context 栈仅包含平台代码时运行
5、onFulfilled 和 onRejected 必须被当做函数调用 (i.e. 即函数体内的 this 为undefined) 
6、对于一个promise，它的then方法可以调用多次
     

 - 当promise fulfilled后，所有onFulfilled都必须按照其注册顺序执行。
 - 当promise rejected后，所有OnRejected都必须按照其注册顺序执行

7、then 必须返回一个promise 

> promise2 = promise1.then(onFulfilled, onRejected);

 - 如果onFulfilled 或 onRejected 返回了值x, 则执行Promise 解析流程[[Resolve]](promise2, x)
 - 如果onFulfilled 或 onRejected抛出了异常e, 则promise2应当以e为reason被拒绝
 - 如果 onFulfilled 不是一个函数且promise1已经fulfilled，则promise2必须以promise1的值fulfilled
 - 如果 OnReject 不是一个函数且promise1已经rejected, 则promise2必须以相同的reason被拒绝

###Promise解析过程
Promise解析过程 是以一个promise和一个值做为参数的抽象过程，可表示为[[Resolve]](promise, x). 过程如下；

 - 如果promise 和 x 指向相同的值, 使用 TypeError做为原因将promise拒绝
 - 如果 x 是一个promise, 采用其状态 [3.4]:
        1、如果x是pending状态，promise必须保持pending走到x fulfilled或rejected.
      2、如果x是fulfilled状态，将x的值用于fulfill promise.
      3、如果x是rejected状态, 将x的原因用于reject promise..
 - 如果x是一个对象或一个函数： 
      1、将 then 赋为 x.then. [3.5]
2、如果在取x.then值时抛出了异常，则以这个异常做为原因将promise拒绝。
3、如果 then 是一个函数， 以x为this调用then函数， 且第一个参数是resolvePromise，第二个参数是rejectPromise，且：
       
         - 当 resolvePromise 被以 y为参数调用, 执行 [[Resolve]](promise, y)
         - 当 rejectPromise 被以 r 为参数调用, 则以r为原因将promise拒绝
         - 如果 resolvePromise 和 rejectPromise 都被调用了，或者被调用了多次，则只第一次有效，后面的忽略
         - 如果在调用then时抛出了异常，则：
                 1、如果 resolvePromise 或 rejectPromise 已经被调用了，则忽略它
                 2、否则, 以e为reason将 promise 拒绝。

      4、如果 then不是一个函数，则 以x为值fulfill promise

 - 如果 x 不是对象也不是函数，则以x为值 fulfill promise。


 [promise原理部分参考这位大佬](https://mengera88.github.io/2017/05/18/Promise%E5%8E%9F%E7%90%86%E8%A7%A3%E6%9E%90/)   
[promiseA+英文原文地址](https://promisesaplus.com/)
