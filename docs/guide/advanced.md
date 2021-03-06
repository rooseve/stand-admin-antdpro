---
title: 进阶使用
order: 3
toc: menu
---

## 提示文案

StandAdmin 会根据配置信息拼接一些显示内容，比如：

|                                      | 文案来源                                                            | 效果示例                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| ------------------------------------ | ------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 新增、编辑、删除成功提示、弹窗 title | `recordModel`中配置的`StoreNsTitle`、`nameFieldName`、`idFieldName` | <img src="https://gw.alipayobjects.com/mdn/rms_9ac13c/afts/img/A*3vCxRrYitNcAAAAAAAAAAAAAARQnAQ" /><br/><img src="https://gw.alipayobjects.com/mdn/rms_9ac13c/afts/img/A*k7NwSIuMZq0AAAAAAAAAAAAAARQnAQ" /><br/><img src="https://gw.alipayobjects.com/mdn/rms_9ac13c/afts/img/A*2JA3RLftzCkAAAAAAAAAAAAAARQnAQ" /><br/><img src="https://gw.alipayobjects.com/mdn/rms_9ac13c/afts/img/A*rKS8QrcTqpQAAAAAAAAAAAAAARQnAQ"/><img src="https://gw.alipayobjects.com/mdn/rms_9ac13c/afts/img/A*0GY8S4mAyv8AAAAAAAAAAAAAARQnAQ"/> |
| `callService`成功提示                | `callService`传入的`serviceTitle`参数                               | <img src="https://gw.alipayobjects.com/mdn/rms_9ac13c/afts/img/A*ZHJFTIgwylQAAAAAAAAAAAAAARQnAQ" />                                                                                                                                                                                                                                                                                                                                                                                                                          |
| 请求出错提示                         | 失败请求（`success:false`）返回的 `message` 字段                    | <img src="https://gw.alipayobjects.com/mdn/rms_9ac13c/afts/img/A*6J-kQpJ86XsAAAAAAAAAAAAAARQnAQ" />                                                                                                                                                                                                                                                                                                                                                                                                                          |

<a id="statemanagement"></a>

## 状态管理

### 状态存储

