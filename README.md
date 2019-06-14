# spring-boot-redis-sentinel-example

Spring Boot连接[Redis Sentinel][redis-sentinel]的例子。

## 基本信息

拓扑（M代表redis-master，S代表redis-sentinel，R代表redis-slave，C代表Spring Boot Application）：

```txt
     +---------------------------+
     |                           |
  +-----+       +-----+       +-----+
  |  M  |-------|  S  |-------|  R  |
  +-----+       +-----+       +-----+
     |             |
     |          +-----+
     +----------|  C  |
                +-----+
```

`application.yaml`配置：

```yaml
spring:
  redis:
#    host: redis-master
#    port: 6379
    password: abc
    sentinel:
      master: springboot
      nodes:
        - redis-sentinel:26379
```

注意这里不需要配置master的host和port，这些信息会从Redis Sentinel中得到。

## 演示步骤

打包并构建镜像：`mvn clean install dockerfile:build`

进入`docker`目录，执行`docker-compose up -d`

观察Spring Boot Application的日志：`docker logs -f docker_spring-boot_1`，会发现每隔3秒执行`INCR foo`：

```txt
07:53:49.205  INFO  hello.Application                    : INCR foo: 1
07:53:52.212  INFO  hello.Application                    : INCR foo: 2
07:53:55.213  INFO  hello.Application                    : INCR foo: 3
07:53:58.216  INFO  hello.Application                    : INCR foo: 4
07:54:01.217  INFO  hello.Application                    : INCR foo: 5
```

停止redis-master：`docker stop docker_redis-master_1`，会看到Spring Boot Application的Redis链接出现了问题：

```txt
07:54:37.206  INFO  hello.Application                    : INCR foo: 17
07:54:40.204  INFO  hello.Application                    : INCR foo: 18
07:54:42.238  INFO  i.l.core.protocol.ConnectionWatchdog : Reconnecting, last destination was /10.0.19.4:6379
07:54:52.247  WARN  i.l.core.protocol.ConnectionWatchdog : Cannot reconnect: io.netty.channel.ConnectTimeoutException: connection timed out: /10.0.19.4:6379
...
07:55:22.560  INFO  i.l.core.protocol.ConnectionWatchdog : Reconnecting, last destination was 10.0.19.4:6379
07:55:22.842  WARN  i.l.core.protocol.ConnectionWatchdog : Cannot reconnect: io.netty.channel.AbstractChannel$AnnotatedNoRouteToHostException: Host is unreachable: /10.0.19.4:6379
...
07:55:29.582  INFO i.l.core.protocol.ConnectionWatchdog  : Reconnecting, last destination was 10.0.19.4:6379
07:55:32.353  WARN i.l.core.protocol.ConnectionWatchdog  : Cannot reconnect: io.netty.channel.AbstractChannel$AnnotatedNoRouteToHostException: Host is unreachable: /10.0.19.4:6379
...
```

等待大约60秒，Redis Sentinel介入，将redis-slave提拔为master，链接恢复：

```txt
07:55:43.860  INFO i.l.core.protocol.ConnectionWatchdog  : Reconnecting, last destination was 10.0.19.4:6379
07:55:43.882  INFO i.l.core.protocol.ReconnectionHandler : Reconnected to 10.0.19.6:6379
07:55:43.887  INFO hello.Application                     : INCR foo: 20
07:55:43.889  INFO hello.Application                     : INCR foo: 21
07:55:43.891  INFO hello.Application                     : INCR foo: 22
07:55:43.892  INFO hello.Application                     : INCR foo: 23
```

此时拓扑变成这样：

```txt
     +-------------//------------+
     |                           |
  +-----+       +-----+       +-----+
  |  M  |--//---|  S  |-------| [M] |
  +-----+       +-----+       +-----+
     |             |             |
     |          +-----+          |
     +----//----|  C  |----------+
                +-----+
```

清理容器：`docker-compose down`。

## Master重启之后的问题

这个问题和Spring Boot没有关系，是Redis本身的。如果我们把前面停掉的master重启，sentinel是不会感知到这个master的，因为这个master的ip变了（见这个[comment][link]）：

你可以观察重启之以后的master的INFO：

```bash
$ docker exec docker_redis-master_1 redis-cli -a abc INFO replication
# Replication
role:master
...
```

可以看到它启动之后还是master，不是slave。这样的话就等于出现了两个master，这就出问题了。

BTW，redis的配置中可以使用hostname，比如`slaveof redis-master`。但是redis-sentinel，使用的是ip，即使你配置的是hostname，最终也是ip。执行下面命令可以看见sentinel的配置：

```bash
$ docker exec docker_redis-sentinel_1 cat /bitnami/redis-sentinel/conf/sentinel.conf | grep springboot

sentinel monitor springboot 10.0.25.2 6379 1
sentinel down-after-milliseconds springboot 60000
sentinel auth-pass springboot abc
sentinel config-epoch springboot 0
sentinel leader-epoch springboot 0
sentinel known-slave springboot 10.0.25.5 6379
```

### 解决办法1：使用host network

使用host network来部署redis-master、redis-slave，使用`<host-ip>:<container-port>`来访问它们，因为host的ip是比较固定的，可以缓解这个问题。

用host network则还有一个限制：不能在同一个host上启动两个相同container-port的容器。

### 解决办法2：publish端口

把redis-master、redis-slave的端口publish到host上，redis.config中把`slave-announce-ip`和`slave-announce-port`设置为host-ip和host-port，最后使用`<host-ip>:<host-port>`访问，同样也是利用host的ip固定特性来解决这个问题。

[redis-sentinel]: https://redis.io/topics/sentinel
[link]: https://github.com/bitnami/bitnami-docker-redis-sentinel/issues/2#issuecomment-501613810