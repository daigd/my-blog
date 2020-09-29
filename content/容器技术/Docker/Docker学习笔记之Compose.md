---
title: "Docker学习笔记之Compose"
date: 2020-09-29
summary: "Docker学习笔记之Compose"
tags: ["Docker", "Docker Compose"]
---

### Docker Compose概述

Compose是一个通过`yaml`文件来定义、运行、管理Docker容器的工具，负责实现对Docker容器集群的快速编排。

### 安装Compose

在Linux环境安装Compose:

```shell
sudo curl -L "https://github.com/docker/compose/releases/download/1.26.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

下载完毕后，分配执行权限：

```shell
sudo chmod +x /usr/local/bin/docker-compose
```

查看版本，确定是否安装成功：

```shell
docker-compose version 
# 可以成功看到版本信息了，表明已安装成功
docker-compose version 1.26.0, build d4451659
docker-py version: 4.2.1
CPython version: 3.7.7
OpenSSL version: OpenSSL 1.1.0l  10 Sep 2019
```

如果想要卸载Composer，直接将对应文件删除即可：

```shell
sudo rm /usr/local/bin/docker-compose
```

### Compose的使用

Compose 里涉及到两个重要概念：`service`、`project`。

> service：一个应用容器，实际上可以运行多个相同镜像的实例；
>
> project：由一组相互关联的应用容器组成的一个完整业务单元。

由此可知，一个项目由多个服务（应用容器）组成，Compose面向项目进行管理。

#### Compose 模板文件

Compose 模板文件是使用`Compose`的核心，默认名称为`docker-compose.yml`,文件格式为`yaml`。一个简单的模板文件如下所示：

```yaml
version: "3" 

services: 
	webapp: 
		image: examples/web 
		ports: 
			- "80:80" 
		volumes:
        	- "/data"

```

`version`：指定模板文件的版本号，当前最新版本为`3.8`,详情可看[官网](https://docs.docker.com/compose/compose-file/)。

`services`：定义应用服务，`webapp`是自定义的服务名称，`image`指定镜像名称，`ports`对容器端口进行映射，`volumes`指定数据卷挂载。

虽然模板文件里参数特别多，但是很多都跟Docker里的参数含义一致，所以理解起来也比较容易，下面以官方的例子来进行讲解。

```yaml
# 指定模板文件版本号
version: "3.8"
services:
 # 自定义的服务名
  redis:
   # 指定服务镜像
    image: redis:alpine
    # 端口映射，这里只指定了容器端口，没有和宿主机端口进行映射
    ports:
      - "6379"
      # 指定服务网络
    networks:
      - frontend
      # 指定服务部署相关配置，仅用于 Swarm 集群部署的时候，如果仅运行 docker-compose up 或 docker-compose run 将被忽略。
    deploy:
      replicas: 2
      update_config:
        parallelism: 2
        delay: 10s
      restart_policy:
        condition: on-failure

  db:
    image: postgres:9.4
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - backend
    deploy:
      placement:
        constraints:
          - "node.role==manager"

  vote:
    image: dockersamples/examplevotingapp_vote:before
    ports:
      - "5000:80"
    networks:
      - frontend
      # 表示该服务依赖于redis服务
    depends_on:
      - redis
    deploy:
      replicas: 2
      update_config:
        parallelism: 2
      restart_policy:
        condition: on-failure

  result:
    image: dockersamples/examplevotingapp_result:before
    ports:
      - "5001:80"
    networks:
      - backend
    depends_on:
      - db
    deploy:
      replicas: 1
      update_config:
        parallelism: 2
        delay: 10s
      restart_policy:
        condition: on-failure

  worker:
    image: dockersamples/examplevotingapp_worker
    networks:
      - frontend
      - backend
    deploy:
      mode: replicated
      replicas: 1
      labels: [APP=VOTING]
      restart_policy:
        condition: on-failure
        delay: 10s
        max_attempts: 3
        window: 120s
      placement:
        constraints:
          - "node.role==manager"

  visualizer:
    image: dockersamples/visualizer:stable
    ports:
      - "8080:8080"
      # 在容器响应中断信号之前，允许等待的时间，即等待1m30s后再把容器停掉
    stop_grace_period: 1m30s
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
    deploy:
      placement:
        constraints:
          - "node.role==manager"

networks:
  frontend:
  backend:

volumes:
  db-data:
```

#### Compose 实战

下面我们来搭建一个`ElasticSearch+Kibana+Logstash`的服务项目来介绍Compose的使用。

创建一个`compose-test`目录并进入该目录：

```shell
mkdir compose-test
cd compose-test/
```

创建Compose 模板文件：

```shell
touch docker-compose.yml
```

编辑该文件：

```shell
vi docker-compose.yml
```

输入以下内容：

```yaml
version: '3'
services:
  es01:
    image: elasticsearch:7.7.1
    container_name: es01
    environment:
      - discovery.type=single-node
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms128m -Xmx512m"
    volumes:
      - es-data01:/usr/share/elasticsearch/data
    ports:
      - 9200:9200
      - 9300:9300
    networks:
      - elastic

  kib01:
    image: kibana:7.7.1
    container_name: kib01
    ports:
      - 5601:5601
    environment:
      ELASTICSEARCH_URL: http://es01:9200
      ELASTICSEARCH_HOSTS: http://es01:9200
    networks:
      - elastic
    depends_on:
      - es01
  
  logstash01:
    image: logstash:7.7.1
    container_name: logstash01   
    networks:
      - elastic
    volumes:
      - logstash01-config:/usr/share/logstash/config
    environment:
      - "xpack.management.enabled=false"
      - "monitoring.enabled=false"
    depends_on:
      - es01
      
