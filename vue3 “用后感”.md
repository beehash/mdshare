
vue3的源码已经开放很久了，官网也火速出了新文档。但奈何我的速度总是能和蜗牛相比。正好，最近有做到vue3的项目。那我来写一篇 `“用后感”` 吧！把项目中学习到的关于vue3的理解分享给大家。

这里做一个简单的归纳，个人认为vue3中值得大家注意的点：
    
* 1、数据拦截机制的变更
* 2、组合API的引入
* 3、setup函数带来的一系列惊喜
* 4、响应式API的宽泛应用
* 5、Diff算法优化
* 6、vue3状态更新和服务端渲染性能测评

#### 一、数据拦截机制的变更
  vue2 通过 Object.defineProperty() 方式来截持数据的变更。而在升级为 vue3 之后，作者不再借助 Object.defineProperty 来劫持数据，而是通过 es6 的新特性 `proxy` 来劫持数据。因为在 vue2 中，响应式数据需要通过 data() 初始化预先定义好，但是在 vue3 之后，data 中的数据不再进行响应式追踪，而是将它转化为`proxy` 代理再进行追踪更新。
  
`注意`： 在vue3中，它将任何在data()函数中定义的数据，会使用getter或setter遍历所有的property，将其转化成proxy代理。


#### 二、组合API的引入
vue3中最大的改变还是compositionApi的引入，这是vue3中的最核心的部分。

 包括以下几方面：
>
> + a、setup函数
>
>  下一节有详细介绍，这里就不赘述了。

> + b、生命周期钩子
>
>  vue3抛出了全局的API，通过引入 `onX` 的形式，使setup函数中支持钩子函数的调用。

> + c、provide/Inject
>
> vue3 中也支持在 setup 函数中使用 provide和inject，只要在对应的位置引入即可：

	import { provide } from 'vue';
    setup(){
    	provide(name, 'hello!');
    }
   
    import {inject} from 'vue';
    setup(){
    	inject('name', [default-value]: string);
    }

> + d、getCurrentInstance
>
> 支持在setup函数中访问自己的实例
	
    import { getCurrentInstance } from 'vue';
	setup() {
    	const internalInstance = getCurrentInstance();
    }


#### 三、setup函数带来的一系列惊喜

在vue3中，setup函数的引入，这种函数式的开发方式，很好地将代码整合到一起。

* 在vue2中，data数据需要在使用的时候调用this访问，现在vue3中使用reactive，ref来设置响应式数据。

```
data() {
    return {
        name: 'aaa',
    };
}

=>

const name = ref('aaa');  
就像简单的定义变量一样，使用ref就可以为变量初始化
```

* setup函数取代了methods的函数定义方式，我们可以选择性地对外暴露我们定义的函数。在methods的函数，都绑定到了this上，`this得有多累啊 (> 3 >)`，很好地保护了我们自己定义的私有变量。


```
methods: {
    function unkouwnFunc() {
        console.log(3333);
    }
},

=>

setup () {
    function unkownFunc() {
        console.log(3333);
    }

    return {
        unkownFunc, // return之后，该函数绑定到了组件实例上，可以在模版中直接使用
    };
}
```

* 关于setup函数中使用emit事件

```
import { SetupContext } from 'vue';
  	
emits: ['eventName'],
setup(props, ctx: SetupContext){
    ctx.emit('event-name', value);
}
    
```

* 另外作者为了支持setup函数，向外抛出了全局的生命周期钩子，只在setup函数中使用有效。

```
import { onBeforeMount, onMounted, onBeforeUpdate, onUpdated, onBeforeUnmount, onUnmounted } from 'vue';

setup() {
    onBeforeMount() {
        your code
    }
    onMounted () {
        your code
    }
    onBeforeUpdate() {
        your code
    }
    onUpdated() {
        your code
    }
    onBeforeUnmount() { 
        your code
    }
    onUnmounted() {
        your code
    }
}
```
#### 四、响应式API的宽泛应用
* Ref和reactive API带来的便利

    vue3中的 ref的reactive 非常非常的好用，为什么这么说？因为vue在这里直接给出了他的全局API。不管在任何场景下，我们都可以直接引入，并创建这个响应式的数据。通过导入，导出，实现代码的 `集中化管理` 。

   所谓 `集中化管理`, 在这里指的是代码的运用灵活度提升了，vue2中，使用store来集中管理状态，而现在，我们可以直接真正抛弃store了。因为vue3的响应式API 给我们带来了非常nice的体验。

```
>> state.js 
    
export function useCount () {
    const count = ref(0);

    setCount (newv = 0) {
        count.value = newv;
    }

    return {
      count,
      setCount,
    };
}
        
=>
    
import {useCount} from 'state.js';
setup() {
    const { count } = useCount();
}
```
   
* 关于computed和watch在setup函数中的使用

```
computed: {
	aaa() => {
        	return 333
        }
    },
    watch: {
    	bbb(newv, oldv) {
        	console.log(newv, oldv);
        }
    },
}
    

=>
    
import { computed, watch } from 'vue';

const aaa = computed(() => {
  return 333;
})

const bbb = watch();
```

##### Diff算法优化
