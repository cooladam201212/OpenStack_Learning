# 02-Gitlab的安装配置

<!-- create time: 2015-08-14 09:48:10  -->

<!-- This file is created from $MARBOO_HOME/.media/starts/default.md
本文件由 $MARBOO_HOME/.media/starts/default.md 复制而来 -->
[toc]
## 设置源

设置国内163的源

```bash
# vim /etc/apt/sources.list
deb http://mirrors.163.com/ubuntu/ precise main universe restricted multiverse
deb http://mirrors.163.com/ubuntu/ precise-security universe main multiverse restricted
deb http://mirrors.163.com/ubuntu/ precise-updates universe main multiverse restricted
deb http://mirrors.163.com/ubuntu/ precise-proposed universe main multiverse restricted
deb http://mirrors.163.com/ubuntu/ precise-backports universe main multiverse restricted

deb-src http://mirrors.163.com/ubuntu/ precise main universe restricted multiverse
deb-src http://mirrors.163.com/ubuntu/ precise-security universe main multiverse restricted
deb-src http://mirrors.163.com/ubuntu/ precise-proposed universe main multiverse restricted
deb-src http://mirrors.163.com/ubuntu/ precise-backports universe main multiverse restricted
deb-src http://mirrors.163.com/ubuntu/ precise-updates universe main multiverse restricted
# apt-get update
```

## 安装依赖包

Gitlab 依赖包、库

```bash
sudo apt-get install -y build-essential zlib1g-dev libyaml-dev libssl-dev libgdbm-dev libreadline-dev \
                        libncurses5-dev libffi-dev curl openssh-server redis-server checkinstall \
                        libxml2-dev libxslt-dev libcurl4-openssl-dev libicu-dev logrotate
```

安装 markdown 文档风格依赖包

```bash
sudo apt-get install -y python-docutils
```

安装 git，Gerrit 依赖 gitweb，同时 GitLab 依赖 git 版本 >= 1.7.10，Ubuntu apt-get 默认安装的是 1.7.9.5

**git-review用于发送代码审查请求**

```bash
sudo apt-get install -y git gitweb git-review
sudo apt-get install -y libcurl4-openssl-dev libexpat1-dev gettext libz-dev libssl-dev build-essential
cd /tmp
curl --progress https://git-core.googlecode.com/files/git-1.8.4.1.tar.gz | tar xz
cd git-1.8.4.1/
cd git-1.8.4.1/
make prefix=/usr/local all
sudo make prefix=/usr/local install
```

Gitlab 需要收发邮件，安装邮件服务器

```bash
sudo apt-get install -y postfix
```

如果安装了 ruby1.8，卸载掉，Gitlab 依赖 2.0 以上

```bash
sudo apt-get remove ruby1.8
```

下载编译 ruby2.0

```bash
mkdir /tmp/ruby && cd /tmp/ruby
wget ftp://ftp.ruby-lang.org/pub/ruby/2.0/ruby-2.0.0-p353.tar.gz | tar xz
cd ruby-2.0.0-p353
./configure --disable-install-rdoc
make
sudo make install
```

修改 gem 源指向 taobao

```bash
gem source -r https://rubygems.org/
gem source -a http://ruby.taobao.org/
```

安装 Bundel 命令

```bash
sudo gem install bundler --no-ri --no-rdoc
```

## 创建用户

给 Gitlab 创建一个 git 用户

```bash
sudo adduser --disabled-login --gecos 'GitLab' git
```

## Gitlab Shell

下载 Gitlab Shell，用来 ssh 访问仓库的管理软件

==**注意需要切换分支,高版本的gitlab-shell API有改动**==

```bash
cd /home/git
sudo -u git -H git clone https://github.com/gitlabhq/gitlab-shell.git
cd gitlab-shell
git checkout -b v1.8.0 v1.8.0
sudo -u git -H cp config.yml.example config.yml
```

修改 gitlab-shell/config.yml

```bash
sudo -u git -H vim /home/git/gitlab-shell/config.yml
gitlab_url: "http://gitlab.pystack.com/"
```

安装 GitLab Shell

```bash
cd /home/git/gitlab-shell
sudo -u git -H ./bin/install
```

## Mysql

安装 Mysql 包

```bash
sudo apt-get install -y mysql-server mysql-client libmysqlclient-dev
```

给 Gitlab 创建 Mysql 数据库并授权用户访问

```bash
sudo mysql -uroot -p
> create database gitlabdb;
> grant all on gitlabdb.* to 'gitlabuser'@'localhost' identified by 'gitlabpass';
```

## Gitlab

下载 GitLab 源代码，并切换到对应的分支上

```bash
cd /home/git
sudo -u git -H git clone https://github.com/gitlabhq/gitlabhq.git gitlab
cd gitlab
sudo -u git -H git checkout 6-4-stable
```

