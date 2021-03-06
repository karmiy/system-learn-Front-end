## async

ES8新增async函数，使得异步操作变得更加方便

async函数是Generator函数的语法糖

什么是语法糖？

没有给计算机语言添加新功能（新增API），只是在原本的功能上添加新的东西，更高阶的功能，使之更为实用、易读

### async、Generator区别

- Generator执行必须靠执行器（需要一直调用next），而async自带执行器

- async与await，相比Generator的星号*与yield，语义更为清楚。async表示函数里有异步操作，await表示紧跟其后的表达式是异步的，需要等待结果

- 正常情况下await命令后面是一个Promise对象，如果不是，会被转换为立即resolve的Promise对象（即被执行Promise.resolve）

- 返回值是Promise，相比Generator返回的是Iterator对象更为的方便，可以直接.then下一步操作

```js
// Generator示例
const fn = time => {
    setTimeout(() => {
        console.log('fn finish');
        iter.next();
    }, time);
}
function * g() {
    console.log('start');
    yield fn(3000);
    yield fn(1000);
    yield fn(2000);
    console.log('end');
}
const iter = g();
iter.next();

// async示例
const fn = time => {
    return new Promise(resolve => {
        setTimeout(() => {
            console.log('fn finish');
            resolve();
        }, time);
    })
}
(async () => {
    console.log('start');
    await fn(3000);
    await fn(1000);
    await fn(2000);
    console.log('end');
})();
    
```

### 用法

async函数有多种使用形式

```js
// 函数声明
async function f() {...}

// 函数表达式
let f = async function() {...}

// 对象的方法
const a = {
    async f(){...}
}
a.f().then(...)

// Class的方法
class K {
    async f() {...}
}

// 箭头函数
const f = async () => {...}
```

async函数的return值，即函数本身的返回值，会是一个Promise对象

```js
async function f() {}
const p = f();
console.log(p instanceof Promise); // true,返回的是Promise对象
```

如果return值是一个Promise对象，则直接作为async函数的返回值

```js
// 示例一
async function f() {
    return Promise.resolve(99);
}
const p = f();
p.then(data => console.log(data)); // 99

// 示例二
async function f() {
    return Promise.reject(98);
}
const p = f();
p.catch(err => console.log(err)); // 98
```

如果return值是非Promise对象，则会被Promise.resolve转为Promise对象并返回

```js
async function f() {
    return 99; // 会把返回值，转为状态为resolve的Promise对象
    // 等价于return Promise.resolve(99);
}
const p = f();
p.then(data => console.log(data)); // 99
```

await关键字代表等待其后方的异步操作，否则不会执行下面的代码

```js
async function f() {
    console.log(1);
    await new Promise(resolve => setTimeout(resolve , 2000));
    console.log(2);
}
f(); // 输出1，两秒后输出2
```
    
await可以接收其后Promise对象的返回值

```js
async function f() {
    console.log(1);
    const a1 = await new Promise(resolve => {
        setTimeout(() => {
            resolve(99);
        }, 2000);
    });
    console.log(a1); // 99
}
f();
```
    
await后的表达式，如果是个非Promise，会被转换成状态为resolved的Promise对象
    
```js
async function f() {
    console.log(1);
    const a1 = await 99;
    console.log(a1); // 99
}
f();
```
    
await可以自执行
    
```js
// 箭头函数的()要放在外面
(async () => {
    const a = await 100;
    return a;
})().then(data => console.log(data));

// function函数符合自执行写法即可
(async function() {
    const a = await 100;
    return a;
})().then(data => console.log(data));
```
    
### async错误处理
    
