# windows下redis设置后台启动

1、在redis 安装路径中，打开cmd窗口；

2、将redis 绑定为 Windows 服务，并设置后台启动

```sh
redis-server --service-install redis.windows.conf --loglevel verbose
```

3、启动服务

```sh
redis-server --service-start
```

4、停止服务

```sh
redis-server --service-stop
```

5、查看任务是否启动

```sh
tasklist | find "redis"
```