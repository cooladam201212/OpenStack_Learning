# 02-fabric远程部署

<!-- create time: 2015-08-18 13:30:10  -->

<!-- This file is created from $MARBOO_HOME/.media/starts/default.md
本文件由 $MARBOO_HOME/.media/starts/default.md 复制而来 -->


样例代码

```python
from fabric.api import *
import time
import re
import sys
import os
import json

env.passwords = {
    'root@vip.plcloud.com:22': 'xxxxxxxxxx',
    'root@192.168.215.101:22': '111111',
    'root@58.67.194.90:2288': 'xxxxxxxxxx',
    'root@10.0.0.14:22': 'xxxxxxxxxx',
    'root@10.0.0.15:22': 'xxxxxxxxxx',
    'root@10.0.0.18:22': 'xxxxxxxxxx',
    'root@10.0.0.23:22': 'xxxxxxxxxx',
}

gitpwd = 'gitdeploy'

@hosts('vip.plcloud.com')
def plcloud_front_master():
    with cd('/var/plcloud/plcloud-front'):
        run('git pull http://git:%s@127.0.0.1:9080/powerleader/plcloud-front.git' % gitpwd)
        run('python manage.py collectstatic -c --noinput')
        run('touch /etc/uwsgi/vassals/front_old.xml')


@hosts('vip.plcloud.com')
def plcloud_front_v2_0():
    with cd('/var/plcloud/v2_0/plcloud-front'):
        run('git pull http://git:%s@127.0.0.1:9080/powerleader/plcloud-front.git v2.0' % gitpwd)
        run('python manage.py collectstatic -c --noinput')
        run('touch /etc/uwsgi/vassals/front_uwsgi_2.xml')


@hosts('vip.plcloud.com')
def plcloud_admin_master():
    with cd('/var/plcloud/plcloud-admin'):
        run('git pull http://git:%s@127.0.0.1:9080/powerleader/plcloud-admin.git master' % gitpwd)
        run('python manage.py collectstatic -c --noinput')
        run('touch /etc/uwsgi/vassals/admin_uwsgi.xml')


@hosts('vip.plcloud.com')
def plcloud_vip_master():
    with cd('/var/plcloud/plcloud-vip'):
        run('git pull http://git:%s@127.0.0.1:9080/powerleader/plcloud-vip.git master' % gitpwd)
        run('python manage.py collectstatic -c --noinput')
        run('touch /etc/uwsgi/vassals/vip_uwsgi.xml')


@hosts('192.168.215.101')
def horizon_master():
    with cd('/opt/horizon'):
        run('git pull origin master')
        run('rm -rf static')
        run('python manage.py compress -f')
        run('python manage.py collectstatic -c --noinput')
        run('touch /etc/uwsgi/vassals/openstack.ini')

def trigger_ci_master():
    with lcd('/var/plcloud/trigger-ci'):
        local('git checkout -f master')
        local('git clean -f -d')
        local('git pull origin master')
        local('touch /etc/uwsgi/vassals/trigger-ci.ini')

@hosts('192.168.215.101')
def plcloud_operation_master():
    with cd('/var/plcloud/plcloud-operation'):
        run('git pull origin master')
        #run('rm -rf static')
        #run('python manage.py collectstatic -c --noinput')
        run('touch /etc/uwsgi/vassals/operation.ini')

def normalize(value):
    return re.sub('[-\s\.]+', '_', value.lower())

def test():
    local('uname -a')

def main(*argc):
    #repo = os.environ.get('repo')
    #branch = os.environ.get('branch')
    if len(argc) == 0:
        gitlab_ci()
        return
    repo = normalize(argc[0])
    branch = normalize(os.environ.get('gitlabBranch'))
    print 'event: (%s, %s)' % (repo, branch)
    func_name = '%s_%s' % (repo, branch)
    task = globals().get(func_name, test)
    print 'task: %s' % task
    out = execute(task)
    print 'out: %s' % out

def gitlab_ci():
    context = json.loads(os.environ.get('CONTEXT'))
    repo = normalize(context['repository']['name'])
    branch = normalize(context['ref'].split('/')[-1])
    print 'event: (%s, %s)' % (repo, branch)
    func_name = '%s_%s' % (repo, branch)
    task = globals().get(func_name, test)
    print 'task: %s' % task
    out = execute(task)
    print 'out: %s' % out

def run_git_pull(url, branch='master', gitpwd=None, gituser='git', is_local=False):
    schema, path = url.split('//')
    fastprint('git pull %s//%s:%s@%s %s' % (schema, gituser, '***', path, branch), show_prefix=True, end='\r\n')
    with hide('running'):
        if is_local:
            local('git pull %s//%s:%s@%s %s' % (schema, gituser, gitpwd, path, branch))
        else:
            run('git pull %s//%s:%s@%s %s' % (schema, gituser, gitpwd, path, branch))

@hosts('10.0.0.14', '10.0.0.15')
@with_settings(gateway='root@58.67.194.90:2288')
def horizon():
    with cd('/var/plcloud/horizon'):
        run_git_pull('http://10.0.0.10:9080/powerleader/horizon.git', 'master', gitpwd)
        run('rm -rf static')
        run('python manage.py compress -f')
        run('python manage.py collectstatic -c --noinput')
        run('touch /etc/uwsgi/vassals/horizon.ini')

@hosts('10.0.0.18', '10.0.0.23')
@with_settings(gateway='root@58.67.194.90:2288')
def ceilometer():
    with cd('/var/plcloud/ceilometer'):
        run_git_pull('http://10.0.0.10:9080/icehouse/ceilometer.git', 'master', gitpwd)
        run('killall -e ceilometer-agent-notification')
        run('/etc/init.d/openstack-ceilometer-notification start')
        time.sleep(2)
        run('tail -n 10 /var/log/ceilometer/agent-notification.log')

@hosts('10.0.0.14', '10.0.0.15')
@with_settings(gateway='root@58.67.194.90:2288')
def keystone():
    with cd('/var/plcloud/keystone'):
        run_git_pull('http://10.0.0.10:9080/icehouse/keystone.git', 'master', gitpwd)
        run('service openstack-keystone restart')
        run('tail -n 10 /var/log/keystone/keystone.log')

@hosts('10.0.0.14', '10.0.0.15', '10.0.0.18', '10.0.0.23')
@with_settings(gateway='root@58.67.194.90:2288')
def keystoneclient():
    with cd('/var/plcloud/python-keystoneclient'):
        run_git_pull('http://10.0.0.10:9080/icehouse/python-keystoneclient.git', 'master', gitpwd)

if __name__ == '__main__':
    main(*sys.argv[1:])

```

更多用法参考以下文档

> [1] [fabric官方文档](http://docs.fabfile.org/en/latest/tutorial.html)
> 
> [2] [fabric入门参考](http://wklken.me/posts/2013/03/25/python-tool-fabric.html)