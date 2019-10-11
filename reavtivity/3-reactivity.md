### 用jest进行测试, 导入依赖:
```js
import { reactive, isReactive, toRaw, markNonReactive } from '../src/reactive'
import { mockWarn } from '@vue/runtime-test'
```

1. 语法介绍  reactve isReactive  支持对象也支持数组
 ```js
const original = { foo: 1 }
const observed = reactive(original)
expect(observed).not.toBe(original)
expect(isReactive(observed)).toBe(true) // isReactive 是判断在reactiveToRaw 这个WeakMap中是否存在observed依赖 通过reactive初始化的是存在的
expect(isReactive(original)).toBe(false)
// 获取值
expect(observed.foo).toBe(1)
// 判断是否存在值
expect('foo' in observed).toBe(true)
// 判断返回对象中存在哪些key
expect(Object.keys(observed)).toEqual(['foo'])
```

2. 克隆reavtive数组是指向观察值的
```js
const original = [{ foo: 1 }]
const observed = reactive(original) // 这是观察值
const clone = observed.slice() // 克隆值
expect(isReactive(clone[0])).toBe(true) // 观察值存在
expect(clone[0]).not.toBe(original[0]) // 不是一个对象
expect(clone[0]).toBe(observed[0]) // 克隆值是指向观察值的
```

3. 嵌套reactive进行监听
```js
const original = {
  nested: {
    foo: 1
  },
  array: [{ bar: 2 }]
}
const observed = reactive(original)
expect(isReactive(observed.nested)).toBe(true)  // 可以监听 对象
expect(isReactive(observed.array)).toBe(true) // 可以监听数组
expect(isReactive(observed.array[0])).toBe(true) // 可以监听数组下面的一项
```

4. 观察值的改变会影响到原始值的变化 对象数组都支持
```js
const original: any = { foo: 1 } // 原始值
const observed = reactive(original) // 观察值
// 观察值的改变
observed.bar = 1
expect(observed.bar).toBe(1) // 观察值变化
expect(original.bar).toBe(1) // 原始值也改变了
delete observed.foo // 删除操作
expect('foo' in observed).toBe(false) // 观察值被删除了
expect('foo' in original).toBe(false) // 原始值被删除了
```

5.设置一个没有被观察的属性 要用reactive包装 
```js
const observed = reactive<{ foo?: object }>({})
const raw = {}
observed.foo = raw
expect(observed.foo).not.toBe(raw)  // 没有被reactive包装的属性 是不等于 被reactive包装的对象的
expect(isReactive(observed.foo)).toBe(true)
```

6. 被观察过的值再被观察返回还是拿个被观察过的值
```js
const original = { foo: 1 }
const observed = reactive(original) // 观察original
const observed2 = reactive(observed) // 再次观察被观察过的original 所以observed2等于observed
expect(observed2).toBe(observed)
```

7. 相同值被多次观察返回的值也是相同的
```js
const original = { foo: 1 }
const observed = reactive(original)
const observed2 = reactive(original)
expect(observed2).toBe(observed)
```

8. 不应该污染原始对象  使用多个reactive
```js
const original: any = { foo: 1 }
const original2 = { bar: 2 }
const observed = reactive(original)
const observed2 = reactive(original2)
observed.bar = observed2   // 这个最好不要这么使用
expect(observed.bar).toBe(observed2)
expect(original.bar).toBe(original2)
```

9. toRaw方法返回监测WeakMap内的值
```js
export function toRaw<T>(observed: T): T {
  return reactiveToRaw.get(observed) || readonlyToRaw.get(observed) || observed
}

const original = { foo: 1 }
const observed = reactive(original)
expect(toRaw(observed)).toBe(original)  // 去reactiveToRaw中查找是否有 查到有的话返回观察值
expect(toRaw(original)).toBe(original)  // 如果查询不到就直接返回传入的值
```

10. 不可观察的值
> number string boolean null undefined symbol Promise RegExp Date


11. markNonReactive: 标记不活跃
```js
const obj = reactive({
  foo: { a: 1 },
  bar: markNonReactive({ b: 2 }) // 被标记成不活跃 所以在用 isReactive 判断在 reactiveToRaw 或者 readonlyToRaw 中是否存在时 显示不存在
})
expect(isReactive(obj.foo)).toBe(true)
expect(isReactive(obj.bar)).toBe(false)
```
