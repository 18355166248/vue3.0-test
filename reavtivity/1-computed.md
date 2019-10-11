使用jest进行测试, 导入依赖:
``` js
import { computed, reactive, effect, stop, ref } from '../src'
```


正确的返回更新值
``` js
const value = reactive<{ foo?: number }>({})
const cValue = computed(() => value.foo)
expect(cValue.value).toBe(undefined)
// 计算属性里面使用了value.foo 当value.foo的值更改了以后 触发了computed的执行 使Cvalue重新赋值了一次
value.foo = 1
expect(cValue.value).toBe(1)
```


2. 懒计算
``` js
const value = reactive<{ foo?: number }>({})
const getter = jest.fn(() => value.foo)
const cValue = computed(getter)

expect(getter).not.toHaveBeenCalled()

expect(cValue.value).toBe(undefined)
expect(getter).toHaveBeenCalledTimes(1)

// 获取计算属性值的时候是不会执行计算
cValue.value
expect(getter).toHaveBeenCalledTimes(1)

// 计算属性内有被依赖值, 依赖值在改变的时候不会触发getter直到被需要获取计算属性值的时候
value.foo = 1 // 这里给被依赖值重新赋值 不过并不需要重新计算 所以getter没有执行 还是一次
expect(getter).toHaveBeenCalledTimes(1)

// 获取计算属性的值 所以计算属性getter执行了一次 加上面一次总共执行了2次
expect(cValue.value).toBe(1)
expect(getter).toHaveBeenCalledTimes(2)

// 获取计算属性值的时候是不会执行计算  所以getter执行次数还是2次
cValue.value
expect(getter).toHaveBeenCalledTimes(2)
```

3. 触发effect
``` js
const value = reactive<{ foo?: number }>({})
const cValue = computed(() => value.foo) // 声明一个计算属性 cValue 依赖 value下的foo值
let dummy
effect(() => { // 调用effect方法 依赖计算属性 cValue下的value值
  dummy = cValue.value
})
expect(dummy).toBe(undefined) // 一开始dummy的值是cValue.value  cValue.value是reactive初始化的value值也就是空对象{} 所以是undefined
value.foo = 1  // 给value的foo赋值 不过并不需要重新计算 所以computed内的函数并没有执行
expect(dummy).toBe(1) // dummy依赖cValue.value 所以computed执行了一次 将value.foo赋值给Cvalue.value.foo 所以dummy的值为1
```


4. 多个computed是可以嵌套使用并相互影响
``` js
const value = reactive({ foo: 0 })
const c1 = computed(() => value.foo)
const c2 = computed(() => c1.value + 1)
expect(c1.value).toBe(0) // c1依赖于value.foo的影响
expect(c2.value).toBe(1) // c2下引用了c1的值 受c1影响
value.foo++
// c1, c2 都受value的影响
expect(c2.value).toBe(2)
expect(c1.value).toBe(1)
```

5. 多个computed是可以嵌套使用并相互影响, 如果使用effect则也会受影响
``` js
const value = reactive({ foo: 0 })
const getter1 = jest.fn(() => value.foo)
const getter2 = jest.fn(() => {
  return c1.value + 1
})
const c1 = computed(getter1)
const c2 = computed(getter2)

let dummy
effect(() => {
  dummy = c2.value  // dummy在effect内赋值c2.value, c2.value依赖c1.value  c1.value依赖value.foo
})
// 获取dummy值时 getter1, getter2都执行了一次
expect(dummy).toBe(1)
expect(getter1).toHaveBeenCalledTimes(1)
expect(getter2).toHaveBeenCalledTimes(1)
value.foo++
// 获取dummy值时 getter1, getter2都执行了一次 dummy最终依赖value.foo  所以值更新为2. getter1, getter2都执行了一次变成2次
expect(dummy).toBe(2)
// should not result in duplicate calls
expect(getter1).toHaveBeenCalledTimes(2)
expect(getter2).toHaveBeenCalledTimes(2)
```

6. 基于上一个案例( 可以使用混合调用 )
``` js
const value = reactive({ foo: 0 })
const getter1 = jest.fn(() => value.foo)
const getter2 = jest.fn(() => {
  return c1.value + 1
})
const c1 = computed(getter1)
const c2 = computed(getter2)

let dummy
effect(() => {
  dummy = c1.value + c2.value // 在这里混合调用
})
expect(dummy).toBe(1)

expect(getter1).toHaveBeenCalledTimes(1)
expect(getter2).toHaveBeenCalledTimes(1)
value.foo++
expect(dummy).toBe(3)
// should not result in duplicate calls
expect(getter1).toHaveBeenCalledTimes(2)
expect(getter2).toHaveBeenCalledTimes(2)
```

7. 停止的时候不再更新 (stop方法)
``` js
const value = reactive<{ foo?: number }>({})
const cValue = computed(() => value.foo)
let dummy
effect(() => {
  dummy = cValue.value
})
expect(dummy).toBe(undefined)
value.foo = 1
expect(dummy).toBe(1)
stop(cValue.effect) // 使用stop方法(packages/reactivity/src/effect/ts) effect方法将不再被调用, 里面的数据也将不会被更新
value.foo = 2
expect(dummy).toBe(1)
```

8. 支持setter
``` js
const n = ref(1) // 暂时理解为react的hook
const plusOne = computed({
  // computed支持get
  get: () => n.value + 1,
  // computed支持set
  set: val => {
    n.value = val - 1
  }
})

expect(plusOne.value).toBe(2)
n.value++
expect(plusOne.value).toBe(3)

plusOne.value = 0 // 赋值的时候触发set方法, 所以结果为-1
expect(n.value).toBe(-1)
```

9. setter触发effect
``` js
const n = ref(1)
const plusOne = computed({
  get: () => n.value + 1,
  set: val => {
    n.value = val - 1
  }
})

let dummy
effect(() => {
  dummy = n.value
})
expect(dummy).toBe(1)

plusOne.value = 0 // 触发set n.value被设置为-1
expect(dummy).toBe(-1)
```
