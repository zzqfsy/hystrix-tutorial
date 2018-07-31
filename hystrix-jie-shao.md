# Hystrix介绍

在目前流行的微服务框架，我们简单的抛出几个问题：

1. 服务之间的调用非常频繁，复杂度这么高的系统可用性如何？
2. 如果服务的依赖服务发生故障，会发生什么情况？
3. 故障如何快速定位？

Hystrix官方回答了这个，并给出了解决方案：

> 在分布式环境中，许多服务依赖项中的一些不可避免地会失败。Hystrix是一个库，可通过添加延迟容错和容错逻辑来帮助您控制这些分布式服务之间的交互。 Hystrix通过隔离服务之间的访问点，阻止它们之间的级联故障以及提供后备选项来实现这一点，所有这些都可以提高系统的整体弹性。

Hystrix旨在执行以下操作：

* 通过第三方客户端库访问（通常通过网络）依赖关系，以防止和控制延迟和故障。
* 在复杂的分布式系统中停止级联故障。
* 快速失败并迅速恢复。
* 在可能的情况下，后退并优雅地降级。
* 实现近实时监控，警报和操作控制。

## Hystrix解决了什么问题？Hystrix如何实现其目标？

> 请查看[wiki-homepage](https://github.com/Netflix/Hystrix/wiki)​

## Hystrix的工作原理？

> 请查看[How-it-Works](https://github.com/Netflix/Hystrix/wiki/How-it-Works)​

hystrix通过\*command来包装业务逻辑，下图展示包装层干了什么事情

![](.gitbook/assets/image%20%281%29.png)

## Hystrix的使用方法？

> 请查看[How-To-Use](https://github.com/Netflix/Hystrix/wiki/How-To-Use)​

