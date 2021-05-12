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
  // 上述代码省略
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
effect 中设定了一个targetMap，用于保存各个对象的依赖，track函数便是用来收集依赖的。
```
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
  // 判断当前的dep是否存在activeEffect，若没有，添加一个activeEffect，将当前状态记录到activeEffect的deps中，如果activeEffect存在effect，并且存在onTrack，执行onTrack。
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
track 函数收集了依赖，那么 trigger 便是执行了依赖，代码如下：
```
export function trigger(
  target: object,
  type: TriggerOpTypes,
  key?: unknown,
  newValue?: unknown,
  oldValue?: unknown,
  oldTarget?: Map<unknown, unknown> | Set<unknown>
) {
  const depsMap = targetMap.get(target)
  // 如果不存在 target，直接返回
  // 此处代码省略

  // 创建一个 effect 存储器 Set
  const effects = new Set<ReactiveEffect>()
  // 创建add函数 为 effects 添加 依赖元素
  const add = (effectsToAdd: Set<ReactiveEffect> | undefined) => {
    if (effectsToAdd) {
      effectsToAdd.forEach(effect => {
        if (effect !== activeEffect || effect.allowRecurse) {
          effects.add(effect)
        }
      })
    }
  }

  // 将target中的每一个key 的 依赖 存入 effects中
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

    // 如果有 scheduler属性，执行scheduler，如果没有，直接执行
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
#### 小测试
为了能显著看到 vue2 到 vue3 的性能提升效果，我这里加了一组测试数据。
本地 mock 了一些数据测试效果，同样的代码在vue2 和 vue3 的项目里面运行：

`由于单独计算响应式的时间太小，这段代码实际测试的时间是包括渲染到dom结束的时间`

![测试小代码](https://beehash.github.io/shared/images/21.5.12.0.png)

##### 结果：
> 1000 条数据情况下

|     | vue3 | vue2 |
| --- | ---  | ---  |
|  1  |57.30615234375 ms      |344.779052734375 ms      |
|  2  |58.239990234375 ms      |311.524169921875 ms      |
|  3  |54.858154296875 ms      |345.263916015625 ms      |
|  4  |50.19482421875 ms      |323.7578125 ms      |
|  5  |71.603271484375 ms      |344.85986328125 ms      |
|  6  |53.6259765625 ms      |343.16796875 ms      |

> 500 条数据情况下

|     | vue3 | vue2 |
| --- | ---  | ---  |
|  1  |36.75927734375 ms      |183.14111328125 ms      |
|  2  |32.89794921875 ms     |186.121337890625 ms      |
|  3  |37.52880859375 ms      |199.150146484375 ms      |
|  4  |36.14013671875 ms      |177.706787109375 ms      |
|  5  |35.85498046875 ms      |220.203857421875 ms      |
|  6  |48.372802734375 ms      |189.227783203125 ms      |

> 300条数据情况下

|     | vue3 | vue2 |
| --- | ---  | ---  |
|  1  |27.139892578125 ms      |133.0322265625 ms     |
|  2  |25.74072265625 ms      |131.471923828125 ms    |
|  3  |25.9609375 ms      |143.6669921875 ms    |
|  4  |25.519775390625 ms      |137.19287109375 ms     |
|  5  |25.31787109375 ms      |192.2646484375 ms     |
|  6  |42.575927734375 ms      |132.372314453125 ms     |

> 100 条数据情况下

|     | vue3 | vue2 |
| --- | ---  | ---  |
|  1  |7.583251953125 ms      |54.409912109375 ms      |
|  2  |7.73095703125 ms      |59.298095703125 ms      |
|  3  |8.09814453125 ms     |59.15625 ms      |
|  4  |10.105712890625 ms      |54.1337890625 ms      |
|  5  |7.506103515625 ms      |72.745849609375 ms      |
|  6  |7.953369140625 ms      |52.799072265625 ms      |

> 50 条数据情况下

|     | vue3 | vue2 |
| --- | ---  | ---  |
|  1  |4.31787109375 ms      |34.56298828125 ms      |
|  2  |5.32568359375 ms      |30.47021484375 ms     |
|  3  |4.961181640625 ms      |33.158935546875 ms      |
|  4  |5.325927734375 ms      |30.56787109375 ms      |
|  5  |5.2119140625 ms      |31.27978515625 ms      |
|  6  |4.718994140625 ms      |35.82080078125 ms      |

> 10 条数据情况下

|     | vue3 | vue2 |
| --- | ---  | ---  |
|  1  |1.635009765625 ms      |13.67578125 ms     |
|  2  |2.39111328125 ms      |11.723876953125 ms     |
|  3  |1.657958984375 ms      |11.34814453125 ms      |
|  4  |1.595947265625 ms      |12.61572265625 ms     |
|  5  |1.963134765625 ms      |11.649658203125 ms      |
|  6  |2.4208984375 ms      |11.113037109375 ms      |

> 3 条数据情况下

|     | vue3 | vue2 |
| --- | ---  | ---  |
|  1  |1.037841796875 ms      |8.458251953125 ms     |
|  2  |3.235107421875 ms      |8.485107421875 ms      |
|  3  |2.16015625 ms      |8.08203125 ms      |
|  4  |1.337890625 ms      |8.10595703125 ms      |
|  5  |1.172119140625 ms      |8.68017578125 ms      |
|  6  |0.999755859375 ms      |10.0869140625 ms      |



##### 截屏数据
> ***1000条数据情况下：***

> `vue3`

![1000条数据下的 vue3 测试效果](https://beehash.github.io/shared/images/21.5.12.1.png)

> `vue2`

![1000条数据下的 vue2 测试效果](https://beehash.github.io/shared/images/21.5.12.2.png)

> ***再来看看100 条数据情况下：***

> `vue3`

![100条数据下的 vue3 测试效果](https://beehash.github.io/shared/images/21.5.12.3.png)

> `vue2`

![100条数据下的 vue2 测试效果](https://beehash.github.io/shared/images/21.5.12.4.png)


测试结果很明显地看到，在数据量庞大的情况下，vue2 的数据渲染速度已经很慢了，但 vue3 的速度还是很快，几乎能在 1 ms 之内完成。
当然在数据量较少的情况下，vue2 和 vue3 的差别不会太大。

#### 在结尾
很明显 vue3 的响应式设计，是对 vue2 的观察者模式的重新定义，弥补了之前的不完美，实现了数据拦截，代码，性能上的提升。
对比 vue2 的观察者模式，我们可以发现，defineProperty 对对象的拦截需要遍历属性，proxy则是对象整体进行监听，
在遇见多层嵌套的情况下时，defineProperty 进行整体递归，消耗的性能对比 proxy 太大了。除此之外，defineProperty 实现是以类来实现的，也就是在递归时 需要 new 实例化整个 Observer，Proxy 不会这样。

`小知识`
- 为什么使用Proxy 替换 deineProperty 数据拦截机制？
> 因为 Proxy 提供更完善的数据拦截操作，不仅有 get、set 还有 defineProperty 所没有的 has、
> deleteProperty、ownKeys、defineProperty等属性。defineProperty 的操作只限于拦截 get、set操作。
> 并且，vue2 之前讲过 defineProperty 的一些限制，如数组操作直接索引改变不生效、不能修改数组的长度。
