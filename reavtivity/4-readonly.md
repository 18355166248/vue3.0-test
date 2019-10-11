### 使用jest进行测试, 导入依赖:
```js
import { reactive,readonly,toRaw,isReactive,isReadonly,markNonReactive,markReadonly,lock,unlock,effect,ref} from '../src'
import { mockWarn } from '@vue/runtime-test'
```

1. 支持Object, Array  使用readonly时嵌套值为只读
```js
const original = { foo: 1, bar: { baz: 2 } }
const observed = readonly(original)
expect(observed).not.toBe(original)
expect(isReactive(observed)).toBe(true)
expect(isReadonly(observed)).toBe(true)
expect(isReactive(original)).toBe(false) // 原始数据是不存在reactiveToRaw(WeakMap)中的
expect(isReadonly(original)).toBe(false) // 原始数据是不存在readonlyToRaw(WeakMap)中的
expect(isReactive(observed.bar)).toBe(true)
expect(isReadonly(observed.bar)).toBe(true)
expect(isReactive(original.bar)).toBe(false) // 原始数据的键值也是不存在reactiveToRaw(WeakMap)中的
expect(isReadonly(original.bar)).toBe(false) // 原始数据的键值也是不存在readonlyToRaw(WeakMap)中的
// 获取
expect(observed.foo).toBe(1)
// 判断是否存在
expect('foo' in observed).toBe(true)
// 判断观察值存在哪些key值
expect(Object.keys(observed)).toEqual(['foo', 'bar'])
```

2. 不允许改变
```js
const qux = Symbol('qux')
const original = {
    foo: 1,
    bar: {
      baz: 2
    },
    [qux]: 3
}
const observed: Writable<typeof original> = readonly(original) // 设置observed为只读

observed.foo = 2 // 尝试修改readonly的值 修改失败 并报错
expect(observed.foo).toBe(1)
expect(
`Set operation on key "foo" failed: target is readonly.`
).toHaveBeenWarnedLast()

observed.bar.baz = 3 // 尝试修改readonly的值 修改失败 并报错
expect(observed.bar.baz).toBe(2)
expect(
`Set operation on key "baz" failed: target is readonly.`
).toHaveBeenWarnedLast()

observed[qux] = 4 // 尝试修改readonly的值 修改失败 并报错
expect(observed[qux]).toBe(3)
expect(
`Set operation on key "Symbol(qux)" failed: target is readonly.`
).toHaveBeenWarnedLast()

delete observed.foo  
expect(observed.foo).toBe(1) // 尝试删除readonly的值 删除失败 并报错
expect(
`Delete operation on key "foo" failed: target is readonly.`
).toHaveBeenWarnedLast()

delete observed.bar.baz // 尝试删除readonly的值 删除失败 并报错
expect(observed.bar.baz).toBe(2)
expect(
`Delete operation on key "baz" failed: target is readonly.`
).toHaveBeenWarnedLast()

delete observed[qux]  // 尝试删除readonly的值 删除失败 并报错
expect(observed[qux]).toBe(3)
expect(
`Delete operation on key "Symbol(qux)" failed: target is readonly.`
).toHaveBeenWarnedLast()
```

3. unlocked和lock控制readonly是否可以观察编辑更新
```js
const observed: any = readonly({ foo: 1, bar: { baz: 2 } })
unlock() // 解锁 使得 执行 readonlyHandlers时 LOCKED 为false 执行更新方法
observed.prop = 2
observed.bar.qux = 3
delete observed.bar.baz
delete observed.foo
lock() // 锁上 之后的修改都不会起效
expect(observed.prop).toBe(2)
expect(observed.foo).toBeUndefined()
expect(observed.bar.qux).toBe(3)
expect('baz' in observed.bar).toBe(false)
expect(`target is readonly`).not.toHaveBeenWarned() // 不会报错
```

4. 锁上的时候不会触发effect unlock解锁的时候是可以触发的
```js
const observed: any = readonly({ a: 1 })
let dummy
effect(() => {
    dummy = observed.a
})
expect(dummy).toBe(1)
observed.a = 2 // 修改readonly会失败 并报错
expect(observed.a).toBe(1)
expect(dummy).toBe(1)
expect(`target is readonly`).toHaveBeenWarned()

unlock() // 解锁 可以执行effect
observed.a = 33
lock()
expect(observed.a).toBe(33)
expect(dummy).toBe(33)
expect('target is readonly').not.toHaveBeenWarned() // 不会报错
```

5. Map WeakMap Set WeakSet都是支持readonly的
> Map和Set因为支持枚举  所以也是可以枚举的


6. 用reactive包装readonly的话返回的还是readonly
```js
const a = readonly({})
const b = reactive(a) // 将readonly放进reactive包装 返回的还是readonly
expect(isReadonly(b)).toBe(true) // b是只读的
// reactive和readonly指向的都是同一个原始数据
expect(toRaw(a)).toBe(toRaw(b))
```

7. 用readonly包装一个reactive返回的是readonly
```js
const a = reactive({})
const b = readonly(a)  // 用readonly包装reactive 返回的是readonly
expect(isReadonly(b)).toBe(true)
// reactive和readonly指向的都是同一个原始数据
expect(toRaw(a)).toBe(toRaw(b))
```

8. 观察已经被观察的值的时候返回的相同的
```js
const original = { foo: 1 }
const observed = readonly(original) // 初始化readonly
const observed2 = readonly(observed) // 将已经readonly过的observerd再readonly一下是不会进行任何处理的 只会原路返回
expect(observed2).toBe(observed) // 所以两个readonly的值指向的同一个WeakMap的地址
```

9. 观察同一个对象多次 返回的也是相同的
```js
const original = { foo: 1 }
const observed = readonly(original)
const observed2 = readonly(original)
expect(observed2).toBe(observed)
```

10. markNoneReactive 标记不活跃
```js
const obj = readonly({
  foo: { a: 1 },
  bar: markNonReactive({ b: 2 })
})
expect(isReactive(obj.foo)).toBe(true)
expect(isReactive(obj.bar)).toBe(false) // 标记不活跃的值是不存在在观察队列中的
```

11. markReadonly 标记只读
```js
const obj = reactive({
  foo: { a: 1 },
  bar: markReadonly({ b: 2 })
})
expect(isReactive(obj.foo)).toBe(true) // reactive初始化后值存在于被观察队列
expect(isReactive(obj.bar)).toBe(true) // reactive初始化后值存在于被观察队列
expect(isReadonly(obj.foo)).toBe(false) // 没有标记只读 所以是可以被观察的
expect(isReadonly(obj.bar)).toBe(true) // 标记了只读 所以这个值只能读不会被观察
obj.bar.b = 233
expect(obj.bar.b).toBe(2)
expect('Set operation on key "b" failed: target is readonly.').toHaveBeenWarned() //尝试修改只读属性 会报错
```

12. 让ref变成readonly
```js
const n: any = readonly(ref(1))
n.value = 2
expect(n.value).toBe(1)
expect(
  `Set operation on key "value" failed: target is readonly.`
).toHaveBeenWarned() // 报错
```










