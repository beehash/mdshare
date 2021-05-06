学习完 vue3 的家伙们都知道，vue3 的响应式设计很酷炫，一个很大的特点是改变了旧有的 Object.defineProperty，换成了新的 es2015 中的 Proxy 来拦截数据变化。

那么，vue3是如何实现响应式设计的呢？让我们来看看这几个关键函数：
- reactive（）
> 这个函数也就是我们平常使用定义的 reactive API，主要的任务是创建一个响应式数据
- createReactiveObject（）
> 这个函数将定义的 object 转化为了 Proxy 代理形式，同时，定义了属性获取或设置时的处理方法。
```
// 将对象设置成代理形式
  const proxy = new Proxy(
    target,
    targetType === TargetType.COLLECTION ? collectionHandlers : baseHandlers
  )
```
- baseHandlers()
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
- baseHandlers.get
> 当获取Object、Array类型数据时，会自动执行这个函数，返回数据，同时会执行 track 函数
```
const targetIsArray = isArray(target)
if (!isReadonly) {
      track(target, TrackOpTypes.GET, key);
}
```
- baseHandlers.set
> 当设置Object、Array类型数据时，会自动执行 set 函数，当属性被设置时会执行这个函数，并且执行trigger方法
```
      if (!hadKey) {
        trigger(target, TriggerOpTypes.ADD, key, value)
      } else if (hasChanged(value, oldValue)) {
        trigger(target, TriggerOpTypes.SET, key, value, oldValue)
      }
```
- collectionHandlers.get
> 当获取map、set、weakmap、weakset类型的数据，执行这个函数顺便执行下 track
- collectionHandlers.set
> 当改变map、set、weakmap、weakset类型的数据时，执行这个函数 trigger
- track
这个函数会为属性添加标记，并且执行生命周期
```
      activeEffect.options.onTrack({
        effect: activeEffect,
        target,
        type,
        key
      })
```
- trigger
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
```
这几个函数，是响应式设计实现的关键，以proxy 拦截数据变化，通过 track、trigger 及时通知更新变化。通过一个 weakmap 来将所有的响应式数据存放起来，然后通过 map 数据类型来记录要记录的数据， set 数据类型存放 key 属性的 effect。

