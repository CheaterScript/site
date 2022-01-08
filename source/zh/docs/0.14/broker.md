title: Broker
---
`ServiceBroker`是Moleculer的主要组成部分。 它处理服务、调用动作、发送事件与远程节点通信等。 您必须在每个节点上创建一个 `ServiceBroker` 实例。

<div align="center">
    <img src="assets/service-broker.svg" alt="Broker logical diagram" />
</div>

## 创建 ServiceBroker

{% note info %}
**Quick tip:** 不需要在你的项目中手动创建 ServiceBroker。 使用 [Moleculer Runner](runner.html) 创建、执行服务管理器并加载服务。 [阅读更多关于 Moleculer Runner 的内容](runner.html)。
{% endnote %}

**使用默认设置创建服务管理器：**
```js
const { ServiceBroker } = require("moleculer");
const broker = new ServiceBroker();
```

**使用自定义设置创建服务管理器：**
```js
const { ServiceBroker } = require("moleculer");
const broker = new ServiceBroker({
    nodeID: "my-node"
});
```

**创建使用推送系统的服务管理器与远程节点通信：**
```js
const { ServiceBroker } = require("moleculer");
const broker = new ServiceBroker({
    nodeID: "node-1",
    transporter: "nats://localhost:4222",
    logLevel: "debug",
    requestTimeout: 5 * 1000
});
```


