### 使用jest进行测试, 导入依赖:
``` js
import { reactive,effect,stop,toRaw,OperationTypes,DebuggerEvent,markNonReactive } from '../src/index'
import { ITERATE_KEY } from '../src/effect' // export const ITERATE_KEY = Symbol('iterate')
```

1. 初始化effect函数会运行一次( 由effect包装 )
``` js
const fnSpy = jest.fn(() => {})
effect(fnSpy)
expect(fnSpy).toHaveBeenCalledTimes(1)
```

不支持普通对象
``` js
let dummy
const obj = {foo: 1}
const fnSpy = jest.fn(() => (dummy = obj.foo))
effect(fnSpy)
expect(dummy).toBe(1)
obj.foo = 100 // 这里不会触发effect
expect(dummy).toBe(1)
```


2. 支持基本属性
``` js
let dummy
const counter = reactive({ num: 0 })
effect(() => (dummy = counter.num))

expect(dummy).toBe(0)
counter.num = 7
expect(dummy).toBe(7)
```

3. 支持观察多个属性
``` js
let dummy
const counter = reactive({ num1: 0, num2: 0 })
effect(() => (dummy = counter.num1 + counter.num1 + counter.num2))

expect(dummy).toBe(0)
counter.num1 = counter.num2 = 7
expect(dummy).toBe(21)
```

4. 处理多个effect
``` js
let dummy1, dummy2
const counter = reactive({ num: 0 })
effect(() => (dummy1 = counter.num))
effect(() => (dummy2 = counter.num))

expect(dummy1).toBe(0)
expect(dummy2).toBe(0)
counter.num++
expect(dummy1).toBe(1)
expect(dummy2).toBe(1)
```

5. 支持嵌套属性
``` js
let dummy
const counter = reactive({ nested: { num: 0 } })
effect(() => (dummy = counter.nested.num))

expect(dummy).toBe(0)
counter.nested.num = 8
expect(dummy).toBe(8)
```

6. 支持删除属性
``` js
let dummy
const obj = reactive({ prop: 'value' })
effect(() => (dummy = obj.prop))

expect(dummy).toBe('value')
delete obj.prop
expect(dummy).toBe(undefined)
```

7. 支持将数据绑定到原型链
``` js
let dummy
const counter = reactive({ num: 0 })
const parentCounter = reactive({ num: 2 })
Object.setPrototypeOf(counter, parentCounter)
effect(() => (dummy = counter.num))

expect(dummy).toBe(0) // 从本身查找 counter.num = 0
delete counter.num
expect(dummy).toBe(2) // counter.num被删除 向原型链上查找 找到parentCounter.num = 2
parentCounter.num = 4
expect(dummy).toBe(4) // parentCounter.num更新了新值 dummy相对于进行更新
counter.num = 3
expect(dummy).toBe(3) // 添加了counter.num 从本身查找


let dummy
const counter = reactive({ num: 0 })
const parentCounter = reactive({ num: 2 })
Object.setPrototypeOf(counter, parentCounter)
effect(() => (dummy = 'num' in counter))

expect(dummy).toBe(true)
delete counter.num
expect(dummy).toBe(true)
delete parentCounter.num
expect(dummy).toBe(false)
counter.num = 3
expect(dummy).toBe(true)
```

8. 支持将reactive设置成另外一个reactive的原型
``` js
let dummy, parentDummy, hiddenValue: any
const obj = reactive<{ prop?: number }>({})
const parent = reactive({
  set prop(value) {
    hiddenValue = value
  },
  get prop() {
    return hiddenValue
  }
})
Object.setPrototypeOf(obj, parent) // 将parent设置成obj的原型
effect(() => (dummy = obj.prop)) // dummy依赖obj.prop
effect(() => (parentDummy = parent.prop)) // parentDummy依赖parent.prop

expect(dummy).toBe(undefined)
expect(parentDummy).toBe(undefined)
obj.prop = 4
expect(dummy).toBe(4)
// this doesn't work, should it?
// expect(parentDummy).toBe(4) // 这个是错误的, parentDummy应该是undefined
parent.prop = 2
expect(dummy).toBe(2) // parent.prop设置了值 dummy依赖于obj.prop obj.prop是没有值的 所以去它的原型上查找 它的原型parentDummy有值
expect(parentDummy).toBe(2) // parent.prop设置了值 所以parentDummy有值
```

