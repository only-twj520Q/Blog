## 前言

什么是babel？你可以理解它就是一个语法转器，简单来说就是 ES6、ES7等等的新语法转化为ES5或能让低端浏览器正常运行的代码。比如我们经常使用的async、promise语法，在低端浏览器上可能就无法使用，会引起故障，但是只要我们合理使用babel，我们就可以放心大胆地使用新语法。

下面阐述的内容都是基于Babel 7的使用和总结，因babel 6 和babel 7在使用上存在较大差异，所以需要提前说明一下。



## 核心原理

这边简单介绍一下babel的工作原理，其实就是利用了抽象语法树，又称AST。所有的babel插件也是基于AST。

#### Babel的处理步骤

* 解析。将源代码变成AST。babel的解析器使用的是babylon。在解析过程中有两个阶段：**词法分析**和**语法分析**，词法分析阶段把字符串形式的代码转换为**令牌**（tokens）流，令牌类似于AST中节点；而语法分析阶段则会把一个令牌流转换成 AST的形式，同时这个阶段会把令牌中的信息转换成AST的表述结构。

* 转化。操作AST，去改变代码。babel转化器使用的是babel-traverse。babel使用提供的API对AST进行深度优先遍历，在此过程中对节点进行添加、更新及移除操作。

* 生成。将更改后的AST，再变回代码。babel生成器使用的是babel-generator。将经过转换的AST通过babel-generator再转换成js代码，过程就是深度优先遍历整个AST，然后构建可以表示转换后代码的字符串。

我们编写的代码在编译阶段就被解析成AST，为什么要转化成AST，是因为方便计算机更好地理解代码，方便开发人员更好地操作代码。因为对于计算机或者编辑器而言，代码其实本质上就是字符串。

