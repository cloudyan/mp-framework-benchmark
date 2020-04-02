# 小程序框架运行时性能大测评

> 作者：董宏平(hiyuki)，滴滴出行小程序负责人，mpx框架负责人及核心作者

随着小程序在商业上的巨大成功，小程序开发在国内前端领域越来越受到重视，为了方便广大开发者更好地进行小程序开发，各类小程序框架也层出不穷，呈现出百花齐放的态势。但是到目前为止，业内一直没有出现一份全面、详细、客观、公正的小程序框架测评报告，为小程序开发者在技术选型时提供参考。于是我便筹划推出一系列文章，对业内流行的小程序框架进行一次全方位的、客观公正的测评，本文是系列文章的第一篇——运行时性能篇。

在本文中，我们会对下列框架进行运行时性能测试(排名不分先后):
* wepy2(https://github.com/Tencent/wepy) @2.0.0-alpha.20
* uniapp(https://github.com/dcloudio/uni-app) @2.0.0-26120200226001
* mpx(https://github.com/didi/mpx) @2.5.3
* chameleon(https://github.com/didi/chameleon) @1.0.5
* mpvue(https://github.com/Meituan-Dianping/mpvue) @2.0.6
* kbone(https://github.com/Tencent/kbone) @0.8.3
* taro next(https://github.com/NervJS/taro) @3.0.0-alpha.5

其中对于kbone和taro next均以vue作为业务框架进行测试。

运行时性能的测试内容包括以下几个维度：
* 框架运行时体积
* 页面渲染耗时
* 页面更新耗时
* 局部更新耗时
* setData调用次数
* setData发送数据大小

框架性能测试demo全部存放于https://github.com/hiyuki/mp-framework-benchmark 中，欢迎广大开发者进行验证纠错及补全；

## 测试方案

为了使测试结果真实有效，我基于常见的业务场景构建了两种测试场景，分别是动态测试场景和静态测试场景。

### 动态测试场景

动态测试中，视图基于数据动态渲染，静态节点较少，视图更新耗时和setData调用情况是该测试场景中的主要测试点。

动态测试demo模拟了实际业务中常见的长列表+多tab场景，该demo中存在两份优惠券列表数据，一份为可用券数据，另一份为不可用券数据，其中同一时刻视图中只会渲染展示其中一份数据，可以在上方的操作区模拟对列表数据的各种操作及视图展示切换(切tab)。

<img src="https://dpubstatic.udache.com/static/dpubimg/PWsGL0GkuQ/dynamic.jpeg" width="300px"></img>

*动态测试demo*

在动态测试中，我在外部通过函数代理的方式在初始化之前将App、Page和Component构造器进行代理，通过mixin的方式在Page的onLoad和Component的created钩子中注入setData拦截逻辑，对所有页面和组件的setData调用进行监听，并统计小程序的视图更新耗时及setData调用情况。该测试方式能够做到对框架代码的零侵入，能够跟踪到小程序全量的setData行为并进行独立的耗时计算，具有很强的普适性，代码具体实现可以查看https://github.com/hiyuki/mp-framework-benchmark/blob/master/utils/proxy.js

### 静态测试场景

静态测试模拟业务中静态页面的场景，如运营活动和文章等页面，页面内具备大量的静态节点，而没有数据动态渲染，初始ready耗时是该场景下测试的重心。

静态测试demo使用了我去年发表的一篇技术文章的html代码进行小程序适配构建，其中包含大量静态节点及文本内容。

<img src="https://dpubstatic.udache.com/static/dpubimg/5TOToHvunN/static.jpeg" width="300px"></img>

*静态测试demo*

## 测试流程及数据

> 以下所有耗时类的测试数据均为微信小程序中真机进行5次测试计算平均值得出，单位均为ms。Ios测试环境为手机型号iPhone 11，系统版本13.3.1，微信版本7.0.12，安卓测试环境为手机型号小米9，系统版本Android10，微信版本7.0.12。

> 为了使数据展示不过于混乱复杂，文章中所列的数据以Ios的测试结果为主，安卓测试结论与Ios相符，整体耗时比Ios高3~4倍左右，所有的原始测试数据存放在https://github.com/hiyuki/mp-framework-benchmark/blob/master/rawData.csv

> 由于transform-runtime引入的core-js会对框架的运行时体积和运行耗时带来一定影响，且不是所有的框架都会在编译时开启transform-runtime，为了对齐测试环境，下述测试均在transform-runtime关闭时进行。

### 框架运行时体积

由于不是所有框架都能够使用`webpack-bundle-analyzer`得到精确的包体积占用，这里我通过将各框架生成的demo项目体积减去native编写的demo项目体积作为框架的运行时体积。

|     | demo总体积(KB) | 框架运行时体积(KB) |
| --- | --------- | ----------- |
|native|27|0|
|wepy2|66|39|
|uniapp|114|87|
|mpx|78|51|
|chameleon|136|109|
|mpvue|103|76|
|kbone|395|368|
|taro next|183|156|

该项测试的结论为：  
native > wepy2 > mpx > mpvue > uniapp > chameleon > taro next > kbone

结论分析：
* wepy2和mpx在框架运行时体积上控制得最好；
* taro next和kbone由于动态渲染的特性，在dist中会生成递归渲染模板/组件，所以占用体积较大。

### 页面渲染耗时(动态测试)

我们使用`刷新页面`操作触发页面重新加载，对于大部分框架来说，页面渲染耗时是从触发刷新操作到页面执行onReady的耗时，但是对于像kbone和taro next这样的动态渲染框架，页面执行onReady并不代表视图真正渲染完成，为此，我们设定了一个特殊规则，在页面onReady触发的1000ms内，在没有任何操作的情况下出现setData回调时，以最后触发的setData回调作为页面渲染完成时机来计算真实的页面渲染耗时，测试结果如下：

|     | 页面渲染耗时 |
| --- | --------- |
|native|60.8|
|wepy2|64|
|uniapp|56.4|
|mpx|52.6|
|chameleon|56.4|
|mpvue|117.8|
|kbone|98.6|
|taro next|89.6|

> 该项测试的耗时并不等同于真实的渲染耗时，由于小程序自身没有提供performance api，真实渲染耗时无法通过js准确测试得出，不过从得出的数据来看该项数据依然具备一定的参考意义。

该项测试的结论为：  
mpx ≈ chameleon ≈ uniapp ≈ native ≈ wepy2 > taro next ≈ kbone ≈ mpvue

结论分析：
* 由于mpvue全量在页面进行渲染，kbone和taro next采用了动态渲染技术，页面渲染耗时较长，其余框架并无太大区别。

### 页面更新耗时(无后台数据)

这里后台数据的定义为data中存在但当前页面渲染中未使用到的数据，在这个demo场景下即为不可用券的数据，当前会在不可用券为0的情况下，对可用券列表进行各种操作，并统计更新耗时。

更新耗时的计算方式是从数据操作事件触发开始到对应的setData回调完成的耗时

> mpvue中使用了当前时间戳(new Date)作为超时依据对setData进行了超时时间为50ms的节流操作，该方式存在严重问题，当vue内单次渲染同步流程执行耗时超过50ms时，后续组件patch触发的setData会突破这个节流限制，以50ms每次的频率对setData进行高频无效调用。在该性能测试demo中，当优惠券数量超过500时，界面就会完全卡死。为了顺利跑完整个测试流程，我对该问题进行了简单修复，使用setTimeout重写了节流部分，确保在vue单次渲染流程同步执行完毕后才会调用setData发送合并数据，之后mpvue的所有性能测试都是基于这个patch版本来进行的，该patch版本存放在https://github.com/hiyuki/mp-framework-benchmark/blob/master/frameworks/mpvue/runtime/patch/index.js

> 理论上来讲native的性能在进行优化的前提下一定是所有框架的天花板，但是在日常业务开发中我们可能无法对每一次setData都进行优化，以下性能测试中所有的native数据均采用修改数据后全量发送的形式来实现。

第一项测试我们使用`新增可用券(100)`操作将可用券数量由0逐级递增到1000：

|     | 100 | 200 | 300 | 400 | 500 | 600 | 700 | 800 | 900 | 1000 | 
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | ---- |
|native|84.6|69.8|71.6|75|77.2|78.8|82.8|93.2|93.4|105.4|
|wepy2|118.4|168.6|204.6|246.4|288.6|347.8|389.2|434.2|496|539|
|uniapp|121.2|100|96|98.2|97.8|99.6|104|102.4|109.4|107.6|
|mpx|110.4|87.2|82.2|83|80.6|79.6|86.6|90.6|89.2|96.4|
|chameleon|116.8|115.4|117|119.6|122|125.2|133.8|133.2|144.8|145.6|
|mpvue|112.8|121.2|140|169|198.8|234.2|278.8|318.4|361.4|408.2|
|kbone|556.4|762.4|991.6|1220.6|1468.8|1689.6|1933.2|2150.4|2389|2620.6|
|taro next|470|604.6|759.6|902.4|1056.2|1228|1393.4|1536.2|1707.8|1867.2|

然后我们按顺序逐项点击`删除可用券(all)` > `新增可用券(1000)` > `更新可用券(1)` > `更新可用券(all)` > `删除可用券(1)`：

|     | delete(all) | add(1000) | update(1) | update(all) | delete(1) |
| --- | ----------- | --------- | --------- | ----------- | --------- |
| native |32.8|295.6|92.2|92.2|83|
| wepy2 |56.8|726.4|49.2|535|530.8|
| uniapp |43.6|584.4|54.8|144.8|131.2|
| mpx |41.8|489.6|52.6|169.4|165.6|
| chameleon |39|765.6|95.6|237.8|144.8|
| mpvue |103.6|669.4|404.4|414.8|433.6|
| kbone |120.2|4978|2356.4|2419.4|2357|
| taro next |126.6|3930.6|1607.8|1788.6|2318.2|

> 该项测试中初期我update(all)的逻辑是循环对每个列表项进行更新，形如`listData.forEach((item)=>{item.count++})`，发现在chameleon框架中执行界面会完全卡死，追踪发现chameleon框架中没有对setData进行异步合并处理，而是在数据变动时直接同步发送，这样在数据量为1000的场景下用该方式进行更新会高频触发1000次setData，导致界面卡死；对此，我在chameleon框架的测试demo中，将update(all)的逻辑调整为深clone产生一份更新后的listData，再将其整体赋值到this.listData当中，以确保该项测试能够正常进行。


该项测试的结论为：  
native > mpx ≈ uniapp > chameleon > mpvue > wepy2 > taro next > kbone

结论分析：
* mpx和uniapp在框架内部进行了完善的diff优化，随着数据量的增加，两个框架的新增耗时没有显著上升；
* wepy2会在数据变更时对props数据也进行setData，在该场景下造成了大量的无效性能损耗，导致性能表现不佳；
* kbone和taro next采用了动态渲染方案，每次新增更新时会发送大量描述dom结构的数据，与此同时动态递归渲染的耗时也远大于常规的静态模板渲染，使得这两个框架在所有的更新场景下耗时都远大于其他框架。

### 页面更新耗时(有后台数据)

刷新页面后我们使用`新增不可用券(1000)`创建后台数据，观察该操作是否会触发setData并统计耗时

|     | back add(1000) | 
| --- | -------------- |
| native | 45.2
| wepy2 | 174.6
| uniapp | 89.4
| mpx | 0
| chameleon | 142.6
| mpvue | 134
| kbone | 0
| taro next | 0

> mpx进行setData优化时inspired by vue，使用了编译时生成的渲染函数跟踪模板数据依赖，在后台数据变更时不会进行setData调用，而kbone和taro next采用了动态渲染技术模拟了web底层环境，在上层完整地运行了vue框架，也达到了同样的效果。

然后我们执行和上面无后台数据时相同的操作进行耗时统计，首先是递增100：

|     | 100 | 200 | 300 | 400 | 500 | 600 | 700 | 800 | 900 | 1000 | 
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | ---- |
| native |88|69.8|71.2|80.8|79.4|84.4|89.8|93.2|99.6|108|
| wepy2 |121|173.4|213.6|250|298|345.6|383|434.8|476.8|535.6|
| uniapp |135.4|112.4|110.6|106.4|109.6|107.2|114.4|116|118.8|117.4|
| mpx |112.6|86.2|84.6|86.8|90|87.2|91.2|88.8|92.4|93.4|
| chameleon |178.4|178.2|186.4|184.6|192.6|203.8|210|217.6|232.6|236.8|
| mpvue |139|151|173.4|194|231.4|258.8|303.4|340.4|384.6|429.4|
| kbone |559.8|746.6|980.6|1226.8|1450.6|1705.4|1927.2|2154.8|2367.8|2617|
| taro next |482.6|626.2|755|909.6|1085|1233.2|1384|1568.6|1740.6|1883.8|

然后按下表操作顺序逐项点击统计

|     | delete(all) | add(1000) | update(1) | update(all) | delete(1) |
| --- | ----------- | --------- | --------- | ----------- | --------- |
| native |43.4|299.8|89.2|89|87.2|
| wepy2 |43.2|762.4|50|533|522.4|
| uniapp |57.8|589.8|62.6|160.6|154.4|
| mpx |45.8|490.8|52.8|167|166| 
| chameleon |93.8|837|184.6|318|220.8|
| mpvue |124.8|696.2|423.4|419|430.6|
| kbone |121.4|4978.2|2331.2|2448.4|2348
| taro next |129.8|3947.2|1610.4|1813.8|2290.2|

该项测试的结论为：  
native > mpx > uniapp > chameleon > mpvue > wepy2 > taro next > kbone

结论分析：
* 具备模板数据跟踪能力的三个框架mpx，kbone和taro next在有后台数据场景下耗时并没有显著增加；
* wepy2当中的diff精度不足，耗时也没有产生明显变化；
* 其余框架由于每次更新都会对后台数据进行deep diff，耗时都产生了一定提升。

### 页面更新耗时(大数据量场景)

> 由于mpvue和taro next的渲染全部在页面中进行，而kbone的渲染方案会额外新增大量的自定义组件，这三个框架都会在优惠券数量达到2000时崩溃白屏，我们排除了这三个框架对其余框架进行大数据量场景下的页面更新耗时测试

首先还是在无后台数据场景下使用`新增可用券(1000)`将可用券数量递增至5000：

|     | 1000 | 2000 | 3000 | 4000 | 5000 |
| --- | ---- | ---- | ---- | ---- | ---- |
| native |332.6|350|412.6|498.2|569.4|
| wepy2 |970.2|1531.4|2015.2|2890.6|3364.2| 
| uniapp |655.2|593.4|655|675.6|718.8| 
| mpx |532.2|496|548.6|564|601.8| 
| chameleon |805.4|839.6|952.8|1086.6|1291.8| 

然后点击`新增不可用券(5000)`将后台数据量增加至5000，再测试可用券数量递增至5000的耗时：

|     | back add(5000) | 
| --- | -------------- |
| native |117.4|
| wepy2 |511.6|
| uniapp |285|
| mpx |0|
| chameleon |824|

|     | 1000 | 2000 | 3000 | 4000 | 5000 |
| --- | ---- | ---- | ---- | ---- | ---- |
| native |349.8|348.4|430.4|497|594.8|
| wepy2 |1128|1872|2470.4|3263.4|4075.8| 
| uniapp |715|666.8|709.2|755.6|810.2|
| mpx |538.8|501.8|562.6|573.6|595.2| 
| chameleon |1509.2|1672.4|1951.8|2232.4|2586.2| 

该项测试的结论为：  
native > mpx > uniapp > chameleon > wepy2

结论分析：
* 在大数据量场景下，框架之间基础性能的差异会变得更加明显，mpx和uniapp依然保持了接近原生的良好性能表现，而chameleon和wepy2则产生了比较显著的性能劣化。

### 局部更新耗时

我们在可用券数量为1000的情况下，点击任意一张可用券触发选中状态，以测试局部更新性能

|     | toggleSelect(ms) |
| --- | ------------ |
| native |2|
| wepy2 |2.6| 
| uniapp |2.8| 
| mpx |2.2| 
| chameleon |2| 
| mpvue |289.6|
| kbone |2440.8| 
| taro next |1975| 

该项测试的结论为：  
native ≈ chameleon ≈ mpx ≈ wepy2 ≈ uniapp > mpvue > taro next > kbone

结论分析：
* 可以看出所有使用了原生自定义组件进行组件化实现的框架局部更新耗时都极低，这足以证明小程序原生自定义组件的优秀性和重要性；
* mpvue由于使用了页面更新，局部更新耗时显著增加；
* kbone和taro next由于递归动态渲染的性能开销巨大，导致局部更新耗时同样巨大。

### setData调用

我们将`proxySetData`的count和size选项设置为true，开启setData的次数和体积统计，重新构建后按照以下流程执行系列操作，并统计setData的调用次数和发送数据的体积。

操作流程如下：
1. 100逐级递增可用券(0->500)
2. 切换至不可用券
3. 新增不可用券(1000)
4. 100逐级递增可用券(500->1000)
5. 更新可用券(all)
6. 切换至可用券

操作完成后我们使用`getCount`和`getSize`方法获取累积的setData调用次数和数据体积，其中数据体积计算方式为JSON.stringify后按照utf-8编码方式进行体积计算，统计结果为：

|     | count | size(KB) |
| --- | ----- | ---- |
| native |14|803|
| wepy2 |3514|1124|
| mpvue |16|2127| 
| uniapp |14|274| 
| mpx |8|261|
| chameleon |2515|319|
| kbone |22|10572| 
| taro next |9|2321|


该项测试的结论为：  
mpx > uniapp > native > chameleon > wepy2 > taro next > mpvue > kbone

结论分析：
* mpx框架成功实现了理论上setData的最优；
* uniapp由于缺失模板追踪能力紧随其后；
* chameleon由于组件每次创建时都会进行一次不必要的setData，产生了大量无效setData调用，但是数据的发送本身经过diff，在数据发送量上表现不错；
* wepy2的组件会在数据更新时调用setData发送已经更新过的props数据，因此也产生了大量无效调用，且diff精度不足，发送的数据量也较大；
* taro next由于上层完全基于vue，在数据发送次数上控制到了9次，但由于需要发送大量的dom描述信息，数据发送量较大；
* mpvue由于使用较长的数据路径描述数据对应的组件，也产生了较大的数据发送量；
* kbone对于setData的调用控制得不是很好，在上层运行vue的情况依然进行了22次数据发送，且发送的数据量巨大，在此流程中达到了惊人的10MB。



### 页面渲染耗时(静态测试)

此处的页面渲染耗时与前面描述的动态测试场景中相同，测试结果如下：

|     | 页面渲染耗时 |
| --- | --------- |
| native |70.4| 
| wepy2 |86.6|
| mpvue |115.2| 
| uniapp |69.6| 
| mpx |66.6| 
| chameleon |65| 
| kbone |144.2| 
| taro next |119.8| 

该项测试的结论为：  
chameleon ≈ mpx ≈ uniapp ≈ native > wepy2 > mpvue ≈ taro next > kbone

结论分析：
* 除了kbone和taro next采用动态渲染耗时增加，mpvue使用页面模板渲染性能稍差，其余框架的静态页面渲染表现都和原生差不多。

## 结论

综合上述测试数据，我们得到最终的小程序框架运行时性能排名为：  
mpx > uniapp > chameleon > wepy2 > mpvue > taro next > kbone

## 一点私货

虽然kbone和taro next采用了动态渲染技术在性能表现上并不尽如人意，但是我依然认为这是很棒的技术方案。虽然本文从头到位都在进行性能测试和对比，但性能并不是框架的全部，开发效率和高可用性仍然是框架的重心，开发效率相信是所有框架设计的初衷，但是高可用性却在很大程度被忽视。从这个角度来说，kbone和taro next是非常成功的，不同于过去的转译思路，这种从抹平底层渲染环境的做法能够使上层web框架完整运行，在框架可用性上带来非常大的提升，非常适合于运营类简单小程序的迁移和开发。

我主导开发的mpx框架(https://github.com/didi/mpx) 选择了另一条道路解决可用性问题，那就是基于小程序原生语法能力进行增强，这样既能避免转译web框架时带来的不确定性和不稳定性，同时也能带来非常接近于原生的性能表现，对于复杂业务小程序的开发者来说，非常推荐使用。在跨端输出方面，mpx目前能够完善支持业内全部小程序平台和web平台的同构输出，滴滴内部最重要最复杂的小程序——滴滴出行小程序完全基于mpx进行开发，并利用框架提供的跨端能力对微信和支付宝入口进行同步业务迭代，大大提升了业务开发效率。