9. effect支持函数调用链
``` js
let dummy
const counter = reactive({ num: 0 })
effect(() => (dummy = getNum())) // effect的dummy赋值是通过函数返回值进行设置

function getNum() {
  return counter.num
}

expect(dummy).toBe(0)
counter.num = 2 // 函数返回值里面使用了counter.num进行设置effect 所以是算依赖收集的
expect(dummy).toBe(2)
```

10. 观察迭代
``` js
let dummy
const list = reactive(['Hello'])
effect(() => (dummy = list.join(' ')))

expect(dummy).toBe('Hello')
list.push('World!') // 改变数组后 打印dummy的时候相应的会执行effect
expect(dummy).toBe('Hello World!')
list.shift() // 改变数组后 打印dummy的时候相应的会执行effect
expect(dummy).toBe('World!')
```

11. 可以指定索引进行赋值
``` js
let dummy
const list = reactive(['Hello'])
effect(() => (dummy = list.join(' ')))

expect(dummy).toBe('Hello')
list[1] = 'World!'
expect(dummy).toBe('Hello World!')
list[3] = 'Hello!' // 可以指定索引进行赋值
expect(dummy).toBe('Hello World!  Hello!')


let dummy
const list = reactive<string[]>([])
list[1] = 'World!'
effect(() => (dummy = list.join(' ')))

expect(dummy).toBe(' World!')
list[0] = 'Hello'
expect(dummy).toBe('Hello World!')
list.pop()
expect(dummy).toBe('Hello')
```

12. 枚举
``` js
let dummy = 0
const numbers = reactive<Record<string, number>>({ num1: 3 })
effect(() => {
  dummy = 0
  // 支持枚举
  for (let key in numbers) {
    dummy += numbers[key]
  }
})

expect(dummy).toBe(3)
numbers.num2 = 4
expect(dummy).toBe(7)
delete numbers.num1
expect(dummy).toBe(4)
```

13. 设置key值为Symbol类型
``` js
const key = Symbol('symbol keyed prop')
let dummy, hasDummy
const obj = reactive({ [key]: 'value' }) // key为 Symbol类型
effect(() => (dummy = obj[key]))
effect(() => (hasDummy = key in obj))

expect(dummy).toBe('value')
expect(hasDummy).toBe(true)
obj[key] = 'newValue'
expect(dummy).toBe('newValue')
delete obj[key]
expect(dummy).toBe(undefined)
expect(hasDummy).toBe(false)
```

14. effect中使用 Symbol 不支持依赖收集 进行动态改变值
``` js
const key = Symbol.isConcatSpreadable
let dummy
const array: any = reactive([])
effect(() => (dummy = array[key]))

expect(array[key]).toBe(undefined)
expect(dummy).toBe(undefined)
array[key] = true // 给reactive初始化的array进行赋值 key值如果为Symbol类型 是可以修改成功的  如果在effect中使用array 是不能监听改变并更新成功的
expect(array[key]).toBe(true)
expect(dummy).toBe(undefined)
```

15. 观察函数值属性 
``` js
const oldFunc = () => {}
const newFunc = () => {}

let dummy
const obj = reactive({ func: oldFunc }) // 将 oldFunc  赋值给 obj.func
effect(() => (dummy = obj.func)) // 调用effect 给 dummy赋值为obj.func 也就是指向 oldFunc

expect(dummy).toBe(oldFunc)
obj.func = newFunc // 将 obj.func 指向 newFunc 所以 在获取dummy的时候重新执行了effect 赋值为newFunc
expect(dummy).toBe(newFunc)
```

16. 不改变值的情况下不会观察或者执行操作
``` js
let hasDummy, getDummy
const obj = reactive({ prop: 'value' })

const getSpy = jest.fn(() => (getDummy = obj.prop))
const hasSpy = jest.fn(() => (hasDummy = 'prop' in obj))
effect(getSpy)
effect(hasSpy)

expect(getDummy).toBe('value')
expect(hasDummy).toBe(true)
obj.prop = 'value' // obj.prop 值没有变化 getSpy, hasSpy不会执行 值也不会有变化
expect(getSpy).toHaveBeenCalledTimes(1)
expect(hasSpy).toHaveBeenCalledTimes(1)
expect(getDummy).toBe('value')
expect(hasDummy).toBe(true)
```

