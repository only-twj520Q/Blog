## promise介绍

**Promise**是异步编程的一种解决方案。ES6提供原生支持，并为**Promise**提供了统一的 API。

从语法上说，**Promise**是一个JS的原生对象，从它可以获取异步操作的消息。

**Promise** 本身是一个构造函数，可以像下面这样构造一个**Promise**实例

```javascript
const p = new Promise((resolve, reject) => {
  resolve('success');
});
console.log(p);
```

打印结果如下

![promise结果](https://ws1.sinaimg.cn/large/006tKfTcly1g0whvqb1jdj3082024jrg.jpg)

**Promise**实例有两个私有属性，其中`[[PromiseStates]] ` 代表状态，一共有三种状态，[[PromiseValue]]代表结果。

- `[[PromiseStates]] ` 
  * pending：等待中
  * resolved：成功
  * rejected：失败
- `[[PromiseValue]]`



基本介绍就写这么多了，网上资料太多了。

下面主要是对一些特性的试验，基本以代码为主。



## 特点

下面是我自己使用下来总结的一些特点，不具有代表性。

### 特点一：立即执行性和异步性

所谓立即执行性，是指**Promise** 构造函数新建实例后立即执行，它也是同步代码。

所谓异步性，是指**promise** 的`then()`具有异步性，当执行到`.then()`部分，这部分会自动进入到**Promise**的异步事件队列，不会阻塞同步代码的执行。需要注意的是**promise**这种异步任务是属于一个微任务，它和以setTimeout为代表的宏任务在执行顺序上有一点区别。

```javascript
const p = new Promise((resolve, reject) => {
  console.log('promise内部')
  resolve('success');
});

p.then((value) => {
  console.log('then内部', value);
});

console.log('主程序');
```

输出结果

```javascript
promise内部
主程序
then内部 success
```

### 特点二：状态不可逆

```javascript
const p1 = new Promise((resolve, reject) => {
  resolve('p1 success-1');
  resolve('p1 success-2');
});

const p2 = new Promise((resolve, reject) => {
  resolve('p2 success');
  reject('p2 reject');
});

const p3 = new Promise((resolve, reject) => {
  reject('p3 reject');
  resolve('p3 success');
});

p1.then((value) => {
  console.log(value);
});

p2.then((value) => {
  console.log(value);
}, (err) => {
  console.log(err);
});

p3.then((value) => {
  console.log(value);
}, (err) => {
  console.log(err);
});
```

输出结果

```javascript
p1 success-1
p2 success
p3 reject
```

**Promise**一旦使用了resolve或者reject的时候，状态就不能再次变化，这就是**Promise**的不可逆性。

### 特点三：链式调用

```js
const p = new Promise(((resolve, reject) => {
  resolve(5);
}));


const p1 = p.then((value) => {
  console.log(value);
  return value * 10;
})

console.log(p1)

```

输出结果

![image-20190313151958689](/Users/apple/Library/Application Support/typora-user-images/image-20190313151958689.png)

可以看到`then`返回的就是一个**Promise**，而且是立即返回。这边之所以展开是resolved状态，是因为console.log的异步性，或者说在展开对象属性之前，它保留的是一个对象的引用。如果需要实时查看状态，加个断点即可。

也正是因为`then`返回的是一个新的**Promise**，所以可以通过链式调用`then` 方法。

`then`方法接收两个函数作为参数，第一个参数是**Promise**执行成功时的回调，第二个参数是**Promise**执行失败时的回调。两个函数只会有一个被调用，函数的返回值将被用作创建then返回的Promise对象。

#### then中的返回值

大概总结了下面四种情况

* `return`了一个同步的值，经过测试，不管返回null，undefined或者其他类型，`then`方法将返回一个resolved状态的**Promise**对象，**Promise**对象的值就是这个返回值。

  ```javascript
  const p = new Promise(((resolve, reject) => {
      resolve(1);
    }));
  
  
  p
    .then((value) => {
      return null;
    })
    .then((value) => {
      console.log(value);
      console.log('对象类型', {}.toString.call(value));
    })
  ```

  输出结果

* ```javascript
  null
  对象类型 [object Null]
  ```

* 没有`return`任何值

  输出结果

  ```javascript
  undefined
  对象类型 [object Undefined]
  ```

  

*  `return`了另一个 **Promise**，`then`方法将根据这个**Promise**的状态和值创建一个新的**Promise**对象返回。

  比如返回的是`Promise.resolved(value)`，则**Promise**的状态为resolved，值为value。

* ```javascript
  const p1 = new Promise(function (resolve, reject) {
    resolve(1);
  })
  
  const p2 = p1.then((value) => {
    return p1;
  })
  
  p2.then((value) => {
    console.log(value)
  })
  
  console.log(p1 === p2)
  ```

  输出结果

  ```javascript
  false
  1
  ```

* `throw` 一个同步异常，`then`方法将返回一个**Promise**，状态是rejected，值是`throw`的参数。



## 不常见的例子

#### 例子1

```javascript
const p1 = new Promise(function (resolve, reject) {
  setTimeout(() => {
    console.log('p1-promise');
    reject('这个是错误')
  }, 3000)
});
const p2 = new Promise(function (resolve, reject) {
  console.log('p2-promise');
  resolve(p1);
})

p2
  .then(result => console.log('then内部', result))
  .catch(error => console.log('catch内部', error))
```

输出结果

![](https://ws4.sinaimg.cn/large/006tKfTcly1g11bev0otug30c604eaaa.gif)

#### 例子2

```javascript
const p1 = new Promise(function (resolve, reject) {
  setTimeout(() => {
    console.log('p1-promise');
    resolve('这个是错误')
  }, 3000)
});
const p2 = new Promise(function (resolve, reject) {
  console.log('p2-promise');
  reject(p1);
})

p2
  .then(result => console.log('then内部', result))
  .catch(error => console.log('catch内部', error))
```

输出结果

![](https://ws4.sinaimg.cn/large/006tKfTcly1g11bjjs1ewg30c604et9p.gif)

#### 例子3

```javascript
const p1 = new Promise(function (resolve, reject) {
  setTimeout(() => {
    console.log('p1-promise');
    reject('这个是错误')
  }, 3000)
});
const p2 = new Promise(function (resolve, reject) {
  console.log('p2-promise');
  reject(p1);
})

p2
  .then(result => console.log('then内部', result))
  .catch(error => console.log('catch内部', error))
```

输出结果

![](https://ws4.sinaimg.cn/large/006tKfTcly1g11c12ksu1g30c604emxg.gif)



三个例子的结果各异，对于这样的现象，我个人的理解是这样。

p1和p2都是一个**Promise**。一方面，在p2代码中，如果使用reject方法，p2的状态变成了一个rejected，值就是p1，错误直接被catch住，而且error本身就是p1。三秒以后，如果p1的状态变成了resolved，可以添加`then`添加回调获取p1的值，p1的状态变成了rejected，则就抛出一个错误了。

另一方面，在p2代码中，如果使用resolve方法，p2的状态不会马上变成resolved，但是这个时候，它会依赖p1的结果，所以它会等到三秒以后p1的状态，如果p1变成了resolved，则会执行`then`中的代码，如果p1变成了rejected，则会执行`catch`中的代码，并且很重要的一点是它会对p1进行类似拆箱的操作，会直接拿到p1的结果，作为`then`或者`catch`回调的参数。

#### 例子4

```javascript
const p1 = Promise.resolve(1);
const p2 = Promise.resolve(p1);
const p3 = new Promise((resolve, reject) => {
  resolve(1);
});

const p4 = new Promise((resolve, reject) => {
  resolve(p1);
});

console.log(p1 === p2);
console.log(p1 === p3);
console.log(p3 === p4);
```

输出结果

```javascript
true
false
false
```

p1接收了一个普通值1，所以会返回一个resolved状态的**Promise**对象，值为1。
p2接收了一个**Promise**对象p1，会直接返回这个**Promise**对象。
p3和p4通过`new`方式创建了一个新的**Promise**对象，所以p3和p1,p4都不会相等。



## 错误

```javascript
const p = new Promise(((resolve, reject) => {
  reject('promise内部的错误');
}));
p
  .then((value) => {
    console.log(value);
    return value * 10;
  })
setTimeout(() => {console.log('主程序')}, 1000)
```



![](https://ws1.sinaimg.cn/large/006tKfTcly1g11877rvvtj30ns01d3yl.jpg)



如果**Promise**抛出一个错误，但是在`then`中没有第二个参数来捕获这个错误的话，就会在控制台打印错误信息，但是不会阻塞代码继续执行。

#### then中对于错误的处理

```javascript
const p = new Promise(((resolve, reject) => {
   resolve(1);
 }));
p
  .then((value) => {
    console.log('第一个then的第一个回调', value);
    return Promise.reject('then中错误啦')
  }, (value) => {
    console.log('第一个then的第二个回调', value);
  })
  .then((value) => {
    console.log('第二个then的第一个回调', value);
  }, (value) => {
    console.log('第二个then的第二个回调', value);
  })
  .then((value) => {
    console.log('第三个then的第一个回调', value);
  }, (value) => {
    console.log('第三个then的第二个回调', value);
  }).catch(error => console.log('catch内部', error))
```

输出结果

```javascript
第一个then的第一个回调 1
第二个then的第二个回调 then中错误啦
第三个then的第一个回调 undefined
```

因为在第二个`then`的参数中有对错误的处理，所以它可以捕获之前的error信息，并且自身的状态也变成了resolved。所以它能执行第三个`then`的回调。

如果不在`then`中进行对之前**Promise**的捕获，则一旦发生错误，会中断**Promise**链后面的代码，直接被`catch`到错误。

```javascript
const p = new Promise(((resolve, reject) => {
    // throw new Error('promise内部的错误')
  resolve(1);
}));
p
  .then((value) => {
    console.log('第一个then的第一个回调', value);
    return Promise.reject('then中错误啦')
  })
  .then((value) => {
    console.log('第二个then的第一个回调', value);
  })
  .then((value) => {
    console.log('第三个then的第一个回调', value);
  }).catch(error => console.log('catch内部', error))
```

输出结果

```javascript
第一个then的第一个回调 1
catch内部 then中错误啦
```

#### throw了一个error

如果在**Promise**中直接throw了一个错误的话，则会让**Promise**的状态变成rejected，不会直接**Promise**下面的代码。

```javascript
const p = new Promise(((resolve, reject) => {
  throw new Error('promise内部的错误')
  resolve(1);
}));
p
  .then((value) => {
    })
  .catch(error => console.log('catch内部', error))
```

输出结果

```javascript
catch内部 Error: promise内部的错误
    at Promise ((index):38)
    at new Promise (<anonymous>)
    at (index):37
```

#### try catch

在**Promise**代码外部使用`try catch`并不会得到想要的结果。因为`try catch`捕获的是同步代码中的错误。

关于更多捕获在异步代码中错误的问题，可以参考我的另一篇日记。



## 顺序执行

如何让Promise顺序执行，可以利用async和await。下面是一个很简单的demo。

```javascript
let p1 = () => new Promise(resolve => {
  setTimeout(() => resolve(1), 1000)
})
let p2 = () => new Promise(resolve => {
  setTimeout(() => resolve(2), 2000)
})
let p = [p1, p2];
async function queue() {
  for (let i=0; i<p.length; i++){
    let result = await p[i]();
    console.log(result)
  }
}
queue()
```



## 实际使用

简单封装一下ajax

```javascript
const fetchData = function(url, method, headerConfig) {
  return new Promise(function(resolve, reject){
    const handleChange = function() {
      if (this.readyState !== 4) {
        return;
      }
      if (this.status === 200 || this.status === 304) {
        resolve(this.response);
      } else {
        reject(new Error(this.statusText));
      }
    };
    const xhr = new XMLHttpRequest();
    xhr.open(method, url);
    xhr.onreadystatechange = handleChange;
    xhr.responseType = "json";
    
    if (typeof headerConfig === 'object' && headerConfig !== null) {
   	  for (let headerKey in headerConfig) {
	    xhr.setRequestHeader(headerKey, headerConfig[headerKey]);
      }
    }
    
    xhr.send();
  });
};
```



