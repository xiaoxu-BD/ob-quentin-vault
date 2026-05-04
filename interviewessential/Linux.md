# Linux

***

# Linux 虚拟机网络无法使用问题排查与解决文档

## 一、问题现象

* `ping www.baidu.com` 无法访问
* `ip a` 显示网卡存在，但 `state DOWN`，没有 `inet` IP 地址
* 突然断电 / 强制关机后再次启动虚拟机，常见这种情况

示例：

```plain
2: ens33: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 00:0c:29:ee:a7:94 brd ff:ff:ff:ff:ff:ff
```

说明网卡驱动存在，但处于 **关闭状态**，因此无法联网。

***

## 二、原因分析

1. **强制关机/断电** 导致网络配置文件未正确加载
2. **网卡未启用**，系统启动时没有自动 `up`
3. **DHCP 租约丢失**，导致没有分配到 IP
4. **虚拟机网络模式异常**（NAT / 桥接模式配置错误）

***

## 三、解决步骤

### 1. 查看网卡状态

```bash
ip a
```

重点看 `ens33` 是否是 `DOWN`，以及是否有 `inet` 地址。

***

### 2. 手动启动网卡

```bash
sudo ip link set ens33 up
```

或：

```bash
sudo nmcli device connect ens33
```

***

### 3. 申请 IP 地址

```bash
sudo dhclient -v ens33
```

执行后再 `ip a` 检查是否获取到 IP。

***

### 4. 设置开机自启

#### CentOS / RHEL 系

编辑：

```bash
sudo vi /etc/sysconfig/network-scripts/ifcfg-ens33
```

确认有：

```plain
DEVICE=ens33
BOOTPROTO=dhcp
ONBOOT=yes
```

保存后重启网络：

```bash
sudo systemctl restart network
```

#### Ubuntu / Debian (Netplan)

编辑：

```bash
sudo vi /etc/netplan/01-netcfg.yaml
```

内容：

```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: true
```

应用配置：

```bash
sudo netplan apply
```

***

### 5. 检查虚拟机网络模式

* 打开 VMware / VirtualBox → 设置 → 网络适配器
* 确保 **NAT / 桥接** 模式正确，必要时切换一次后再测试
* 如果是 VMware，可在 **虚拟网络编辑器**里重置 `VMnet8 (NAT)` 配置

***

## 四、推荐一键修复脚本

保存为 `fix-net.sh`：

```bash
#!/bin/bash
DEV=ens33

echo ">>> 尝试启用网卡 $DEV"
sudo ip link set $DEV up

echo ">>> 重新申请 DHCP 地址"
sudo dhclient -v $DEV

echo ">>> 当前网络状态："
ip a show $DEV
```

执行：

```bash
chmod +x fix-net.sh
./fix-net.sh
```

***

## 五、预防建议

1. 尽量使用 `shutdown -h now` 或 VMware 菜单里的“正常关机”
2. 定期备份 `/etc/sysconfig/network-scripts/` 或 `/etc/netplan/` 配置
3. 如果经常遇到，考虑在开机自启动脚本里加：

```bash
nmcli device connect ens33 || dhclient ens33
```

***

# 安装JDK

第一步

下载安装包

```powershell
wget  https://download.oracle.com/java/21/latest/jdk-21_linux-x64_bin.tar.gz 
```

解压:

tar  -zxvf  \*.tar.gz -C /opt/java

第三步:

配置环境变量:

```bash
sudo vi /etc/profile
```

```powershell
export JAVA_HOME=/opt/java/jdk-21.0.8
export JRE_HOME=$JAVA_HOME/jre
export CLASSPATH=.:$JAVA_HOME/lib:$JRE_HOME/lib
export PATH=$JAVA_HOME/bin:$JRE_HOME/bin:$PATH
```

重启刷新: source /etc/profile

# 网络相关命令:

端口占用问题:

ss -tuln

查看端口是否开放: `nc` 或者`ncat`测试端口是否开放

查看某一个端口是否被哪一个进程占用  lsof -i :8080