StandAdmin 内部使用[Dva](https://dvajs.com/guide/concepts.html)做状态管理，`StoreNs`其实就是 Dva 的 `namespace`。

简单来说，存在一个全局的 `state` 对象，`StoreNs`就是访问这个 `state` 的 key，而 value 就是[`context.storeRef`](/api#IStoreRef)。

状态改变通过 context 中的 [API](https://standadmin.github.io/stand-admin-antdpro-demo/#/stand-admin-antdpro-demo/admin-demo/big-context) 进行，比如调用`context.goSearch(params)`会触发`storeRef`中的`records`、`searchParams`发生变化，进而触发 re-render，整体上是一种[受控模式](https://reactjs.org/docs/forms.html#controlled-components)。

### 全局 `state` 的优劣

直接看代码：

```jsx | pure
const recordModel = buildStandRecordModelPkg({
  StoreNs: 'DemoRecord',
  ///...
});
const AdminComp = StandContextHoc({ recordModel })(MainComp);
```

`StoreNs`是`recordModel`的一个属性，`recordModel` 又是组件定义（`AdminComp`）的一部分，这样多个组件实例`<AdminComp />`其实是使用同一个`StoreNs`，或者说，共享同一份`state`数据（即 `storeRef`）。

全局`state`有其优点：

- 可以方便的共享、监听数据，不需要额外的[状态提升](https://reactjs.org/docs/lifting-state-up.html)或者事件机制
- 对热更新比较友好，局部刷新后通常不会出现状态丢失

但也有个明显的缺点：

- 共用一个`StoreNs`的多个组件实例会[相互影响](https://standadmin.github.io/stand-admin-antdpro-demo/#/stand-admin-antdpro-demo/admin-demo/same-ns)（毕竟都是受同一份状态数据的控制），很多场景下这并不是期望结果。

<a id="cloneModelPkg"></a>

### 数据空间隔离

利用[`cloneModelPkg`](/api#clonemodelpkg)或者[`getDynamicModelPkg`](/api#getdynamicmodelpkg)（亦可使用[`StandContextHoc`](/api#definecontexthocparams)的`makeRecordModelPkgDynamic`参数）复制出一个新的`recordModel`，除了`StoreNS`，其他属性一模一样。

一个简单的例子，两个规则列表，逻辑一致，只是一个限制`state = 1`的，一个限制`state = 2`的。

**<font color="red">错误</font>做法，简单写两个实例**

```jsx | pure
const render = () => (
  <>
    {/* 错误做法，实例之间会相互影响 */}
    <AdminComp specSearchParams={{ status: 1 }} />
    <AdminComp specSearchParams={{ status: 2 }} />
  </>
);
```

**<font color="green">正确</font>做法，利用`makeRecordModelPkgDynamic`创建不同的组件**

[示例](https://standadmin.github.io/stand-admin-antdpro-demo/#/stand-admin-antdpro-demo/admin-demo/multi-ns)，[代码](http://github.com/StandAdmin/stand-admin-antdpro-demo/blob/main/src/pages/Demos/MultiNs/index.js)

```jsx | pure
const DynamicCompCache = {};
const getDynamicComp = namespace => {
  if (!DynamicCompCache[namespace]) {
    DynamicCompCache[namespace] = StandContextHoc({
      recordModel,
      makeRecordModelPkgDynamic: namespace, // 创建新的recordModel
    })(MainComp);
  }

  return DynamicCompCache[namespace];
};

const AdminCompA = getDynamicComp('A');
const AdminCompB = getDynamicComp('B');

const render = () => (
  <>
    {/* 正确做法 */}
    <AdminCompA specSearchParams={{ status: 1 }} />
    <AdminCompB specSearchParams={{ status: 2 }} />
  </>
);
```

<a id="searchParams"></a>

## 查询参数

### 参数构建

查询参数有几个来源：

- defaultSearchParams，默认的查询参数，通过[defineContextHocParams](/api#definecontexthocparams)或者组件的 [prop](/api#standcontexthoc) 设置。
- activeSearchParams，当前查询参数，来自查询 Form 或者 Url（[`syncParamsToUrl`](/api#definecontexthocparams) 开启）
- specSearchParams，强制指定的查询参数，设置方式同 defaultSearchParams。

三者的优先级从低到高，拼合后还会做一次[名称转换](/guide/service#分页参数)，再传递给[`searchRecords`](/api#buildstandrecordmodelpkg)方法。

### Url 参数同步

[`syncParamsToUrl`](/api#definecontexthocparams) 开启后：

- 查询（调用`context.goSearch`）功能会把计算出的参数编码（[`stringifyQueryParams`](/api#standutils)）后 push 到 url 上去

  > `stringifyQueryParams`是 StandAdmin 的内置实现，可以保持字段类型（支持 moment 格式），比如
  >
  > ```javascript | pure
  > { a: 1, b: 'str', c: { d: moment() }
  >  // 编码的结果是： _j_c=%7B%22d%22%3A%22_moment%3A1622535359583%22%7D&_n_a=1&b=str
  > ```

* 组件 didMount 时，会解码 url 中的参数当做 activeSearchParams

### 参数转换

查询表单值和接口参数常常不一致。比如日期，表单值通常是 `moment` 类型，接口参数往往是 `string` 类型；另外还有些复杂的场景，查询表单和接口参数差异巨大，查询表单的一个 switch 切换可能对应着接口的一批参数。

做参数转换有两个位置可供选择：

- 在[`searchRecords`](/api#buildstandrecordmodelpkg)处做[单向转换](https://admin-demo.abf.alibaba-inc.com/admin-demo/weird-query)，适用于转换逻辑偏**复杂**的场景。
- 在查询表单中利用[useStandSearchForm](/api#usestandsearchform)中做[双向转换](https://admin-demo.abf.alibaba-inc.com/admin-demo/data-convert-search)，适用于转换逻辑偏**简单**的场景。

<a name="FormHistroy"></a>

## 表单草稿

StandAdmin 内置实现了一个表单草稿功能，UI 效果如下：

<img src="https://gw.alipayobjects.com/mdn/rms_9ac13c/afts/img/A*Gc1uT59vyOMAAAAAAAAAAAAAARQnAQ"/>
<img src="https://gw.alipayobjects.com/mdn/rms_9ac13c/afts/img/A*GmHdRZOB2QAAAAAAAAAAAAAAARQnAQ"/>

### 调用方法

`useStandUpsertForm`、`useStandSearchForm`返回了一个函数：`renderFormHistroyTrigger`，直接调用后即可输出草稿功能的 UI

```jsx | pure
export default props => {
  const {
    formProps,
    modalProps,
    renderFormHistroyTrigger, // 表单草稿的render方法
  } = useStandUpsertForm({
    ...getOptsForStandUpsertForm(props),
  });

  return (
    <Modal
      // forceRender
      {...modalProps}
    >
      <div style={{ float: 'right' }}>
        {/* 输出草稿功能的UI */}
        {renderFormHistroyTrigger()}
      </div>
      <Form {...formProps}>{/* 表单内容*/}</Form>
    </Modal>
  );
};
```

### 存储位置

表单草稿的 CRUD 接口使用 [localforage](https://www.npmjs.com/package/localforage) 实现，默认存储在在浏览器本地的 IndexedDB 中

<img src="https://gw.alipayobjects.com/mdn/rms_9ac13c/afts/img/A*HMYtSqp0EywAAAAAAAAAAAAAARQnAQ" />
