---
title: 一次node服务的内存泄漏排查记录(DevTool使用)
description: 正在连接云数据库🧠
tags: NodeJs
---

<a name="nWFkX"></a>
## 背景：
<a name="h9Cdi"></a>
## ![](https://raw.githubusercontent.com/sultan-young/picture-bed/master/assets/20240410172529.png#height=321&id=SB2Uc&originHeight=556&originWidth=443&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=&width=256)![](https://raw.githubusercontent.com/sultan-young/picture-bed/master/assets/20240410172550.png#height=302&id=u2xR7&originHeight=312&originWidth=471&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=&width=456)
公司的一个node服务在某台ECS服务器上经常发生内存或CPU告警(部署到其它服务器的不存在)，且有一定概率会造成机器宕机。本文记录一下笔者排查的思路。
<a name="VYoGR"></a>
## 分析:
**既然只在某台服务器上发生，那么自然会想到是不是这台服务器的环境存在什么问题, 遂进行排查：**<br />正常服务器的配置(内存8G):<br />![](https://raw.githubusercontent.com/sultan-young/picture-bed/master/assets/20240410172451.png#id=EGJ4I&originHeight=141&originWidth=1285&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)<br />发生内存告警的服务器配置(内存4G):<br />![](https://raw.githubusercontent.com/sultan-young/picture-bed/master/assets/20240410172435.png#id=pbD73&originHeight=134&originWidth=1300&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)<br />内存确实是存在差距的，但是转念一想，我们在业务上的体量也是不一样的呀，8G服务器所在地区的业务体量是4G服务器所在的N倍(N > 2)。<br />而且，我们的这个node服务本质是一个网关，不会处理太多业务逻辑，也就不需要太多的常驻内存。也就是说，**一定存在严重的内存泄漏问题**。

找运维同学，要了一下出问题时候机器的运行情况。![](https://raw.githubusercontent.com/sultan-young/picture-bed/master/assets/20240410173247.png#id=qipG9&originHeight=718&originWidth=1571&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)<br />这张图是内存报警时候的图，机器还没挂掉。挂掉的时候，CPU、内存、BPS全部打满了...

我们使用devTools分析一波。<br />开启测试环境Node服务的inspect模式，便于观测内存的变化。<br />`node --inspect-brk=0.0.0.0:9229 dist/index.js`
:::danger
一定要保证在网络安全的环境下使用这种方式，如果你的服务器是外网可访问的，那么任何人都可以调试你的服务。
:::
在chrome打开 `chrome://inspect/#devices`<br />![](https://raw.githubusercontent.com/sultan-young/picture-bed/master/assets/20240411113218.png#height=312&id=Fd6sf&originHeight=879&originWidth=1429&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=&width=508)<br />配置好之后，你会看到 **Remote Target** 下方出现了一条，点击inspect即刻打开devTool<br />![](https://raw.githubusercontent.com/sultan-young/picture-bed/master/assets/20240411113626.png#height=349&id=zz9qg&originHeight=403&originWidth=702&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=&width=608)<br />点击后，切换到Memory，你会看到如下图的面板![](https://raw.githubusercontent.com/sultan-young/picture-bed/master/assets/20240411171620.png#id=jaIn9&originHeight=950&originWidth=1802&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)<br />三个选项的区别和使用场景：<br />**Heap snapshot(堆快照): **<br />将当前的Node服务的堆栈情况生成一个快照。通过快照我们可以清晰某一刻，系统中都有哪些数据在内存中。我们也可以通过隔一段时间打一个快照，对比两个快照的内存变化，从而分析系统的内存监控情况。<br />![](https://raw.githubusercontent.com/sultan-young/picture-bed/master/assets/20240411173011.png#id=turlJ&originHeight=590&originWidth=1378&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
> 当你对系统内存泄漏的信息不多时，可以使用打快照的方式，对比两个时间节点的内存变化。

**Allocation instrumentation on timeline(内存仪表盘):**<br />与`Heap snapshot`不同的是，启用这个模式后，分析面板会多一个时间线，且分析会一直进行下去。指点你点击终止。<br />![](https://raw.githubusercontent.com/sultan-young/picture-bed/master/assets/image%20(1).png)<br />Timeline 中，蓝色柱代表未被回收的内存(可能存在内存泄漏)，灰色代表被GC已经回收的内存。<br />选中对应的柱状图，即可看到当时的内存情况。
> 当你知道某些操作可能会引发内存泄漏的时候，可以用这个用来验证。


**Allocation sampling(分配抽样):**<br />像内存仪表盘一样，它也会一直运行。<br />不同的是，它记录的是内存分配的过程，而不是每个时间点内存的所有情况。<br />它周期性的对内存分配进行抽样，对性能的损耗更小，适合需要长时间收集的场景。<br />![](https://raw.githubusercontent.com/sultan-young/picture-bed/master/assets/image%20(2).png)
> 当你想观测系统的内存分配情况时候，可以使用这种方式。


有了分析工具，笔者决定先使用 `Heap snapshot` 模式，先对刚启动的服务打一个快照，并模拟用户，在前端页面点一点(为了更加真实，笔者这里使用了压测工具，将事先写好的接口并行调用)。调用了100次之后再次打了一个快照，神奇的事情出现了，内存竟然从`33.1MB`变成了`37.4MB`。<br />![](https://raw.githubusercontent.com/sultan-young/picture-bed/master/assets/20240411193629.png#id=HfkU2&originHeight=297&originWidth=1736&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)<br />查看两个快照之间分配的对象发现，有 `(string)`的占用了3.4MB的内存（Shallow Size），占用了内存增长的大部分。

Memory面板名词解释，部分引用自[https://zhuanlan.zhihu.com/p/80792297](https://zhuanlan.zhihu.com/p/80792297)
> (string) 字符串原始值。指字符串的引用。除了明确使用 new String创建的字符串外，其它的均属于(string)  
> (array) 通常指的是数组索引属性的集合, 它是指那些由于元素增加或删除而发生变化的索引.  
> Array 代表实际的Array对象实例,指数组对象的整个生命周期变化——新数组的创建或现有数组的回收  
> Contructor - 表示使用此构造函数创建的所有对象  
> Distance - 显示使用节点最短简单路径时距根节点的距离  
> Shallow Size - 显示通过特定构造函数创建的所有对象浅层大小的总和。浅层大小是指对象自身占用的内存大小（一般来说，数组和字符串的浅层大小比较大）  
> Retained Size - 显示同一组对象中最大的保留大小。某个对象删除后（其依赖项不再可到达）可以释放的内存大小称为保留大小。
> New - Comparison 特有 - 新增项  
> Deleted - Comparison 特有 - 删除项  
> Delta - Comparison 特有 - 增量 （Delta = New - Deleted）  
> Alloc. Size - Comparison 特有 - 内存分配大小  
> Freed Size - Comparison 特有 - 释放大小  
> Size Delta - Comparison 特有 - 内存增量 （Size Delta = Alloc.Size - Freed Size）  


![](https://raw.githubusercontent.com/sultan-young/picture-bed/master/assets/20240411193923.png#id=eNmEe&originHeight=398&originWidth=1405&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)<br />点开进一步分析，发现这个string正是我们刚才接口请求的响应体。定位到之指示的`apiProxy.service.js`对应的代码片段:
```typescript
public static proxy(ctx: KoaContext, path: string, method: string, headers: AnyObject): Promise<any> {
        const {
            port,
            hostname,
        } = ApiProxyService.getRoute(path, headers);

        return new Promise((resolve, reject) => {
            const options: RequestOptions = {
                hostname: hostname,
                path: path,
                method: method,
                port: port,
                headers: headers,
            };

            const request = http.request(options, (res) => {
                if (res.statusCode !== 200) {
                    ...
                    return;
                }
                let data = '';
                res.on('data', (chunk) => {
                    data = data + chunk;
                });
                res.on('end', () => {
                  ...
                });
            });

            request.on('error', (e) => {
              ...
            });

            request.end();
            // 自动定位到了这里
            process.on('uncaughtException', function (err) {
                console.warn(err.stack);
                console.warn('NOT exit...');
            });

        });
    }
```
从代码上看，在 `proxy`中增加了 一个 `uncaughtException`事件，相当于每个请求，都会增加一个全局事件，当这个事件触发时候，会将之前监听的所有事件全部触发。笔者将监听 `uncaughtException` 的事件放置到了全局，确保应用启动只触发一次，再次inspect Memory后，发现一切恢复了正常！
> 这个问题存在了很久，只是我们一直使用PM2进行管理，当内存超出预设值时候，会对服务进行自动重启。在宕机的这台服务器上，由于机器内存较小，当多个服务的内存同时飙高时候，虽然没达到单个服务预设的内存阈值，但是整体的内存已经达到了机器的上线，从而使node服务频繁触发GC，导致CPU随之飙高，最终机器宕机。

:::danger
各位看官在使用PM2时候，一定要有对应的监控手段。否则可能会像笔者本次一样，系统存在问题而不自知。
:::
<a name="Dvovq"></a>
## 总结：
:::info
到这里，我们把已经存在的内存泄漏问题解决了。但是我们的node服务无限制使用系统资源的问题还是存在(即一个node服务负载会影响到其他的node服务)。
:::
这个问题反映出来了我们对团队对服务整体的运行情况缺少把控的手段，强依赖于运维对机器设置的整体预警, 这是笔者针对本次问题，做出的一些后续的防范思考:

1. **增加监控：**每个node服务都需要增加Monitor, 并有相关的告警机制和面板查看。这个可以直接接入笔者之前私有化部署的Sentry体系中。
2. **动态扩容：**业务中有明显的高低峰，且运行在机器上的服务也有着高低权重。考虑使用Docker对所有服务进行管理，对所有应用设置资源上限，并给高权重的应用设置较为宽松的扩容条件，给低权重的应用设置严格的扩容条件(或者不设置)。 

<a name="cRfxQ"></a>
## 问题探讨:
![](https://raw.githubusercontent.com/sultan-young/picture-bed/master/assets/20240411214642.png#id=PKBuR&originHeight=580&originWidth=1718&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)<br />从`Memory`面板来看，每次请求后，内存中都会将接口响应的字符串存下来。字符串的内存泄漏是本次的罪魁祸首，重复监听只是次要的。<br />但是从代码层面分析
```typescript
process.on('uncaughtException', function (err) {
                console.warn(err.stack);
                console.warn('NOT exit...');
            });
```
`uncaughtException`事件只会造成重复监听，它的回调函数中也没有对外部的引用，不会触发闭包, 理论上来讲不会和响应体字符串产生关联。<br />求各位看官答疑解惑。