Ubuntu 镜像

<https://blog.csdn.net/weixin_44807874/article/details/146281832>

下载安装

# Ubuntu 安装配置 测试镜像:

安装 docker

<https://www.cnblogs.com/lanyusky/p/19187095>

详细的步骤操作;

mysql nacos xxljob seata-server

mysql (等待健康检查通过)

├── nacos

├── xxl-job

└── seata-server (等 mysql healthy + nacos started)

redis (并行)

namesrv (并行)

└── broker

```
  └── rocketmq-dashboard
```

```shell
ls mysql/config mysql/data mysql/logs mysql/init
ls redis/config/redis.conf redis/data
ls nacos/env/nacos-standlone-mysql.env nacos/logs
ls rocketMQ/data/broker/conf/broker.conf
ls xxl-job/config/application.properties
ls seata-server/resources/application.yml seata-server/logs	  
```

```yaml
version: '3.8'

services:
  mysql:
    image: mysql:8.4.1
    container_name: mysql-8
    environment:
      - MYSQL_ROOT_PASSWORD=NFTurbo666
      - TZ=Asia/Shanghai
      - SET_CONTAINER_TIMEZONE=true
      - CONTAINER_TIMEZONE=Asia/Shanghai
      - LANG=C.CUTF-8
      - LC_ALL=C.UTF-8
      - MYSQL_CHARACTER_SET_SERVER=utf8mb4
      - MYSQL_COLLATION_SERVER=utf8mb4_unicode_ci
    volumes:
      - ./mysql/config:/etc/mysql/conf.d
      - ./mysql/data:/var/lib/mysql
      - ./mysql/logs:/var/log/mysql
      - ./mysql/init:/docker-entrypoint-initdb.d
      - /etc/localtime:/etc/localtime:ro
    ports:
      - 3306:3306
    restart: always
    healthcheck:
      test: [ "CMD", "mysqladmin" ,"ping", "-h", "localhost" ]
      interval: 5s
      timeout: 10s
      retries: 10
    networks:
      - app-network
    profiles:
      - basic
      - full  # 所有容器共享full配置

  redis:
    image: redis:latest
    container_name: redis
    restart: always
    ports:
      - 6379:6379
    volumes:
      - ./redis/config/redis.conf:/usr/local/etc/redis/redis.conf:rw
      - ./redis/data:/data:rw
    command:
      /bin/bash -c "redis-server /usr/local/etc/redis/redis.conf"
    networks:
      - app-network
    profiles:
      - basic
      - full

  nacos:
    image: nacos/nacos-server:v2.5.1
    container_name: nacos
    env_file:
      - ./nacos/env/nacos-standlone-mysql.env
    volumes:
      - ./nacos/logs/:/home/nacos/logs
    ports:
      - "8848:8848"
      - "9848:9848"
    restart: always
    depends_on:
      mysql:
        condition: service_healthy
    networks:
      - app-network
    profiles:
      - mirserver
      - full

  namesrv:
    image: apache/rocketmq:5.2.0
    container_name: rmqnamesrv
    environment:
      - JAVA_OPT_EXT=-Xms512m -Xmx1g -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m
    ports:
      - 9876:9876
    volumes:
      - ./rocketMQ/data/namesrv/logs:/home/rocketmq/logs
    command: sh mqnamesrv -n 192.168.200.128:9876
    networks:
      - app-network
    profiles:
      - mq
      - full

  broker:
    image: apache/rocketmq:5.2.0
    container_name: rmqbroker
    environment:
      - JAVA_OPT_EXT=-Xms512m -Xmx1g -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m
    ports:
      - 10909:10909
      - 10911:10911
      - 10912:10912
    volumes:
      - ./rocketMQ/data/broker/logs:/home/rocketmq/logs
      - ./rocketMQ/data/broker/store:/home/rocketmq/store
      - ./rocketMQ/data/broker/conf/broker.conf:/home/rocketmq/rocketmq-5.2.0/conf/broker.conf
    command: sh mqbroker -n 192.168.200.128:9876  -c /home/rocketmq/rocketmq-5.2.0/conf/broker.conf
    depends_on:
      - namesrv
    networks:
      - app-network
    profiles:
      - mq
      - full

  rocketmq-dashboard:
    image: apacherocketmq/rocketmq-dashboard:latest
    container_name: rocketmq-dashboard
    environment:
      - JAVA_OPTS=-Drocketmq.namesrv.addr=rmqnamesrv:9876  -Dserver.port=8080
    ports:
      - "8080:8080"
    restart: always
    depends_on:
      - namesrv
      - broker
    networks:
      - app-network
    profiles:
      - mq
      - full

  xxl-job:
    image: xuxueli/xxl-job-admin:2.4.1
    container_name: xxl-job
    volumes:
      - ./xxl-job/config/application.properties:/application.properties
    ports:
      - "8082:8080"
    restart: always
    depends_on:
      mysql:
        condition: service_healthy
    networks:
      - app-network
    profiles:
      - job
      - full

  seata-server:
    image: seataio/seata-server:2.0.0
    container_name: seata-server
    environment:
      - STORE_MODE=db
      - SEATA_IP=192.168.200.128
      - SEATA_PORT=8091
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - ./seata-server/resources/application.yml:/seata-server/resources/application.yml
      - ./seata-server/logs:/seata-server/logs
    ports:
      - "7091:7091"
      - "8091:8091"
    restart: always
    depends_on:
      mysql:
        condition: service_healthy
      nacos:
        condition: service_started
    networks:
      - app-network
    profiles:
      - mirserver
      - full

networks:
  app-network:
    driver: bridge
```

