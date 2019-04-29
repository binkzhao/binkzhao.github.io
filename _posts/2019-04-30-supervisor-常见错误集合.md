# Supervisor常见错误集合

### 1、日志权限错误

```
IOError: [Errno 13] Permission denied: '/var/log/supervisor/supervisord.log'
```

原因，/var/log/supervisor/supervisord.log没有写权限，赋予权限即可：

```shell
$ sudo chmod -R 777 /var/log/supervisor/supervisord.log
```

### 2、开启HTTP Server错误

```
Error: Cannot open an HTTP server: socket.error reported errno.EACCES (13)
```

配置文件中 /var/run 文件夹，没有授予启动 supervisord 的相应用户的写权限。/var/run 文件夹实际上是链接到 /run，因此我们修改 /run 的权限

```shell
$ sudo chmod 777 /run
```

一般情况下，我们可以用 root 用户启动 supervisord 进程，然后在其所管理的进程中，再具体指定需要以那个用户启动这些进程。

### 3、日志权限问题

```
'INFO spawnerr: unknown error making dispatchers for 'app_name': EACCES'
```

修改日志文件的权限

```shell
$ sudo chmod 777 /usr/log/supervisor/supervisor.log
$ sudo chmod 777 /usr/log/supervisor/youAppName.log
```

**4,指定运行太多问题,**

```
 Exited too quickly (process log may have details)
```

有可能是当前文件已经运行

```shell
$ kill 调当前的进程,再试试运行
```

5,找不到supervisor==3.81等版本

```
明明你已经安装了supervisor,但是还是报错
pkg_resources.DistributionNotFound: The 'supervisor==3.1.3' distribution was not found and is required by the application
```

有可能是因为你没有python2中没有下载supervisor

```shell
$ sudo easy_install supervisor
# 或者使用如下命令安装
$ pip2 install supervisor
```