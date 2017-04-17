title: 记一次 yum 操作抛出异常的问题
author: bliver
tags:
  - yum
  - 隐藏权限
categories:
  - 技术
date: 2016-04-29 13:38:02
---
遇到一个奇怪的问题，在用普通用户执行`yum`相关操作时没有问题，但是切换到`root`之后就抛出异常。找到问题的原因之后，才发现其实是用户的隐藏权限问题。

<!-- more -->

异常内容如下：

```
Loaded plugins: security
Traceback (most recent call last):
  File "/usr/bin/yum", line 29, in <module>
    yummain.user_main(sys.argv[1:], exit_code=True)
  File "/usr/share/yum-cli/yummain.py", line 285, in user_main
    errcode = main(args)
  File "/usr/share/yum-cli/yummain.py", line 114, in main
    base.doLock()
  File "/usr/lib/python2.6/site-packages/yum/__init__.py", line 1817, in doLock
    while not self._lock(lockfile, mypid, 0644):
  File "/usr/lib/python2.6/site-packages/yum/__init__.py", line 1883, in _lock
    errmsg = _('Could not create lock at %s: %s ') % (filename, str(msg))
UnicodeDecodeError: 'ascii' codec can't decode byte 0xe6 in position 11: ordinal not in range(128)  
```

上网查了一下资料，原因是Python的`str`默认是`ascii`编码，和`unicode`编码冲突。还有一个主要的原因是我的`shell`环境是中文的，执行`yum`命令时打印的错误信息中也包含中文。所以导致`python`在输出错误信息报错。解决办法是修改`/usr/lib/python2.6/site-packages/yum/__init__.py`，加入如下代码：

```
import sys
reload(sys)
sys.setdefaultencoding('utf8')
```

然后再去执行`yum`操作时，不再抛出异常，而是打印了真正的错误信息。

```
Loaded plugins: fastestmirror, security
Could not create lock at /var/run/yum.pid: [Errno 13] 权限不够: '/var/run/yum.pid'
Can't create lock file; exiting
```

至此才找到真正的原因是`root`用户对`/var/run`目录没有写入权限。这就涉及到`Linux`中的`lsattr`和`chattr`两个命令，即对目录或文件的隐藏属性的操作。执行`lsattr`命令:

```
[root@test ~]# lsattr -d /var/run
----i--------e- /var/run
```

发现其中多了一个隐藏属性`i`，而它的作用就是让一个档案**不能被删除、改名，也无法写入或新增数据**。这时需要通过`chattr`命令去掉这个属性，具体命令如下：

```
[root@test ~]# chattr -i /var/run
```

这时再去执行`lsattr -d /var/run`时发现隐藏属性`i`已经没有了。执行`yum`相关的命令也正常了。