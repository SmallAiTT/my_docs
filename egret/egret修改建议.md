# 关于egret建议

## 启动方式

现在的启动代码太乱了，而且用户体验很差。

个人建议要在2.0正式发布的时候要优先处理掉，毕竟这是个大版本的改变。

基本的思路是添加一个boot模块，启动的时候只引用一个boot模块，然后通过egret.boot()方法来进行启动。
通过配置文件的方式进行加载。

为了保持原有方式与新的方式并存，我个人是这么想的，如下定义目录结构：

```
game
  launcher 这个是原有的启动目录，这个保持不变用于支持1.x方式
  resource
  src
    Main.js 这个新旧模式的写法都一模一样

  egretProperties.json  这个是原有模式的配置文件，保持不变支持1.x方式

  index.html  这个是新的模式的启动
  egret.json  这个是新模式的配置文件（PS:感觉不知道为什么以前你们要定成egretProperties，又长又难记）
  myEgret.json  这个是个性化配置文件，用这个文件中的项替换egret.json中的项。
  release.html  这个是新模式的发布模式的入口html。
```

用上面的这种，用于兼容之前的模式，egret命令发现有egret.json的时候，优先使用新模式。

#### 启动代码

使用新的模式，启动代码非常精简，要做到只通过egret.json以及index.html简短代码就可以了。

例如在index.html中：

```html
<script src="libs/boot/boot.js"></script>

<script>
    egret.boot();
</script>
```

然后`egret.json`：

```json
{
  "name" : "game",
  "package" : "com.lightyeargame",
  "appName" : "我的游戏",
  "design" : {
    "width" : 720,
    "height" : 480
  },
  "main" : "Main",

  "fps" : true,

  "egretModuls" : ["core"],
  "frameworksDir" : "your/frameworks/dir",
  "frameworks" : ["res", "mo"],
  "modulesDir" : "your/modules/dir/default/is/modules",
  "modules" : ["g-consts", "g-home", "g-hero", "g-fight"]
}
```


* `name`
  这个作为工程名字，同时通过命令构建android项目的时候，也是用这个构建项目名。

* `package`
  包名，但实际上，真正的包名是在加上`name`，如上创建出来的android工程的包名就是`com.lightyeargame.game`。

* `appName`
  自动替换`index.html`里面的`title`以及native里面的app名字。

* `design`
  设计分辨率。

* `main`
  入口文档类，自动对应src/xxx.ts里面的这个xxx类。

* `fps`
  用于控制帧率显示与否。

* `egretModules`
  egret引擎模块。

* `frameworks`
  开发这定义的框架模块。方便开发者使用，1.x的那种模式需要配路径，相当难用。
  通过`frameworksDir`来指定frameworks目录。

* `modules`
  项目模块，这样可以比较好的支持如果开发者希望再开发项目时也用模块进行划分的情况。
  都写在`src`里面，如果没定义js顺序那么有时由于ts编译的原因很容易出现一些蛋疼的问题，
  如果定义了顺序，大项目太多js又不方便管理。
  同时定义成模块，开发时可以只编译自己的当前模块，比较快。
  就我个人而言，我现在倾向大项目按模块分。
  通过`modulesDir`指定项目模块目录。

通过3中modules的划分，编译的时候还可以指定，
例如，我只需要编译引擎，那么就是`egret build -e`，
只需要frameworks那么就是`egret build -f`，
只需要modules那么就是`egret build -m`，
只需要src那么就是`egret build`，
编译具体模块跟原来的还是一样，
编译全部就是`egret build -a`。

但是现在你们`-e`是编译全部，所以这个命名你们可以再该，我只是举个例子。

上面的参数项是我举例的而已，根据你们的研究需要进行添加。

#### 第三方模块

上述配置文件提到了关于模块的改动，注意到是直接通过名字，而不是路径，这样项目中很很好处理。

所以一个第三方模块的目录变成这样：

```
my-module
  src
  module.json
```

其中，`module.json`里面内容尽可能简单，
个人认为去掉之前的`name`这种，直接使用该模块的目录名称作为模块名就好了。
现有的方式非常繁琐，还要保持`xxx.json`与`name`是一个名字，
而且目录名可以和模块名不一样反而不规范不统一，容易造成误解。
也就是说，目录名就是模块名，是我认为最好的体验。


#### 网络模块需要修改

如果要提供配置文件模式的支持，那么网络模块我觉得需要改造下。
现在是需要主循环启动之后才能使用网络模块，那么这时候canvas就初始化好了。
这么一来，我就没办法通过`egret.json`的`design`来控制设计分辨率了。
这个设定感觉有点搓。