#

# Linux 常用

vi 操作

1. <code><font style="color:rgb(96, 125, 139);">/字符串：文本查找操作，用于从当前光标所在位置开始向文件尾部查找指定字符串的内容，查找的字符串会被加亮显示；</font></code>
2. <code><font style="color:rgb(96, 125, 139);">?字符串：文本查找操作，用于从当前光标所在位置开始向文件头部查找指定字符串的内容，查找的字符串会被加亮显示；</font></code>

<font style="color:rgb(96, 125, 139);"></font>

tail

scp

解压:

-zxvf

解压并解包： tar -zxvf \[原文件名].tar.gz

打包并压缩

打包并压缩： tar -zcvf \[目标文件名].tar.gz \[原文件名/目录名]

tar.gz

解压: gunzip filename.tar.gz

压缩 gzip \[原文件名].tar

find . -name  "\*.txt"

ss -tuln

# JDK 版本切换脚本

```powershell
D:\Java\
    ├── jdk8
    ├── jdk17
    ├── jdk21
```

```powershell
@echo off

REM 设置 JDK 路径
setx JAVA8_HOME "D:\Java\jdk8" /M
setx JAVA17_HOME "D:\Java\jdk17" /M
setx JAVA21_HOME "D:\Java\jdk21" /M

REM 默认版本（可改）
setx JAVA_VERSION "%%JAVA17_HOME%%" /M

REM JAVA_HOME 指向 JAVA_VERSION
setx JAVA_HOME "%%JAVA_VERSION%%" /M

REM PATH 设置（只需要一次）
setx PATH "%%JAVA_HOME%%\bin;%%PATH%%" /M

echo 初始化完成，请重启终端生效
pause
```

```powershell
@echo off

if "%1"=="" (
    echo 用法: switch-java 8 ^| 17 ^| 21
    goto end
)

if "%1"=="8" (
    setx JAVA_VERSION "%%JAVA8_HOME%%" /M
)

if "%1"=="17" (
    setx JAVA_VERSION "%%JAVA17_HOME%%" /M
)

if "%1"=="21" (
    setx JAVA_VERSION "%%JAVA21_HOME%%" /M
)

echo 已切换到 JDK %1
echo 请重新打开终端

:end
pause
```


> 更新: 2026-04-23 20:31:59  
> 原文: <https://www.yuque.com/alice-hv75k/mtczog/zult9w0w9r3y232a>