volumes:
  es-data01:
    driver: local
  logstash01-config:
    driver: local
    
networks:
  elastic:
    driver: bridge
```

执行```docker-compose -p elk up -d```命令，会看到控制台输出以下内容：

```shell
Creating es01       ... done
Creating logstash01 ... done
Creating kib01      ... done
```

执行``` curl localhost:9200```,控制台输出以下内容则说明```elasticsearch```已启动成功：

```shell
{
  "name" : "2b9932a79edd",
  "cluster_name" : "docker-cluster",
  "cluster_uuid" : "TR59-aFHRDWXuz1VCrcr4Q",
  "version" : {
    "number" : "7.7.1",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "ad56dce891c901a492bb1ee393f12dfff473a423",
    "build_date" : "2020-05-28T16:30:01.040088Z",
    "build_snapshot" : false,
    "lucene_version" : "8.5.1",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

在浏览器输入```ip:5601```,可以看到Kibana也能正常访问。

#### 常用命令说明

`docker-compose`命令的基本使用格式：

```shell
docker-compose [-f<arg>...] [options] [COMMAND] [ARGS...]
```

命令选项

- `-f,--file FILE`指定使用的模板文件，默认为`docker-compose.yml`，可以多次指定

- `p，--project-name NAME` 指定项目名称，默认使用所在目录名称作为项目名称

`up`
启动容器项目，该命令参数比较多，下面说下常用的：

- `-d` 在后台运行服务。
- `--force-recreate` 强制构建容器。
- `--no-crcreate` 如果容器已经存在，则不重新创建，不能与`--force-recreate`同时使用。
- `no-build` 不自动构建缺失的服务镜像。

示例：

```shell
docker-compose -p elk up -d
```

`down`

停止`up`命令所启动的容器，并移除对应网络。

示例：

```shell
docker-compose down
```

提示：

```shell
Removing network compose-test_elastic
WARNING: Network compose-test_elastic not found.
```

看来停止也要指定项目名，命令改成如下：

```shell
docker-compose -p elk down
```

控制台输出以下内容：

```shell
Stopping es01       ... done
Stopping kib01      ... done
Stopping logstash01 ... done
Removing es01       ... done
Removing kib01      ... done
Removing logstash01 ... done
Removing network elk_elastic
```

即`down`命令会先把对应容器服务停掉，随后删除，最后再把对应网络移除。

`ps`

列出项目中所有容器。

示例：

```shell  
docker-compose -p elk ps
```

输出：

```shell
   Name                 Command               State                       Ports                     
----------------------------------------------------------------------------------------------------
es01         /tini -- /usr/local/bin/do ...   Up      0.0.0.0:9200->9200/tcp, 0.0.0.0:9300->9300/tcp
kib01        /usr/local/bin/dumb-init - ...   Up      0.0.0.0:5601->5601/tcp                        
logstash01   /usr/local/bin/docker-entr ...   Up      5044/tcp, 9600/tcp               
```

`images`

列出Compose文件包含的镜像。

示例：

```shell
docker-compose -p elk images
```

输出：

```shell
Container     Repository      Tag      Image Id       Size  
------------------------------------------------------------
es01         elasticsearch   7.7.1   830a894845e3   804.3 MB
kib01        kibana          7.7.1   6de54f813b39   1.201 GB
logstash01   logstash        7.7.1   7f059e3dee67   787.9 MB
```

`logs`

格式为：`docker-compose logs [options] [SERVICE...]`

查看项目容器服务的日志，默认情况下不同容器日志颜色不一样。

示例：

```shell
docker-compose -p elk logs --tail 2
```

输出：

```shell
es01          | "at io.netty.util.internal......",
es01          | "at java.lang......"] }
kib01         | {"type":"log"......}
kib01         | {"type":"log"......}
logstash01    | [2020-06-09T09:02:41,917]......
logstash01    | [2020-06-09T09:02:42,445]......
```

如果只想看指定服务的日志，命令如下：

```shell
docker-compose -p elk logs --tail 2 kib01
```

`top`

查看项目中容器运行的进程。

示例：

```shell
# 查看全部容器的进程
docker-compose -p elk top
# 查看指定容器的进程
docker-compose -p elk top kib01
```

`port`

查看指定容器对外映射的端口。

示例：

```shell
docker-compose -p elk port es01 9200
```

输出：

```shell
0.0.0.0:9200
```

`rm`

移除项目容器，如果指定`-v`参数，还会同时把对应数据卷删除，注意，**项目中使用的网络不会移除**。

格式为：

```shell
Usage: rm [options] [SERVICE...]
Options:
    -f, --force   Don't ask to confirm removal
    -s, --stop    Stop the containers, if required, before removing
    -v            Remove any anonymous volumes attached to containers
```

示例：

```shell
docker-compose -p elk rm -s -v logstash01
```

输出：

```shell
Stopping logstash01 ... done
Going to remove logstash01
Are you sure? [yN] Y
Removing logstash01 ... done
```

其它命令可参看[官网](https://docs.docker.com/compose/reference/)。