#### h5和native的加载整合

将js加载直接整合到boot里面去，不要暴露给用户。所以精简到`egret.boot()`就可以了。
native的差异相关代码也不要暴露给用户，例如现在的`runtime_required.js`这些。
一些native差异性初始化代码可以整合到引擎层面，自动在boot之前引用，不要暴露给用户。

#### 支持myEgret.json和url参数功能

这个模式之前演讲的时候说了。而且这个功能只需要在h5模式下就可以了，因为这个是dev模式，native模式就可以不用了。

#### 启动阶段监听

需要对启动阶段进行监听，我现在自己的需求是这几种：

* egret引擎初始化之后同步与异步模式监听：AFTER_EGRET

* 配置文件加载后同步与异步监听：AFTER_CONFIG

* 入口文档类添加完之后同步监听：AFTER_MAIN

通过这几个监听，可以实现每个第三方进行自己的初始化，而不是在每个项目的main里面进行初始化。
大大提高第三方模块的用户体验。
开发者只需要关心引用第三方模块就可以了，不需要具体细节。

#### 小结

目前我自己写了个boot模块，如果你们需要我可以提供。
同时也写了一个编译命令，就是基本按照刚刚上面说的思路来进行管理整合的，所以还是兼容1.x模式的。
也就是说，该升级方式对用户代码基本没什么影响，就是用户使用的时候需要注意而已。
我觉得，这个你们最好花点时间，在2.0正式版本发布的时候就支持。
我是这么想的，启动模式相当重要，版本的变动其他的还好说，启动模式变了就算好用也会被吐槽。
但是这次是大版本的更新，升级成一个更好的模式我觉得开发者是可以接受的。
更何况还兼容1.x的模式。
如果在2.0的时候没有，2.x小版本进行更新的时候再加上，我觉得就不好了。
所以我个人觉得，就算延迟2.0的发布也要将这个启动部分先做好。
这个就像一个地基一样，影响着建筑结构。
如果还要等到3.0的话，我只能说醉了。现在虽然可以用，但是实在是不好用。

## logger改进

这个演讲的时候我有提过，我可以提供代码。

## 事件模块

现在的事件模块是按照flash来写的，这点我可以理解。
但是现在的模块确实不好用，让我们在自己进行拓展不是不行，是太烦了。
而且，我认为完全可以引擎进行支持就行了，因为就我看来，实在太常用了。
我在我自己的`boot`模块中实现了一个`Emitter`。
提供了这些接口：

```
on
onPriority
onAsync
onAsnycPriority
onNextTick
onPriorityNextTick
once
oncePriority
onceAsync
onceAsyncPriority
onceNextTick
oncePriorityNextTick
single
singleAsync
singleNextTick
un
unNextTick
unOnce
unOnceNextTick
unSingle
unSingleNextTick
unAll
unAllNextTick
emit
emitAsync
emitNextTick
emitPhase
emitPhaseNextTick
```

接口很多，但是主要只是进行了归类，所以变多了。
整理下就很好记而且实用。
主要是按这几个进行划分：

1. 是否需要设置优先级
2. 监听可以重复响应(on)、监听只响应一次(once)、监听只能有一个(single)
3. 立即响应还是下一帧才响应(xxxNextTick)
4. 是同步模式还是异步模式(xxxAsync为异步模式)

例如：

将优先级单独拆分到另一个api，原因是通常情况下我们很少用到优先级这个方式，分开写反而容易记忆。

api名字不使用原来的`addEventListener`的原因是，如果这么做的话，对现有需求的api就太长太难记了。
而且现有的`on`这种相当容易记忆。

然后事件需要提供类式注册。

例如`MyEmitter`也可以注册监听，即`MyEmitter.on("myEvent", function(){})`。
如此一来，相当于所有的`MyEmitter`的实例进行监听。这个相当使用。

上述的功能点是在我们项目中的一个非常重要的需求，对于我们项目是非常核心的一点。
例如，我们项目自己写的MVC就是使用了NextTick模式。而且现有的event模块实在太不好用了。

## native工程的创建

将native工程创建集成到命令行中，结合`egret.json`进行使用。
要不然现在要打一个native包太繁琐了。

## ui工具需要提供命令行发布功能

这个是刚需，否则项目中一个一个手动通过gui操作要死人的。

## 总结

`boot`的修改建议希望你们认真考虑下，是我个人从项目使用过程中得出的总结。
而且，我是希望这种“地基”部分的东西，是大版本改动的时候才有，但是我又不希望到了3.0才有看到你们类似的改进。
所以强烈建议你们在2.0发布的时候就支持，就算版本推迟。