```js
// 示例一
async function f() {
    console.log(1);
    const a1 = await new Promise((resolve, reject) => {
        setTimeout(() => {
            reject(98);
        }, 2000);
    }).catch(err => {
        console.log('inside err is: ' + err);
        return 99;
    });
    console.log(a1);
}
f().catch(err => {
    console.log('outside err is: ' + err);
})

    // 输出1，两秒后输出'inside err is: 98'、99，没有走f()的catch

    注：如果await后的Promise对象有自己的catch，则会自己catch捕获，不会立即交给async函数执行后的catch。
        这里catch后继续往下走，是因为catch里return 99，会得到一个Promise.resolve(99)即resolved状态的Promise给await，
        对await而言这是resolved成功状态的。
        有疑问的话看下一个示例

// 示例二
async function f() {
    console.log(1);
    const a1 = await new Promise((resolve, reject) => {
        setTimeout(() => {
            reject(98);
        }, 2000);
    }).catch(err => {
        console.log('inside err is: ' + err);
        return Promise.reject(99); // 这里返回一个reject状态的Promise
    });
    console.log(a1);
}
f().catch(err => {
    console.log('outside err is: ' + err);
})

    // 输出1，两秒后输出'inside err is: 98'、'outside err is: 99'，没有console.log(a1);
    
    注：如果catch返回一个reject状态的Promise，会停止执行接下来的函数内容，直接抛出async函数，交给async函数执行后的catch
        
    结论：从示例一、二可以看出，async函数内部直接抛出去执行async函数执行后的catch，取决于内部是否await有接收reject状态的Promise
    
// 示例三
async function f() {
    console.log(1);
    const a1 = await new Promise((resolve, reject) => {
        setTimeout(() => {
            reject(98);
        }, 2000);
    });
    console.log(a1);
}
f().catch(err => {
    console.log('outside err is: ' + err);
})

    // 输出1，两秒后输出'outside err is: 98'，没有执行到console.log(a1)

    原因：await后的Promise对象没有自己的catch捕获，且返回的是reject状态的Promise，证实上面的结论，await遇到reject状态的Promise后会抛出
    
// 示例四
async function f() {
    console.log(1);
    try{
        const a1 = await new Promise((resolve, reject) => {
            reject(98);
        });
        const a2 = await new Promise((resolve, reject) => {
            resolve(99);
        });
    }catch(err) {
        console.log('try catch error');
    }
    console.log(100);
    return 101;
}
f().then(data => console.log(data))
    .catch(err => console.log('outside err is: ' + err));
    
    // 输出1、'try catch error'、100、101

    注：await被try catch包起时，当await后的Promise是reject状态时，不会直接抛出整个async函数，
        而是由try的catch接收。
        如果catch里没有return，则不影响代码继续执行！
        这个点要特别注意，我们在下一个示例看看，如果catch中有return会怎么样
        
// 示例五
async function f() {
    console.log(1);
    try{
        const a1 = await new Promise((resolve, reject) => {
            reject(98);
        });
        const a2 = await new Promise((resolve, reject) => {
            resolve(99);
        });
    }catch(err) {
        console.log('try catch error');
        return 777;
    }
    console.log(100);
    return 101;
}
f().then(data => console.log(data))
    .catch(err => console.log('outside err is: ' + err));

    // 输出1、'try catch error'、777
    
    注：如果try catch的catch中有return，其实就相当于给async函数内部执行return操作，直接返回
        这里return 777，相当于async函数return Promise.resolve(777)，所以会走fn().then输出777
        让我们在下一个示例返回reject状态的Promise验证
        
// 示例六
async function f() {
    console.log(1);
    try{
        const a1 = await new Promise((resolve, reject) => {
            reject(98);
        });
        const a2 = await new Promise((resolve, reject) => {
            resolve(99);
        });
    }catch(err) {
        console.log('try catch error');
        return Promise.reject(888);
    }
    console.log(100);
    return 101;
}
f().then(data => console.log(data))
    .catch(err => console.log('outside err is: ' + err));
    
    // 输出1、'try catch error'、'outside err is: 888'，证实了上述说法
    
        

总结：
    在没有try catch的情况下：
    async的抛出取决对await接收到reject状态的Promise对象
    
    在使用try catch的情况下：
    当await接收到reject状态的Promise对象时，会由try的catch捕获
    如果catch中没有做任何return操作，async内部继续执行不抛出
    如果catch中做了return操作，会根据return的是什么状态的Promise来决定是什么状态的抛出
```
        
### 在循环中使用async/await
    
```js
// 示例数据源
const store = {
    a: 100,
    b: 99,
    c: 98,
}
const keys = ['a', 'b', 'c']
function sleep() {
    return new Promise(resolve => setTimeout(resolve, 2000));
}
// 模拟异步获取num
function getStoreNum(name) {
    return sleep().then(() => store[name]);
}
```
    
#### for中使用await
    
```js
// 示例一（for）
async function f() {
    for(let i = 0, len = keys.length; i < len; i++) {
        const num = await getStoreNum(keys[i]);
        console.log(num);
    }
}
f();
// 间隔2秒的输出100、99、98

// 示例二
async function f() {
    for(let key in store) {
        const num = await getStoreNum(key);
        console.log(num);
    }
}
f();
// 间隔2秒的输出100、99、98

结论：for循环、for in、for of(此处不示例)支持async/await
```

#### forEach中使用await
    
```js
function f() {
    console.log('start');
    keys.forEach(async key => {
        const num = await getStoreNum(key);
        console.log(num);
    })
    console.log('end');
}
f();
// 立即输出'start'、'end'，2秒后同时输出100、99、98

结论：forEach不支持Promise感知，也不支持async/await

解决：封装如下$each函数模拟forEach

const $each = (function(){
    let index = 0;
    return (collection, callback) => {
        if(index === collection.length) {
            index = 0;
            return Promise.resolve();
        }else {
            return callback(collection[index], index).then(() => {
                ++index;
                return $each(collection, callback);
            });
        }
    }
}());

console.log('start');
$each(keys, async key => {
    const num = await getStoreNum(key);
    console.log(num);
}).then(() => console.log('end'))
```
    
