## !intro reactivity

## !!steps 实现目标
reactive, ref, computed, effect, watch, nextTick。[codesandbox url](https://codesandbox.io/p/devbox/awesome-snowflake-ht4353?file=%2Fsrc%2Fmain.ts%3A7%2C1)

```js !
const reactiveValue = reactive({
  count: 0,
})
const refValue = ref(0)
const reactiveDoubleValue = computed(
  () => reactiveValue.count * 2,
)
const refDoubleValue = computed(
  () => refValue.value * 2,
)

watch(
  () => reactiveValue.count,
  (newValue, oldValue) => {
    console.log("watch reactiveValue: ")
    console.log("new value: ", newValue)
    console.log("old value: ", oldValue)
  },
)

watch(
  () => refValue.value,
  (newValue, oldValue) => {
    console.log("watch refValue: ")
    console.log("new value: ", newValue)
    console.log("old value: ", oldValue)
  },
)
```

## !!steps 如何监听读取和修改变量？

在 Vue2 是使用的 Object.defineProperties 进行的数据劫持，而在 Vue3 中是通过 ES6 新增的 Proxy 进行的数据劫持

```js ! reactive.js
function reactive(obj) {
  return new Proxy({
    get(target, key, receiver) {
      const res = Reflect.get(
        target,
        key,
        receiver,
      )
      // 劫持 get 操作
      // TODO: track()

      if (isObject(res)) {
        return reactive(res)
      }

      return res
    },
    set(target, key, newValue, receiver) {
      const res = Reflect.set(
        target,
        key,
        newValue,
        receiver,
      )
      // 劫持 set 操作
      // TODO: trigger()
      return res
    },
  })
}
```

## !!steps 依赖是什么？

在 get 操作中，我们需要进行依赖的收集，在 set 操作中，我们需要进行依赖的触发。那依赖是什么呢？依赖其实就是 ReactiveEffect 的实例，我们只需要收集该实例，然后在数据变更时触发该实例的 run 方法即可

```js !
class ReactiveEffect {
  constructor(public fn: Function, public scheduler: Function | null = null) {}

  run() {
    const res = this.fn()

    return res
  }
}

export function effect(fn, options = {}) {
  const _effect = new ReactiveEffect(fn, options.scheduler)

  _effect.run()

  const runner = _effect.run.bind(_effect)
  runner.effect = _effect

  return runner
}

```

## !!steps 使用什么数据结构进行依赖的收集?

响应式数据 a = { name: "a", age: 10 }, 假如 effect 依赖了 a 中的 age 属性，那么当 a 的 name 属性发生变化时，effect 是不应该重新执行的。同样的 a.age 可能被多个 effect 依赖，那么当 a.age 修改时，这些依赖了 a.age 的所有 effect 都应该重新执行。所以选择一个合适的依赖收集的数据结构是非常重要的。

```js !
export const targetMap = new WeakMap()
/**
 * weakmap
 *  target -> map
 *    key -> set
 */
const a = { name: "a", age: 10 }
effect(() => console.log(a.age)) // effect1
effect(() => console.log(a.name)) // effect2
effect(() => console.log(a.name, a.age)) // effect3

/**
 * weakMap
 *  a -> map
 *    name -> [effect2, effect3]
 *    age -> [effect1, effect3]
 */
```

## !!steps 如何收集依赖？

在 get 操作里进行依赖收集，但是在 get 时，我们如何能知道当前依赖是什么？

```js !
export let activeEffect: ReactiveEffect | null = null
export const targetMap = new WeakMap()
class ReactiveEffect {
  constructor(public fn: Function, public scheduler: Function | null = null) {}
  run() {
    activeEffect = this
    const res = this.fn()
    activeEffect = null
    return res
  }
}
export function track(target: object, key: string) {
  if (activeEffect) {
    let depsMap = targetMap.get(target)
    if (!depsMap) {
      targetMap.set(target, (depsMap = new Map()))
    }
    let dep = depsMap.get(key)
    if (!dep) {
      depsMap.set(key, (dep = new Set()))
    }
    trackEffect(dep)
  }
}
export function trackEffect(dep: Set<any>) {
  dep.add(activeEffect)
}
export function triggerEffects(dep: Set<any>) {
  if (dep) {
    dep.forEach(effect => {
      if (effect.scheduler) {
        effect.scheduler()
      } else {
        effect.run()
      }
    })
  }
}
export function trigger(target: object, key: string) {
  const depsMap = targetMap.get(target)
  if (!depsMap) {
    return
  }
  const dep = depsMap.get(key)
  triggerEffects(dep)
}
```

## !!steps 如何实现 ref ?

```js !
function trackRefValue(ref: RefImpl) {
  if (activeEffect) {
    trackEffect(ref.dep || (ref.dep = new Set()))
  }
}

class RefImpl {
  public dep: Set<any>
  private _value: any
  constructor(value: any) {
    this._value = isObject(value) ? reactive(value) : value
  }

  get value() {
    trackRefValue(this)
    return this._value
  }

  set value(newValue) {
    if (!Object.is(newValue, this._value)) {
      this._value = isObject(newValue) ? reactive(newValue) : newValue
      triggerEffects(this.dep)
    }

  }
}


export function ref(value: any) {
  return new RefImpl(value)
}
```

## !!steps 如何实现 computed

computed 内部依赖了 ReactiveEffect 这个类，同时会传递 scheduler 参数

```js !
class ComputedRefImpl {
  public readonly effect: ReactiveEffect;
  private _value!: any;

  public _dirty = true
  private _getter: Function
  constructor(getter) {
    this._getter = getter
    this.effect = new ReactiveEffect(this._getter, () => {
      this._dirty = true
    });
  }

  get value() {
    if (this._dirty) {
      this._value = this.effect.run()
      this._dirty = false
    }
    return this._value
  }
}

export function computed(getterOrOptions: any) {
  const onlyGetter = typeof getterOrOptions === "function";

  let getter
  if (onlyGetter) {
    getter = getterOrOptions;
  }

  const cRef = new ComputedRefImpl(getter);

  return cRef as any;
}
```

## !!steps 如何实现 watch ?

```js !
export function watch(
  source,
  cb,
  options: any = {},
) {
  let getter, oldValue, newValue
  const job = () => {
    newValue = _effect.run()
    cb(newValue, oldValue)
    oldValue = newValue
  }
  if (typeof source === "function") {
    getter = source // () => obj.name
  } else {
    getter = () => traverse(source) // obj
  }
  const _effect = new ReactiveEffect(
    getter,
    () => {
      if (options.flush === "post") {
        const p = Promise.resolve()
        p.then(job)
      } else {
        job()
      }
    },
  )
  if (options.immediate === true) {
    job()
  } else {
    oldValue = _effect.run()
  }
}
function traverse(
  value,
  seen = new Set(),
) {
  if (
    typeof value !== "object" ||
    typeof value === null ||
    seen.has(value)
  )
    return

  seen.add(value)

  for (const key in value) {
    traverse(value[key], seen)
  }
  return value
}
```

## !!steps 如何实现视图异步更新（nextTick）

在 vue 中进行视图更新时是异步更新的，这样就是为什么在更新完 dom 之后立即获取 dom 的相关数据，获取到的会是旧数据。如果想要获取最新的数据我们可以通过 await nexttick 来获取最新的数据

```js !
const queue: any[] = []
let isFlushPending = false
const p = Promise.resolve()

export function nextTick(fn) {
  return fn ? p.then(fn) : p
}

export function queueJobs(fn) {
  if (!queue.includes(fn)) {
    queue.push(fn)
  }
  queueFlush()
}

function queueFlush() {
  if (isFlushPending) return
  isFlushPending = true
  nextTick(flushJobs)
}

function flushJobs() {
  isFlushPending = false
  let job
  while ((job = queue.shift())) {
    job && job()
  }
}
```

## !outro
