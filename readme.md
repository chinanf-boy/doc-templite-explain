# doc-templite [![explain]][source] [![translate-svg]][translate-list]

[explain]: http://llever.com/explain.svg
[source]: https://github.com/chinanf-boy/Source-Explain
[translate-svg]: http://llever.com/translate.svg
[translate-list]: https://github.com/chinanf-boy/chinese-translate-list

「 为 多个 md 文件准备的 模版工具🔧 」

<!-- [中文](./readme.md) | ~~[english](./readme.en.md)~~ -->

---

## explian ✅

<!-- doc-templite START generated -->
<!-- name = 'doc-templite' -->
<!-- version = '1.1.1' -->
<!-- time = '2018 8.17' -->
版本 | 与日期 | 最新更新 | 更多
---|---|---|---
[1.1.1][commit] | ⏰ 2018 8.17 | ![version] | [源码解释][more]

[commit]: https://github.com/chinanf-boy/doc-templite/tree/v1.1.1
[version]: https://img.shields.io/npm/v/doc-templite.svg
[more]: https://github.com/chinanf-boy/Source-Explain

<!-- doc-templite END generated -->

## 生活

[help me live , live need money 💰](https://github.com/chinanf-boy/live-need-money)

---

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [doc-templite 的 使用 可看](#doc-templite-%E7%9A%84-%E4%BD%BF%E7%94%A8-%E5%8F%AF%E7%9C%8B)
- [准备:借鉴doctoc](#%E5%87%86%E5%A4%87%E5%80%9F%E9%89%B4doctoc)
  - [钉下注释](#%E9%92%89%E4%B8%8B%E6%B3%A8%E9%87%8A)
- [准备:模版](#%E5%87%86%E5%A4%87%E6%A8%A1%E7%89%88)
  - [模版库](#%E6%A8%A1%E7%89%88%E5%BA%93)
  - [什么模版](#%E4%BB%80%E4%B9%88%E6%A8%A1%E7%89%88)
  - [输入对象](#%E8%BE%93%E5%85%A5%E5%AF%B9%E8%B1%A1)
- [package.json](#packagejson)
  - [cli.js](#clijs)
  - [doc-templite.js](#doc-templitejs)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

萌生这个工具的念头是, 因为, 要修改多个中文翻译的库中`readme.md`, 关于 要翻译的原文提供的相关信息

如这样的格式: 

```
翻译的原文 | 与日期 | 最新更新 | 更多
---|---|---|---
 [commit] | ⏰ 2018 7.31 | ![last] | [more]
```

要修改十几20个文件, 很烦，而且还会有后续的添加, 所以就有了`doc-templite`

## [doc-templite 的 使用 可看](https://github.com/chinanf-boy/doc-templite/blob/master/readme.zh.md#%E4%BE%8B)

好了, 让我们来剖析一下 `doc-templite` 是如何实现的吧, 我们从准备开始

## 准备:借鉴doctoc

### 钉下注释

我一直都有使用[doctoc](https://github.com/thlorenz/doctoc), 来为 md文件 生成目录

而要生成目录, 你需要在文件中添加 doctoc 注释

```
<!-- START doctoc -->
<!-- END doctoc -->
```

以确定目录位置

1. 借鉴这一点, 我定义

<details>

<summary> doc-templite的注释 </summary>

```
    <!-- doc-templite START -->
    <!-- docTempliteId = 'readme' -->

    <!-- doc-templite END -->
```

2. 自然`doctoc`是可以查找目录下的md文件, 继续借鉴这个函数

3. 每次doctoc生成目录, 就会替换成新的

说明有一个, 更换 doctoc 生成目录的函数 借鉴

</details>

## 准备:模版

### 模版库

既然是模版工具, 那么就是要有 模版 与 输入值

机缘巧合下, 我看到了 [templite](https://github.com/lukeed/templite) 轻量级模板, 只有 150字节

使用简单, 只要两个值: 1. `什么模版` 2. `一个输入对象`

``` js
const templite = require('templite');

templite('Hello, {{name}}!', { name: 'world' });
//=> Hello, world!
```

### 什么模版

1. 模版应该唯一, 且只需要写一次, 相关的输入对象应该由 对应的md文件承担

所以`doc-templite`需要一个在命令行运行目录存在的

```
.doc-templite.js
```

它像这样, 简单输出一个js对象

``` js
module.exports = {
  "yobrave":
`name | age
---------|----------
{{ name }} | {{ age }}`,
"readme":
`name | age
---------|----------
{{ name }} | {{ age }}`
}
```

2. 我想模版总是不同的, 所以对象中的`key`可以很好的, 起到引领不同模版的效果

- "yobrave"
- "readme"

### 输入对象

1. 模版的输入对象, 由相应的md文件定义, 且包裹在 `doc-teplite注释` 中

> some.md

```
    <!-- doc-templite START -->
    <!-- docTempliteId = 'readme' -->
    <!-- name = 'yobrave' -->
    <!-- age = 18 -->
    <!-- doc-templite END -->
```

我想有几个点要讲的

- docTempliteId 正如我所说的, 不同的id模版由此定义
- name 而这个
- age  和这个 自然是 模版的输入对象

那么如何把这些 单行注释 变为 js对象了

2. 我想我是时候要选择一种 对象解析器, `Toml`格式

[Toml](https://github.com/toml-lang/toml) 格式, 就是我选择的

为什么, 不选择自带的 JSON解析, 那么你看看两者的对比

```
Toml - nice
<!-- name = 'yobrave' -->
JSON - bad
<!-- {"name":"yobrave"} -->
```

可以看出 JSON 的 字符串转换的 书写, 真的不太行

## package.json

> 我的node包初始化都是使用[generator-yobrave](https://github.com/chinanf-boy/generator-yobrave)生成的

``` js
"main": "doc-templite.js",
"bin": "cli.js",
```

这个工具主要在命令行中使用

### cli.js

- [explian cli](./cli.md)

文字讲讲做了什么

1. `twoLog(cli.flags['D'])` 根据命令行的是否Debug, 切换 [ora] / [winston]

[ora]: https://github.com/sindresorhus/ora
[winston]: https://github.com/winstonjs/winston

2. 确认用户的输入和模版文件`.doc-templite.js`, 否则 帮助/错误

3. 根据输入文件/目录找md文件, 汇总组合成数组

4. 逐个{读取文件内容 运行 `doc-templite.js` 转换, 拿到结果},拿到结果数组, 期间发生错误, 打印错误退出

5. 根据`--OR`命令选项与结果中成功转换的文件, 是否写入覆盖文件

### doc-templite.js

- [explain doc-templite](./doc-templite.md)

文字讲讲做了什么

1. 拿到单个文件的内容, 看看有没有`doc-templite注释`,且将 单一注释模版的内容组合 添加到`数组`

> 期间可能会出现 `doc-templite注释不闭合` 的错误

2. 对`数组`中每个注释模版操作

2.1 逐个注释模版, 找 toml 注释(单行/多行)

- 其中 找toml的注释 是分 找单行 和 找多行 注释

2.2 切掉 前后注释符号, 交给 toml 解析成js对象

> 期间出现 `toml注释不闭合` 与 `toml解析` 的错误

2.3 从js对象中拿到 模版id (docTempliteId 默认:`yobrave), 并运行模版引擎

> 期间 会出现`id模版找不到` 错误

2.4 组合 生成的文档模版, 覆盖上一文件内容(原文/上一模版的覆盖)拿到全新的文件内容, 汇总一下数据, 继续下个模版

> 注意: 只有文件中所有模版都成功后, 才能说这个文件成功转换

3. `数组`循环结束, 组合数据 返回

供给`cli.js`

``` js
	let result = {
		path: opts.path, // 文件路径
		toml: blocksTomls, // 每个模版中的toml对象 :[{},{}]
		transformed: transformed, // 是否成功转换
		data: data // 新文件内容
	}
```