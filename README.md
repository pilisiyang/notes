# 如何清理Docker占用的磁盘空间?

> 用了Docker，好处挺多的，但是有一个不大不小的问题，它会一不小心占用太多磁盘，这就意味着我们必须及时清理。

全Docker化架构，所有服务，包括数据库都运行在Docker里面。这样做当然不是为了炫技，看得清楚的好处还是不少的：
- 所有服务器的配置都非常简单，只安装了Docker，这样新增服务器的时候要简单很多。
- 可以非常方便地在服务器之间移动各种服务，下载Docker镜像就可以运行，不需要手动配置运行环境。
- 开发/测试环境与生产环境严格一致，不用担心由于环境问题导致部署失败。

Docker一直非常稳定，没有出什么问题。但是，它有一个不大不小的问题，会比较消耗磁盘空间。


如果Docker一不小心把磁盘空间全占满了，你的服务也就算玩完了，因此所有Docker用户都需要对此保持警惕。当然，大家也不要紧张，这个问题还是挺好解决的。

1. Docker System命令

在《谁用光了磁盘？Docker System命令详解》中，详细介绍了Docker System命令，它可以用于管理磁盘空间。

docker system df命令，类似于Linux上的df命令，用于查看Docker的磁盘使用情况：

```shell
$ docker system df
TYPE                TOTAL               ACTIVE              SIZE                RECLAIMABLE
Images              147                 36                  7.204GB             3.887GB (53%)
Containers          37                  10                  104.8MB             102.6MB (97%)
Local Volumes       3                   3                   1.421GB             0B (0%)
Build Cache                                                 0B                  0B
```

可知，Docker镜像占用了7.2GB磁盘，Docker容器占用了104.8MB磁盘，Docker数据卷占用了1.4GB磁盘。

docker system prune命令可以用于清理磁盘，删除关闭的容器、无用的数据卷和网络，以及dangling镜像（即无tag的镜像）。docker system prune -a命令清理得更加彻底，可以将没有容器使用Docker镜像都删掉。注意，这两个命令会把你暂时关闭的容器，以及暂时没有用到的Docker镜像都删掉了……所以使用之前一定要想清楚吶。

执行docker system prune -a命令之后，Docker占用的磁盘空间减少了很多：
```shell
$ docker system df
TYPE                TOTAL               ACTIVE              SIZE                RECLAIMABLE
Images              10                  10                  2.271GB             630.7MB (27%)
Containers          10                  10                  2.211MB             0B (0%)
Local Volumes       3                   3                   1.421GB             0B (0%)
Build Cache                                                 0B                  0B
```

2. 手动清理Docker镜像/容器/数据卷

对于旧版的Docker（版本1.13之前），是没有Docker System命令的，因此需要进行手动清理。这里给出几个常用的命令：

删除所有关闭的容器：
```shell
docker ps -a | grep Exit | cut -d ' ' -f 1 | xargs docker rm
```

删除所有dangling镜像（即无tag的镜像）：
```shell
docker rmi $(docker images | grep "<none>" | awk '{print $3}')
```

删除所有dangling数据卷（即无用的Volume）：
```shell
docker volume rm $(docker volume ls -qf dangling=true)
```

3. 限制容器的日志大小
有一次，当我使用1与2提到的方法清理磁盘之后，发现并没有什么作用，于是，我进行了一系列分析。

在Ubuntu上，Docker的所有相关文件，包括镜像、容器等都保存在/var/lib/docker/目录中：
```shell
du -hs /var/lib/docker/
97G /var/lib/docker/
```

Docker竟然使用了将近100GB磁盘，这也是够了。使用du命令继续查看，可以定位到真正占用这么多磁盘的目录：
```shell
92G  /var/lib/docker/containers/a376aa694b22ee497f6fc9f7d15d943de91c853284f8f105ff5ad6c7ddae7a53
```

由docker ps可知，Nginx容器的ID恰好为a376aa694b22，与上面的目录/var/lib/docker/containers/a376aa694b22的前缀一致：
```shell
$ docker ps
CONTAINER ID        IMAGE                                       COMMAND                  CREATED             STATUS              PORTS               NAMES
a376aa694b22        192.168.59.224:5000/nginx:1.12.1            "nginx -g 'daemon off"   9 weeks ago         Up 10 minutes                           nginx
```

因此，Nginx容器竟然占用了92GB的磁盘。进一步分析可知，真正占用磁盘空间的是Nginx的日志文件。

使用truncate命令，可以将Nginx容器的日志文件“清零”：
```shell
truncate -s 0 /var/lib/docker/containers/a376aa694b22ee497f6fc9f7d15d943de91c853284f8f105ff5ad6c7ddae7a53/*-json.log
```

当然，这个命令只是临时有作用，日志文件迟早又会涨回来。要从根本上解决问题，需要限制Nginx容器的日志文件大小。这个可以通过配置日志的max-size来实现，下面是Nginx容器的docker-compose配置文件：
```yaml
nginx:
image: nginx:1.12.1
restart: always
logging:
driver: "json-file"
options:
  max-size: "5g"
```

重启Nginx容器之后，其日志文件的大小就被限制在5GB，再也不用担心了~

4. 重启Docker
还有一次，当我清理了镜像、容器以及数据卷之后，发现磁盘空间并没有减少。根据Docker disk usage提到过的建议，我重启了Docker，发现磁盘使用率从83%降到了19%。根据高手指点，这应该是与内核3.13相关的Bug，导致Docker无法清理一些无用目录：

> it's quite likely that for some reason when those container shutdown, docker couldn't remove the directory because the shm device was busy. This tends to happen often on 3.13 kernel. You may want to update it to the 4.4 version supported on trusty 14.04.5 LTS.


> The reason it disappeared after a restart, is that daemon probably tried and succeeded to clean up left over data from stopped containers.

我查看了一下内核版本，发现真的是3.13：

```shell
uname -r
3.13.0-86-generic
```

如果你的内核版本也是3.13，而且清理磁盘没能成功，不妨重启一下Docker。当然，这个晚上操作比较靠谱。

K8S nodeport 模式无法联通原因:

docker 在 1.13 版本之后，将系统iptables 中 FORWARD 链的默认策略设置为 DROP

需要在每台node上修改 iptables -P FORWARD ACCEPT

sudo find /var/lib/docker/containers/ -name *-json.log | xargs sudo rm -rf