#### map中使用await

```js
function f() {
    console.log('start');
    const maps = keys.map(async key => {
        const num = await getStoreNum(key);
        return num;
    })
    console.log(maps);
    console.log('end');
}
f();
// 输出'start'、[Promise、Promise、Promise]、'end'

原因：map返回的数组，是回调return结果，而回调又是async，async返回Promise，所以得到一个Promise数组，异步操作在数组内各自执行互不影响

解决：使用Promise.all处理map返回的Promise数组

async function f() {
    console.log('start');
    const maps = keys.map(async key => {
        const num = await getStoreNum(key);
        return num;
    })
    const nums = await Promise.all(maps);
    console.log(nums);
    console.log('end');
}
f();
// 输出'start'，2秒后输出[100, 99, 98]、'end'
```
    
#### filter中使用await

```js
async function f() {
    console.log('start');
    const filters = keys.filter(async key => {
        const num = await getStoreNum(key);
        return num > 99;
    })
    console.log(filters);
    console.log('end');
}
f();

// 立即输出'start'、['a', 'b', 'c']、'end'

原因：filter过滤是根据回调的返回值，而它的回调是async函数，async返回Promise对象，
        Promise对象对true/false而言永远是true，所以filters没有过滤掉任何一项，且没有等待异步，返回['a', 'b', 'c']

解决：无法单独使用，需要先map再filter

async function f() {
    console.log('start');
    const maps = keys.map(async key => {
        const num = await getStoreNum(key);
        return num;
    })
    const nums = await Promise.all(maps);
    const filters = nums.filter(item => item > 99);
    console.log(filters);
    console.log('end');
}
f();
// 立即输出'start'，2秒后输出100、'end'
```
    
#### reduce中使用await

```js
async function f() {
    console.log('start');
    const sum = await keys.reduce(async (sum, key) => {
        const num = await getStoreNum(key);
        return sum + num;
    }, 0)
    console.log(sum);
    console.log('end');
}
f();

// 立即输出'start'，2秒后输出[object Promise]98、'end'

原因：我们来解析一下reduce的过程
步骤一：sum为0，key为'a'，num得到100，return 0 + 100即100，但回调是async函数，返回值会被转换为Promise对象
步骤二：sum为Promise对象，num得到99，return Promise对象 + 99，得到'[object Promise]99'，返回时又被转为Promise对象
步骤三：sum为Promise对象，num得到98，return Promise对象 + 99，得到'[object Promise]98'，返回时又被转为Promise对象
    
    注：我们可以从步骤看出，reduce里是可以使用await的，之所以没有等待6s而是2s就出来，
        是因为对于reduce而言，只要回调有返回值就会进入下一次累加！
        第一次执行回调后，async回调会立即返回一个Promise对象(只是内部还在异步getStoreNum而言)，
        所以立即进入了第二次回调，所以2s就结束了


解决：既然sum是Promise，我们也加个await等待即可
async function f() {
    console.log('start');
    const sum = await keys.reduce(async (promiseSum, key) => {
        const sum = await promiseSum; // await上一次promise
        const num = await getStoreNum(key);
        return sum + num;
    }, 0)
    console.log(sum);
    console.log('end');
}
f();
// 立即输出'start'，6秒后输出297、'end'


有种加速reduce的方法：把sum和num对调
async function f() {
    console.log('start');
    const sum = await keys.reduce(async (promiseSum, key) => {
        const num = await getStoreNum(key); // 挪到上面
        const sum = await promiseSum; // 挪到取num的下面
        return sum + num;
    }, 0)
    console.log(sum);
    console.log('end');
}
f();
// 立即输出'start'，2秒后输出297、'end'

    原因：promiseSum内部是单独在执行异步操作的，将getStoreNum放上面，这样在getStoreNum这2s异步过程中，
            promiseSum由于上一次回调已经开始执行，所以是和getStoreNum同步在跑的，
            2个promise都是2s，所以当num取到时，sum也刚好跑完了
            
            
配合map循环更有效：
async function f() {
    console.log('start');
    const maps = keys.map(async key => {
        const num = await getStoreNum(key);
        return num;
    })
    const nums = await Promise.all(maps);
    const sum = nums.reduce((sum, num) => sum + num, 0);
    console.log(sum);
    console.log('end');
}
f();
// 立即输出'start'，2秒后输出297、'end'
```