17. 使用toRaw函数调用的reactive是不能被监测到的
``` js
let dummy
const obj = reactive<{ prop?: string }>({})
effect(() => (dummy = toRaw(obj).prop))

expect(dummy).toBe(undefined)
obj.prop = 'value' // 这里修改是不能被监测到的
expect(dummy).toBe(undefined)


let dummy
const obj = reactive<{ prop?: string }>({})
effect(() => (dummy = obj.prop))

expect(dummy).toBe(undefined)
toRaw(obj).prop = 'value' // 这里修改是不能被监测到的
expect(dummy).toBe(undefined)


let dummy, parentDummy, hiddenValue: any
const obj = reactive<{ prop?: number }>({})
const parent = reactive({
  set prop(value) {
    hiddenValue = value
  },
  get prop() {
    return hiddenValue
  }
})
Object.setPrototypeOf(obj, parent)
effect(() => (dummy = obj.prop))
effect(() => (parentDummy = parent.prop))

expect(dummy).toBe(undefined)
expect(parentDummy).toBe(undefined)
toRaw(obj).prop = 4 // 这里修改是不能被监测到的
expect(dummy).toBe(undefined)
expect(parentDummy).toBe(undefined)
```

19. effect支持有暂停条件的递归循环
``` js
const counter = reactive({ num: 0 })
const numSpy = jest.fn(() => {
  counter.num++
  if (counter.num < 10) {
    numSpy()
  }
})
effect(numSpy) // 在这里初始化 递归了10次
expect(counter.num).toEqual(10)
expect(numSpy).toHaveBeenCalledTimes(10)
```

20. 避免和其他effect今后混用造成无限循环
``` js
const nums = reactive({ num1: 0, num2: 1 })

const spy1 = jest.fn(() => (nums.num1 = nums.num2))
const spy2 = jest.fn(() => (nums.num2 = nums.num1))
effect(spy1)
effect(spy2) // 这两个effect不要混用 不然得到的值就跟下面一样  会互相影响 最后达不到预期效果
expect(nums.num1).toBe(1)
expect(nums.num2).toBe(1)
expect(spy1).toHaveBeenCalledTimes(1)
expect(spy2).toHaveBeenCalledTimes(1)
nums.num2 = 4
expect(nums.num1).toBe(4)
expect(nums.num2).toBe(4)
expect(spy1).toHaveBeenCalledTimes(2)
expect(spy2).toHaveBeenCalledTimes(2)
nums.num1 = 10
expect(nums.num1).toBe(10)
expect(nums.num2).toBe(10)
expect(spy1).toHaveBeenCalledTimes(3)
expect(spy2).toHaveBeenCalledTimes(3)
```

21. effect返回的是一个函数不过不是新的函数
``` js
function greet() {
  return 'Hello World'
}
const effect1 = effect(greet)
const effect2 = effect(greet)
expect(typeof effect1).toBe('function')
expect(typeof effect2).toBe('function') // 返回值为函数
expect(effect1).not.toBe(greet) // 返回值不是之前的函数
expect(effect1).not.toBe(effect2)
```

22. effect做了优化 当执行函数内被依赖属性没有被执行或者利用的时候 函数是不会被调用的 ( 有效的优化 )
``` js
let dummy
const obj = reactive({ prop: 'value', run: false })

const conditionalSpy = jest.fn(() => {
  dummy = obj.run ? obj.prop : 'other'
})
effect(conditionalSpy)

expect(dummy).toBe('other')
expect(conditionalSpy).toHaveBeenCalledTimes(1)
obj.prop = 'Hi'  // 修改了obj.prop的值 不过这个时候obj的run为false 所以obj.prop的值不会用到 effect函数不会执行
expect(dummy).toBe('other')
expect(conditionalSpy).toHaveBeenCalledTimes(1) // 还是一次
obj.run = true
expect(dummy).toBe('Hi')  // obj的run值被改变了 effect函数逻辑会执行到obj.prop  所以effect执行了一次 变成了2次
expect(conditionalSpy).toHaveBeenCalledTimes(2)
obj.prop = 'World'
expect(dummy).toBe('World') // effect函数逻辑会执行到obj.prop 而且obj.prop也发成了改变 所以 effect又执行了一次 总次数为3次
expect(conditionalSpy).toHaveBeenCalledTimes(3)
```

