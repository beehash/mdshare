学习完 vue3 的家伙们都知道，vue3 的响应式设计很酷炫，一个很大的特点是改变了旧有的 Object.defineProperty，换成了新的 es2015 中的 Proxy 来拦截数据变化。

那么，vue3是如何实现响应式设计的呢？让我们来看看下面这张图：
![这张图片](https://beehash.github.io/shared/images/21.5.6.1.png)



---



###### 从上面这张图我们再来看下其中涉及的几个函数吧
#####  reactive
> 这个函数也就是我们平常使用定义的 reactive API，主要的任务是创建一个响应式数据
```
export function reactive(target: object) {
  // 判断对象是否是 只读，若是只读，直接返回
  if (target && (target as Target)[ReactiveFlags.IS_READONLY]) {
    return target;
  }
  // 通过 createReactiveObject 创建响应式数据
  return createReactiveObject(
    target,
    false, // isreadOnly
    mutableHandlers, // baseHandlers
    mutableCollectionHandlers, // collectionHandlers
    reactiveMap // proxyMap: weakMap
  );
}
```
- createReactiveObject（）
> 这个函数将定义的 object 转化为了 Proxy 代理形式，同时，定义了属性获取或设置时的处理方法。
```
  function createReactiveObject(
    target: Target,
    isReadonly: boolean,
    baseHandlers: ProxyHandler<any>,
    collectionHandlers: ProxyHandler<any>,
    proxyMap: WeakMap<Target, any>
  ) {
  // 判断是否是对象类型，如果不是，直接返回
  // 对象已经是 proxy 或者 只读的对象直接返回
  // 判断该对象是否已经存在，已存在直接返回
  // 判断对象类型是否无效类型，有效类型为：COMMON：（Array、Object）  COLLECTION：（Map、weakMap、Set、weakSet），无效类型直接返回
  // 上述代码省略
  const proxy = new Proxy(
    target,
    targetType === TargetType.COLLECTION ? collectionHandlers : baseHandlers
  )
  proxyMap.set(target, proxy)
  return proxy
}
```
##### baseHandlers()
> baseHandlers处理器主要用来处理object、array类型的数据
```
export const mutableHandlers: ProxyHandler<object> = {
  get,
  set,
  deleteProperty,
  has,
  ownKeys
}
```
- collectionHandlers()
> 用来处理map、set、weakmap、weakset几种类型的数据
```
const mutableInstrumentations: Record<string, Function> = {
  get(this: MapTypes, key: unknown) {
    return get(this, key)
  },
  get size() {
    return size((this as unknown) as IterableCollections)
  },
  has,
  add,
  set,
  delete: deleteEntry,
  clear,
  forEach: createForEach(false, false)
}
```
- baseHandlers.get
> 当获取Object、Array类型数据时，会自动执行这个函数，返回数据，同时会执行 track 函数
```
function createGetter(isReadonly = false, shallow = false) {
  return function get(target: Target, key: string | symbol, receiver: object) {
    // 判断key是否等于'is_reactive' 'is_readonly' ,如果是返回isRedonly ，如果key等于'__v_raw' 返回 target 中的 key
    // 判断 target 是否是数组，如果是数组，使用 arrayInstrumentations 返回对应数据
    // 判断 key 是否是 Symbol 类型的 或者 没有可 track 的 key、或者 shallow、ref对象， 返回 target 中的 key
    const res = Reflect.get(target, key, receiver)
    // 上述代码省略
    ...
    if (!isReadonly) {
      track(target, TrackOpTypes.GET, key)
    }
    // 部分代码省略
    ...
    return res
  }
}
```

#### baseHandlers.set
> 当设置Object、Array类型数据时，会自动执行 set 函数，当属性被设置时会执行这个函数，并且执行trigger方法
```
function createSetter(shallow = false) {
  return function set(
    target: object,
    key: string | symbol,
    value: unknown,
    receiver: object
  ): boolean {
    let oldValue = (target as any)[key]
    // 获取value和oldvalue，如果是更改的值不是ref类型，原来的是ref类型，直接更改ref.value = 新的value，函数返回true
    // 判断target中是否有key属性， 如果没有 trigger 的type参数为 'add'，有参数为 'set'

    const result = Reflect.set(target, key, value, receiver)
    if (target === toRaw(receiver)) {
      if (!hadKey) {
        trigger(target, TriggerOpTypes.ADD, key, value) 
      } else if (hasChanged(value, oldValue)) {
        trigger(target, TriggerOpTypes.SET, key, value, oldValue)
      }
    }
    return result
  }
}
```
#### collectionHandlers.get
> 当获取map、set、weakmap、weakset类型的数据，执行这个函数顺便执行下 track
```
function get(
  target: MapTypes,
  key: unknown,
  isReadonly = false,
  isShallow = false
) {
  target = (target as any)[ReactiveFlags.RAW]
  const rawTarget = toRaw(target) // 取 target 的真实 value
  const rawKey = toRaw(key) // 如果是响应式取 key 的真实value
  if (key !== rawKey) {
    !isReadonly && track(rawTarget, TrackOpTypes.GET, key)
  }
  !isReadonly && track(rawTarget, TrackOpTypes.GET, rawKey)
  // 返回数据
}
```
#### collectionHandlers.set
> 当改变map、set、weakmap、weakset类型的数据时，执行这个函数 trigger
```
function set(this: MapTypes, key: unknown, value: unknown) {
  value = toRaw(value)
  const target = toRaw(this)
  const { has, get } = getProto(target)
  // 确认 key 和 toRaw(key) 是否存在在target中
  // 如果不存在, 执行 trigger 方法并且type参数为'add'
  // 如果存在，执行trigger 方法并且type参数为 'set'
  const oldValue = get.call(target, key)
  target.set(key, value)
  if (!hadKey) {
    trigger(target, TriggerOpTypes.ADD, key, value)
  } else if (hasChanged(value, oldValue)) {
    trigger(target, TriggerOpTypes.SET, key, value, oldValue)
  }
  return this
}
```
#### track
这个函数在
```
activeEffect.options.onTrack({
  effect: activeEffect,
  target,
  type,
  key
})

export function track(target: object, type: TrackOpTypes, key: unknown) {
  // 如果当前 track 状态为 disable， activeEffect 为 undefined，直接返回null
  // 判断当前的target是否被记录，若没有，则创建一个空 Map
  let depsMap = targetMap.get(target)
  if (!depsMap) {
    targetMap.set(target, (depsMap = new Map()))
  }
  // 判断当前target中是否存在记录的key，若没有 创建一个为空的 Set
  let dep = depsMap.get(key)
  if (!dep) {
    depsMap.set(key, (dep = new Set()))
  }
  // 判断当前的dep是否存在activeEffect，若没有，添加一个activeEffect，将当前状态记录到activeEffect的deps中，如果activeEffect存在effect，并且存在onTrack，执行onTrack，记录当前的依赖。
  if (!dep.has(activeEffect)) {
    dep.add(activeEffect)
    activeEffect.deps.push(dep)
    if (__DEV__ && activeEffect.options.onTrack) {
      activeEffect.options.onTrack({
        effect: activeEffect,
        target,
        type,
        key
      })
    }
  }
}
```
#### trigger
```
effect.options.onTrigger({
  effect,
  target,
  key,
  type,
  newValue,
  oldValue,
  oldTarget
})


export function trigger(
  target: object,
  type: TriggerOpTypes,
  key?: unknown,
  newValue?: unknown,
  oldValue?: unknown,
  oldTarget?: Map<unknown, unknown> | Set<unknown>
) {
  const depsMap = targetMap.get(target)
  // 如果不存在target， 直接返回
  // 此处代码省略

  // 创建一个effect存储器Set
  const effects = new Set<ReactiveEffect>()
  // 为effects 添加 effect 元素
  const add = (effectsToAdd: Set<ReactiveEffect> | undefined) => {
    if (effectsToAdd) {
      effectsToAdd.forEach(effect => {
        if (effect !== activeEffect || effect.allowRecurse) {
          effects.add(effect)
        }
      })
    }
  }

  // 将target中的每一个key 的effect 存入 effects中
  // 此处代码省略

  const run = (effect: ReactiveEffect) => {
    if (__DEV__ && effect.options.onTrigger) {
      effect.options.onTrigger({
        effect,
        target,
        key,
        type,
        newValue,
        oldValue,
        oldTarget
      })
    }

    // 如果有 scheduler属性，暂时存入 effect栈中，如果没有，直接执行
    if (effect.options.scheduler) {
      effect.options.scheduler(effect)
    } else {
      effect()
    }
  }

  // 执行effects中的effect
  effects.forEach(run);
}
```

这几个函数，是响应式设计实现的关键，以proxy 拦截数据变化，通过 track、trigger 及时通知更新变化。通过weakmap 来将所有的响应式数据存放起来，然后通过 map 数据类型来记录要记录的数据， set 数据类型存放 key 属性的 effect。






