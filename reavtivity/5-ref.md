### 使用jest进行测试, 导入依赖:
```js
import { ref, effect, reactive, isRef, toRefs } from '../src/index'
import { computed } from '@vue/runtime-dom'
```

1. ref声明的值可以被reactive观察 嵌套值的改变也可以
```js
const a = ref(1) // const a = ref({count: 1}) 
let dummy
effect(() => {
  dummy = a.value
})
expect(dummy).toBe(1)
a.value = 2 // ref是可以被观察到的
expect(dummy).toBe(2)
```

2.  就算嵌套多层 也是可以被监测到的
```js
const a = ref(1)
const obj = reactive({
  a,
  b: {
    c: a,
    d: [a]
  }
})
let dummy1
let dummy2
let dummy3
effect(() => {
  dummy1 = obj.a
  dummy2 = obj.b.c
  dummy3 = obj.b.d[0]
})
expect(dummy1).toBe(1)
expect(dummy2).toBe(1)
expect(dummy3).toBe(1)
a.value++
expect(dummy1).toBe(2) // 依然可以监测到 任何一个值的改变都会造成其他值同步更新
expect(dummy2).toBe(2)
expect(dummy3).toBe(2)
obj.a++
expect(dummy1).toBe(3)
expect(dummy2).toBe(3)
expect(dummy3).toBe(3)
```

3. 支持嵌套ref
```js
const a = {
  b: ref(0)
}

const c = ref(a)

expect(typeof (c.value.b + 1)).toBe('number')
```

4. 用isRef判断是否是ref
```js
let dummy
const obj = { foo: 1 }
expect(isRef(ref(1))).toBe(true) // 是
expect(isRef(computed(() => 1))).toBe(true) // 计算属性也是
expect(isRef(reactive(() => (dummy = obj.foo)))).toBe(false) // reactive不是
dummy

expect(isRef(0)).toBe(false) // 不是
expect(isRef(1)).toBe(false) // 不是
// 对象看起来像其实不是
expect(isRef({ value: 0 })).toBe(false)
```

5. 使用toRefs转换成ref
```js
const a = reactive({
  x: 1,
  y: 2
})

const { x, y } = toRefs(a)

expect(isRef(x)).toBe(true)
expect(isRef(y)).toBe(true)
expect(x.value).toBe(1)
expect(y.value).toBe(2)

// source -> proxy 源数据的改变影响转换ref后的数据
a.x = 2
a.y = 3
expect(x.value).toBe(2)
expect(y.value).toBe(3)

// proxy -> source 转换ref后的数据影响源数据
x.value = 3
y.value = 4
expect(a.x).toBe(3)
expect(a.y).toBe(4)

// reactivity effect后的数据就是转换ref后的数据
let dummyX, dummyY
effect(() => {
  dummyX = x.value
  dummyY = y.value
})
expect(dummyX).toBe(x.value)
expect(dummyY).toBe(y.value)

// 修改源数据会触发使用ref的effect方法
a.x = 4
a.y = 5
expect(dummyX).toBe(4)
expect(dummyY).toBe(5)
```
