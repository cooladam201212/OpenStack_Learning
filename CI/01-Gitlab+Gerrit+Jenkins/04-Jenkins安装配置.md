# 04-Jenkins安装配置

<!-- create time: 2015-08-18 09:52:11  -->

<!-- This file is created from $MARBOO_HOME/.media/starts/default.md
本文件由 $MARBOO_HOME/.media/starts/default.md 复制而来 -->

## 下载包

```bash
wget http://pkg.jenkins-ci.org/debian/binary/jenkins_1.544_all.deb
```

## 安装

安装依赖

```bash
apt-get install daemon
```

安装 Jenkins

```bash
dpkg -i jenkins_1.544_all.deb
```

jenkins 默认监听了 8080 端口，修改为 8083

```bash
vim /etc/default/jenkins
HTTP_PORT=8083
```

重新 jenkins 服务

```bash
/etc/init.d/jenkins restart
```

## Nginx

配置 Nginx 端口转发，在文件末尾加入下面配置

```bash
# vim /etc/nginx/sites-available/gitlab
server {
  listen *:80;
  server_name jenkins.ci-example.com;

  location / {
    proxy_pass  http://jenkins.ci-example.com:8083;
  }
}
```

重启 Nginx，就可以用 jenkins.thstack.com 访问 Jenkins 了

```bash
/etc/init.d/nginx restart
```

## 访问

http://jenkins.ci-example.com/

开启用户注册功能，点击 -> 系统管理 -> Configure Global Security -> 勾上启用安全，就可以看到下图 

保存后，会自动跳转到登录页面，点击右上角注册按钮

输入管理员信息

为了安全，设置 Jenkins 不对普通用户开放登录权限，只有管理员可以设置、构建任务，普通用户可以查看任务状态 点击 系统管理 -> Configure Global Security -> 去掉开放用户注册勾

接下来要安装 Jenkins 的插件，来支持 Gerrit 点击 系统管理 -> 管理插件，会看到下图显示

如果上图种 可更新、可选插件、已安装 三个菜单点开为空白的话，需要获取更新下 Jenkins 的信息，之后就可以看到插件信息了 点击 系统管理 -> 管理插件 -> 高级

安装所需插件,由于国外网络连接很差,需要手动下载插件安装

```bash
su - jenkins # jenkins安装后会创建jenkins用户,家目录在/var/lin/jenkins
cd plugins/
wget http://mirror.xmission.com/jenkins/plugins/gerrit-trigger/2.10.1/gerrit-trigger.hpi
wget http://mirror.xmission.com/jenkins/plugins/scm-api/0.2/scm-api.hpi
wget http://mirror.xmission.com/jenkins/plugins/git/2.4.0/git.hpi
wget http://mirror.xmission.com/jenkins/plugins/git-client/1.9.0/git-client.hpi
service jenkins restart # 重启 jenkins 会自动安装下载的插件
```

在系统中给 Jenkins 用户生成 ssh 密钥，Jenkins 用户在安装包的时候自动创建了，家目录在 /var/lib/jenkins

```bash
su - jenkins
ssh-keygen -C git@plcloud.com
cat ~/.ssh/id_rsa.pub         # 把公钥内容复制一下，后面需要添加到 Gerrit 中
```

用 htpasswd 创建 jenkins 用户访问控制密码

```bash
htpasswd /etc/gerrit/etc/htpasswd.conf jenkins
```

继续换个浏览器或者清除浏览器记录，用 jenkins 用户访问 Gerrit

http://jenkins.ci-example.com

设置 jenkins 用户邮箱，顺便去邮箱里确认

添加刚才对 git@plcloud.com 邮箱生成的 ssh 密钥
