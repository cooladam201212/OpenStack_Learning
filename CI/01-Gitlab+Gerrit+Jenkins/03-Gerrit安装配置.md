# 03-gerrit安装配置

<!-- create time: 2015-08-17 16:09:41  -->

<!-- This file is created from $MARBOO_HOME/.media/starts/default.md
本文件由 $MARBOO_HOME/.media/starts/default.md 复制而来 -->

## 下载 Gerrit 安装包

```bash
wget http://gerrit-releases.storage.googleapis.com/gerrit-2.8.war
```

## 安装依赖包

Gerrit 的安装包是 java 格式,需要安装 jre

```bash
apt-get install default-jre daemon
```

## 创建 Gerrit 数据库

```bash
sudo mysql -uroot -p
> create database gerritdb;
> grant all on gerritdb.* to 'gerrituser'@'localhost' identified by 'gerritpass';
```

## 开始安装

把 gerrit 安装再 /etc/gerrit/ 下

```bash
# java -jar gerrit-2.8.war init -d /etc/gerrit/

*** Gerrit Code Review 2.8
***

Create '/etc/gerrit'           [Y/n]? y

*** Git Repositories
***

Location of Git repositories   [git]:

*** SQL Database
***

Database server type           [h2]: mysql

Gerrit Code Review is not shipped with MySQL Connector/J 5.1.21
**  This library is required for your configuration. **
Download and install it now [Y/n]? y
Downloading http://repo2.maven.org/maven2/mysql/mysql-connector-java/5.1.21/mysql-connector-java-5.1.21.jar ... OK
Checksum mysql-connector-java-5.1.21.jar OK
Server hostname                [localhost]:
Server port                    [(mysql default)]:
Database name                  [reviewdb]: gerritdb
Database username              [root]: gerrituser
gerrituser's password          :
              confirm password :

*** User Authentication
***

Authentication method          [OPENID/?]: http
Get username from custom HTTP header [y/N]? n
SSO logout URL                 :

*** Email Delivery
***

SMTP server hostname           [localhost]: smtp.exmail.qq.com
SMTP server port               [(default)]: 25
SMTP encryption                [NONE/?]: 
SMTP username                  [root]: git@plcloud.com
review@ci-example.com's password  :
              confirm password :

*** Container Process
***

Run as                         [root]:
Java runtime                   [/usr/lib/jvm/java-6-openjdk-amd64/jre]:
Copy gerrit-2.8.war to /etc/gerrit/bin/gerrit.war [Y/n]? y
Copying gerrit-2.8.war to /etc/gerrit/bin/gerrit.war

*** SSH Daemon
***

Listen on address              [*]:
Listen on port                 [29418]:

Gerrit Code Review is not shipped with Bouncy Castle Crypto v144
  If available, Gerrit can take advantage of features
  in the library, but will also function without it.
Download and install it now [Y/n]? y
Downloading http://www.bouncycastle.org/download/bcprov-jdk16-144.jar ... OK
Checksum bcprov-jdk16-144.jar OK
Generating SSH host key ... rsa... dsa... done

*** HTTP Daemon
***

Behind reverse proxy           [y/N]? y
Proxy uses SSL (https://)      [y/N]? n
Subdirectory on proxy server   [/]:
Listen on address              [*]:
Listen on port                 [8081]: 8082
Canonical URL                  [http://www.ci-example.com/]: http://review.ci-example.com/

*** Plugins
***

Install plugin reviewnotes version v2.8 [y/N]? y
Install plugin download-commands version v2.8 [y/N]? y
Install plugin replication version v2.8 [y/N]? y
Install plugin commit-message-length-validator version v2.8 [y/N]? y

Initialized /etc/gerrit
Executing /etc/gerrit/bin/gerrit.sh start
Starting Gerrit Code Review: OK
Waiting for server on review.ci-example.com:80 ... OK
Opening http://review.ci-example.com/#/admin/projects/ ...FAILED
Open Gerrit with a JavaScript capable browser:

http://review.ci-example.com/#/admin/projects/
```

在安装完后 Gerrit 默认会打开浏览器，由于我的系统是 Server 版没有桌面和浏览器，所以才会出现上面倒数第三行的 …FAILED 同时从上面的信息看出我使用了 http 方式验证、启用代理服务器，指定了 Gerrit 端口为 8082，URL 为 review.ci-example.com。需要在 nginx 里设置下端口转发。

gerrit邮件配置sendmail

```bash
vim /etc/gerrit/etc/gerrit.config
[sendmail] 下增加 from = git@plcloud.com 否则邮件会被拦截
```

Gerrit 启动脚本

```bash
cp /etc/gerrit/bin/gerrit.sh /etc/init.d/gerrit
vim /etc/init.d/gerrit
GERRIT_SITE=/etc/gerrit/       # 在代码 47 行增加
sudo update-rc.d gerrit defaults 21
service gerrit restart
```

## Nginx

修改 Nginx 配置文件，给 Gerrit 做端口转发和访问控制，把下面内容写入到文件最后

```bash
# vim /etc/nginx/sites-available/gitlab
server {
  listen *:80;
  server_name review.ci-example.com;
  allow   all;
  deny    all;
  auth_basic "Review System Login";
  auth_basic_user_file /etc/gerrit/etc/htpasswd.conf;

  location / { 
    proxy_pass  http://review.ci-example.com:8082;
  }   
}
```

创建 htpasswd.conf 文件，并添加 admin 用户、密码到文件中

```bash
# touch /etc/gerrit/etc/htpasswd.conf
# htpasswd /etc/gerrit/etc/htpasswd.conf admin
  New password: 
  Re-type new password: 
  Adding password for user admin
```

默认第一个登录 Gerrit 的用户是 Admin。

## 访问

在浏览器 url 输入：http://review.ci-example.com/，记得添加 hosts 解析。由于做了访问控制，会出现如下验证框，输入刚才创建的 admin 和密码

![](/Users/frank/Documents/marboo/media/CI/gerrit/00-gerrit-admin-login.png)

注册 admin 的邮箱，去邮箱验证，然后添加 git@plcloud.com 密钥 

![](/Users/frank/Documents/marboo/media/CI/gerrit/01-gerrit-bind-email.png)
![](/Users/frank/Documents/marboo/media/CI/gerrit/02-gerrit-add-ssh-key.png)

使用 htpasswd 创建 frank 用户和密码

```bash
htpasswd /etc/gerrit/etc/htpasswd.conf frank
```

换个浏览器或清除下浏览器保存的 admin 的密码 访问 http://review.ci-example.com/ 输入 frank 和密码：

注册邮箱和添加 rainyday_hubei@163.com 的密钥 

## 最后

如果想 Gitlab 上创建的项目使用 Gerrit 的 Code Review 功能，两个系统的用户必须统一，也就是说不管哪个用户使用 Gerrit，前提是这个用户在 Gitlab 和 Gerrit 上都已注册，邮箱一致、sshkey 一致。当然 Nginx 访问控制用户的密码那就随意了。至于 Gitlab 上创建的项目如何同步到 Gerrit 上、Gitlab 如何使用 Gerrit 的 Code Review 功能等等都在 Jenkins 安装完之后会一起整合在一起。请关注后面的文章。