可以使用这个链接[在线测试](https://astexplorer.net/)观察代码生成的AST是什么样的。

```javascript
{
    "type": "Program",
    "start": 0,
    "end": 52,
    "body": [...]
}
```

你会发现抽象语法树中不同层级有着相似的结构，这样的结构叫做节点，一个AST是由多个或单个这样的节点组成，节点内部还可以嵌套有多个这样的子节点。

#### 实际使用

```javascript
import * as babylon from 'babylon';
import traverse from 'babel-traverse';
import generate from 'babel-generator';

const code = `function double(x) {
  return x;
}`;
const ast = babylon.parse(code);
traverse(ast, {
  enter(path) {
    if (
      path.node.type === "Identifier" &&
      path.node.name === "x"
    ) {
      path.node.name = "xx";
    }
  }
});
const output = generate(ast, code);
console.log('output', output)

// 输出结果
{
  code: "function double(xx) {↵  return xx;↵}"
  map: null
  rawMappings: null
}
```



## 基本用法

一般而言，babel的使用存在两种方式。

* 命令行
* 结合构建工具

通常说来，后者更常见。如果结合webpack使用，我们需要在`webpack.config.js`中的配置项`rules`属性中配置`babel-loader`，然后在项目根目录中新增`.babelrc`文件，这个文件就是对babel的配置。

`webpack.config.js`部分配置

```javascript
module: {
  rules: [
    {
      test: /\.js$/,
      include: path.resolve('src'),
      loader: 'babel-loader'
    },
    ...
  ]
 }
```

`.babelrc`部分配置

```javascript
{
  "presets": [
    [ 
      "@babel/preset-env", {
        "targets": {
          "chrome": "52",
          "ie": "8"
        },
        "useBuiltIns": false,
        "corejs": false
    	}
    ]
  ]
 }
```

`babel-loader`使用前需要使用`npm`安装，并且它默认会读取根目录下的`.babelrc`的配置，或者直接在`babel-loader`的配置options也是可以的。注意`.babelrc`文件的内容格式是JSON。



## 常见名词

#### @babel/core

babel的核心包，它主要的作用就是编译。

#### @babel/cli

只有@babel/core是无法在命令行使用这些功能的，@babel/cli支持你直接在命令行中编译代码。

```javascript
./node_modules/.bin/babel src --out-dir lib
```

上面的命令会编译src目录下的所有js代码，并编译成最终的代码（`babel.config.js`或者`.babelrc`配置的），并输出到lib目录下。`--out-dir`代表输出到哪个目录下。

#### @babel/node 

`babel-node` 命令并非独立安装，在babel 7以前，需要通过安装 `babel-cli` 包获得。而在babel 7以后，babel 的模块被拆分。因此需要安装 `@babel/core`和 `@babel/node` 两个包来获取。

```javascript
// 安装
npm i -g @babel/core @babel/node
// 使用
babel-node test.js
```

#### @babel/polyfill

babel默认只转化js语法，比如对let ，const，class等。它不转换新的API，比如Set、Map、Reflect、Symbol、Promise 等全局对象，包括全局对象上的方法，比如`Object.assign`。

我们使用babel官网提供的的`try out`功能做演示。

![](https://img.ikstatic.cn/MTU3MzczMTMxOTQ5OCM1MzkjanBn.jpg)

如图，我们可以看到babel插件默认只会对class，箭头函数这些js语法糖进行转化。但是对新的API它默认不会进行支持，比如上图中的Promise。所以我们可以引入`@babel/polyfill`。这个库包含了`core-js`和`regenerator-runtime`。它是对完整的ES6环境的模拟，我们需要在入口文件或者所有代码的前面引入这个库。

```js
import '@babel/polyfill'
// 等同于
import "core-js/shim";
import "regenerator-runtime/runtime";
```

但是这个库有两个很不可忽略的问题。

* 体积太大。在非压缩的前提下，能达到几百K，而且如果我们只是使用了其中某个特性的情况下，这会造成很大的浪费。
* 会污染全局变量，因为它会增加很多全局变量API，或者在很多类的原型链上都作了修改，从而实现增加实例方法。如果我们开发的也是一个类库供其他开发者使用，这种情况就会变得非常不可控。

至于说解决方案下面会提到。



## 核心使用

需要记住的是，babel本身不具有任何转化能力，我们把它转化的功能都分解到一个个插件中去。所以，当我们不配置任何插件的时候，输入和输出的代码都是一样的。

在配置文件中，我们主要是由两个配置项，分别是预设preset和插件plugins。

新的babel 7已经使用了新的插件语法。

### 预设preset

先解释一下什么是预设，在没有预设这个概念之前，当我们需要使用箭头函数的时候，我们使用`@babel/plugin-transform-arrow-functions`插件，它的作用就是将ES6的箭头函数转换成普通函数，但是ES6的语法太多了，如果我们单纯依靠引入一个个依赖的插件就太麻烦，而且容易出错。所以预设的出现就解决了这个问题，它可以理解为插件的组合，比如`@babel/preset-es2015`是对整个ES6语法的插件集合，还有`@babel/preset-es2016`等等，不过现在这些类似的这些预设官方也不推荐使用了，推荐使用`@babel/preset-env`。目前常见的预设有`@babel/preset-env`，`@babel/preset-react`，`@babel/preset-typescript`。

#### @babel/preset-env（重点）

重点介绍`@babel/preset-env`。这是一个能根据配置的运行环境特点，为代码做相应和必要的编译，同时支持浏览器和node，比如如果你要求的运行环境浏览器版本比较低，转化的代码可能会比较多，如果要求支持的是高版本浏览器，可能你都不需要转化ES6的语法，从而节省了很多空间。如果不写任何配置项，env 等价于 latest，也等价于` es2015 + es2016 + es2017` 三个相加(不包含 `stage-x` 中的插件)。env 包含的插件列表维护在[这里](https://github.com/babel/babel-preset-env/blob/master/data/plugin-features.js)

```javascript
// 浏览器
{
  "presets": [
    ["@babel/preset-env", {
      "targets": {
        "browsers": ["last 2 versions", "safari >= 7"]
      }
    }]
  ]
}
// node
{
  "presets": [
    ["@babel/preset-env", {
      "targets": {
        "node": "6.10"
      }
    }]
  ]
}

```

此外babel 7的还有一个重要的变化是删除了`stage-x`的相关插件，这是一个实验性的语法插件，所有针对处于标准提案阶段的功能所编写的预设（stage preset）都已被弃用。`@babel/preset-env`将只支持到`stage-4`。stage 是向下兼容 0>1>2>3>4 所包含的插件数量依次减少。

简单说明一下，TC39 将提案分为以下几个阶段：

* Stage 0 - 设想（Strawman）：只是一个想法，可能有 Babel插件。
* Stage 1 - 建议（Proposal）：这是值得跟进的。
* Stage 2 - 草案（Draft）：初始规范。
* Stage 3 - 候选（Candidate）：完成规范并在浏览器上初步实现。
* Stage 4 - 完成（Finished）：将添加到下一个年度版本发布中。

#### useBuiltIns配置

这是一个`preset-env`的重要配置选项，它的出现是为了解决上述提到`@babel/polyfill`的缺陷，它将会自动检测语法帮你require你代码中使用的部分polyfill。

```javascript
{
  "presets": [
    ["@babel/preset-env", {
      "useBuiltIns": "usage",
      "corejs": 2,
      "targets": {
        "browsers": ["last 2 versions", "safari >= 7"]
      }
    }]
  ]
}
```

这个字段的可选值包括：usage、entry 和 false， 默认为 false，表示不对 polyfills 处理。

* usage：它会在你使用到 ES6 新特性时，自动从`core-js`中添加相关的模块和方法，不会造成全局污染。
* entry：需要在入口使用`import "@babel/polyfill"`，将 polyfill 拆分引入，仅引入有浏览器不支持的 polyfill

![image-20191114175843871](https://img.ikstatic.cn/MTU3MzczMTQyMTY3MiM4NjEjanBn.jpg)

如上图，配置了entry以后，`import @babel/polyfill`会被分拆成很多`require("core-js/modules/xxx")`，它其实会根据`@babel/preset-env`配置的目标环境将一个大的库分拆引入。事实上，`@babel/polyfill`这个包本身是没有内容的，它依赖于`core-js`和`regenerator-runtime`这两个包，这两个包提供了ES6规范的运行时环境。因此当我们不需要按需polyfill时直接引入`@babel-polyfill`就行了，它会把`core-js`和`regenerator-runtime`全部导入，当我们配置了entry以后，它会根据目标环境自动按需引入`core-js`和`regenerator-runtime`。

![](https://img.ikstatic.cn/MTU3MzczMTU1MzU4NCMgIDQjanBn.jpg)

如上图，当配置了usage以后，我们无需手动`import '@babel/polyfill'`，它会根据我们的实际代码结合配置的目标环境，自动引入相应的库。

所以，比较这两个配置字段，我更倾向于使用usage这种方式，事实上，它才是真正做到了按需引入。

### 插件plugin

插件的作用是为了实现某个具体功能，比如预设不能支持的，比如`@babel/plugin-proposal-decorators`是为了解决装饰器的问题，比如`@babel/plugin-syntax-dynamic-import`是为了解决webpack动态引入某个模块的问题。

#### @babel/plugin-transform-runtime（重点）  

这个插件能解决`@babel/polyfill`提供的类或者实例方法污染全局作用域的情况的。它的作用和上面的useBuiltIns很类似。

`@babel/plugin-transform-runtime`插件是为了解决

- 运行时引入，为了解决多个文件重复引用相同helpers 
- 局部引入，为了解决新API方法全局污染

![](https://img.ikstatic.cn/MTU3MzczMTcyNjEwNyM2NTIjanBn.jpg)

![](https://img.ikstatic.cn/MTU3MzczMTg4MTc5MyM2MjUjanBn.jpg)

对比上面的两个图，我们用class举例，可以发现，如果使用这个插件，它会在每个模块的内部重复定义这个相同的函数方法，虽然最后的结果也没有问题，但是会造成出现很多重复代码，并且增大了体积。而如果使用了插件以后，我们发现从定义方法改成引用，那重复定义就变成了重复引用，就不存在代码重复的问题了。如上图，可以发现，class这个helper 就是通过 `require` 引入的，这样就不会存在代码重复的问题了。

在使用 `babel-plugin-transform-runtime` 的时候必须把 `babel-runtime` 当做依赖。

 `babel-runtime`内部集成了

* `core-js`：转换一些内置类 (`Promise`, `Symbols`等等) 和静态方法 (`Array.from` 等)。绝大部分转换是这里做的，自动引入。
* `regenerator`：作为 `core-js` 的拾遗补漏，主要是对 `generator/yield` 和 `async/await` 两组的支持。当代码中有使用 `generators/async` 时自动引入。
* `helpers`： 如上面的 `asyncToGenerator` 就是其中之一，其他还有如 `jsx`, `classCallCheck` 等等，可以查看 [babel-helpers](https://github.com/babel/babel/blob/6.x/packages/babel-helpers/src/helpers.js)。

但是这个插件也有**缺点：**`babel-plugin-transform-runtime` **不支持** 类的实例方法 (比如数组的`filter`方法等)

还有一个需要**注意**的是，我发现默认情况下这个插件不会对ES6新的API进行polyfill，就是说它默认不会转化新的特性比如Promise、`Object.assign`。所以我们还需要配置corejs这个字段，并安装对应的依赖`@babel/runtime-corejs2`。

```javascript
npm install --save @babel/runtime-corejs2

{
 "plugins": [
    [
      "@babel/plugin-transform-runtime",
      {
        "helpers": true,
        "regenerator": true,
        "useESModules": false,
        "corejs": 2
      }
    ]
	] 
}
```

### 注意

当plugins 与 presets 同时存在的情况，执行顺序如下

> 这意味着如果两个转换插件都将处理程序的某个代码片段，则将根据转换插件或 preset 的排列顺序依次执行。
>
> - 先执行 plugins 的配置项,再执行 Preset 的配置项；
> - plugins 配置项，按照声明顺序执行。
> - Preset 配置项，按照声明逆序执行。

### Babel插件方案对比

|              方案               |                             优点                             |                    缺点                    |
| :-----------------------------: | :----------------------------------------------------------: | :----------------------------------------: |
| @babel/plugin-transform-runtime |                     按需引入, 打包体积小                     |              不能兼容实例方法              |
|         @babel/polyfill         |                      完整模拟 ES6 环境                       | 打包体积过大, 污染全局对象和内置的对象原型 |
|        @babel/preset-env        | 按需引入, 可配置性高，并且通过配置useBuiltIns实现按需加载@babel/polyfill |                  暂未发现                  |

### 结论

`@babel/plugin-transform-runtime`和`useBuiltIns`配置可以解决替代@babel/polyfill。



## babel使用总结

综上，我个人对于babel 7的配置方案总结如下

* @babel/preset-env + targets + useBuiltins: usage
* @babel/plugin-transform-runtime
* 引入必要的plugin