23. effect内部依赖的值如果不是reactive返回的值 如果这个值改变了 当需要实时更新的话 需要接收返回值并在更新后调用一次返回值
``` js
let dummy
let run = false
const obj = reactive({ prop: 'value' })
const runner = effect(() => {
  dummy = run ? obj.prop : 'other'
})

expect(dummy).toBe('other')
runner()
expect(dummy).toBe('other')
run = true // run并不在reactive返回的值 所以改变后需要 调用下effect的返回值才能更新dummy
runner()
expect(dummy).toBe('value')
obj.prop = 'World' // obj的prop的reactive返回的值 所以不需要重新调用effect
expect(dummy).toBe('World')
```

24. 在effect内对值进行了多次赋值并不会让effect执行多次
``` js
let dummy
const obj = reactive<Record<string, number>>({})
const fnSpy = jest.fn(() => {
  for (const key in obj) {
    dummy = obj[key] // 这里会赋值0-n次
  }
  dummy = obj.prop // 这里会赋值一次
})
effect(fnSpy)

expect(fnSpy).toHaveBeenCalledTimes(1)
obj.prop = 16
expect(dummy).toBe(16)
expect(fnSpy).toHaveBeenCalledTimes(2) // dummy赋值了2次 effect函数只执行一次
```

25.  遵循嵌套效果
``` js
const nums = reactive({ num1: 0, num2: 1, num3: 2 })
const dummy: any = {}

const childSpy = jest.fn(() => (dummy.num1 = nums.num1))
const childeffect = effect(childSpy)
const parentSpy = jest.fn(() => {
  dummy.num2 = nums.num2
  childeffect()
  dummy.num3 = nums.num3
})
effect(parentSpy)

expect(dummy).toEqual({ num1: 0, num2: 1, num3: 2 })
expect(parentSpy).toHaveBeenCalledTimes(1)
expect(childSpy).toHaveBeenCalledTimes(2)
// childSpy被执行了一次
nums.num1 = 4
expect(dummy).toEqual({ num1: 4, num2: 1, num3: 2 })
expect(parentSpy).toHaveBeenCalledTimes(1)
expect(childSpy).toHaveBeenCalledTimes(3)
// parentSpy和childSpy各执行了一次 因为childSpy在parentSpy内部嵌套执行了一次
nums.num2 = 10
expect(dummy).toEqual({ num1: 4, num2: 10, num3: 2 })
expect(parentSpy).toHaveBeenCalledTimes(2)
expect(childSpy).toHaveBeenCalledTimes(4)
// parentSpy和childSpy各执行了一次 因为childSpy在parentSpy内部嵌套执行了一次
nums.num3 = 7
expect(dummy).toEqual({ num1: 4, num2: 10, num3: 7 })
expect(parentSpy).toHaveBeenCalledTimes(3)
expect(childSpy).toHaveBeenCalledTimes(5)
```

25. reactive, effect不仅支持普通函数也支持class类
``` js
class Model {
  count: number
  constructor() {
    this.count = 0
  }
  inc() {
    this.count++
  }
}
const model = reactive(new Model()) // 这里使用的是类有自带的初始化和方法
let dummy
effect(() => {
  dummy = model.count
})
expect(dummy).toBe(0) // 初始化成功
model.inc() // 支持调用class方法
expect(dummy).toBe(1)
```

26. effect第二个参数scheduler 如果传入了第二个参数 当依赖值改变了 effect的值并不会变 必须调用下 第二个参数内的返回值才会更新
``` js
let runner: any, dummy
const scheduler = jest.fn(_runner => {
  runner = _runner
})
const obj = reactive({ foo: 1 })
effect(
  () => {
    dummy = obj.foo
  },
  { scheduler }
)
expect(scheduler).not.toHaveBeenCalled()
expect(dummy).toBe(1)
// 这里obj的foo的值改变了  effect依赖此值 不过第一个参数并不会调用 而且调用了第二个参数
obj.foo++
expect(scheduler).toHaveBeenCalledTimes(1)
// 所以现在dummy值没有变化
expect(dummy).toBe(1)
// 执行effect第二个参数内返回的值 触发effect第一个函数的执行
runner()
// 所以更新了dummy的值
expect(dummy).toBe(2)
```

