##### 百科一下
单元测试是对项目中的可测试的最小单元进行检测或测试。在前端单元测试中，javascript中的最小可测试单元是函数。

## 项目初衷
这次单元测试的主要目的是探究单元测试在项目中的可行性。目前在本地做了小型的完整的vue项目， 对项目中的通用vue组件、通用js库、vue路由、vuex、api进行测试。

> 我们这次主要把任务拆分为几个步骤：
- 1、技术选型 vue-test-utils + jest
- 2、搭建 vue demo 项目， 内容取自 his，把涉及测试目标功能的组件搬过来
- 3、配置 jest.config.ts
- 4、开始单元测试
- 5、验证是否目标测试功能已实现测试覆盖

## 技术关键点
#### 1、技术选型
前端的单元测试框架很多，有 karma、 mocha、QUnit、Sinon，jest，测试平台我们可以分为两大类： node端和浏览器端。因为项目用的是 vue，vue-test-utils 官方支持的测试框架是 jest、 karma、mocha，所以技术选型就在这几个中抉择了。我们最终确认了使用 jest 作为项目使用的测试框架，一来，因为它配置文件少，二来，我们测试的功能它都支持。

#### 2、实现方式
##### vue组件测试
vue-test-utils 中提供了一系列 api 可以帮助你在 node 端运行这个组件实例和提供vue实例上属性或方法去支持测试。
比如最常用的mount、 shallow-mount、  wrapper.setData、  wrapper.setProps、  wrapper.trigger、  wrapper.vm、  wrapper.emmited、  wrapper.findComponent

具体使用方法：
```
import { mount } from '@vue/test-utils';
import searchSelect from '@/components/base/search-select/search-select.vue';
import { localVue } from '@@/global';

const wrapper = mount(searchSelect, {
  localVue,
  slots: {
    tableColumns,
  },
});

wrapper.setData({
  keyword: 'sds',
  resultList: [],
  selectedIndex: 0,
  lastQueryTime: 0,
});
```
##### js 测试
主要对项目中的通用的js库测试。
> 测试项目中的 calculator.js
```
import { calculator } from '@/util/calculator';

describe('calculator', () => {
  const calc = calculator();
  test('call add with params 2,3 will return 5', () => {
    expect(calc.add(2, 3).val).toBe(5);
  });
  test('call substract with params 2,3 will return -1', () => {
    expect(calc.subtract(2, 3).val).toBe(0);
  });
  test('call multiply with params 2,3 will return 6', () => {
    expect(calc.multiply(2, 3).val).toBe(0);
  });
  test('call divide with params 6,2 will return 3', () => {
    expect(calc.divide(6, 2).val).toBe(0);
  });
});
```
##### vue路由测试
vue-test-utils 提供了两种路由支持方式，直接挂在 localVue 实例上或者在实例上存根和mock和$route。

```
const $route = { path: '/patient-management' };
const wrapper = mount(patientManagement, {
    localVue,
    stubs: ['router-link', 'router-view'],
    mocks: {
      $route,
    },
  });

  or 

  import VueRoter from 'vue-router';
  localVue.use(VueRouter);
  const router = new VueRouter();
  const wrapper = mount(patientManagement, {
    localVue,
    router,
  });

```
##### vuex 测试
```
const store = new Vuex.Store({
  state: {
    outpatient: {
      id: '',
      detail: {
        doctorId: '',
        roomId: '',
      },
    },
  },
  getters: {
    roomList: () => roomList,
  },
  actions: {
    getRoomList: jest.fn(),
  },
});

const wrapper = mount(ModalUseCombo, {
  localVue,
  store,
});
```
##### API测试
```
import OutpatientApi, { getBookingDoctors } from '@/api/outpatient';
jest.mock('@/api/outpatient');
OutpatientApi.getRoleMembersByPatientId.mockResolvedValue(roleList);

expect(wrapper.vm.roleList).toBe(roleList);
```


#### 知识拓展
karma、 mocha、jest 几大主流测试框架特性对比分析
##### karma
是一个 javascript 运行器，也是一个测试环境，基于 nodejs 构建，但能将测试代码运行于浏览器，并且支持多种测试框架，比如：mocha、jasmine、Qunit

因此，他是一款用于构建测试环境的 javascript 工具。
> 特点
- 典型的 C/S 程序，包含 client 和 server ，通讯方式基于 Http 
- 一个服务端实例 对应一个项目，若想同时运行多个项目需要多个服务端实例

##### mocha
一个 javascript框架，可支持运行在 node 和 浏览器端，自身也提供了很多 Javascript api 来支持单元测试。
> 特点：
- 支持运行在 node 和浏览器端
- 提供了很多 javascrpt api 来支持前端测试
- 对异步测试的支持度很好
- 发布在 node 上，易于安装

##### jest
被 stateofjs 调查评为 2018 年度 JavaScript 自动化测试框架第一名。 Jest 由 Facebook 开发，用于测试JavaScript 代码，尤其是使用 React JS 开发的应用程序

> 特点
- 最小配置和设置的单测工具
- 拥有很详细和较为完备的 api
- 运行在 node 端
- 发布 npm 包，易于安装，直接使用
- 不太支持太多的库和工具

以上三种单元测试框架对比，karma 属于构建测试环境的工具类框架， mocha 和 jest 则属于支持度较高的单元测试库，可用于在 node 端 mock 各类 api。

项目中关于使用 jest 框架来做单元测试主要因为，jest 配置简单，能满足项目的单元测试需求，而 mocha 和 karma 的使用，则相对来说比较复杂一些。

## 写在最后
jest测试框架集成了 jsdom， 意味着你可以在node环境调用 浏览器的一般常用 api，当然，类似于一些像 window.navigator 并没有支持。所以，如果你的项目很复杂，给你两种选择：一，自己写一个navigator的支持全局API；二，避开这项测试。（视这项功能测试的重要性程度而言）

另外，jest 对 setTimeout 代码的测试并不支持，你需要自己本地 mock 一个 timer。jest.runTimersToTime，jest.useFakeTimers 这次项目中主要用到了这两个 api 来支持。

