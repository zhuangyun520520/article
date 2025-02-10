
### 1 前言

记录一下这几日学async/await的李姐，可能有问题，请谅解。

### 2 async/await的简介
`async/await` 是现代 JavaScript 中处理异步编程的重要工具，提升了代码的可读性和可维护性。通过结合使用 `async` 函数和 `await` 关键字，开发者可以更方便地编写异步代码，简化了处理复杂异步逻辑的流程。

#### 2.1 作用：

-   **以同步的方式去执行异步任务，简化异步流程，避免回调地狱**。
-   **一致和简洁的错误处理**，使得 `try...catch` 结构能够捕获异步代码中的错误。
-   **控制异步任务的执行顺序**，适用于有依赖关系的异步任务。
-   **提高代码可读性和可维护性**，让异步代码写起来像同步代码一样清晰直观。

#### 2.2 一些语法规则：

-   `async` 是 `function` 的一个前缀，被修饰的函数被调用时，会返回一个Promise对象，如果函数没有return，也会返回一个Promise对象，不过都都对象的PromiseResult为undefined。
-   `async` 函数返回的Promise对象的状态。未捕获的异常、await的Promise被拒绝、操作出错、直接返回一个reject的Promise。这些情况下返回的Promise对象都是rejected状态。
-   await`后面跟的是一个`Promise`对象，如果不是，则会包裹一层`Promise.resolve()。await的结果都是promise对象里面的PromiseState

### 3 async/await的理解

我在学习async/await的时候，看见有些人说它是基于Promise和Generator函数的，也有人说它是基于Promise和浏览器的事件循环的。

有些人有一些困惑，包括我自己。下面就说一下我的理解。

#### 3.1 async/await跟Promise的关系

-   执行async函数，返回的是Promise对象。
-   await之后的代码作为一个整体(不包括await同行的代码)在Promise的then方法里执行。
-   *try...catch可捕获异常，代替了Promise 的catch。

#### 3.2 异步的本质

在 JavaScript 中，异步的本质是**通过事件循环（Event Loop）机制来处理非阻塞的操作**。JavaScript 是单线程的，这意味着它一次只能执行一个任务，但通过异步机制，它能够处理多个任务而不会阻塞主线程，使程序在执行耗时操作（例如网络请求或文件 I/O）时，能够继续响应用户的操作。

#### 3.3 async/await的执行

下面从代码的执行来看一看async/await是怎么利用Event Loop的

```
async function async1() {
  console.log('async1 start');    //2
  await async2();
  console.log('async2 end');   //5
  await async3();
  console.log('async3 end');   //7
  console.log('async1 end');   //8
}

async function async2() {
  console.log('async2 start');   //3
}
async function async3() {
  console.log('async3 start');   //6
}

console.log('js start');    //1
async1();
console.log('js end');     //4

    
结果：
js start
async1 start
async2 start
js end
async2 end
async3 start
async3 end
async1 end
```

整个代码的执行

-   先执行同步代码。（async修饰的函数里面也是同步执行的，包括await之后的async2,也是同步执行）
-   在await async2();执行完之后，返回的是一个resolve的Promise对象，在这个promise.then()方法里面以微任务去调用之后的4行代码。
-   此时还需要先执行同步代码console.log('js end');
-   然后一次类推去执行。

#### 3.4 协程

从上面可以看出，async/await他们有一个特征是：**能够暂停执行并能恢复执行**

协程在 JavaScript 中的主要作用是**让代码能够暂停和恢复执行，从而更优雅地处理异步操作和复杂的控制流**。JavaScript 的协程特性，虽然不像一些编程语言那样原生支持，但可以通过 **`Generator`** 和 **`async/await`** 实现。这些协程实现让 JavaScript 在异步处理、资源管理和复杂任务协调方面变得更加简洁和高效。

#### 3.5 跟Generator的关系

Generator其实就是JS在语法层面对协程的支持。协程就是主程序和子协程直接控制权的切换，并伴随通信的过程，那么，从generator语法的角度来讲，yield，next就是通信接口，next是主协程向子协程通信，而yield就是子协程向主协程通信。

在执行到await时，会暂停执行后续的函数，看await得到的promise对象的状态是什么，再继续执行。这时候就需要协程去控制暂停和执行。
所以在手写async/await时，会基于Generator函数去实现。

### 4 手写async/await


```
function asyncToGenerator(generatorFunc) {
  return function () {
    const gen = generatorFunc.apply(this, arguments); // 创建生成器实例
    return new Promise((resolve, reject) => { // 返回一个新的 Promise
      function step(key, arg) { // 定义 step 函数，负责执行生成器的步骤
        let generatorResult;
        try {
          generatorResult = gen[key](arg); // 调用生成器的 next 或 throw 方法
        } catch (error) {
          return reject(error); // 捕获同步错误并拒绝 Promise
        }
        const { value, done } = generatorResult; // 解构出 value 和 done

        if (done) {
          return resolve(value); // 如果生成器完成，解析 Promise
        } else {
          return Promise.resolve(value).then(val => step("next", val), val => step("throw", err));
        }
      }
      step("next"); // 开始执行生成器
    });
  }
}


// 1. 定义生成器函数
function* myAsyncGenerator() {
  const result1 = yield new Promise(resolve => {
    setTimeout(() => {
      console.log("任务 1 完成");
      resolve("任务 1 的结果");
    }, 1000);
  });

  console.log(result1); // 输出: 任务 1 的结果

  const result2 = yield new Promise(resolve => {
    setTimeout(() => {
      console.log("任务 2 完成");
      resolve("任务 2 的结果");
    }, 1000);
  });

  console.log(result2); // 输出: 任务 2 的结果

  return "所有任务完成"; // 返回最终结果
}

// 2. 使用 asyncToGenerator 转换生成器函数
const asyncFunction = asyncToGenerator(myAsyncGenerator);

// 3. 调用转换后的函数并处理 Promise
asyncFunction()
  .then(result => {
    console.log(result); // 输出: 所有任务完成
  })
  .catch(error => {
    console.error("出现错误:", error);
  });
```

对应的async/await的实际代码如下：

```
async function myAsyncFunction() {
      const result1 = await new Promise(resolve => {
        setTimeout(() => {
          console.log("任务 1 完成");
          resolve("任务 1 的结果");
        }, 1000);
      });

      console.log(result1); // 输出: 任务 1 的结果

      const result2 = await new Promise(resolve => {
        setTimeout(() => {
          console.log("任务 2 完成");
          resolve("任务 2 的结果");
        }, 1000);
      });

      console.log(result2); // 输出: 任务 2 的结果

      return "所有任务完成"; // 返回最终结果
    }

    // 2. 调用异步函数并处理结果
    myAsyncFunction()
      .then(result => {
        console.log(result); // 输出: 所有任务完成
      })
      .catch(error => {
        console.error("出现错误:", error);
      });
```

### 5 结束语
以上是个人浅浅的理解，感谢批评指正。