配置 GitLab，修改 gitlab.yml，其中 host： 项和 gitlab-shell 中 gitlab_url 的主机一致

```bash
cd /home/git/gitlab
sudo -u git -H cp config/gitlab.yml.example config/gitlab.yml
sudo -u git -H vim config/gitlab.yml
host: gitlab.pystack.com
email_from: gitlab@pystack.com
support_mail: gitlab@pystack.com
signup_enabled: true             # 开启用户注册
bin_path: /usr/local/bin/git     # git可执行文件路径
```

创建相关目录

```bash
cd /home/git/gitlab
sudo -u git -H mkdir tmp/pids/ tmp/sockets/ public/uploads /home/git/repositories /home/git/gitlab-satellites
```

修改相关目录权限

```bash
sudo chown -R git:git log/ tmp/
sudo chmod -R u+rwX  log/ tmp/ public/uploads
```

修改 unicorn.rb 监听端口为：8081

```bash
cd /home/git/gitlab/
sudo -u git -H cp config/initializers/rack_attack.rb.example config/initializers/rack_attack.rb
sudo -u git -H cp config/unicorn.rb.example config/unicorn.rb
sudo -u git -H vim config/unicorn.rb
listen "gitlab.pystack.com:8081", :tcp_nopush => true
```

配置 GitLab 访问 mysql 数据库设置

```bash
cd /home/git/gitlab/
sudo -u git cp config/database.yml.mysql config/database.yml
sudo -u git -H vim config/database.yml  
*修改 Production 部分:*
production:
  adapter: mysql2
  encoding: utf8
  reconnect: false
  database: gitlabdb
  pool: 10
  username: gitlabuser
  password: "gitlabpass"
  host: localhost
  socket: /var/run/mysqld/mysqld.sock
```

根据所选用的邮件服务厂商

设置 GitLab 使用指定邮箱发送邮件，注意 production.rb 的文件格式，开头空两格

```bash
cd /home/git/gitlab/
sudo -u git -H vim config/environments/production.rb
  #修改 :sendmail 为 :smtp
  config.action_mailer.delivery_method = :smtp
  config.action_mailer.smtp_settings = {
    :address              => "smtp.exmail.qq.com",
    :port                 => 25,
    :domain               => 'plcloud.com',
    :user_name            => 'git@plcloud.com',
    :password             => 'ljm140920gumi',
    :authentication       =>  :plain,
    :enable_starttls_auto => true
  }
end          # 上面内容加入到 end 里面
```

安装 gem, 修改 Gemfile 文件中源指向为 taobao

```bash
cd /home/git/gitlab/
sudo -u git -H vim Gemfile
source "http://ruby.taobao.org/"
```

```bash
cd /home/git/gitlab/
sudo -u git -H bundle install --deployment --without development test postgres aws
```
如果报 modernizer-2.6.2 无法找到的错误,是因为modernizer-2.6.2包已经下架