### 元数据选项
使用`metadata`属性来存储自定义值。 它可以用于自定义 [middleware](middlewares.html#Loading-amp-Extending) 或 [strategy](balancing.html#Custom-strategy)。

```js
const broker = new ServiceBroker({
    nodeID: "broker-2",
    transporter: "NATS",
    metadata: {
        region: "eu-west1"
    }
});
```
{% note info %}
`metadata`属性可以通过运行 `$node.list` 动作获得。
{% endnote %}

{% note info %}
`metadata`属性会传送到其他节点。
{% endnote %}

## Ping
若要ping 远程节点，请使用 `broker.ping` 方法。 您可以ping 一个节点，或者所有可用的节点。 它返回一个`Promise`，其中包含已收到的 ping信息(latency, time difference)。 超时是可定义的。

### 以1秒超时Ping 节点
```js
broker.ping("node-123", 1000).then(res => broker.logger.info(res));
```

**Output**
```js
{ 
    nodeID: 'node-123', 
    elapsedTime: 16, 
    timeDiff: -3 
}
```
> `timeDiff`值是这两个节点之间系统时钟的差异。

### Ping 多个节点
```js
broker.ping(["node-100", "node-102"]).then(res => broker.logger.info(res));
```

**Output**
```js
{ 
    "node-100": { 
        nodeID: 'node-100', 
        elapsedTime: 10, 
        timeDiff: -2 
    },
    "node-102": { 
        nodeID: 'node-102', 
        elapsedTime: 250, 
        timeDiff: 850 
    } 
}
```

### Ping 所有可用节点
```js
broker.ping().then(res => broker.logger.info(res));
```

**Output**
```js
{ 
    "node-100": { 
        nodeID: 'node-100', 
        elapsedTime: 10, 
        timeDiff: -2 
    } ,
    "node-101": { 
        nodeID: 'node-101', 
        elapsedTime: 18, 
        timeDiff: 32 
    }, 
    "node-102": { 
        nodeID: 'node-102', 
        elapsedTime: 250, 
        timeDiff: 850 
    } 
}
```

## ServiceBroker 属性

| 名称                        | 类型                     | 说明                             |
| ------------------------- | ---------------------- | ------------------------------ |
| `broker.options`          | `Object`               | 服务管理器选项                        |
| `broker.Promise`          | `Promise`              | Bluebird Promise 类.            |
| `broker.started`          | `Boolean`              | 服务管理器状态                        |
| `broker.namespace`        | `String`               | Namespace.                     |
| `broker.nodeID`           | `String`               | Node ID.                       |
| `broker.instanceID`       | `String`               | Instance ID.                   |
| `broker.metadata`         | `Object`               | 来自服务管理器选项的 Metadata            |
| `broker.logger`           | `Logger`               | ServiceBroker 的日志类.            |
| `broker.cacher`           | `Cacher`               | Cacher 实例                      |
| `broker.serializer`       | `Serializer`           | Serializer 实例.                 |
| `broker.validator`        | `Any`                  | 参数验证器实例。                       |
| `broker.services`         | `Array<Service>` | 本地服务.                          |
| `broker.metrics`          | `MetricRegistry`       | 内置的 Metric Registry.           |
| `broker.tracer`           | `Tracer`               | 内置 Tracer 实例。                  |
| `broker.errorRegenerator` | `Regenerator`          | Built-in Regenerator instance. |

## ServiceBroker 方法

| 名称                                                        | 响应                    | 说明                        |
| --------------------------------------------------------- | --------------------- | ------------------------- |
| `broker.start()`                                          | `Promise`             | 启动服务管理器                   |
| `broker.stop()`                                           | `Promise`             | 停止服务管理器                   |
| `broker.repl()`                                           | -                     | 以 REPL (交互式方式) 启动         |
| `broker.errorHandler(err, info)`                          | -                     | 调用全局错误处理器。                |
| `broker.getLogger(module, props)`                         | `Logger`              | 获取日志子类。                   |
| `broker.fatal(message, err, needExit)`                    | -                     | 抛出错误并退出进程。                |
| `broker.loadServices(folder, fileMask)`                   | `Number`              | 从文件夹中加载服务。                |
| `broker.loadService(filePath)`                            | `Service`             | 从文件载入服务。                  |
| `broker.createService(schema, schemaMods)`                | `Service`             | 依据 schema 创建一个服务。         |
| `broker.destroyService(service)`                          | `Promise`             | 销毁加载的本地服务。                |
| `broker.getLocalService(name)`                            | `Service`             | 通过全名获取本地服务实例(如：`v2.poss`) |
| `broker.waitForServices(serviceNames, timeout, interval)` | `Promise`             | 等待服务(完成启动)。               |
| `broker.call(actionName, params, opts)`                   | `Promise`             | 调用一个服务动作                  |
| `broker.mcall(def)`                                       | `Promise`             | 调用多个服务动作                  |
| `broker.emit(eventName, payload, opts)`                   | -                     | 发出事件                      |
| `broker.broadcast(eventName, payload, opts)`              | -                     | 广播一个事件。                   |
| `broker.broadcastLocal(eventName, payload, opts)`         | -                     | 仅向本地服务广播一个事件。             |
| `broker.ping(nodeID, timeout)`                            | `Promise`             | Ping 远程节点。                |
| `broker.hasEventListener("eventName")`                    | `Boolean`             | 服务管理器是否在侦听指定的事件。          |
| `broker.getEventListeners("eventName")`                   | `Array<Object>` | 返回指定事件的所有已注册事件监听器。        |
| `broker.generateUid()`                                    | `String`              | 生成一个 UUID/token。          |
| `broker.callMiddlewareHook(name, args, opts)`             | -                     | 在注册的 middlewares 中调用异步钩子. |
| `broker.callMiddlewareHookSync(name, args, opts)`         | -                     | 在注册的 middlewares 中调用同步钩子. |
| `broker.isMetricsEnabled()`                               | `Boolean`             | 检查计量功能是否已启用。              |
| `broker.isTracingEnabled()`                               | `Boolean`             | 检查跟踪功能是否已启用。              |

## 全局错误处理器
全局错误处理程序是处理异常的通用方式。 它会捕获未处理的动作 & 事件处理器错误。

**捕获，处理 & 把错误记录到日志**
```js
const broker = new ServiceBroker({
    errorHandler(err, info) {
        // Handle the error
        this.logger.warn("Error handled:", err);
    }
});
```

**捕获 & 重新抛出错误**
```js
const broker = new ServiceBroker({
    errorHandler(err, info) {
        this.logger.warn("Log the error:", err);
        throw err; // Throw further
    }
});
```

{% note info %}
`info`对象包含服务管理器和服务实例、当前 context 和动作或事件定义。
{% endnote %}
