# 利用lerna管理多框架日历组件

> 最近在开发公司内部使用的小程序框架，第一次接触到了多包管理工具 -- [lerna](https://github.com/lerna/lerna/)

本篇将会以个人开发的[日历组件](https://juejin.cn/post/6946154756721115166)为例，简述使用lerna管理项目的历程。

也许大家会问了，你这日历组件不是vue写的🐴要lerna管理个啥。

![时代变了](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/95987c4030c244bda407ea1076d1182a~tplv-k3u1fbpfcp-watermark.image)

在构思组件开发时就想着能够为多框架的用户提供支持，所以就有了**react/vue/小程序**版本。

## 初探lerna

回到正题，我们先看一看项目的目录结构。

![WX20210618-205140@2x.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c517857c5fc447369f340b13ccc00f17~tplv-k3u1fbpfcp-watermark.image)

按照lerna项目默认的项目结构，我们会将需要管理的包都放进`packages`文件夹内，并在根目录的`lerna.json`中编写lerna相关的配置。

![WX20210618-205226@2x.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/79a9c5b6d86643c88328b64b205ec6e8~tplv-k3u1fbpfcp-watermark.image)

`packages`文件夹内的结构如上图，代表我使用lerna同时在管理4个包，分别基于vue/react/原生微信小程序开发而成的日历组件，以及负责处理日历核心逻辑的`core`包。

[日历组件](https://juejin.cn/post/6946154756721115166)文中有提及，笔者对一个上古时期的日历组件做了一次优化，优化的重点是**抽离**及**易读**。

在抽离的过程中发现，其实**生成日历数据**这件事，不论这个组件是vue开发的还是react开发甚至是小程序组件，都需要这段逻辑。所以我们可以将其进行更彻底的抽离，直接将这类逻辑收束成依赖包，供所有类型的日历组件进行调用。

核心逻辑具体实现就不再赘述了，基本和前一篇文章阐述的一致。关于日历视图层react以及小程序的实现，基本就是将Vue中的实现“翻译”成框架对应的写法。如果大家有兴趣的话可以点击下方的传送门↓↓

### React组件实现[(Github传送门)](https://github.com/mykurisu/calendar/tree/master-next/packages/react)

在React实践中，比较特别的有两个地方：

- 用useMemo替代vue实现中的computed

```javascript
const isFirstMonth = useMemo(() => selectedMonth === 0);

const isLastMonth = useMemo(() => selectedMonth === 11);
```

- 用useLayoutEffect替代vue实现中的nextTick

```javascript

useLayoutEffect(() => {

    setBlockHeight(document.querySelector('.__main__block-head').offsetWidth + "px");

}, [calendarData]);

```

### 微信小程序组件实现[(Github传送门)](https://github.com/mykurisu/calendar/tree/master-next/packages/miniapp)

小程序本身的开发模式就和vue谜之相似，基本上可以说是无痛移植，就是在获取日历方块宽度的时候没法用`querySelector`，必须得使用小程序自己的API`createSelectorQuery`。

但是在样式这块就不太一样，微信小程序只认与wxml同名的wxss文件，不支持样式的导入，所以在进行核心逻辑打包的时候同时执行一个`miniapp-script.js`，将css文件复制到miniapp目录中并更改其后缀名。

## lerna实践

结合上面的描述，我们只能认为lerna是一种包管理的思维，将原来的单项目对单依赖包的模式变成了单项目对多依赖包，并看不出它有什么其他实质性的帮助。

但是当我准备将写好的4个依赖包发布到npm时，遇到了比较棘手的问题，这4个依赖的打包方式完全不同，发布的时候岂不是得cd到每个包里进行install、build、version等机械式的操作。（其实并不用）

首先，先看看创建lerna项目的第一步，`lerna.json`的配置。

```json
{

    "version": "independent",

    "packages": [

        "packages/*"

    ],

    "npmClient": "yarn",

    "useWorkspaces": true,

    "command": {

        "publish": {

            "allowBranch": "master-next"

        }

    }

}
```

`version`字段是用来定义lerna项目的版本号，如果此字段声明了版本号，则内部的所有子项目都会按此版本号进行发布。但是可以选择通过不填版本号，如上述配置一样填写`independent`，来进行另一种发布模式，在这种发布模式下每个子项目维护自己的版本号。

`packages`字段则是用于声明作用目录，只有在数组内的子项目才会被lerna检索到。

`npmClient`字段用于声明lerna执行指令时使用的包管理工具，默认是`npm`。配置里声明的是`yarn`，所以可以开启`workspaces`模式。workspaces是yarn的一大特色，使用了workspaces之后绝大多数的依赖包都被提升到了根目录下的node_modules之内，各个子项目的node_modules里面不会重复存在依赖。

`command`字段则是对lerna指令内置的参数进行改动。[具体用法](https://github.com/lerna/lerna#lernajson)

接下来，笔者通过一次日历组件的打包发布全流程，给大家直观的展示接入lerna之后的玩法。

> lerna ls
>
> 获取本地项目内子项目列表

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f8a48605c1674f8283ba1b1678368b5f~tplv-k3u1fbpfcp-watermark.image)

> lerna bootstrap
>
> 安装作用目录下全项目的依赖

> lerna changed or lerna diff
>
> 用于确认本次修改涉及到的范围，lerna会将有改动的子项目都列出来，方便我们及时回顾改动

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2989ac4b9f2043998eb715180f3107a8~tplv-k3u1fbpfcp-watermark.image)

> lerna run (any script)
>
> 在所有子项目依次执行某条指令，图中展示的是build指令，等同于在每个子项目中进行了yarn build

![WX20210811-112000@2x.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4f9fcf109835416c9c6bb3de86f52db6~tplv-k3u1fbpfcp-watermark.image)

> lerna publish
>
> 将本次改动的子项目批量进行npm发布，如下图所示，在输入指令之后可以为每个项目指定想要发布的版本

![WX20210811-112236@2x.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c70ec8b5413848e3b7cd80684370fb08~tplv-k3u1fbpfcp-watermark.image)

![WX20210811-112254@2x.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f2d8c7a5dedb42cda40bb65537ab2b5d~tplv-k3u1fbpfcp-watermark.image)

![WX20210811-112329@2x.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2be216f970b847f2829690c4cf682e2b~tplv-k3u1fbpfcp-watermark.image)

> lerna clean
>
> 后续迭代时如果想重新安装各项目依赖，可以先执行clean指令，它会将项目底下所有node_modules都清理干净

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c631a8a26ee44979a89865bec56053c3~tplv-k3u1fbpfcp-watermark.image)

## 闲聊

历时2年的日历组件，总算有了大概雏形，每个框架都有开箱即用的npm包：

[@mykurisu/calendar-component-vue](https://www.npmjs.com/package/@mykurisu/calendar-component-vue)

[@mykurisu/calendar-component-miniapp](https://www.npmjs.com/package/@mykurisu/calendar-component-miniapp)

[@mykurisu/calendar-component-react](https://www.npmjs.com/package/@mykurisu/calendar-component-react)

如果大家有需要的话可以直接install，有任何问题都可以到[@mykurisu/calendar](https://github.com/mykurisu/calendar)中进行反馈。

如果是关于样式上的问题，我建议fork一下项目，直接修改core中的样式，修改完之后改一下package.json里面的配置，甚至可以发布成自己的私有包。另外，如果有时间的话我也会更新多个日历皮肤，争取做到真正的开箱即用。

最后，如果本项目对大家有帮助麻烦帮忙点点[@mykurisu/calendar](https://github.com/mykurisu/calendar)的star，后续的更新也会及时推送给大家。