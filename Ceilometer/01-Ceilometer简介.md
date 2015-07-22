[toc]

# ceilometer概略

## ceilometer主要概念
- **resource:**可监控资源,云主机、硬盘、镜像、路由器、交换机...
- **meter:**监控项,cpu_util,memory.usage,network.incoming.bytes.rate...
- **sample:**样本数据点
- **statistics:**sample处理过的数据,包含max,min,avg,sum,count等
- **alarm:**告警,定义监控对象、监控周期、触发条件等参数
- **evaluator:**评估是否满足告警条件,满足条件发消息到notifier
- **notifier:**通知器,定义告警条件满足后的动作,邮件、短信、日志
- **collector:**数据收集器,将收集到的数据存入后端数据库
- **pushing agents:**
- **polling agents:**
- [完整术语表参见](http://docs.openstack.org/developer/ceilometer/glossary.html#term-metering)

## 获取测量值数据方式
- 第三方的数据发送者把数据以通知消息(Notification Message)的方式发送到消息总线(Notification Bus)上,Ceilometer 中的 Notification Agent 会获取这些通知事件,从中提取测量数据
- Ceilometer 中的 Polling Agent 会根据配置,定期轮询,主动通过各种 API 或者其他通信协议去远端或者本地的不同服务实体中获取所需要的测量数据
- 用户通过调用 Ceilometer RESTful API 直接把利用其他方式获取的任意测量数据送达给 Ceilometer
- [**ceilometer路由方式**](https://pecan.readthedocs.org/en/latest/routing.html)


## ceilometer服务
由 setup.cfg [entry_points]配置块下配置及 ceilometer/cli.py 下对应函数名,可见有以下几个主要服务


[参考RDO文档](https://www.rdoproject.org/CeilometerQuickStart)

- **compute agent**: polls the local libvirt daemon to acquire performance data for the local instances, messages and emits these data as AMQP notifications
- **central agent**: polls the public RESTful APIs of other openstack services such as nova and glance, in order to keep tabs on resource existence
- **collector service**: consumes AMQP notifications from the agents and other openstack services, then dispatch these data to the metering store
- **API service**: presents aggregated metering data to consumers (such as billing engines, analytics tools etc.)
- **alarm-evaluator service**: determines when alarms fire due to the associated statistic trend crossing a threshold over a sliding time window
- **alarm-notifier service**: initiates alarm actions, for example calling out to a webhook with a description of the alarm state transition

```python
console_scripts =
    # 提供 RestfulAPI 调用接口
    ceilometer-api = ceilometer.cli:api
    
    #
    ceilometer-agent-central = ceilometer.cli:agent_central
    
    # 运行在计算节点上,查询实例的性能信息,相关信息,通过 AMQP 消息队列发送数据给 collector
    ceilometer-agent-compute = ceilometer.cli:agent_compute
    
    # 通过 AMQP 消息队列货期其他 OpenStack 服务的事件通知
    ceilometer-agent-notification = ceilometer.cli:agent_notification
    
    #
    ceilometer-send-sample = ceilometer.cli:send_sample
    
    #
    ceilometer-dbsync = ceilometer.cli:storage_dbsync
    
    #
    ceilometer-expirer = ceilometer.cli:storage_expirer
    
    #
    ceilometer-collector = ceilometer.cli:collector_service
    
    # 告警触发评估
    ceilometer-alarm-evaluator = ceilometer.cli:alarm_evaluator
    
    # 告警通知器,执行触发后的动作 sms,log,email...
    ceilometer-alarm-notifier = ceilometer.cli:alarm_notifier
```


## 系统架构

**总体架构**

![ceilometer架构](http://ww2.sinaimg.cn/mw690/663a9daagw1et65y1o6usj20o40hnmyr.jpg)

**0.1版本概略图**

![0.1版本概略图](http://ww1.sinaimg.cn/large/663a9daagw1etuo5bi3y6j20kc0giaas.jpg)

**收集数据**

![收集数据](http://ww1.sinaimg.cn/large/663a9daagw1etuo5do7a1j20w80gp75d.jpg)

**pipeline 处理流程**

![pipeline 处理流程](http://ww2.sinaimg.cn/large/663a9daagw1etup94p7lmj20tn0dsmxz.jpg)

**转换数据**

![转换数据](http://ww1.sinaimg.cn/large/663a9daagw1etup95yoaxj20l909u755.jpg)

**发布数据**

![发布数据](http://ww4.sinaimg.cn/large/663a9daagw1etup97ftw4j20hk08pwey.jpg)

- 如果发送到 collector,则通过数据抽象层存储至对应数据库(mongo/mysql/hbase...)

**访问数据**

![访问数据](http://ww4.sinaimg.cn/large/663a9daagw1etup93lj34j20tv0avwfe.jpg)

- 为后续评估数据,触发事件提供待处理数据

>[1] [RDO Ceilometer介绍](https://www.rdoproject.org/CeilometerQuickStart)
>
>[2] [Ceilometer模块学习](http://www.aboutyun.com/forum-196-1.html)
>
>[3] [Ceilometer 高层架构图](http://docs.openstack.org/developer/ceilometer/architecture.html#storing-the-data)
>
>[4] [Ceilometer0.1规划图](http://ceilometer.readthedocs.org/en/latest/architecture.html#high-level-description)