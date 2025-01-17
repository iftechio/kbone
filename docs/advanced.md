## 代码优化

代码优化主要在于代码体积的精简和 dom 树的精简：

1. 代码体积精简

在编译到小程序代码的时候，会将整个 vue runtime 和一些 vue 特性插件（如 vue-router、vuex 等）给打包进来，这样会导致代码包比较庞大，而这些代码是无法去除的，因此得从业务代码上着手进行一些缩减。业务上存在一些代码可以用小程序接口替代，这部分是完全不需要打包进来的，因此可以使用一个行内 loader 和环境变量来进行代码的去除，简单做法如下：

```
npm install --save-dev reduce-loader
```

```js
import Vue from 'vue'
import ActionSheet from 'reduce-loader!./action-sheet' // 使用行内 loader，剔除 action-sheet 文件的引入

// 通过注入的环境变量判断代码运行环境，进而执行不同的逻辑
if (!process.env.isMiniprogram) {
    // web 端
    ActionSheet.show([1, 2, 3], success)
} else {
    // 小程序端
    wx.showActionSheet({
        itemList: [1, 2, 3],
        success,
    })
}
```

2. dom 树的精简

对于一些站点会使用响应式设计，即 pc 端和 h5 端会共用一套代码，通常 pc 端很多节点在 h5 端是不需要展示出来的，这就需要在样式上对节点设置 `display: none`，而这些节点仍旧存在于 dom 树上，只是不渲染在视图上。如果这套代码直接转成小程序代码，也必定会创建一些无需展示的 dom 节点，这些节点本身是可以直接剔除。

因此可以使用另外一个 loader 对这些节点进行删减，在 webpack 配置中 vue loader 执行之前再添加一个 `vue-improve-loader`：

```
npm install --save-dev vue-improve-loader
```

```js
{
    test: /\.vue$/,
    use: [
        {
            loader: 'vue-loader',
            options: {
                compilerOptions: {
                    preserveWhitespace: false
                }
            }
        },
        'vue-improve-loader',
    ]
},
```

然后在 vue 文件中给要剔除的节点加上 check-reduce 属性：

```html
<!-- 删减前代码 -->
<template>
    <div>
        <span>some text</span>
        <a check-reduce>
            <span>some text other</span>
        </a>
    </div>
</template>
```

因为 web 端代码构建和小程序端代码构建走不同的配置，所以 web 端代码会忽略这个属性，而小程序端代码则会删减掉带这个属性的节点。以下便是会输出给 vue-loader 的代码，从构建层面上剔除掉不需要的节点。

```html
<!-- 删减后代码 -->
<template>
    <div>
        <span>some text</span>
    </div>
</template>
```

> PS：vue-improve-loader 必须在 vue-loader 之前执行，这样 vue-loader 才会接收到被删减后的代码。

## 使用小程序内置组件

部分内置组件可以直接使用 html 标签替代，比如 input 组件可以使用 input 标签替代。目前已支持的可替代组件列表：

* `<input />` --> input 组件
* `<input type="radio" />` --> radio 组件
* `<input type="checkbox" />` --> checkbox 组件
* `<label><label>` --> label 组件
* `<textarea></textarea>` --> textarea 组件
* `<img />` --> image 组件
* `<video></video>`  --> video 组件
* `<canvas></canvas>` --> canvas 组件

还有一部分内置组件在 html 中没有标签可替代，那就需要使用 `wx-component` 标签或者使用 `wx-` 前缀，基本用法如下：

```html
<!-- wx-component 标签用法 -->
<wx-component behavior="picker" mode="region" @change="onChange">选择城市</wx-component>
<wx-component behavior="button" open-type="share" @click="onClickShare">分享</wx-component>

<!-- wx- 前缀用法 -->
<wx-picker mode="region" @change="onChange">选择城市</wx-picker>
<wx-button open-type="share" @click="onClickShare">分享</wx-button>
```

如果使用 `wx-component` 标签表示要渲染小程序内置组件，然后 behavior 字段表示要渲染的组件名；其他组件属性传入和官方文档一致，事件则采用 vue 的绑定方式。

`wx-component` 或 `wx-` 前缀已支持内置组件列表：

* cover-image 组件
* cover-view 组件
* scroll-view 组件
* view 组件
* icon 组件
* progress 组件
* text 组件
* button 组件
* editor 组件
* picker 组件
* slider 组件
* switch 组件
* navigator 组件
* camera 组件
* image 组件
* live-player 组件
* live-pusher 组件
* map 组件
* ad 组件
* official-account 组件
* open-data 组件
* web-view 组件

> PS：button 标签不会被渲染成 button 内置组件，如若需要请按照上述原生组件使用说明使用。
> PS：原生组件的表现在小程序中表现会和 web 端标签有些不一样，具体可[参考原生组件说明文档](https://developers.weixin.qq.com/miniprogram/dev/component/native-component.html)。
> PS：原生组件下的子节点，div、span 等标签会被渲染成 cover-view，img 会被渲染成 cover-image，如若需要使用 button 内置组件请使用 wx-component。
> PS：如果将插件配置 runtime.wxComponent 的值配置为 `noprefix`，则可以用不带前缀的方式使用内置组件。

## 开发建议

1. 虽然此方案将完整的 vue runtime 包含进来了，但必然存在一些无法直接适配的接口，比如 getBoundingClientRect，一部分会通过 dom/bom 扩展 api 间接实现，一部分则完全无法支持。**[查看 dom/bom 扩展 api 文档](./domextend.md)**。
2. 可能存在部分逻辑在 web 端和小程序端需要使用不同的实现，该部分代码可以抽离成一个单独的模块或者插件，暴露接口给业务端代码使用。在模块内可以使用上述提到的 `process.env.isMiniprogram` 环境变量进行判断区分当前运行环境。比如上述提到的 actionSheet 实现就可以抽离成一个 vue 插件实现。

> PS：注意这里使用 process.env.isMiniprogram 环境变量时尽量不要加其他动态条件，以方便 webpack 编译时剪除死代码，比如 `if (false) { console.log('xxxx') }` 就属于死代码

```js
// 正确使用方式
if (!process.env.isMiniprogram) {
    // web 端
    if (isIPhone) {
        // do something
    }
}

// 错误使用方式
if (!process.env.isMiniprogram && isIPhone) {
    // web 端
    // do something
}
```

3. 如果需要使用第三方库，尽量选择使用轻量的库，以缩减构建出来的代码体积。
4. vue 组件命名尽量不要和小程序内置组件同名。
5. 避免使用 id 选择器、属性选择器，尽量少用标签选择器和 * 选择器，尽可能使用 class 选择器代替。
6. 为了确保模板解析不出问题，标签上布尔值属性建议使用 = 号赋值的写法，如下例子所示：

```html
<input type="checkbox" checked="checked" />
<input type="checkbox" :checked="{{true}}" />
```
