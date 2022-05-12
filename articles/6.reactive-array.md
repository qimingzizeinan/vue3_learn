# 数组
js中数组也是一种对象，所以前面的代码大多还能用。只是总会有些特殊的情况发生，需要我们做一些兼容。

> 所有代码均在 src/非原始值/Array.html  
> 代码中已加好注释

先来看看数组的读取和设置都有哪些操作,在根据具体情况处理。  

### 读取：  
- 索引读取 arr[0]
- 访问长度 arr.length
- for ... in 遍历
- for ... of 遍历
- 使用原型方法： concat/join/every/some等

### 设置：
- 索引设置 arr[0] = 3
- 修改长度 arr.length = 0
- 栈方法 pop/push/shift/unshift 等
- 修改原数组的原型方法 splice/fill/sort等

### 索引与length的处理
当前代码中直接代理一个数组，并且修改已存在的元素的值，是可以正常响应的，但是无法正确处理新增元素的情况。

下面这段代码是目前无法正常运行。
```javascript
const arr = reactive(['foo'])

effect(() => {
    console.log(arr[0])
})

arr[1] = 'bar'
```
解决的方法很简单。在set方法中判断代理目标为数组同时判断是新增还是设置操作。如果是新增则在trigger方法中将所有有关于length属性的effectFn取出并执行。

```javascript

// 判断是否已经存在的属性
const type = Array.isArray(target) ?
    // 如果被设置的索引小于数组长度说明是设置操作，反之是新增操作
    Number(key) < target.length ? TriggerType.SET : TriggerType.ADD
    : Object.prototype.hasOwnProperty.call(target, key) ? TriggerType.SET : TriggerType.ADD

// 新增且是数组
if (type === TriggerType.ADD && Array.isArray(target)) {
    // 取出与length有关的所有副作用函数
    const lengthEffect = depsMap.get('length')
    // 将与length有关的副作用函数加到effectsToRun中去
    lengthEffect && lengthEffect.forEach((effectFn) => {
        if (effectFn !== activeEffect) {
            effectsToRun.add(effectFn)
        }
    })
}
```

如果对数组的长度进行修改,导致数组元素发生变化，也应该触发相应,即下面这段代码的情况。
```javascript
const arr = reactive(['foo'])
effect(() => {
    console.log(arr[0])
})
arr.length = 0
```
解决的办法是在调用trigger函数触发时，将新的属性值(也就是新的length值)传递到trigger中去，将所有大于等于新的Length值的effectFn函数全部取出添加到effectToRun中去
```javascript
// 相等时说明receiver是targeet的代理对象
if (target === receiver.raw) {
    // 新旧值不相等切两个值都不为NaN
    if (oldValue !== newValue && (oldValue === oldValue && newValue === newValue)) {
        // 将新的值传递到trigger中去
        trigger(target, key, type, newValue)
    }

}

// 判断是否为数组且同时操作的属性为length
if (Array.isArray(target) && key === 'length') {
    depsMap.forEach((effects, key) => {
        // // 索引大于等于新的Length值的effectFn全部加入effectsToRun
        if (key >= newValue) {
            effects.forEach(effectFn => {
                if (effectFn !== activeEffect) {
                    effectsToRun.add(effectFn)
                }
            })
        }
    })
}
```

### for ... in 与for ... of的支持
前面的内容讲过for ... in 的支持是在ownKeys中。针对数组的追踪，则需要判断target是否为数组，如果为数组则将track的key值设置为lenght即可。
for ... of 现有代码就已经支持了。只是有一点要注意出于一些性能的影响不追踪symbol值。
```javascript
track(target, Array.isArray(target) ? 'length' : ITERATE_KEY)
```

### 数组的查找方法

#### 支持includes等方法

先来看一个无法正常工作的例子
```javascript
const obj = {}
const arr = reactive([obj])

console.log(arr.includes(arr[0]))  // false
```
为什么会返回false呢？  
js的语言规范中includes方法会先指定调用方法的this，这里的this是哪个呢？是arr这个代理对象。在includes的内部也会用arr去访问每一项元素，前面的get代码代码中明确歇着，如果访问的元素也是可以被代理的，则返回代理对象。
arr[0]是一个代理对象,includes内部访问的也是代理对象。每个对象都是通过createReactive创建出来的，所以两个不同对象是无法判定相等的。
因此这里返回了false.  

如何解决呢？  
创建一个用来存贮原始对象和代理对象的映射，每次调用reactive时先在映射中查找，找了则直接返回即可。

```javascript
const reactiveMap = new Map()

function reactive(obj) {

    const existionProxy = reactiveMap.get(obj)
    if (existionProxy) return existionProxy

    const proxy = createReactive(obj)

    reactiveMap.set(obj, proxy)
    return proxy
}
```

接下来看另一个failure的情况
```javascript
const obj1 = {}
const arr = reactive([obj1])

console.log(arr.includes(obj1)) // false
```
失败的原因很简单。arr是原始对象,调用includes方法时，内部循环拿到的也是代理对象，将代理对象和原始对象相比较，自然无法相等，也就返回false了。
解决的办法就是重写includes方法.需要重写的方法还有indexOf和lastIndexOf方法。一并重写.
```javascript
const arrayInstrumentations = {}

;['includes', 'indexOf', 'lastIndexOf'].forEach(method => {
    const originMethod = Array.prototype[method]
    arrayInstrumentations[method] = function(...args) {
        // this 是代理对象，先在代理对象中查找，将结果存储到 res 中
        let res = originMethod.apply(this, args)

        if (res === false) {
            // res 为 false 说明没找到，在通过 this.raw 拿到原始数组，再去原始数组中查找，并更新 res 值
            res = originMethod.apply(this.raw, args)
        }
        // 返回最终的结果
        return res
    }
})


// 判断为数组切操作的key在arrayInstrumentations对象上，则我们将Reflect.get的第一个参数设置为arrayInstrumentations对象。从该对象上获取值.
if (Array.isArray(target) && arrayInstrumentations.hasOwnProperty(key)) {
    return Reflect.get(arrayInstrumentations, key, receiver)
}
```

### 数组的栈方法
例如 push/pop/shift/unshift等
```javascript
const arr = reactive([])

effect(() => {
    arr.push(1)
})

effect(() => {
    arr.push(1)
})
```
这个测试例子运行时会栈溢出。为什么呢？  
push 方法会向数组添加了一项的同时会间接访问length属性。第一个effect函数运行之后,响应式中就会建立与length相关的依赖。第二个effect函数执行的时候，再次读取Length的时候则会触发已经收集到的依赖，然后第二次effect函数被收集，第一个effect函数触发后又执行与lenght属性相关的依赖，再次触发第二个effect函数，循环往复，则出现栈溢出。

解决办法呢？
屏蔽对length属性的读取。

'pop', 'shift', 'unshift', 'splice'方法也要同样处理。
```javascript
let shouldTrack = true
;['push', 'pop', 'shift', 'unshift', 'splice'].forEach(method => {
    const originMethod = Array.prototype[method]
    arrayInstrumentations[method] = function(...args) {
        // 在调用原始方法前，禁止追踪
        shouldTrack = false
        // 调用原始的方法
        let res = originMethod.apply(this, args)
        shouldTrack = true
        return res
    }
})
```






