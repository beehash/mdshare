为什么要写这篇文章呢？干嘛不写？原因如下：

1. typescript目前是趋势，不管是react，angular都在积极引入typescript来增强框架的严格性，vue2也开发出了一些支持typescript的插件
		
    #####     比如vue-class-component， vue-property-decorator

2. 语法规范上，typescript的类型声明使代码开发更更清晰，更严谨。可以在很大程度上提升代码的开发水准。前有eslint作为前端规范代码工具，遭到广大开发者的喜爱. 

		况且大家都懂得，一个蹩脚的代码会让你有多恼火  >_>

3. typescript是javascript的超集，这意味着任何javascript都可以转化成typscript，而他对javascript包容性是极强的



或许你会说，typescript不是一个类型声明的支持嘛，又麻烦又不好用，但是基于以上原因
你还有理由去逃避现实吗？

## typescript 热点讨论

### 1. typescript 泛型
typscript实现了类似c++，java语言的泛型模式。本着可复用原则
泛型的实现确实可以省下不少的代码空间 >_>

具体使用如下
```

	1. 函数泛型类型：
    <T>（args:T）: T 
        
  2. 接口泛型类型：
    interface myInterface<T>{
      <T> (args: T)
    }
    
  3. 类泛型类型：
    class myclass<T> {
      name: <T>;
      add = (name: T) : T => T; 
    }
    
	
```


### 2. typescript 高级类型
```
1 交叉类型：
  将多个类型合并为一个类型
  
  特殊关键符号： &
  function myfunc <T, U>（first: T, second: U）: T & U {
    Coding is here
  }
    
 2. 联合类型
 	有时代码中需要传入多个指定的类型
    
 	特殊关键符号： ｜
  function myfunc (first: string, second: padding | string) : {
    Coding is here
  }
```


### 3. typescript 命名空间
```
namspace 定义的空间
	namespace mySpace {
    export class goat {
      name: 'goat',
    }
      
    export class fish {
      name: 'fish',
    }
  }
    
使用对应的namespace
  import mySpace from './myspace';
  const aa = new mySpace.fish();
  const bb = new mySpace.goat();
  
  console.log(aa.name); // fish
  console.log(bb.name); // goat
    
```


## vue中支持ts

### 1. vue-types类型介绍

为了支持typescript，vue官方也出了
关于typescript的类型声明，具体介绍一些比较常用的类型： 

```
vue Component：
    Component
    AsyncComponent
    ComponentOptions
 
vue props
	  PropType,
  	PropOptions,
  	ComputedOptions,

VNode
	  VNode,
  	VNodeComponentOptions,

```
具体情况使用哪种类型，请查看
https://github.com/vuejs/vue/tree/dev/types

另外vue-router，vuex也提供了类型支持


### 2.在vue配置中配置如下：

*.d.ts

```
import Vue,{ ComponentOptions } from 'vue'
import VueRouter, { Route, RawLocation, NavigationGuard } from 'vue-router'
import { Store } from 'vuex';

// ts支持vue
declare module '*.vue' {
  import Vue from 'vue';
  export default Vue;
}

// ts支持vue-router
declare module 'vue/types/vue' {
  interface Vue {
    $router: VueRouter
    $route: Route
  }
}

//ts 支持组件内路由守卫
declare module 'vue/types/options' {
  interface ComponentOptions<V extends Vue> {
    router?: VueRouter
    beforeRouteEnter?: NavigationGuard<V>
    beforeRouteLeave?: NavigationGuard<V>
    beforeRouteUpdate?: NavigationGuard<V>
  }
}

// vue支持store
declare module "vue/types/options" {
  interface ComponentOptions<V extends Vue> {
    store?: Store<any>;
  }
}

// vue支持组件内访问store树
declare module "vue/types/vue" {
  interface Vue {
    $store: Store<any>;
  }
}

```

从typescrip的应用角度的出发，还有更多的理由去使用他，我只是基本介绍下他的简单应用。
后续有更多心得体会也会及时更新上去的！

@@大家，如果有哪里写得不好的，请帮我指正下，我一定会在第一时间去改正的！

さよらら、皆さん


typescript因为目前项目中使用比较多，之前的使用得从一年半前说起了，写这篇文章，一方面再次复习下typescript，另一方面帮助有需要的同志迅速熟悉typescript在项目中的应用。文章主要从tyepscript热点和vue项目中的从相关使用配置情况作为切入点展开讨论的。有啥写得不好的，请同志们帮我纠正，谢谢啦



