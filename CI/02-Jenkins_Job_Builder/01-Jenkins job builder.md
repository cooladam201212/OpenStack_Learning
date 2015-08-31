# 01-Jenkins job builder

<!-- create time: 2015-08-18 13:29:38  -->

<!-- This file is created from $MARBOO_HOME/.media/starts/default.md
本文件由 $MARBOO_HOME/.media/starts/default.md 复制而来 -->

默认的 Jenkins jobs 使用图形界面创建, 然后存储为 xml 格式,阅读性差、不便于版本管理

jenkins job builder 是一个 Python 工具包, 使用 yaml 格式配置, 然后调用 jenkins 接口写入到 jenkins中,更加方便阅读和进行版本管控

## 安装

```bash
git clone https://github.com/openstack-infra/jenkins-job-builder.git
cd jenkins-job-builder
sudo python setup.py install
```

## 配置

- 配置 jenkins 连接

```bash
vim /etc/jenkins_jobs/jenkins_jobs.ini
[jenkins]
user=USERNAME
password=PASSWORD
url=JENKINS_URL
```

another example config

```bash
[job_builder]
ignore_cache=True
keep_descriptions=False
include_path=.:scripts:~/git/
recursive=False
allow_duplicates=False

[jenkins]
user=admin
password=xxxxx
url=https://jenkins.plcloud.com

[hipchat]
authtoken=dummy
```

## 编写任务

sample

```bash
- job:
    name: eDeploy-UnitTests-YAML
    description: 'Do not edit this job through the web!'
    project-type: freestyle
    block-downstream: false
    scm:
    - git:
        skip-tag: false
        url: git@github.com:enovance/edeploy.git
    triggers:
      - pollscm: '@hourly'
    builders:
      - shell: |
          git clean -dxf
          sloccount --duplicates --wide --details . | fgrep -v .svn > sloccount.sc || :
          find . -name test\*.py|xargs nosetests --with-xunit --verbose || :
          find . -name \*.py|egrep -v '^./tests/'|xargs pyflakes  > pyflakes.log || :
          rm -f pylint.log
          for f in `find . -name \*.py|egrep -v '^./tests/'`; do
          pylint --output-format=parseable --reports=y $f >> pylint.log
          done || :
          python /usr/local/lib/python2.7/dist-packages/clonedigger/clonedigger.py --cpd-output . || :
    publishers:
      - warnings:
          workspace-file-scanners:
            - file-pattern: pyflakes.log
              scanner: PyFlakes
      - junit:
          results: nosetests.xml
      - sloccount:
          pattern: sloccount.sc
      - violations:
          cpd:
             pattern: output.xml
          pylint:
             pattern: pylint.log
      - email:
          recipients: devops@mycompany.com
```

## 测试

正式写入 jenkins 之前最好使用如下命令生成 xml 进行检查后再写入

```bash
jenkins-jobs test my-job.yaml -o .
```

## 写入 Jenkins

写入 jenkins, 可以不指定配置文件, 则使用默认路径配置文件

```bash
jenkins-jobs --conf /etc/jenkins_jobs/jenkins_jobs.ini update my-job.yaml
```

## 删除任务

```bash
jenkins-jobs --conf /etc/jenkins_jobs/jenkins_jobs.ini delete my-job
```

> [jenkins job builder官网](http://docs.openstack.org/infra/jenkins-job-builder/)
> 
> [官方范例](http://git.openstack.org/cgit/openstack-infra/jenkins-job-builder/tree/tests)
> 
> [PyYAML官网](http://pyyaml.org/wiki/PyYAMLDocumentation)
> 
> [enovance样例](http://techs.enovance.com/6006/manage-jenkins-jobs-with-yaml)