参考[modernizr2.6.2依赖修复](http://stackoverflow.com/questions/22825497/installing-gitlab-missing-modernizer)进行修复, 修改相应文件后重新执行上述命令

修改 Gemfile `"modernizr", "2.6.2"` 到 `"modernizr-rails", "2.7.1"`

修改 Gemfile.lock `modernizr (2.6.2)` 到 `modernizr-rails (2.7.1)`

修改 Gemfile.lock `modernizr (= 2.6.2)` 到 `modernizr-rails (= 2.7.1)`

初始化数据库并激活高级功能

```bash
cd /home/git/gitlab/
sudo -u git -H bundle exec rake gitlab:setup RAILS_ENV=production
```

输入 yes 来初始化数据库、创建相关表，最后会输出 GitLab Web 管理员用来登录的账号和密码

```bash
Do you want to continue (yes/no)? yes
...
Administrator account created:
login.........admin@local.host
password......5iveL!fe
```

设置 GitLab 启动服务

```bash
cd /home/git/gitlab/
sudo cp lib/support/init.d/gitlab /etc/init.d/gitlab
sudo update-rc.d gitlab defaults 21
```

设置 GitLab 使用 Logrotate 备份 Log

```bash
cd /home/git/gitlab/
sudo cp lib/support/logrotate/gitlab /etc/logrotate.d/gitlab
```

检查GitLab及其环境的配置是否正确：

```bash
sudo -u git -H bundle exec rake gitlab:env:info RAILS_ENV=production

System information
System:         Ubuntu 12.04
Current User:   git
Using RVM:      no
Ruby Version:   2.0.0p353
Gem Version:    2.0.14
Bundler Version:1.10.6
Rake Version:   10.1.0

GitLab information
Version:        6.4.3
Revision:       3173626
Directory:      /home/git/gitlab
DB Adapter:     mysql2
URL:            http://gitlab.pystack.com
HTTP Clone URL: http://gitlab.pystack.com/some-project.git
SSH Clone URL:  git@gitlab.pystack.com:some-project.git
Using LDAP:     no
Using Omniauth: no

GitLab Shell
Version:        1.8.0
Repositories:   /home/git/repositories/
Hooks:          /home/git/gitlab-shell/hooks/
Git:            /usr/local/bin/git
```

启动 GitLab 服务

```bash
/etc/init.d/gitlab restart
Shutting down both Unicorn and Sidekiq.
GitLab is not running.
Starting both the GitLab Unicorn and Sidekiq..
The GitLab Unicorn web server with pid 17771 is running.
The GitLab Sidekiq job dispatcher with pid 17778 is running.
GitLab and all its components are up and running
```

**选做**:如果启动服务报错,需要执行检查命令并按照提示修复报错信息

也可以参考[官网排错指南](https://github.com/gitlabhq/gitlab-public-wiki/wiki/Trouble-Shooting-Guide)

```bash
# 检查一
sudo -u git -H bundle exec rake gitlab:check RAILS_ENV=production

# 检查二
cd /home/git/gitlab-shell
./bin/check
```

最后编译一下

```bash
sudo -u git -H bundle exec rake assets:precompile RAILS_ENV=production
```

## Nginx

安装 Nginx 包

```bash
apt-get install -y nginx
```

配置 Nginx

```bash
cd /home/git/gitlab
sudo cp lib/support/nginx/gitlab /etc/nginx/sites-available/gitlab
sudo ln -s /etc/nginx/sites-available/gitlab /etc/nginx/sites-enabled/gitlab
sudo vim /etc/nginx/sites-available/gitlab
listen *:80 default_server;
server_name gitlab.pystack.com;
proxy_pass http://gitlab.pystack.com:8081;
```

启动 Nginx

```bash
/etc/init.d/apache2 stop
/etc/init.d/nginx restart
```

访问

```bash
用浏览器访问: http://gitlab.pystack.com
用户名：admin@local.host
密码：5iveL!fe
```

## 界面使用

- 使用默认账户 admin@local.host 登陆后重置 admin 密码,然后重新登陆
![](/Users/frank/Documents/marboo/media/CI/gitlab/00-gitlab-login.png)
![](/Users/frank/Documents/marboo/media/CI/gitlab/01-admin-reset-passwd.png)
![](/Users/frank/Documents/marboo/media/CI/gitlab/02-welcome.png)
- 进入欢迎页面点右上角用户个人设置,重置绑定邮箱,并去邮箱验证
![](/Users/frank/Documents/marboo/media/CI/gitlab/03-reset-email.png)
- 创建用户组 devgroup,创建新项目 openstack 
![](/Users/frank/Documents/marboo/media/CI/gitlab/04-new-group.png)
![](/Users/frank/Documents/marboo/media/CI/gitlab/05-create-project.png)
![](/Users/frank/Documents/marboo/media/CI/gitlab/06-new-project.png)
- 添加 ssh key, 生成 ssh key, 填入文本框
![](/Users/frank/Documents/marboo/media/CI/gitlab/07-add-ssh.png)
![](/Users/frank/Documents/marboo/media/CI/gitlab/08-admin-gen-ssh.png)
![](/Users/frank/Documents/marboo/media/CI/gitlab/09-admin-add-ssh.png)
- 创建测试用户 frank ,生成 ssh key,填入文本框
![](/Users/frank/Documents/marboo/media/CI/gitlab/10-signup-new-user-frank.png)
![](/Users/frank/Documents/marboo/media/CI/gitlab/11-adduser-frank.png)
![](/Users/frank/Documents/marboo/media/CI/gitlab/12-frank-gen-ssh.png)
![](/Users/frank/Documents/marboo/media/CI/gitlab/13-frank-add-ssh.png)
- admin 用户重新登陆后添加 frank 至 devgroup ,分配其在 openstack 项目权限为 reporter
![](/Users/frank/Documents/marboo/media/CI/gitlab/14-admin-area.png)
![](/Users/frank/Documents/marboo/media/CI/gitlab/15-admin-add-frank.png)
- 此时用 frank 重新登陆后可以看见 openstack 项目,

  配置`git config user.name 'frank'`,
  
  配置`git config user.email 'rainyday_hubei@163.com`
  
  进行修改、提交push请求,因为是 reporter 身份, 所以只能 clone 项目,不能 push

> [github安装指导](https://github.com/gitlabhq/gitlabhq/blob/master/doc/install/installation.md)
> 
> [gitlab-shell安装文档](https://github.com/gitlabhq/gitlab-shell)