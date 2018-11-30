# 从 TS 造 Promise 的过程建立前端安全感

你还在用 npm 的 star 数来选择依赖吗？在 **npm 安全性问题随时爆发** 的今天，作为前端开发者的我们应该具备源码阅读的能力，最好知道自己在用什么，这样在使用外部 `npm` 依赖时才有安全感不是么?

> 最近遇到一个细思极恐的问题，笔者最近老是收到防脱广告推送亦或是一些与笔者最近说出来的话相关的广告，以前只知道网上的信息流会被窃据，如今难不成语音也会监听窃取了？？？吓得我赶紧关掉了所有麦的权限。虽然我们不能自己造个手机，但也不能活得没有安全感。

拿 axios 这个我们常用的依赖来说，如果哪天被篡改了，那后果真不敢想象。即便这基本是不可能的，但切图仔中的精英不能就此停止追求进步的脚步。好在笔者前些天写了篇 [axios 重构经验分享](https://juejin.im/post/5bf7f1c0e51d455ed74f625c)，多少可以证明自己努力过。 😂

这还不够，翻了下最常用的依赖，其中 [es6-promise](https://github.com/stefanpenner/es6-promise) 特别惹眼 (源码比较晦涩难懂，说白了就是有点乱)，本来早就想深入了解 Promise ，于是毫不犹豫决定造它。由于笔者在过渡到 TypeScript ，所以本次开发依旧会采用 TypeScript 来敲。

这应该是笔者最后一次用 TypeScript 冠名分享文章，再见 🤞，我已经可以安全上路了。( 喊了那么多次，快上车，都没有多少人上车，那我就先走了。)

本文适合从零了解或者想重新深入研究 Promise 的读者，并且可以得到如下知识：

- Promise **重要**知识点

- Promise API 实现方法

- 前端何来安全感

笔者希望读者可以仅通过看**仅此一篇文章**就可以对 Promise 有个深刻的认知，**并且可以自己实现一个 Promise 类**。所以会从方方面面讲 Promise，内容可能会比较多，建议读者选读。

## 为什么要实现 Promise

Promise 出现在 Es6 ，如果 Es5 需要使用 Promise，通常需要用到 `Promise-polyfill` 。也就是说，我们要实现的是一个 polyfill。实现它不仅有助于我们深入了解 Promise 而且能减少使用中犯错的概率，以至于获得 Promise 最佳实践。

从另外一个角度来说重新实现某个依赖也是一种源码解读的方式，坚持这么做，意味着解读源码能力的提升。

## Promise

> Promise 表示一个异步操作的最终结果，与之进行交互的方式主要是 then 方法，该方法注册了两个回调函数，用于接收 promise 的终值或本 promise 不能执行的原因。

来看笔者用心画的一张 API 结构图 ( 看不清楚的可以进我的 [GitHub](https://github.com/leer0911/myPromise) 看，有大图和 xmind 源文件 )：

![](https://raw.githubusercontent.com/leer0911/myPromise/master/doc/img/1.png)

上图只是一个 **Promise 的 API 蓝图**，其实 `Promises/A+` 规范并不设计如何创建、解决和拒绝 promise，而是专注于提供一个通用的 then 方法。所以，Promise/A+ 规范的实现可以与那些不太规范但可用的实现能良好共存。如果大家都按规范来，那么就没有那么多兼容问题。(PS：包括 web 标准 ) 接着聊下 `Promises/A+`，看过的可以跳过。

## Promises/A+

所有 Promise 的实现都离不开 [Promises/A+](https://promisesaplus.com/) 规范，内容不多，建议大家可以过一遍。这边讲一些规范中重要的点

### 术语

![](https://raw.githubusercontent.com/leer0911/myPromise/master/doc/img/7.png)

- `Promise` 一个拥有 `then` 方法的对象或函数，其行为符合 `Promises/A+` 规范；

- `thenable` 一个定义了 `then` 方法的对象或函数，也可视作 “拥有 `then` 方法”

- `值（value）` 指**任何** JavaScript 的**合法值**（包括 undefined , thenable 和 promise）

- `异常（exception）` 使用 `throw` 语句抛出的一个值

- `据因（reason）` 表示一个 promise 的拒绝原因。

### Promise 的状态

> 一个 Promise 的当前状态**必须**为以下三种状态中的一种：等待态（Pending）、执行态（Fulfilled）和拒绝态（Rejected）。

- `等待态（Pending）`

  **处于等待态时，promise 需满足：`可以`迁移至执行态或拒绝态**

- `执行态（Fulfilled）`

  **处于执行态时，promise 需满足：`不能`迁移至其他任何状态，必须拥有一个`不可变`的`终值`**

- `拒绝态（Rejected）`

  **处于拒绝态时，promise 需满足：`不能`迁移至其他任何状态，必须拥有一个`不可变`的`据因`**

> 这里的不可变指的是恒等（即可用 `===` 判断相等），而不是意味着更深层次的不可变（ 指当 value 或 reason 不是[基本值](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Data_structures)时，只要求其引用地址相等，但属性值可被更改）。

### Then 方法

> 一个 promise 必须提供一个 then 方法以访问其当前值、终值和据因。

promise 的 then 方法接受两个参数：

```js
promise.then(onFulfilled, onRejected);
```

- `onFulfilled` 和 `onRejected` 都是可选参数。

- 如果 `onFulfilled` 是函数，当 promise **执行结束**后其必须被调用，其**第一个参数**为 promise 的**终值**，在 promise 执行结束前其**不可被调用**，其调用次数不可超过一次

- 如果 `onRejected` 是函数，当 promise 被**拒绝**执行后其必须被调用，其**第一个参数**为 promise 的**据因**，在 promise 被拒绝执行前其**不可被调用**，其调用次数不可超过一次

- `onFulfilled` 和 `onRejected` 只有在执行环境堆栈仅包含平台代码 ( 指的是引擎、环境以及 promise 的实施代码 )时才可被调用

- 实践中要确保 `onFulfilled` 和 `onRejected` 方法异步执行，且应该在 `then` 方法被调用的那一轮事件循环之后的新执行栈中执行。

- `onFulfilled` 和 `onRejected` 必须被作为函数调用即没有 this 值 ( 也就是说在 严格模式（strict） 中，函数 this 的值为 undefined ；在非严格模式中其为全局对象。)

- then 方法可以被同一个 promise 调用多次

- then 方法必须返回一个 promise 对象

### Then 参数 (函数) 返回值

希望读者可以认真看这部分的内容，对于理解 promise 的 `then` 方法有很大的帮助。

先来看下 promise 执行过程：

![](https://raw.githubusercontent.com/leer0911/myPromise/master/doc/img/8.png)

大致的过程是，promise 会从 `pending` 转为 `fulfilled` 或 `rejected` ，然后对应调用 `then` 方法参数的 `onFulfilled` 或 `onRejected` ，最终返回 `promise` 对象。

进一步理解，假定 有如下两个 promise：

```js
promise2 = promise1.then(onFulfilled, onRejected);
```

会有以下几种情况：

1. 如果 `onFulfilled` 或者 `onRejected` **抛出异常 e** ，则 promise2 **必须拒绝执行**，并返回 `拒因 e`

2. 如果 `onFulfilled` **不是函数** 且 promise1 成功执行， promise2 必须成功执行并返回 **相同的值**

3. 如果 `onRejected` **不是函数** 且 promise1 拒绝执行， promise2 必须拒绝执行并返回 **相同的据因**

希望进一步搞懂的，可以将下面代码拷贝到 chrome 控制台或其他可执行环境感受一下：

```js
// 通过改变 isResolve 来切换 promise1 的状态
const isResolve = true;

const promise1 = new Promise((resolve, reject) => {
  if (isResolve) {
    resolve('promise1 执行态');
  } else {
    reject('promise1 拒绝态');
  }
});

// 一、promise1 处于 resolve 以及 onFulfilled 抛出异常 的情况
// promise2 必须拒绝执行，并返回拒因
promise1
  .then(() => {
    throw '抛出异常!';
  })
  .then(
    value => {
      console.log(value);
    },
    reason => {
      console.log(reason);
    }
  );

// 二、promise1 处于 resolve 以及 onFulfilled 不是函数的情况
// promise2 必须成功执行并返回相同的值
promise1.then().then(value => {
  console.log(value);
});

// 三、promise1 处于 reject 以及 onRejected 不是函数的情况
// promise2 必须拒绝执行并返回拒因
promise1.then().then(
  () => {},
  reason => {
    console.log(reason);
  }
);

// 四、promise1 处于 resolve 以及 onFulfilled 有返回值时
promise1
  .then(value => {
    return value;
  })
  .then(value => {
    console.log(value);
  });
```

下面还有一个比较重要的情况，它关系到业务场景中传值问题：

4. `onFulfilled` 或者 `onRejected` 返回 **一个 JavaScript 合法值** 的情况

我们先来假定 then 方法内部有一个叫做 `[[Resolve]]` 的方法用于处理这种特殊情况，下面来具体了解下这个方法。

### `[[Resolve]]` 方法

一般像 `[[...]]` 这样的认为是内部实现，如 `[[Resolve]]`，该方法接受两个参数：

```js
[[Resolve]](promise, x);
```

对于 `x` 值，有以下几种情况：

- `x` 有 **then 方法** 且看上去像一个 `Promise`

- `x` 为对象或函数

- `x` 为 `Promise`

另外 promise 不能与 x 相等即 `promise !== x`,否则：

![](https://raw.githubusercontent.com/leer0911/myPromise/master/doc/img/same.png)

下面来看张图，大致了解下各情况的应对方式：

![](https://raw.githubusercontent.com/leer0911/myPromise/master/doc/img/resolve.png)

### `Promise/A+` 小结

至此，`Promise/A+` 需要 了解的就讲完了。主要包括了，术语以及 Then 方法的用法和相关注意事项。需要特别注意的是，then 方法中参数返回值的处理。接下来，我们在规范的基础上，用 TypeScript 来 实现 Promise。

**接下来，文中出现的规范特指 `Promise/A+` 规范**

## Promise 实现

Promise 本身是一个构造函数，即可以实现为类。接下来，主要围绕实现一个 Promise 类来讲。

先来看下一个标准 promise 对象具备的属性和 方法，做到心中有数。

Promise 提供的 API：

![](https://raw.githubusercontent.com/leer0911/myPromise/master/doc/img/3.png)

Promise 内部属性包括：

![](https://raw.githubusercontent.com/leer0911/myPromise/master/doc/img/2.png)

下面开始正式的实现部分

### 声明文件

在开始前，先来了解下，用 TypeScript 写 Promise 涉及的一些类型声明。可以看这个[声明文件](https://github.com/leer0911/myPromise/blob/master/index.d.ts)。

主要包括：

![](https://raw.githubusercontent.com/leer0911/myPromise/master/doc/img/4.png)

TypeScript 中声明文件用于外部模块，是 TypeScript 的核心部分。另外，从一个声明文件就可以大致了解所用模块暴露的 API 情况 (接受什么类型，或者会返回什么类型的数据)。这种事先设计好 API 是一个好的开发习惯，但实际开发中会比较难。

接着来看，Promise 类核心实现的开始部分，构造函数。

### 构造函数

规范提到 Promise 构造函数接受一个 `Resolver` 类型的函数作为第一个参数，该函数接受两个参数 resolve 和 reject，用于处理 promise 状态。

实现如下：

```ts
class Promise {
  // 内部属性
  private ['[[PromiseStatus]]']: PromiseStatus = 'pending';
  private ['[[PromiseValue]]']: any = undefined;

  subscribes: any[] = [];

  constructor(resolver: Resolver<R>) {
    this[PROMISE_ID] = id++;
    // resolver 必须为函数
    typeof resolver !== 'function' && resolverError();
    // 使用 Promise 构造函数，需要用 new 操作符
    this instanceof Promise ? this.init(resolver) : constructorError();
  }

  private init(resolver: Resolver<R>) {
    try {
      // 传入两个参数并获取用户传入的终值或拒因。
      resolver(
        value => {
          this.mockResolve(value);
        },
        reason => {
          this.mockReject(reason);
        }
      );
    } catch (e) {
      this.mockReject(e);
    }
    return null;
  }

  private mockResolve() {
    // TODO
  }
  private mockReject() {
    // TODO
  }
}
```

### [[Resolve]] 实现

通过前面规范部分，我们了解到 `[[Resolve]]` 属于内部实现，用于处理 then 参数的返回值。也就是这里即将要实现的名为 `mockResolve` 的方法。

![](https://raw.githubusercontent.com/leer0911/myPromise/master/doc/img/5.png)

根据规范内容可以得知，`mockResolve` 方法接受的 value 可能为 Promise，thenable，以及其他有效 JavaScript 值。

```ts
private mockResolve(value: any) {
  // 规范提到 resolve 不能传入当前返回的 promise
  // 即 `[[Resolve]](promise,x)` 中 promise ！== x
  if (value === this) {
    this.mockReject(resolveSelfError);
    return;
  }
  // 非对象和函数，直接处理
  if (!isObjectORFunction(value)) {
    this.fulfill(value);
    return;
  }
  // 处理一些像 promise 的对象或函数，即 thenable
  this.handleLikeThenable(value, this.getThen(value));
}
```

#### 处理 Thenable 对象

重点看下 `handleLikeThenable` 实现，可结合前面规范部分提及 Thenable 的几种情况来分析：

```ts
  private handleLikeThenable(value: any, then: any) {
    // 处理 "真实" promise 对象
    if (this.isThenable(value, then)) {
      this.handleOwnThenable(value);
      return;
    }
    // 获取 then 值失败且抛出异常，则以此异常为拒因 reject promise
    if (then === TRY_CATCH_ERROR) {
      this.mockReject(TRY_CATCH_ERROR.error);
      TRY_CATCH_ERROR.error = null;
      return;
    }
    // 如果 then 是函数，则检验 then 方法的合法性
    if (isFunction(then)) {
      this.handleForeignThenable(value, then);
      return;
    }
    // 非 Thenable ，则将该终植直接交由 fulfill 处理
    this.fulfill(value);
  }
```

#### 处理 Thenable 中 Then 为函数的情况

规范提及：

> 如果 then 是函数，将 x 作为函数的作用域 this 调用之。传递两个回调函数作为参数，第一个参数叫做 resolvePromise ，第二个参数叫做 rejectPromise。

此时，`handleForeignThenable` 就是用来检验 then 方法的。

实现如下：

```ts
  private tryThen(then, thenable, resolvePromise, rejectPromise) {
    try {
      then.call(thenable, resolvePromise, rejectPromise);
    } catch (e) {
      return e;
    }
  }
  private handleForeignThenable(thenable: any, then: any) {
    this.asap(() => {
      // 如果 resolvePromise 和 rejectPromise 均被调用，
      // 或者被同一参数调用了多次，则优先采用首次调用并忽略剩下的调用
      // 此处 sealed (稳定否)，用于处理上诉逻辑
      let sealed = false;
      const error = this.tryThen(
        then,
        thenable,
        value => {
          if (sealed) {
            return;
          }
          sealed = true;
          if (thenable !== value) {
            this.mockResolve(value);
          } else {
            this.fulfill(value);
          }
        },
        reason => {
          if (sealed) {
            return;
          }
          sealed = true;
          this.mockReject(reason);
        }
      );

      if (!sealed && error) {
        sealed = true;
        this.mockReject(error);
      }
    });
  }
```

#### fulfill 实现

来看 `[[Resolve]]` 中最后一步，`fulfill` 实现：

```ts
  private fulfill(value: any) {
    this['[[PromiseStatus]]'] = 'fulfilled';
    this['[[PromiseValue]]'] = value;

    // 用于处理异步情况
    if (this.subscribes.length !== 0) {
      this.asap(this.publish);
    }
  }
```

看到这里，大家可能有关注到很多方法都带有 `private` 修饰符，在 TypeScript 中

> private 修饰的属性或方法是私有的，不能在声明它的类的外部访问

规范提到过，`PromiseStatus` 属性不能由外部更改，也就是 promise 状态只能改变一次，而且只能从内部改变，也就是这里私有方法 `fulfill` 的职责所在。

#### `[[Resolve]]` 小结

至此，一个内部 `[[Resolve]]` 就实现了。我们回顾一下，`[[Resolve]]` 用于处理以下情况

```ts
// 实例化构造函数，传入 resolve 的情况
const promise = Promise(resolve => {
  const value: any;
  resolve(value);
});
```

以及

```ts
// then 方法中有 返回值的情况
promise.then(
  () => {
    const value: any;
    return value;
  },
  () => {
    const reason: any;
    return reason;
  }
);
```

对于终值 `value` 有多种情况，在处理 Thenable 的时候，请参考规范来实现。`promise` 除了 `resolve` 的还有 `reject`，但这部分内容比较简单，我们会放到后面再讲解。先来看与 `resolve` 密不可分的 `then` 方法实现。这也是 `promise` 的核心方法。

### Then 方法实现

通过前面的实现，我们已经可以从 Promise 构造函数来改变内部 `[[PromiseStatus]]` 状态以及内部 `[[PromiseValue]]` 值，并且对于多种 value 值我们都有做相应的兼容处理。接下来，是时候把这些值交由 then 方法中的第一个参数 `onFulfilled` 处理了。

在讲解之前先来看下这种情况：

```ts
promise2 = promise1.then(onFulfilled, onRejected);
```

使用 `promise1` 的 `then` 方法后，会返回一个 promise 对象 `promise2` 实现如下：

```ts
class Promise {
  then(onFulfilled?, onRejected?) {
    // 对应上述的 promise1
    const parent: any = this;
    // 对应上述的 promise2
    const child = new parent.constructor(() => {});

    // 根据 promise 的状态选择处理方式
    const state = PROMISE_STATUS[this['[[PromiseStatus]]']];
    if (state) {
      // promise 各状态对应枚举值 'pending' 对应 0 ，'fulfilled' 对应 1，'rejected' 对应 2
      const callback = arguments[state - 1];
      this.asap(() =>
        this.invokeCallback(
          this['[[PromiseStatus]]'],
          child,
          callback,
          this['[[PromiseValue]]']
        )
      );
    } else {
      // 调用 then 方法的 promise 处于 pending 状态的处理逻辑，一般为异步情况。
      this.subscribe(parent, child, onFulfilled, onRejected);
    }

    // 返回一个 promise 对象
    return child;
  }
}
```

这里比较惹眼的 `asap` 后续会单独讲。先来理顺一下逻辑，`then` 方法接受两个参数，由当前 `promise` 的状态决定调用 `onFulfilled` 还是 `onRejected`。

现在大家肯定很关心 then 方法里的代码是如何被执行的，比如下面的 `console.log`：

```ts
promise.then(value => {
  console.log(value);
});
```

接下来看与之相关的 `invokeCallback` 方法

### then 方法中回调处理

then 方法中的 `onFulfilled` 和 `onRejected` 都是可选参数，开始进一步讲解前，建议大家先了解规范中提及的两个参数的特性。

现在来讲解 `invokeCallback` 接受的参数及其含义：

- `settled` (**稳定状态**)，promise 处于**非 pending** 状态则称之为 settled，settled 的值可以为 `fulfilled` 或 `rejected`

- `child` 即将返回的 promise 对象

- `callback` 根据 `settled` 选择的 `onFulfilled` 或 `onRejected` 回调函数

- `detail` 当前调用 then 方法 promise 的 value(终值) 或 reason(拒因)

**注意这里的 `settled` 和 `detail`，`settled` 用于指 `fulfilled` 或 `rejected`， detail 用于指 `value` 或 `reason` 这都是有含义的**

知道这些之后，就只需要参考规范建议实现的方式进行处理相应：

```ts
  private invokeCallback(settled, child, callback, detail) {
    // 1、是否有 callback 的对应逻辑处理
    // 2、回调函数执行后是否会抛出异常，即相应处理
    // 3、返回值不能为自己的逻辑处理
    // 4、promise 结束(执行结束或被拒绝)前不能执行回调的逻辑处理
    // ...
  }
```

需要处理的逻辑已给出，剩下的实现方式读者可自行实现或看本项目源码实现。建议所有实现都应参考规范来落实，在实现过程中可能会出现遗漏或错误处理的情况。(**ps：考验一个依赖健壮性的时候到了**)

截至目前，都是处理同步的情况。promise 号称处理异步的大神，怎么能少得了相应实现。处理异步的方式有回调和订阅发布模式，我们实现 promise 就是为了解决回调地狱的，所以这里当然选择使用 订阅发布模式。

### then 异步处理

这里要处理的情况是指：当调用 then 方法的 promise 处于 pending 状态时。

那什么时候会出现这种情况呢？来看下这段代码：

```ts
const promise = new Promise(resolve => {
  setTimeout(() => {
    resolve(1);
  }, 1000);
});

promise.then(value => {
  console.log(value);
});
```

代码编写到这里，如果出现这种情况。我们的 promise 其实是不能正常工作的。由于 `setTimeout` 是一个异常操作，当内部 then 方法按同步执行的时候，resolve 根本没执行，也就是说调用 then 方法的 promise 的 `[[PromiseStatus]]` 目前还处于 'pending'，`[[PromiseValue]]` 目前为 undefined，此时添加对 `pending` 状态下的回调是没有任何意义的 ，另外规范提及 then 方法的回调必须处于 settled( **之前有讲过** ) 才会调用相应回调。

或者我们不用考虑是不是异步造成的，只需要明确一件事。存在这么一种情况，调用 then 方法的 promise 状态可能为`pending`。

这时就必须有一套机制来处理这种情况，对应代码实现就是：

```ts
  private subscribe(parent, child, onFulfillment, onRejection) {
    let {
      subscribes,
      subscribes: { length }
    } = parent;
    subscribes[length] = child;
    subscribes[length + PROMISE_STATUS.fulfilled] = onFulfillment;
    subscribes[length + PROMISE_STATUS.rejected] = onRejection;
    if (length === 0 && PROMISE_STATUS[parent['[[PromiseStatus]]']]) {
      this.asap(this.publish);
    }
  }
```

`subscribe` 接受 4 个参数 `parent`，`child`,`onFulfillment`,`onRejection`

- `parent` 为当前调用 then 方法的 promise 对象
- `child` 为即将由 then 方法返回的 promise 对象
- `onFulfillment` then 方法的第一个参数
- `onFulfillment` then 方法的第二个参数

用一个数组来存储 `subscribe` ，主要保存即将返回的 promise 对象及相应的 `onFulfillment` 和 `onRejection` 回调函数。

满足 `subscribe` 是新增的情况及调用 then 方法的 promise 对象的 `[[PromiseStatus]]` 值不为 'pending'，则调用 `publish` 方法。也就是说异步的情况下，不会调用该 `publish` 方法。
这么看来这个 `publish` 是跟执行回调相关的方法。

那异步的情况，什么时候会触发回调呢?可以回顾之前讲解过的 `fulfill` 方法:

```ts
  private fulfill(value: any) {
    this['[[PromiseStatus]]'] = 'fulfilled';
    this['[[PromiseValue]]'] = value;

    // 用于处理异步情况
    if (this.subscribes.length !== 0) {
      this.asap(this.publish);
    }
  }
```

当满足 `this.subscribes.length !== 0` 时会触发 `publish`。也就是说当异步函数执行完成后调用 `resolve` 方法时会有这么一个是否调用 `subscribes` 里面的回调函数的判断。

这样就保证了 `then` 方法里回调函数只会在异步函数执行完成后触发。接着来看下与之相关的 `publish` 方法

### publish 方法

首先明确，`publish` 是发布，是通过 `invokeCallback` 来调用回调函数的。在本项目中，只与 `subscribes` 有关。直接来看下代码：

```ts
  private publish() {
    const subscribes = this.subscribes;
    const state = this['[[PromiseStatus]]'];
    const settled = PROMISE_STATUS[state];
    const result = this['[[PromiseValue]]'];
    if (subscribes.length === 0) {
      return;
    }
    for (let i = 0; i < subscribes.length; i += 3) {
      // 即将返回的 promise 对象
      const item = subscribes[i];
      const callback = subscribes[i + settled];
      if (item) {
        this.invokeCallback(state, item, callback, result);
      } else {
        callback(result);
      }
    }
    this.subscribes.length = 0;
  }
```

### then 方法小结

![](https://raw.githubusercontent.com/leer0911/myPromise/master/doc/img/6.png)

到这我们就实现了 promise 中的 then 方法，也就意味着目前实现的 promise 已经具备处理异步数据流的能力了。then 方法的实现离不开规范的指引，只要参考规范对 then 方法的描述，其余就只是逻辑处理了。

至此 promise 的核心功能已经讲完了，也就是内部 `[[Resolve]]` 和 `then` 方法。接下来快速看下其余 API。

## 语法糖 API 实现

catch 和 finally 都属于语法糖

- `catch` 属于 `this.then(null, onRejection)`

- `finally` 属于 `this.then(callback, callback);`

promise 还提供了 `resolve`，`reject`，`all`，`race` 的静态方法，为了方便链式调用，上述方法均会返回一个新的 promise 对象用于链式调用。

之前主要讲 `resolve`，现在来看下

### reject

`reject` 处理方式跟 `resolve` 略微不同的是它不用处理 thenable 的情况，规则提及 reject 的值 reason 建议为 error 实例代码实现如下：

```ts
  private mockReject(reason: any) {
    this['[[PromiseStatus]]'] = 'rejected';
    this['[[PromiseValue]]'] = reason;
    this.asap(this.publish);
  }
  static reject(reason: any) {
    let Constructor = this;
    let promise = new Constructor(() => {});
    promise.mockReject(reason);
    return promise;
  }
  private mockReject(reason: any) {
    this['[[PromiseStatus]]'] = 'rejected';
    this['[[PromiseValue]]'] = reason;
    this.asap(this.publish);
  }
```

### all & race

在前面 API 的基础上，扩展出 all 和 race 并不难。先来看两者的作用：

- all 用于处理一组 promise，当满足所有 promise 都 resolve 或 某个 promise reject 时返回对应由 value 组成的数组或 reject 的 reason

- race 赛跑的意思，也就是在一组 promise 中看谁执行的快，则用这个 promise 的结果

对应实现代码：

```ts
// all
let result = [];
let num = 0;
return new this((resolve, reject) => {
  entries.forEach(item => {
    this.resolve(item).then(data => {
      result.push(data);
      num++;
      if (num === entries.length) {
        resolve(result);
      }
    }, reject);
  });
});

// race
return new this((resolve, reject) => {
  let length = entries.length;
  for (let i = 0; i < length; i++) {
    this.resolve(entries[i]).then(resolve, reject);
  }
});
```

### 时序组合

如果有需要按顺序执行的异步函数，可以采用如下方式：

```ts
[func1, func2].reduce((p, f) => p.then(f), Promise.resolve());
```

在 ES7 中时序组合可以通过使用 `async/await` 实现

```ts
for (let f of [func1, func2]) {
  await f();
}
```

更多使用方式可参考 [这篇](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Guide/Using_promises)

## 知识补充

Promise 的出现是为了更好地处理异步数据流，或者常说的回调地狱。这里说的回调，是在异步的情况下，如果非异步，则一般不需要用到回调函数。下面来了解下在实现 Promise 过程中出现的几个概念：

- 回调函数

- 异步 & 同步

- EventLoop

- asap

### 回调函数

回调函数的英文定义：

> A callback is a function that is passed as an argument to another function and is executed after its parent function has completed。

字面上的理解，回调函数就是一个参数，将这个函数作为参数传到另一个函数里面，当那个函数执行完之后，再执行传进去的这个函数。这个过程就叫做回调。

在 JavaScript 中，回调函数具体的定义为： 函数 A 作为参数(函数引用)传递到另一个函数 B 中，并且这个函数 B 执行函数 A。我们就说函数 A 叫做回调函数。如果没有名称(函数表达式)，就叫做匿名回调函数。

首先需要声明，回调函数只是一种实现，并不是异步模式特有的实现。回调函数同样可以运用到同步（阻塞）的场景下以及其他一些场景。

**回调函数需要和异步函数区分开来。**

#### 异步函数 & 同步函数

- 如果在函数返回的时候，调用者就能够得到预期结果，这个就是同步函数。

- 如果在函数返回的时候，调用者还不能够得到预期结果，而是需要在将来通过一定的手段得到，那么这个函数就是异步的。

那什么是异步，同步呢?

### 异步 & 同步

首先要明确，Javascript 语言的执行环境是"单线程"（single thread）。所谓"单线程"，就是指一次只能完成一件任务。如果有多个任务，就必须排队，前面一个任务完成，再执行后面一个任务，以此类推。

这种模式会造成一个阻塞问题，为了解决这个问题，Javascript 语言将任务的执行模式分成两种：同步（Synchronous）和异步（Asynchronous）。

但需要注意的是：**异步机制是浏览器的两个或以上常驻线程共同完成的**，Javascript 的单线程和异步更多的应该是**属于浏览器的行为**。也就是说 Javascript 本身是单线程的，**并没有异步的特性**。

由于 Javascript 的运用场景是浏览器，浏览器本身是典型的 GUI 工作线程，GUI 工作线程在绝大多数系统中都实现为事件处理，避免阻塞交互，因此产生了 Javascript 异步基因。所有涉及到异步的方法和函数都是由浏览器的另一个线程去执行的。

#### 浏览器中的线程

- **浏览器事件触发线程** 当一个事件被触发时该线程会把事件添加到待处理队列的队尾，等待 JavaScript 引擎的处理。这些事件可以是当前执行的代码块如：**定时任务**、也可来自浏览器内核的其他线程如**鼠标点击**、**AJAX 异步请求**等，但由于 JavaScript 是单线程，所有这些事件都得排队等待 JavaScript 引擎处理；

- **定时触发器线程** 浏览器定时计数器**并不是**由 JavaScript 引擎计数的, 因为 JavaScript 引擎是单线程的, 如果处于阻塞线程状态就会影响记计时的准确, 因此通过单独线程来计时并触发定时是更为合理的方案；

- **异步 HTTP 请求线程** XMLHttpRequest 在连接后是通过浏览器新开一个线程请求，将检测到状态变更时，**如果设置有回调函数，异步线程就产生状态变更事件放到 JavaScript 引擎的处理队列中** 等待处理；

通过以上了解，可以知道其实 JavaScript **是通过 JS 引擎线程与浏览器中其他线程交互协作实现异步**。

但是回调函数具体何时加入到 JS 引擎线程中执行？执行顺序是怎么样的？

接下来了解一下，与之相关的 EventLoop 机制

### EventLoop

先来看一些概念，Stack，Heap，Queue，直接上图：

![](https://raw.githubusercontent.com/leer0911/myPromise/master/doc/img/q.png)

- 那些不需要回调函数的操作都可归为 Stack 这一类

- Heap 用来存储声明的变量、对象

- 一旦某个异步任务有了响应就会被推入 Queue 队列中

一个大致的流程如下：

JS 引擎线程用来执行栈中的**同步任务**，当所有同步任务**执行完毕后，栈被清空**，然后读取消息队列中的一个待处理任务，并把**相关回调函数压入栈中**，单线程开始执行新的同步任务。

JS 引擎线程从消息队列中读取任务是不断循环的，**每次栈被清空后**，都会在消息队列中读取新的任务，如果没有新的任务，就会等待，直到有新的任务，这就叫**事件循环**。

![](https://raw.githubusercontent.com/leer0911/myPromise/master/doc/img/event.png)

这张图不知道谁画的，真的是非常棒!先借来描述下 AJAX 大概的流程：

AJAX 请求属于非常耗时的异步操作，浏览器有提供专门的线程来处理它。当主线程上有调用 AJAX 的代码时，会触发异步任务。执行这个异步任务的工作交由 AJAX 线程，**主线程并没有等待这个异步操作的结果**而是接着执行。假设主线程代码在某个时刻执行完毕，也就是此时的 Stack 为空。而在早些时刻，异步任务执行完成后已经将消息存放在 Queue 中，以便 Stack 为空时从中拿去一个回调函数来执行。

这背后运作的就叫 `EventLoop`，有了上面的大致了解。下面来重新了解下 EventLoop：

> 异步背后的“靠山”就是 event loops。这里的异步准确的说应该叫浏览器的 event loops 或者说是 javaScript 运行环境的 event loops，因为 ECMAScript 中没有 event loops，event loops 是在 HTML Standard 定义的。

event loop 翻译出来就是事件循环，可以理解为实现异步的一种方式，我们来看看 event loop 在 HTML Standard 中的定义章节：

> 为了协调事件，用户交互，脚本，渲染，网络等，用户代理必须使用本节所述的 event loop。

事件，用户交互，脚本，渲染，网络这些都是我们所熟悉的东西，他们都是由 event loop 协调的。触发一个 click 事件，进行一次 ajax 请求，背后都有 event loop 在运作。

#### task

> 一个 event loop 有一个或者多个 task 队列。每一个 task 都来源于指定的任务源，比如可以为鼠标、键盘事件提供一个 task 队列，其他事件又是一个单独的队列。可以为鼠标、键盘事件分配更多的时间，保证交互的流畅。

task 也被称为**macrotask**，task 队列还是比较好理解的，就是一个先进先出的队列，由指定的任务源去提供任务。

task 任务源:

- setTimeout
- setInterval
- setImmediate
- I/O
- UI rendering

#### microtask

每一个 event loop 都有一个 microtask 队列，一个 microtask 会被排进 microtask 队列而不是 task 队列。microtask 队列和 task 队列有些相似，都是先进先出的队列，由指定的任务源去提供任务，不同的是一个 event loop 里只有一个 microtask 队列。

通常认为是 microtask 任务源有：

- process.nextTick
- promises
- Object.observe
- MutationObserver

#### 一道关于 EventLoop 的面试题

```ts
console.log('start');

setTimeout(function() {
  console.log('setTimeout');
}, 0);

Promise.resolve()
  .then(function() {
    console.log('promise1');
  })
  .then(function() {
    console.log('promise2');
  });

console.log('end');

// start
// end
// promise1
// promise2
// setTimeout
```

> 上面的顺序是在 chrome 运行得出的，有趣的是在 safari 9.1.2 中测试，promise1 promise2 会在 setTimeout 的后边，而在 safari 10.0.1 中得到了和 chrome 一样的结果。promise 在不同浏览器的差异正源于有的浏览器将 then 放入了 macro-task 队列，有的放入了 micro-task 队列。

#### EventLoop 小结

event loop 涉及到的东西很多，但本文怕重点偏离，这里只是提及可能与 promise 相关的知识点。如果想深入了解的同学，建议看完[这篇](https://github.com/aooy/blog/issues/5)必定会受益匪浅。

## 什么是 asap

`as soon as possible` 英文的缩写，在 promise 中起到的作用是尽快响应变化。

在 **Promises/A+规范** 的 **Notes 3.1** 中提及了 promise 的 then 方法可以采用“宏任务（macro-task）”机制或者“微任务（micro-task）”机制来实现。

本项目，采用 `macro-task` 机制

```ts
  private asap(callback) {
    setTimeout(() => {
      callback.call(this);
    }, 1);
  }
```

或者 MutationObserver 模式：

```ts
function flush() {
...
}
function useMutationObserver() {
  var iterations = 0;
  var observer = new MutationObserver(flush);
  var node = document.createTextNode('');
  observer.observe(node, { characterData: true });

  return function () {
    node.data = iterations = ++iterations % 2;
  };
}
```

初次看这个 `useMutationObserver` `函数总会很有疑惑，MutationObserver` 不是用来观察 dom 的变化的吗，这样凭空造出一个节点来反复修改它的内容，来触发观察的回调函数有何意义？

答案就是使用 `Mutation` 事件可以异步执行操作（例子中的 flush 函数），一是可以尽快响应变化，二是可以去除重复的计算。

或者 node 环境下：

```ts
function useNextTick() {
  return () => process.nextTick(flush);
}
```

## 前端安全感

既然标题提到了 `前端安全感`，当然要说点什么。现如今前端处于模块化开发的鼎盛时期，面对 [npm](https://www.npmjs.com/) 那成千上万的 "名牌包包" 很多像我这样的小白难免会迷失自我，渐渐没有了安全感。

是做拿来主义，还是像笔者一样偶尔研究下底层原理，看看别人的代码，自己撸一遍来感悟其中的真谛来提升安全感。

做前端也有段时日了，被业务埋没的那些年，真是感叹!回想曾经差点去做设计的我终于要入门前端了，心中难免想去搓一顿，那就写到这吧!

## 结语

写的有点多，好累。希望对大家有所帮助。

[源码](https://github.com/leer0911/myPromise)

## 参考

[Promises/A+](https://promisesaplus.com/)
[MDN Promises](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Promise)
[Promise 使用](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Guide/Using_promises)
[Event loop](https://github.com/aooy/blog/issues/5)
