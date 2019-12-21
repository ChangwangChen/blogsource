---
title: 配置Sentinel监控Redis主从集群
date: 2016-09-28 20:50:38
categories: Redis
tags: [Redis]
---

# 缘起
项目中使用[Redis](http://www.redis.cn)做缓存配置，为了实现HA（High Availability），需要配置 [Redis Sentinel](http://www.redis.cn/topics/sentinel.html)
来做Redis的故障转移监控。

<!--more-->

# 环境搭建
我们测试环境使用的Redis主从复制架构和Sentinel监控都很简单。总的来说就是在2台服务器上面做Redis主从复制，然后在其中一台服务器上面部署
Redis Sentinel服务做监控。

我们测试的两台主机IP地址分别是：`192.168.42.20` 和 `192.168.42.21`

## Redis主从服务搭建
配置Redis主从很简单，其他的配置信息不变，只是需要在从库的配置文件中加上一句`slaveof`即可。

分别在两台服务器中安装 `Redis` 实例，在`192.168.42.21`的服务器上面部署从库。使用的配置信息：

```
 //slaveof <ip> <port>
 
 slaveof 192.168.42.20 6379
```

## 配置Redis Sentinel
Redis Sentinel 的配置也非常简单，安装Redis的时候，自带的就有`sentinel.conf`配置文件。我们需要在配置文件中添加下面几项：

```
port 26379  #Sentinel服务监听的端口号

logfile "/var/log/redis-sentinel/26379.log"  #日志文件的位置

loglevel notice  #日志记录的级别

daemonize yes  #设置为后台运行

# sentinel monitor <master-name> <ip> <redis-port> <quorum>
sentinel monitor mymaster 192.168.42.20 6379 2   #这里的 <master-name> 很关键，不过可以任意修改，但是整个文件中要保持一致

```
修改配置文件之后，通过指令 `redis-sentinel sentinel-26379.conf` 即可启动服务。

通常我们会设置多个 `Redis Sentinel` 服务来监控Redis，保证健壮性。在配置方面只需要修改不同服务监听的端口号即可。

# 注意事项

* 配置Redis的时候，不需要指定`bind` （或者使用`bind 0.0.0.0`来监听所有IP），这样的话Redis会监听`*:6379`，不论内网或者外网都能访问Redis服务
* 配置Sentinel的时候，`sentinel monitor`语句中的IP需要配置成外网的IP，这样保证外部服务能够通过Sentinel来访问到Redis服务
* 配置Sentinel名称的时候，可以任意起名称，但是在整个文件中需要保持一致

# 运行效果

我们使用的是 Laravel 框架来访问 Redis Sentinel 服务，并通过它来间接访问 Redis。

在代码中我们使用了 [Laravel-PSRedis](https://github.com/Indatus/laravel-PSRedis) 这个拓展包，也可通过 [Composer](http://www.phpcomposer.com/)来安装。

```json
{
    "require" : {
        "indatus/laravel-ps-redis": "^1.2"
    }
}
```
项目的配置中，在 `config/database.php` 文件中的 `redis` 项下面添加：

```php

        'nodeSetName' => 'mymaster', //Sentinel的名称
        'masters' => [
            [
                'host' => env('REDIS_HOST', '192.168.20.40'), //IP
                'port' => 26379 //端口号
            ]
        ],
        'backoff-strategy' => [
            'max-attempts' => 10, // 最多10次请求来判断服务是否能够访问
            'wait-time' => 500, // 每次请求的时间间隔 500 毫秒
            'increment' => 1.5, // 每次请求的等待时间会增加 1.5 倍
        ],

```

通过以上配置，可以通过访问 Redis Sentinel 来替代直接访问 Redis 服务。当其中的某个 Redis 服务挂掉之后，Sentinel 会自动在其他的 Redis 服务中
选举一个做为 Master，剩下的作为 Slave。

我们测试的结果是，手动将 `192.168.20.40` 中的 Redis 进程 kill 之后，仍然可以通过访问 Redis Sentinel 来访问 Redis。并且，访问的 Redis 实例已经自动转移到了 `192.168.20.41` 服务器上面。

# 参考文档
* [Redis Replication](http://redis.cn/topics/replication.html)
* [Redis Sentinel](http://www.redis.cn/topics/sentinel.html)
* [Redis 自动故障转移sentinel（哨兵）实践](http://www.tuicool.com/articles/7bye2yv)
* [Laravel-PSRedis](https://github.com/Indatus/laravel-PSRedis)
