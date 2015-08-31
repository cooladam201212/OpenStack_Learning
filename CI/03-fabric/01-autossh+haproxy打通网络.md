# 01-autossh+haproxy打通网络

<!-- create time: 2015-08-24 23:31:30  -->

<!-- This file is created from $MARBOO_HOME/.media/starts/default.md
本文件由 $MARBOO_HOME/.media/starts/default.md 复制而来 -->

## ssh 反向隧道

原理图

![](http://ww3.sinaimg.cn/mw690/663a9daagw1eveljexnluj20qo0k0abr.jpg)

实现

内网主机 A 执行如下命令

```bash
ssh -NCfR remout_port:localhost:local_port remount_user@remout_host
```

命令解释:

- -N 不执行远程命令
- -C 压缩数据
- -f 在认证完成后自动放到后台执行
- -R 创建一个反向代理
- 任何人访问remote_host:remote_port相当于访问localhost:localport

假设 local_port 为 提供 ssh 服务的 22 端口,则执行上述命令后,连接 `remote_host:remote_port` 即可连接上内网主机 ssh

## ssh 免密码

生成 key 执行`ssh-keygen`, 将生成的 `id_rsa.pub` 复制到远程主机, 并追加到 `~/.ssh/authorized_keys` 中

```bash
cat id_rsa.pub >> ~/.ssh/authorized_keys
# 或者 ssh-copy-id user1@123.123.123.123
```

## autossh

作用为监控 ssh 执行状态, 断线自动重连

用法

```bash
autossh -M 5678 -NCR 1234:localhost:2223 user1@123.123.123.123 -p2288
```

要点解释

- -M 5678为监控端口,负责通过该端口监控运行状态
- 去掉了`-f`参数,因为 autossh 本身会在后台执行
- autossh还有很多参数,用来设置重连间隔等等,更复杂的用法可另行查阅或参见底部参考资料

P.S.

```bash
1.家里是ADSL的话，用DDNS，解决ip问题

2.外网有路由的可设下端口映射

3.虽然有密钥和密码保护，但还请小心使用
```

## 设置为 upstart 服务

为防止重启后掉线,加入开机启动服务,此处使用 `upstart`,`SysV`、`Systemd`另行查阅

```bash
# 讲以下内容写入 /etc/init/autossh_1.conf
description "autossh" 
author "Franko <zhonggang.yang@powerleader.com.cn>" 

start on runlevel [2345] 
stop on runlevel [06] 

respawn 
respawn limit 2 5 

exec autossh -M 9030 -CNR 9080:localhost:8080 root@114.119.4.36 -p 2288
```

## haproxy 反向代理,转发至内网机器

如果需要打通内网主机与外网的局域网主机连接,则还需要在远端进行反向代理,可使用`Nginx`或`HAProxy`等工具实现,此处使用`HAProxy`

HAProxy 参考配置文件

```bash
#---------------------------------------------------------------------
# Example configuration for a possible web application.  See the
# full configuration options online.
#
#   http://haproxy.1wt.eu/download/1.4/doc/configuration.txt
#
#---------------------------------------------------------------------

#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    # to have these messages end up in /var/log/haproxy.log you will
    # need to:
    #
    # 1) configure syslog to accept network log events.  This is done
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in
    #    /etc/sysconfig/syslog
    #
    # 2) configure local2 events to go to the /var/log/haproxy.log
    #   file. A line like the following can be added to
    #   /etc/sysconfig/syslog
    #
    #    local2.*                       /var/log/haproxy.log
    #
    log         127.0.0.1 local2

    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

#---------------------------------------------------------------------
# main frontend which proxys to the backends
#---------------------------------------------------------------------
frontend  main *:5000
    acl url_static       path_beg       -i /static /images /javascript /stylesheets
    acl url_static       path_end       -i .jpg .gif .png .css .js

    use_backend static          if url_static
    default_backend             app

#---------------------------------------------------------------------
# static backend for serving up images, stylesheets and such
#---------------------------------------------------------------------
backend static
    balance     roundrobin
    server      static 127.0.0.1:4331 check

#---------------------------------------------------------------------
# round robin balancing between the various backends
#---------------------------------------------------------------------
backend app
    balance     roundrobin
    server  app1 127.0.0.1:5001 check
    server  app2 127.0.0.1:5002 check
    server  app3 127.0.0.1:5003 check
    server  app4 127.0.0.1:5004 check

listen git-proxy
    bind 172.20.0.12:9080
    mode http
    balance roundrobin
    server git-1 127.0.0.1:9080
```

配置完成后,远端主机同一网段访问 `172.20.0.12:9080` 即相当于访问 local_host:local_port

注意配置文件最后几行的 `mode http` 表示访问的模式为 HTTP ,此处的本地端口为 `gitlab` 服务,访问即可实现 `git pull http://...` 操作


> [1] [ssh反向连接及autossh](http://www.cnblogs.com/eshizhan/archive/2012/07/16/2592902.html)
> 
> [2] [ssh反向代理小实验](http://blog.chinaunix.net/uid-29143273-id-4554257.html)