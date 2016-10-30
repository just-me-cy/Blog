---
title: reselect实践
tags: ['reselect','redux']
categories: ['redux']
---

## 目标
最近在cms系统中使用了`redux`，由于系统主要用于产品的编辑配置，用户交互比较多，`store`承载的数据量比较大，采用了[`reselect`](https://github.com/reactjs/reselect)。
使用`reselect`出于以下两点考虑：
1. 优化性能
  如果有从state中大量计算衍生数据的需求，即父组件state通过计算作为props传递给子组件，那么state的任何变化都会引起重新计算。
  通过reselect可以创建带有记忆的selector，只有相关属性变化了才去重新计算衍生数据.
2. 代码组织
  通过reselect创建了数据处理抽象层,专门处理state到子组件的映射。

### 官方对reselect的说明：
> 1. Selectors can compute derived data, allowing Redux to store the minimal possible state.
> 2. Selectors are efficient. A selector is not recomputed unless one of its arguments change.
> 3. Selectors are composable. They can be used as input to other selectors.

使用`selector`可以提高`react`和`redux`的性能
    `react`本身是基于虚拟节点diff来更新相应ui，没有dom的增删改查，性能已经相当好了。为了组件的更新及时反映在屏幕上，`react`都要重新执行渲染周期，但是随着ui复杂度的逐渐提升，过多的更新也会成为性能杀手。react也提供了`componentShouldUpdate`，在此方法内将上一个`props`、`state`和新的`props`、`state`作对比，去决定是否去执行更新，`PureRenderMixin`也是在基础上实现的。
    如果子组件的`props`是父组件通过计算后传递下去的，每当父组件更新，那么要重新计算。可以使用`reselect`省去没必要的重新计算.

## reselectAPI
### createSelector

> createSelector(...inputSelectors | [inputSelectors], resultFunc)

1. 参数`inputSelectors`可是逗号分隔的`selectors`，也可是由`selectors`组成的数组，每当`state`变化了，就会执行`selectors`定义的计算，计算结果作为`resultFunc`的参数。
2. `resultFunc`也就是转换函数
**主要流程**：如果 `state tree`的改变会引起`inputselectors`值变化，那么`selector`会调用转换函数，传入`inputselectors`作为参数，并返回结果。如果`inputSelectors`的值和前一次的一样，它将会直接返回前一次计算的数据，而不会再调用一次转换函数。
  
  ```javascript
    const mySelector = createSelector(
      state => state.values.value1,
      state => state.values.value2,
      (value1, value2) => value1 + value2
    )

    // You can also pass an array of selectors
    const totalSelector = createSelector(
      [
      state => state.values.value1,
      state => state.values.value2
      ],
      (value1, value2) => value1 + value2
    )
  ```

  `createSelector`内部，默认通过===比较判断，由`inputSelector`选择的值和上一次的值有无变化，所以`inputSelectors`的参数要是`immutable`的。
  `createSelector`源码如下，可知`createSelector`是由`createSelectorCreator`指定了默认比较函数创建的偏函数。

  ```javascript
    export const createSelector = createSelectorCreator(defaultMemoize)
  ```
  通过`createSelector`创建的`selector`缓存大小为1，也就说当参数中任意`inputSelectors`的返回值变化了，结果都会重新计算。
  
### defaultMemoize
> defaultMemoize(func, equalityCheck = defaultEqualityCheck)

1. 如果不指定比较函数，那么使用`defaultEqualityCheck`，`createSelector`默认使用的缓存函数。`defaultMemoize`的缓存大小为1，当任意参数的变化了会重新执行计算。
2. equalityCheck是用来决定参数是否变化的辅助函数

  ```javascript
  function defaultEqualityCheck(a, b) {
    return a === b
  }
  ```
  
### createSelectorCreator
> createSelectorCreator(memoize, ...memoizeOptions)

1. 此方法是`createSelector`的自定义版本，`memoize`是自定义的记忆函数，用来替换`defaultMemoize`
2. `...memoizeOptions`是额外的参数，会被传递给记忆函数。记忆函数的第一个参数是resultFunc。
看源码更容易理解
```javascript
export function createSelectorCreator(memoize, ...memoizeOptions) {
  return (...funcs) => {
    let recomputations = 0
    const resultFunc = funcs.pop()
    const dependencies = getDependencies(funcs)

    const memoizedResultFunc = memoize(
      (...args) => {
        recomputations++
        return resultFunc(...args)
      },
      ...memoizeOptions
    )

    const selector = (state, props, ...args) => {
      const params = dependencies.map(
        dependency => dependency(state, props, ...args)
      )
      return memoizedResultFunc(...params)
    }

    selector.resultFunc = resultFunc
    selector.recomputations = () => recomputations
    selector.resetRecomputations = () => recomputations = 0
    return selector
  }
}
```
### createStructuredSelector
> createStructuredSelector({inputSelectors}, selectorCreator = createSelector)
  

## 例子
  比如有个列表
  ![reselect例子](../../../../images/select-160831.png)
  引入相应的函数
  ```javascript
    ...
    import { createSelectorCreator, defaultMemoize } from 'reselect';
    import { isEqual } from 'lodash';
  ```
  创建2个selector
  ```javascript
  const prosSelector = (state) => state.product.productsByQuery.get('items');
  const filterSelector = (state) => state.product.visibilityFilter;
  ```
  使用`lodash.isEqual`比较函数代替默认的`===`比较，还是使用默认的`defaultMemoize`，只是加上了额外的参数`lodash.isEqual`。

  ```javascript
  const createDeepEqualSelector = createSelectorCreator(
    defaultMemoize,
    isEqual
  );
  const selectPros = createDeepEqualSelector(
    [prosSelector, filterSelector],
    (pros, filter) => {
      console.log('重新计算select', pros);
      switch (filter) {
        case Filters.SHOW_ALL:
          return pros;
        case Filters.SHOW_CHANGE:
          return pros.filter(prosItem => prosItem.get('statusCode') === 'change');
        case Filters.SHOW_SHELVE:
          return pros.filter(prosItem => prosItem.get('statusCode') === 'shelve');
        case Filters.SHOW_OFF_SHELVE:
          return pros.filter(prosItem => prosItem.get('statusCode') === 'offShelve');
        case Filters.SHOW_NEW:
          return pros.filter(prosItem => prosItem.get('statusCode') === 'new');
        case Filters.SHOW_OUT_OF_SALE:
          return pros.filter(prosItem => prosItem.get('statusCode') === 'outOfSale');
        case Filters.SHOW_CONFIRM:
          return pros.filter(prosItem => prosItem.get('statusCode') === 'confirm');
        default :
          return pros;
      }
    }
  );
  ```
  在`mapStateToProps`返回一个映射对象，为子组件准备适合自己需要的状态视图。

  ```javascript
  function mapStateToProps(state) {
    ...
    return {
      visiblePros: selectPros(state),
    };
  }
  ```
  在render中将visiblePros传给子组件
  ```javascript
  <ProductLists pros={visiblePros} />
  ```
  只有当决定子组件`props`的`prosSelector`，`filterSelector`任一返回值和上次不相同了，`selectPros`的回调函数才会执行，`visiblePros`才会更新，才会触发子组件`ProductLists`的`react`渲染周期，而在其它部分（非相关）变化时不做计算

  在实际项目中，可以组合使用`selectors`，带来的优点的让代码组织非常清晰。



参考：
https://github.com/reactjs/reselect
http://redux.js.org/docs/recipes/ComputingDerivedData.html












