27. 事件: onTrack ( effect第二个参数固定参数 onTrack onTrigger ) ( 初始化自动触发 )
``` js
export interface DebuggerEvent {
  effect: ReactiveEffect
  target: any
  type: OperationTypes
  key: string | symbol | undefined
}
export const ITERATE_KEY = Symbol('iterate')

let events: DebuggerEvent[] = [] // 申请一个存放事件的数组
let dummy
const onTrack = jest.fn((e: DebuggerEvent) => {
  events.push(e)
})
const obj = reactive({ foo: 1, bar: 2 })
const runner = effect(
  () => {
    dummy = obj.foo
    dummy = 'bar' in obj
    dummy = Object.keys(obj)
  },
  { onTrack } // 这里 onTrack 是必须用这个名字 onTrack 会初始化触发
)
expect(dummy).toEqual(['foo', 'bar'])
expect(onTrack).toHaveBeenCalledTimes(3) // 初始化 onTrack effect执行 dummy被赋值了3次都被effect第二个参数记录下来了放进了events
expect(events).toEqual([
  {
    effect: runner,
    target: toRaw(obj),
    type: OperationTypes.GET,
    key: 'foo'
  },
  {
    effect: runner,
    target: toRaw(obj),
    type: OperationTypes.HAS,
    key: 'bar'
  },
  {
    effect: runner,
    target: toRaw(obj),
    type: OperationTypes.ITERATE, // iterate
    key: ITERATE_KEY // Symbol('iterate')
  }
])
```

28. 事件: onTrigger ( effect第二个参数固定参数 onTrack onTrigger ) ( 初始化不会自动触发 需要改变依赖值才会触发 )
``` js
let events: DebuggerEvent[] = []
let dummy
const onTrigger = jest.fn((e: DebuggerEvent) => {
  events.push(e)
})
const obj = reactive({ foo: 1 })
const runner = effect(
  () => {
    dummy = obj.foo
  },
  { onTrigger } // onTrigger只能用这个名字
)

obj.foo++ // 在改变依赖值的情况下才会触发 onTrigger
expect(dummy).toBe(2)
expect(onTrigger).toHaveBeenCalledTimes(1)
expect(events[0]).toEqual({
  effect: runner,
  target: toRaw(obj),
  type: OperationTypes.SET,
  key: 'foo',
  oldValue: 1,
  newValue: 2
})

delete obj.foo
expect(dummy).toBeUndefined()
expect(onTrigger).toHaveBeenCalledTimes(2)
expect(events[1]).toEqual({
  effect: runner,
  target: toRaw(obj),
  type: OperationTypes.DELETE,
  key: 'foo',
  oldValue: 2
})
```

29. 停止监听( stop )
``` js
let dummy
const obj = reactive({ prop: 1 })
const runner = effect(() => {
  dummy = obj.prop
})
obj.prop = 2
expect(dummy).toBe(2)
stop(runner) // 停止effect
obj.prop = 3
expect(dummy).toBe(2)

// 停止effect后仍然可以手动调用进行更新值
runner() // 更新effect
expect(dummy).toBe(3)
```

30. 时间 onStop  在调用stop的时候 会触发effect第二个参数里面的onStop
``` js
const runner = effect(() => {}, {
  onStop: jest.fn()
})

stop(runner)
expect(runner.onStop).toHaveBeenCalled() // onStop方法被触发
```

31. 标记不活跃参数  markNonReactive
``` js
const obj = reactive({
  foo: markNonReactive({ // 使用 markNonReactive 赋值给 foo为不活跃参数
    prop: 0
  })
})
let dummy
effect(() => {
  dummy = obj.foo.prop
})
expect(dummy).toBe(0)
obj.foo.prop++ // 不活跃参数在直接修改的时候的不能触发更新的
expect(dummy).toBe(0)
obj.foo = { prop: 1 } // 重新给不活跃参数 foo 进行赋值才会触发更新
expect(dummy).toBe(1)
```
